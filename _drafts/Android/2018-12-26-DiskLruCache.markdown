---
layout:     post
title:      "Android源码系列(18) -- DiskLruCache"
date:       2018-12-26
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Android源码系列
---

## 一、类签名

#### 1.1 类特性

这是在文件系统上使用有限空间的缓存类。每个缓存条目都有一个字符串键和固定数量的值。每个键必须匹配正则表达式：__[a-z0-9_-]{1,120}__。值都是字节序列，可通过流或文件访问，长度介于0到 __Integer.MAX_VALUE__。

```java
public final class DiskLruCache implements Closeable
```

缓存数据保存在文件系统的一个目录中。此文件必须排除在缓存之外，缓存必须删除或复写目录里的文件。且不能支持多进程同时操作同一个缓存目录。

此缓存可限制保存在文件系统字节的长度。当已保存字节长度超过限制，会在后台线程逐个移除实体，直到满足长度限制为止。但限制也不是严格执行：需删除文件的时候，缓存大小会暂时超过限制。容量限制不包含文件系统的开销和缓存日志文件的大小，所以对空间大小敏感的应用最好设置一个相对保守的阈值。

客户端调用 __edit()__ 方法创建或更新实体的值。一个实体每次只被一个编辑器持有。如果某个值不能编辑则 __edit()__ 方法返回 __null__。

实体被创建的时候需要提供所有的值，或者在必要时使用 __null__ 作为占位符。实体被编辑的时候不需要为每个值提供数据，值的内容为之前的内容。

每此调用 __edit__ 方法时必须配对使用 __Editor.commit()__ 或 __Editor.abort()__。提交操作是原子性的：每此读取获得的是 __提交之前__ 或 __提交之后__ 完整的值的集合，而不是两个状态的混合值。

客户端调用 __get()__ 读取一个实体的快照。读操作会在 __get__ 方法调用的时候观察值。更新或移除操作不会影响正在进行的读取操作。

此类可容忍少量 I/O 错误。如果文件系统丢失文件，对应的实体会从缓存中删除。假如这个错误发生在缓存写入值的时候，编辑操作会悄无声息地执行失败。调用者需要处理由 __IOException__ 引起的问题。

#### 1.2 日志格式

日志文件命名为"journal"。一个典型的日志文件格式如下：

```java
/*
 *     libcore.io.DiskLruCache
 *     1
 *     100
 *     2
 *
 *     CLEAN 3400330d1dfc7f3f7f4b8d4d803dfcf6 832 21054
 *     DIRTY 335c4c6028171cfddfbaae1a9c313c52
 *     CLEAN 335c4c6028171cfddfbaae1a9c313c52 3934 2342
 *     REMOVE 335c4c6028171cfddfbaae1a9c313c52
 *     DIRTY 1ab96a171faeeee38496d8b330771a7a
 *     CLEAN 1ab96a171faeeee38496d8b330771a7a 1600 234
 *     READ 335c4c6028171cfddfbaae1a9c313c52
 *     READ 3400330d1dfc7f3f7f4b8d4d803dfcf6
 */
```

前五行内容是日志文件的头部。分别是常量字符创 __"libcore.io.DiskLruCache"__、__磁盘缓存版本__、__应用程序版本__、__值总计数量__ 和 一个空行。

```java
/*
 * Each of the subsequent lines in the file is a record of the state of a
 * cache entry. Each line contains space-separated values: a state, a key,
 * and optional state-specific values.
 *   o DIRTY lines track that an entry is actively being created or updated.
 *     Every successful DIRTY action should be followed by a CLEAN or REMOVE
 *     action. DIRTY lines without a matching CLEAN or REMOVE indicate that
 *     temporary files may need to be deleted.
 *   o CLEAN lines track a cache entry that has been successfully published
 *     and may be read. A publish line is followed by the lengths of each of
 *     its values.
 *   o READ lines track accesses for LRU.
 *   o REMOVE lines track entries that have been deleted.
 *
 * The journal file is appended to as cache operations occur. The journal may
 * occasionally be compacted by dropping redundant lines. A temporary file named
 * "journal.tmp" will be used during compaction; that file should be deleted if
 * it exists when the cache is opened.
 */
```


## 二、常量

```java
// 原文件的文件名
static final String JOURNAL_FILE = "journal";

// 临时文件的文件名
static final String JOURNAL_FILE_TEMP = "journal.tmp";

// 备份文件的文件名
static final String JOURNAL_FILE_BACKUP = "journal.bkp";

// 魔数字符串用于标识日志文件的身份
static final String MAGIC = "libcore.io.DiskLruCache";
// 当前DiskLruCache的版本
static final String VERSION_1 = "1";
static final long ANY_SEQUENCE_NUMBER = -1;

// 已清除
private static final String CLEAN = "CLEAN";

// 脏数据
private static final String DIRTY = "DIRTY";

// 已移除
private static final String REMOVE = "REMOVE";

// 已读取
private static final String READ = "READ";
```

## 三、数据成员


```java
private final File directory;
private final File journalFile;
private final File journalFileTmp;
private final File journalFileBackup;
// 使用此库时App的版本号，版本号改变后缓存将失效
private final int appVersion;
private long maxSize;
private final int valueCount;
private long size = 0;
private Writer journalWriter;
private final LinkedHashMap<String, Entry> lruEntries =
    new LinkedHashMap<String, Entry>(0, 0.75f, true);
private int redundantOpCount;

/**
 * To differentiate between old and current snapshots, each entry is given
 * a sequence number each time an edit is committed. A snapshot is stale if
 * its sequence number is not equal to its entry's sequence number.
 */
private long nextSequenceNumber = 0;

/** This cache uses a single background thread to evict entries. */
// 此缓存使用单个后台线程来清除条目
final ThreadPoolExecutor executorService =
    new ThreadPoolExecutor(0, 1, 60L, TimeUnit.SECONDS, new LinkedBlockingQueue<Runnable>());
private final Callable<Void> cleanupCallable = new Callable<Void>() {
  public Void call() throws Exception {
    synchronized (DiskLruCache.this) {
      if (journalWriter == null) {
        return null; // Closed.
      }
      trimToSize();
      if (journalRebuildRequired()) {
        rebuildJournal();
        redundantOpCount = 0;
      }
    }
    return null;
  }
};
```

## 四、构造方法

```java
private DiskLruCache(File directory, int appVersion, int valueCount, long maxSize) {
  this.directory = directory;
  this.appVersion = appVersion;
  this.journalFile = new File(directory, JOURNAL_FILE);
  this.journalFileTmp = new File(directory, JOURNAL_FILE_TEMP);
  this.journalFileBackup = new File(directory, JOURNAL_FILE_BACKUP);
  this.valueCount = valueCount;
  this.maxSize = maxSize;
}
```

## 五、成员方法

#### 5.1 open

打开在目录 __directory__ 中的缓存，文件不存在则创建一个新缓存。

参数解析：

- __directory__：存放文件的可写目录
- __appVersion__：应用的版本号，当版本号改变后缓存会全部清除
- __valueCount__：每个缓存条目的值的数量，必须为正数
- __maxSize__：此缓存用于存储的最大字节数

```java
/**
 * Opens the cache in {@code directory}, creating a cache if none exists
 * there.
 *
 * @param directory a writable directory
 * @param valueCount the number of values per cache entry. Must be positive.
 * @param maxSize the maximum number of bytes this cache should use to store
 * @throws IOException if reading or writing the cache directory fails
 */
public static DiskLruCache open(File directory, int appVersion, int valueCount, long maxSize)
    throws IOException {
    
  // 缓存可用的最大字节数小于等于0
  if (maxSize <= 0) {
    throw new IllegalArgumentException("maxSize <= 0");
  }

  if (valueCount <= 0) {
    throw new IllegalArgumentException("valueCount <= 0");
  }

  // 获取备份文件存在
  File backupFile = new File(directory, JOURNAL_FILE_BACKUP);
  if (backupFile.exists()) {
    File journalFile = new File(directory, JOURNAL_FILE);
    // 如果原journal文件存在
    if (journalFile.exists()) {
      // 则删除备份文件JOURNAL_FILE_BACKUP，即journal.bkp
      backupFile.delete();
    } else {
      // 原journal文件不存在，把JOURNAL_FILE_BACKUP文件重命名为JOURNAL_FILE
      renameTo(backupFile, journalFile, false);
    }
  }

  // Prefer to pick up where we left off.
  DiskLruCache cache = new DiskLruCache(directory, appVersion, valueCount, maxSize);
  if (cache.journalFile.exists()) {
    try {
      cache.readJournal();
      cache.processJournal();
      return cache;
    } catch (IOException journalIsCorrupt) {
      System.out
          .println("DiskLruCache "
              + directory
              + " is corrupt: "
              + journalIsCorrupt.getMessage()
              + ", removing");
      cache.delete();
    }
  }

  // 没找到已存在的文件，则创建全新空的缓存
  directory.mkdirs();
  cache = new DiskLruCache(directory, appVersion, valueCount, maxSize);
  cache.rebuildJournal();
  return cache;
}
```

#### 5.2 readJournal

读取日志

```java
private void readJournal() throws IOException {
  StrictLineReader reader = new StrictLineReader(new FileInputStream(journalFile), Util.US_ASCII);
  try {
    String magic = reader.readLine();
    String version = reader.readLine();
    String appVersionString = reader.readLine();
    String valueCountString = reader.readLine();
    String blank = reader.readLine();
    if (!MAGIC.equals(magic)
        || !VERSION_1.equals(version)
        || !Integer.toString(appVersion).equals(appVersionString)
        || !Integer.toString(valueCount).equals(valueCountString)
        || !"".equals(blank)) {
      throw new IOException("unexpected journal header: [" + magic + ", " + version + ", "
          + valueCountString + ", " + blank + "]");
    }

    int lineCount = 0;
    while (true) {
      try {
        readJournalLine(reader.readLine());
        lineCount++;
      } catch (EOFException endOfJournal) {
        break;
      }
    }
    redundantOpCount = lineCount - lruEntries.size();

    // If we ended on a truncated line, rebuild the journal before appending to it.
    if (reader.hasUnterminatedLine()) {
      rebuildJournal();
    } else {
      journalWriter = new BufferedWriter(new OutputStreamWriter(
          new FileOutputStream(journalFile, true), Util.US_ASCII));
    }
  } finally {
    Util.closeQuietly(reader);
  }
}
```

#### 5.3 readJournalLine

```java
private void readJournalLine(String line) throws IOException {
  int firstSpace = line.indexOf(' ');
  if (firstSpace == -1) {
    throw new IOException("unexpected journal line: " + line);
  }

  int keyBegin = firstSpace + 1;
  int secondSpace = line.indexOf(' ', keyBegin);
  final String key;
  if (secondSpace == -1) {
    key = line.substring(keyBegin);
    if (firstSpace == REMOVE.length() && line.startsWith(REMOVE)) {
      lruEntries.remove(key);
      return;
    }
  } else {
    key = line.substring(keyBegin, secondSpace);
  }

  Entry entry = lruEntries.get(key);
  if (entry == null) {
    entry = new Entry(key);
    lruEntries.put(key, entry);
  }

  if (secondSpace != -1 && firstSpace == CLEAN.length() && line.startsWith(CLEAN)) {
    String[] parts = line.substring(secondSpace + 1).split(" ");
    entry.readable = true;
    entry.currentEditor = null;
    entry.setLengths(parts);
  } else if (secondSpace == -1 && firstSpace == DIRTY.length() && line.startsWith(DIRTY)) {
    entry.currentEditor = new Editor(entry);
  } else if (secondSpace == -1 && firstSpace == READ.length() && line.startsWith(READ)) {
    // This work was already done by calling lruEntries.get().
  } else {
    throw new IOException("unexpected journal line: " + line);
  }
}
```

#### 5.4 processJournal

```java
/**
 * Computes the initial size and collects garbage as a part of opening the
 * cache. Dirty entries are assumed to be inconsistent and will be deleted.
 */
private void processJournal() throws IOException {
  // 删除已经存在的临时日志文件
  deleteIfExists(journalFileTmp);
  // 逐个遍历lruEntries
  for (Iterator<Entry> i = lruEntries.values().iterator(); i.hasNext(); ) {
    Entry entry = i.next();
    if (entry.currentEditor == null) {
      for (int t = 0; t < valueCount; t++) {
        size += entry.lengths[t];
      }
    } else {
      entry.currentEditor = null;
      for (int t = 0; t < valueCount; t++) {
        deleteIfExists(entry.getCleanFile(t));
        deleteIfExists(entry.getDirtyFile(t));
      }
      i.remove();
    }
  }
}
```

#### 5.5 rebuildJournal

创建一个忽略多余信息的日志文件，并把文件替换已经存在的日志文件。

```java
private synchronized void rebuildJournal() throws IOException {
  if (journalWriter != null) {
    journalWriter.close();
  }

  // 给journalFileTmp创建一个缓冲写入
  Writer writer = new BufferedWriter(
      new OutputStreamWriter(new FileOutputStream(journalFileTmp), Util.US_ASCII));
  try {
    // 第一行，写入魔数String
    writer.write(MAGIC);
    writer.write("\n");
    // 第二行，DiskLru的版本号
    writer.write(VERSION_1);
    writer.write("\n");
    // 第三行，App的版本号
    writer.write(Integer.toString(appVersion));
    writer.write("\n");
    writer.write(Integer.toString(valueCount));
    writer.write("\n");
    writer.write("\n");

    for (Entry entry : lruEntries.values()) {
      if (entry.currentEditor != null) {
        writer.write(DIRTY + ' ' + entry.key + '\n');
      } else {
        writer.write(CLEAN + ' ' + entry.key + entry.getLengths() + '\n');
      }
    }
  } finally {
    writer.close();
  }
  
  // 如果已有一份日志文件存在，就把
  if (journalFile.exists()) {
    renameTo(journalFile, journalFileBackup, true);
  }
  renameTo(journalFileTmp, journalFile, false);
  journalFileBackup.delete();

  journalWriter = new BufferedWriter(
      new OutputStreamWriter(new FileOutputStream(journalFile, true), Util.US_ASCII));
}
```

#### 5.6 deleteIfExists

如果文件已存在则删除该文件

```java
private static void deleteIfExists(File file) throws IOException {
  // 仅在文件存在的时候执行，删除失败会抛出IOException
  if (file.exists() && !file.delete()) {
    throw new IOException();
  }
}
```

#### 5.7 renameTo

从命名文件，把 __from__ 文件的名称重命名为 __to__ ，并根据 __deleteDestination__ 决定是否删除已存在的 __to__ 文件。

```java
private static void renameTo(File from, File to, boolean deleteDestination) throws IOException {
  // 是否先删除已存在的目标文件
  if (deleteDestination) {
    deleteIfExists(to);
  }
  
  // 重命名from为to
  if (!from.renameTo(to)) {
    throw new IOException();
  }
}
```

#### 5.8 get

```java
/**
 * Returns a snapshot of the entry named {@code key}, or null if it doesn't
 * exist is not currently readable. If a value is returned, it is moved to
 * the head of the LRU queue.
 */
public synchronized Value get(String key) throws IOException {
  checkNotClosed();
  Entry entry = lruEntries.get(key);
  if (entry == null) {
    return null;
  }

  if (!entry.readable) {
    return null;
  }

  for (File file : entry.cleanFiles) {
      // A file must have been deleted manually!
      if (!file.exists()) {
          return null;
      }
  }

  redundantOpCount++;
  journalWriter.append(READ);
  journalWriter.append(' ');
  journalWriter.append(key);
  journalWriter.append('\n');
  if (journalRebuildRequired()) {
    executorService.submit(cleanupCallable);
  }

  return new Value(key, entry.sequenceNumber, entry.cleanFiles, entry.lengths);
}
```

#### 5.9 edit

返回名为 __key__ 条目的编辑器，如果正在进行其他编辑则返回null。

```java
/**
 * Returns an editor for the entry named {@code key}, or null if another
 * edit is in progress.
 */
public Editor edit(String key) throws IOException {
  return edit(key, ANY_SEQUENCE_NUMBER);
}
```

以上方法调用了此方法，__expectedSequenceNumber__ 参数为 __ANY_SEQUENCE_NUMBER__。

```java
private synchronized Editor edit(String key, long expectedSequenceNumber) throws IOException {
  checkNotClosed();
  Entry entry = lruEntries.get(key);
  if (expectedSequenceNumber != ANY_SEQUENCE_NUMBER && (entry == null
      || entry.sequenceNumber != expectedSequenceNumber)) {
    return null; // 值已经被废弃
  }
  if (entry == null) {
    entry = new Entry(key);
    lruEntries.put(key, entry);
  } else if (entry.currentEditor != null) {
    return null; // 另一个编辑正在进行
  }

  Editor editor = new Editor(entry);
  entry.currentEditor = editor;

  // Flush the journal before creating files to prevent file leaks.
  // 在创建文件之前刷新日志以防止文件泄漏
  journalWriter.append(DIRTY);
  journalWriter.append(' ');
  journalWriter.append(key);
  journalWriter.append('\n');
  journalWriter.flush();
  return editor;
}
```

#### 5.10 getter

```java
// 返回此缓存存储数据的目录
public File getDirectory() {
  return directory;
}

/**
 * Returns the maximum number of bytes that this cache should use to store
 * its data.
 */
// 返回缓存用于存储其数据的最大字节数
public synchronized long getMaxSize() {
  return maxSize;
}

/**
 * Returns the number of bytes currently being used to store the values in
 * this cache. This may be greater than the max size if a background
 * deletion is pending.
 */
// 返回当前用于存储此缓存中的值的字节数
// 如果后台删除待处理，则可能大于最大大小
public synchronized long size() {
  return size;
}
```

#### 5.11 setter

```java
/**
 * Changes the maximum number of bytes the cache can store and queues a job
 * to trim the existing store, if necessary.
 */
// 如有必要，更改缓存可以存储的最大字节数，并将作业排入队列以修剪现有存储
public synchronized void setMaxSize(long maxSize) {
  this.maxSize = maxSize;
  executorService.submit(cleanupCallable);
}
```

#### 5.12 completeEdit

```java
private synchronized void completeEdit(Editor editor, boolean success) throws IOException {
  Entry entry = editor.entry;
  if (entry.currentEditor != editor) {
    throw new IllegalStateException();
  }

  // If this edit is creating the entry for the first time, every index must have a value.
  if (success && !entry.readable) {
    for (int i = 0; i < valueCount; i++) {
      if (!editor.written[i]) {
        editor.abort();
        throw new IllegalStateException("Newly created entry didn't create value for index " + i);
      }
      if (!entry.getDirtyFile(i).exists()) {
        editor.abort();
        return;
      }
    }
  }

  for (int i = 0; i < valueCount; i++) {
    File dirty = entry.getDirtyFile(i);
    if (success) {
      if (dirty.exists()) {
        File clean = entry.getCleanFile(i);
        dirty.renameTo(clean);
        long oldLength = entry.lengths[i];
        long newLength = clean.length();
        entry.lengths[i] = newLength;
        size = size - oldLength + newLength;
      }
    } else {
      deleteIfExists(dirty);
    }
  }

  redundantOpCount++;
  entry.currentEditor = null;
  if (entry.readable | success) {
    entry.readable = true;
    journalWriter.append(CLEAN);
    journalWriter.append(' ');
    journalWriter.append(entry.key);
    journalWriter.append(entry.getLengths());
    journalWriter.append('\n');

    if (success) {
      entry.sequenceNumber = nextSequenceNumber++;
    }
  } else {
    lruEntries.remove(entry.key);
    journalWriter.append(REMOVE);
    journalWriter.append(' ');
    journalWriter.append(entry.key);
    journalWriter.append('\n');
  }
  journalWriter.flush();

  if (size > maxSize || journalRebuildRequired()) {
    executorService.submit(cleanupCallable);
  }
}
```
#### 5.13 journalRebuildRequired
```java
/**
 * We only rebuild the journal when it will halve the size of the journal
 * and eliminate at least 2000 ops.
 */
private boolean journalRebuildRequired() {
  final int redundantOpCompactThreshold = 2000;
  return redundantOpCount >= redundantOpCompactThreshold //
      && redundantOpCount >= lruEntries.size();
}
```

#### 5.14 remove

通过 __key__ 删除存在的实体。

```java
/**
 * Drops the entry for {@code key} if it exists and can be removed. Entries
 * actively being edited cannot be removed.
 *
 * @return true if an entry was removed.
 */
public synchronized boolean remove(String key) throws IOException {
  checkNotClosed();
  Entry entry = lruEntries.get(key);
  if (entry == null || entry.currentEditor != null) {
    return false;
  }

  for (int i = 0; i < valueCount; i++) {
    File file = entry.getCleanFile(i);
    if (file.exists() && !file.delete()) {
      throw new IOException("failed to delete " + file);
    }
    size -= entry.lengths[i];
    entry.lengths[i] = 0;
  }

  redundantOpCount++;
  journalWriter.append(REMOVE);
  journalWriter.append(' ');
  journalWriter.append(key);
  journalWriter.append('\n');

  lruEntries.remove(key);

  if (journalRebuildRequired()) {
    executorService.submit(cleanupCallable);
  }

  return true;
}
```

#### 5.15 其他

缓存已经关闭时返回true

```java
public synchronized boolean isClosed() {
  return journalWriter == null;
}
```

检查 __journalWriter__ 是否已关闭，如果已关闭会抛出 __IllegalStateException__。

```java
private void checkNotClosed() {
  if (journalWriter == null) {
    throw new IllegalStateException("cache is closed");
  }
}
```

强制缓冲所有操作到文件系统

```java
public synchronized void flush() throws IOException {
  checkNotClosed();
  trimToSize();
  journalWriter.flush();
}
```

关闭此缓存，已存储的值将保留在文件系统中。

```java
public synchronized void close() throws IOException {
  if (journalWriter == null) {
    // 文件句柄已关闭
    return; // Already closed.
  }
  // 终止所有正在进行的编辑
  for (Entry entry : new ArrayList<Entry>(lruEntries.values())) {
    if (entry.currentEditor != null) {
      entry.currentEditor.abort();
    }
  }
  trimToSize();
  journalWriter.close();
  journalWriter = null;
}
```

移除缓存直到缓存占用没有超过限制

```java
private void trimToSize() throws IOException {
  while (size > maxSize) {
    Map.Entry<String, Entry> toEvict = lruEntries.entrySet().iterator().next();
    remove(toEvict.getKey());
  }
}
```

关闭缓存并删除所有已保存的值。

```java
/**
 * Closes the cache and deletes all of its stored values. This will delete
 * all files in the cache directory including files that weren't created by
 * the cache.
 */
public void delete() throws IOException {
  close();
  Util.deleteContents(directory);
}
```

把输入流根据 __UTF_8__ 字符集拼接成字符串

```java
private static String inputStreamToString(InputStream in) throws IOException {
  return Util.readFully(new InputStreamReader(in, Util.UTF_8));
}
```

## 六、Value

实体所含值的快照

```java
/** A snapshot of the values for an entry. */
public final class Value {
  private final String key;
  private final long sequenceNumber;
  private final long[] lengths;
  private final File[] files;

    private Value(String key, long sequenceNumber, File[] files, long[] lengths) {
    this.key = key;
    this.sequenceNumber = sequenceNumber;
    this.files = files;
    this.lengths = lengths;
  }

  /**
   * Returns an editor for this snapshot's entry, or null if either the
   * entry has changed since this snapshot was created or if another edit
   * is in progress.
   */
  public Editor edit() throws IOException {
    return DiskLruCache.this.edit(key, sequenceNumber);
  }

  public File getFile(int index) {
      return files[index];
  }

  /** Returns the string value for {@code index}. */
  // 返回指定索引下文件的String值
  public String getString(int index) throws IOException {
    InputStream is = new FileInputStream(files[index]);
    return inputStreamToString(is);
  }

  /** Returns the byte length of the value for {@code index}. */
  // 返回指定索引下字节的长度
  public long getLength(int index) {
    return lengths[index];
  }
}
```

## 七、Editor

编辑一个实体的值

```java
/** Edits the values for an entry. */
public final class Editor {
  private final Entry entry;
  private final boolean[] written;
  private boolean committed;

  private Editor(Entry entry) {
    this.entry = entry;
    this.written = (entry.readable) ? null : new boolean[valueCount];
  }

  /**
   * Returns an unbuffered input stream to read the last committed value,
   * or null if no value has been committed.
   */
  private InputStream newInputStream(int index) throws IOException {
    synchronized (DiskLruCache.this) {
      if (entry.currentEditor != this) {
        throw new IllegalStateException();
      }
      if (!entry.readable) {
        return null;
      }
      try {
        return new FileInputStream(entry.getCleanFile(index));
      } catch (FileNotFoundException e) {
        return null;
      }
    }
  }

  /**
   * Returns the last committed value as a string, or null if no value
   * has been committed.
   */
  // 把最后一次提交的值作为String返回，如果没有值提交过就返回null
  public String getString(int index) throws IOException {
    InputStream in = newInputStream(index);
    return in != null ? inputStreamToString(in) : null;
  }

  public File getFile(int index) throws IOException {
    synchronized (DiskLruCache.this) {
      if (entry.currentEditor != this) {
          throw new IllegalStateException();
      }
      if (!entry.readable) {
          written[index] = true;
      }
      File dirtyFile = entry.getDirtyFile(index);
      if (!directory.exists()) {
          directory.mkdirs();
      }
      return dirtyFile;
    }
  }

  /** Sets the value at {@code index} to {@code value}. */
  // 把index所指的值修改为新值value
  public void set(int index, String value) throws IOException {
    Writer writer = null;
    try {
      // 创建一个输出流
      OutputStream os = new FileOutputStream(getFile(index));
      // 通过输出流构建Writer
      writer = new OutputStreamWriter(os, Util.UTF_8);
      // 把值写入到索引所指
      writer.write(value);
    } finally {
      // 关闭Writer
      Util.closeQuietly(writer);
    }
  }

  /**
   * Commits this edit so it is visible to readers.  This releases the
   * edit lock so another edit may be started on the same key.
   */
  public void commit() throws IOException {
    // The object using this Editor must catch and handle any errors
    // during the write. If there is an error and they call commit
    // anyway, we will assume whatever they managed to write was valid.
    // Normally they should call abort.
    completeEdit(this, true);
    committed = true;
  }

  /**
   * Aborts this edit. This releases the edit lock so another edit may be
   * started on the same key.
   */
  // 终止编辑。此方法会释放编辑锁，并允许其他编辑操作可以在同一个key上开展编辑
  public void abort() throws IOException {
    completeEdit(this, false);
  }

  public void abortUnlessCommitted() {
    if (!committed) {
      try {
        abort();
      } catch (IOException ignored) {
      }
    }
  }
}
```

## 八、Entry

```java
private final class Entry {
  private final String key;

  // 持有文件的长度
  private final long[] lengths;

  /** Memoized File objects for this entry to avoid char[] allocations. */
  File[] cleanFiles;
  File[] dirtyFiles;

  /** True if this entry has ever been published. */
  // 实体已经发布的话此值为True
  private boolean readable;

  /** The ongoing edit or null if this entry is not being edited. */
  // 正在进行的编辑，如果未编辑此条目，则返回null
  private Editor currentEditor;

  /** The sequence number of the most recently committed edit to this entry. */
  // 此条目最近提交的编辑的序列号。
  private long sequenceNumber;

  private Entry(String key) {
    this.key = key;
    this.lengths = new long[valueCount];
    cleanFiles = new File[valueCount];
    dirtyFiles = new File[valueCount];

    // The names are repetitive so re-use the same builder to avoid allocations.
    // 名字都是重复的，所以重用同一个构造器避免内存申请
    StringBuilder fileBuilder = new StringBuilder(key).append('.');
    int truncateTo = fileBuilder.length();
    for (int i = 0; i < valueCount; i++) {
        fileBuilder.append(i);
        cleanFiles[i] = new File(directory, fileBuilder.toString());
        fileBuilder.append(".tmp");
        dirtyFiles[i] = new File(directory, fileBuilder.toString());
        fileBuilder.setLength(truncateTo);
    }
  }

  public String getLengths() throws IOException {
    StringBuilder result = new StringBuilder();
    for (long size : lengths) {
      result.append(' ').append(size);
    }
    return result.toString();
  }

  // 使用类似"10123"的十进制数字设置长度
  private void setLengths(String[] strings) throws IOException {
    if (strings.length != valueCount) {
      throw invalidLengths(strings);
    }

    try {
      for (int i = 0; i < strings.length; i++) {
        lengths[i] = Long.parseLong(strings[i]);
      }
    } catch (NumberFormatException e) {
      throw invalidLengths(strings);
    }
  }

  private IOException invalidLengths(String[] strings) throws IOException {
    throw new IOException("unexpected journal line: " + java.util.Arrays.toString(strings));
  }

  public File getCleanFile(int i) {
    return cleanFiles[i];
  }

  public File getDirtyFile(int i) {
    return dirtyFiles[i];
  }
}
```
