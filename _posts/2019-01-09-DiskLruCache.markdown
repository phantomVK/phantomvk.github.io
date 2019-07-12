---
layout:     post
title:      "Android源码系列(18) -- DiskLruCache"
date:       2019-01-09
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Android源码系列
---

## 一、类签名

#### 1.1 特性

这是基于文件系统的 __LRU(Least Recently Used)__ 缓存类，并支持控制空间使用量。每个缓存条目都有一个字符串键和固定数量的值。每个键必须满足正则表达式：__[a-z0-9_-]{1,120}__。值都是字节序列，可通过流或文件的方式访问，长度在0到 __Integer.MAX_VALUE__ 之间。

```java
public final class DiskLruCache implements Closeable
```

数据保存在文件系统的同一个目录中。此文件必须排除在缓存之外，缓存只能删除或复写目录里的文件。不支持多进程同时操作同一个缓存目录。

此缓存可限制保存在文件系统字节的长度。当已保存字节长度超过限制，会在后台线程逐个移除条目，直到满足长度限制为止。但限制也不是严格执行：需删除文件的时候，缓存大小会暂时超过限制。容量限制不包含文件系统的开销和缓存日志文件的大小，所以对空间大小敏感的应用最好设置一个相对保守的阈值。

客户端调用 __edit()__ 方法才能创建或更新条目的值，每次只被一个编辑器持有。如果不能编辑则 __edit()__ 方法返回 __null__。

条目创建时需提供所有值，或在必要时用 __null__ 作为占位符。条目编辑时不需要为每个值提供数据，值的内容为之前的内容。

每次调用 __edit__ 方法后必须配对使用 __Editor.commit()__ 或 __Editor.abort()__。提交操作是原子性的：每次读取获得 __提交之前__ 或 __提交之后__ 完整的值集合，而不是两个状态的混合值。

客户端调用 __get()__ 读取条目的快照。读操作会在 __get__ 方法调用的时候观察值。更新或移除操作不会影响正在进行的读取操作。

本类可容忍少量 I/O 错误。如果文件系统丢失文件，对应的条目会从缓存中删除。假如这个错误发生在缓存写入值的时候，编辑执行操作会静默失败。注意，调用者需要自行处理因 __IOException__ 引起的问题。

以下源码来自 __Glide 4.4.0__ ，不同来源可能会有细微差别，但是总体逻辑和设计思路是一样的，也都是参考 [Jake Wharton](https://github.com/JakeWharton/DiskLruCache) 写的版本。

#### 1.2 日志格式

日志文件命名为 "journal"，典型的文件格式如下：

```java
libcore.io.DiskLruCache
1
100
2

CLEAN 3400330d1dfc7f3f7f4b8d4d803dfcf6 832 21054
DIRTY 335c4c6028171cfddfbaae1a9c313c52
CLEAN 335c4c6028171cfddfbaae1a9c313c52 3934 2342
REMOVE 335c4c6028171cfddfbaae1a9c313c52
DIRTY 1ab96a171faeeee38496d8b330771a7a
CLEAN 1ab96a171faeeee38496d8b330771a7a 1600 234
READ 335c4c6028171cfddfbaae1a9c313c52
READ 3400330d1dfc7f3f7f4b8d4d803dfcf6
```

前五行内容是日志文件的头部。分别是 __"libcore.io.DiskLruCache"__、__磁盘缓存版本__、__应用程序版本__、__值总计数量__ 和一个空行。

文件随后每一行，代表记录一个缓存条目的状态。内容为：状态值、key、可选的描述状态的值，各自用空格分割。

- __DIRTY__ 意味对应条目是新创建或已被修改。每个正确的 __DIRTY__ 操作必须跟着 __CLEAN__ 或 __REMOVE__ 操作。如果没满足该条件，则需要删除临时文件；
- __CLEAN__ 表示缓存条目已成功发布并可访问。每个发布行后面跟着若干个写入值的长度；
- __READ__ 是访问操作 (访问不会造成副作用)；
- __REMOVE__ 表示该条目内容已被删除。

当发生缓存操作时，内容会追加到日志文件中。缓存类偶尔会删除日志文件多余的行，缩小内容体积。临时文件名为 __"journal.tmp"__，在日志压缩过程中使用，且会在缓存启动时删除该文件。

## 二、常量

```java
// 原文件的文件名
static final String JOURNAL_FILE = "journal";

// 临时文件的文件名
static final String JOURNAL_FILE_TEMP = "journal.tmp";

// 备份文件的文件名
static final String JOURNAL_FILE_BACKUP = "journal.bkp";

// 魔数字符串标识文件的身份
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

数据成员通过 __LinkedHashMap__ 类实现LRU特性，具体源码请看[Java源码系列(11) -- LinkedHashMap](/2018/07/09/LinkedHashMap/)


```java
// 缓存保存的文件夹
private final File directory;

// 日志文件
private final File journalFile;

// 临时日志文件
private final File journalFileTmp;

// 备份的日志文件
private final File journalFileBackup;

// 使用此库时App的版本号，版本号改变后缓存将失效
private final int appVersion;

// 用于存储的最大字节数
private long maxSize;

// 每个条目可保存值的最大数量
private final int valueCount;

// 已保存内容的大小
private long size = 0;
private Writer journalWriter;

// 实现LRU的LinkedHashMap
private final LinkedHashMap<String, Entry> lruEntries =
    new LinkedHashMap<String, Entry>(0, 0.75f, true);

// 多余操作次数的统计
private int redundantOpCount;

// 用于区分当前快照和旧快照的序列号，条目每次提交编辑时都被授予一个序列号
// 如果快照的序列号不等于条目的序列号，则快照是旧的
private long nextSequenceNumber = 0;

// 缓存使用后台单线程清除条目
final ThreadPoolExecutor executorService =
    new ThreadPoolExecutor(0, 1, 60L, TimeUnit.SECONDS, new LinkedBlockingQueue<Runnable>());
```

清除日志文件的任务，在需要的时候加到线程池等待执行

```java
private final Callable<Void> cleanupCallable = new Callable<Void>() {
  public Void call() throws Exception {
    // 操作的时候锁类对象
    synchronized (DiskLruCache.this) {
      // 日志文件无法操作
      if (journalWriter == null) {
        return null; // Closed.
      }
      trimToSize();
      if (journalRebuildRequired()) {
        // 重建日志文件
        rebuildJournal();
        // 日志文件已重建，此统计值置0
        redundantOpCount = 0;
      }
    }
    return null;
  }
};
```

## 四、构造方法

可见性修饰是 __private__，构造方法无法从外部调用，只能通过内部静态方法 __open__ 创建实例。

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

打开在目录 __directory__ 中的缓存，文件不存在则创建新缓存。参数解析：

- __directory__：存放文件的可写目录
- __appVersion__：应用版本号，当版本号改变后缓存会全部重置
- __valueCount__：每个缓存条目保存值的数量，必须为正数
- __maxSize__：此缓存可用于存储的最大字节数

还有，读写文件缓存目录失败时会抛出 __IOException__

```java
public static DiskLruCache open(File directory, int appVersion, int valueCount, long maxSize)
    throws IOException {
    
  // 缓存可用的最大字节数小于等于0
  if (maxSize <= 0) {
    throw new IllegalArgumentException("maxSize <= 0");
  }

  if (valueCount <= 0) {
    throw new IllegalArgumentException("valueCount <= 0");
  }

  // 获取备份文件，文件是重建日志时生成的
  File backupFile = new File(directory, JOURNAL_FILE_BACKUP);
  if (backupFile.exists()) {
    File journalFile = new File(directory, JOURNAL_FILE);
    // 备份文件存在，原journal文件也存在
    if (journalFile.exists()) {
      // 原文件存在，表示重建过程正常完成，已不需要备份文件
      // 则删除备份文件JOURNAL_FILE_BACKUP，即journal.bkp
      backupFile.delete();
    } else {
      // 原journal文件不存在，把JOURNAL_FILE_BACKUP作为JOURNAL_FILE
      renameTo(backupFile, journalFile, false);
    }
  }

  // Prefer to pick up where we left off.
  DiskLruCache cache = new DiskLruCache(directory, appVersion, valueCount, maxSize);
  if (cache.journalFile.exists()) {
    try {
      cache.readJournal();
      cache.processJournal();
      // 缓存读取成功，返回cache句柄
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

  // 没找到文件或读取出错，则创建全新空的缓存
  directory.mkdirs();
  cache = new DiskLruCache(directory, appVersion, valueCount, maxSize);
  cache.rebuildJournal();
  return cache;
}
```

#### 5.2 readJournal

读取日志，主要目的是校验日志文件头内容是否合法，并通过捕获自行抛出异常的方法处理异常情况。常见的应用场景是：App版本号更新后，新版本号和日志内App版本号不匹配抛出异常。然后，该异常被捕获后，此缓存会被丢弃并创建新缓存文件。

```java
private void readJournal() throws IOException {
  StrictLineReader reader = new StrictLineReader(new FileInputStream(journalFile), Util.US_ASCII);
  try {
    String magic = reader.readLine();
    String version = reader.readLine();
    String appVersionString = reader.readLine();
    String valueCountString = reader.readLine();
    String blank = reader.readLine();
    // 检验头部5行内容是否合法
    if (!MAGIC.equals(magic)
        || !VERSION_1.equals(version)
        || !Integer.toString(appVersion).equals(appVersionString)
        || !Integer.toString(valueCount).equals(valueCountString)
        || !"".equals(blank)) {
      // 外部捕获异常，用新建日志文件替换问题日志
      throw new IOException("unexpected journal header: [" + magic + ", " + version + ", "
          + valueCountString + ", " + blank + "]");
    }

    // 日志文件头部内容校验已通过
    int lineCount = 0;
    // 逐行读取日志，并按照固定格式解析行内容
    while (true) {
      try {
        readJournalLine(reader.readLine());
        // 计算读取的总行数，行数不包括日志头部内容
        lineCount++;
      } catch (EOFException endOfJournal) {
        // 读取的内容异常，例如：设备突然断电导致日志没写全
        break;
      }
    }

    // 冗余操作计数 = 日志行数 - 缓存实体数量
    // 若冗余操作计数很大，表示日志可能含有很多冗余的信息，需要进行日志清理
    redundantOpCount = lineCount - lruEntries.size();

    // If we ended on a truncated line, rebuild the journal before appending to it.
    // 上面按照行读取出现异常时，在这里触发日志重建操作
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

日志文件头部内容校验通过后，日志文件的修改记录逐行读取并通过此方法进行处理。

假设现在处理的行内容是 `REMOVE 335c4c6028171cfddfbaae1a9c313c52`

```java
private void readJournalLine(String line) throws IOException {
  // 每行必须包含一个至少空格，否则抛出异常IOException
  int firstSpace = line.indexOf(' ');
  if (firstSpace == -1) {
    throw new IOException("unexpected journal line: " + line);
  }

  // firstSpace索引值为6
  int keyBegin = firstSpace + 1;
  int secondSpace = line.indexOf(' ', keyBegin);
  final String key;
  // 此时不存在secondSpace，所以secondSpace为-1，条件命中
  if (secondSpace == -1) {
    // 从行内容裁出key: 335c4c6028171cfddfbaae1a9c313c52
    key = line.substring(keyBegin);
    // REMOVE.length()为6，所有操作的字符串只有REMOVE的长度为6
    if (firstSpace == REMOVE.length() && line.startsWith(REMOVE)) {
      // 匹配移除操作，则把该条目从LinkedHashMap中移除
      lruEntries.remove(key);
      // REMOVE操作退出方法
      return;
    }
  } else {
    // CLEAN操作进入此分支的，因为只有CLEAN的secondSpace不为-1
    key = line.substring(keyBegin, secondSpace);
  }

  // DIRTY、READ、CLEAN操作到这里
  Entry entry = lruEntries.get(key);
  if (entry == null) {
    // 日志包含的记录在这里初始化并放入内存中
    entry = new Entry(key);
    lruEntries.put(key, entry);
  }

  if (secondSpace != -1 && firstSpace == CLEAN.length() && line.startsWith(CLEAN)) {
    // 处理CLEAN操作，裁剪出key后面的多个值
    String[] parts = line.substring(secondSpace + 1).split(" ");
    entry.readable = true;
    entry.currentEditor = null;
    entry.setLengths(parts);
  } else if (secondSpace == -1 && firstSpace == DIRTY.length() && line.startsWith(DIRTY)) {
    // 处理DIRTY操作
    entry.currentEditor = new Editor(entry);
  } else if (secondSpace == -1 && firstSpace == READ.length() && line.startsWith(READ)) {
    // READ操作不需处理，因为READ操作没有任何副作用
    // This work was already done by calling lruEntries.get().
  } else {
    // 出现未知操作类型，正常来说不会遇到
    throw new IOException("unexpected journal line: " + line);
  }
}
```

总结操作指令：

- __REMOVE__：从 __LinkedHashMap__ 删除条目；
- __CLEAN__：初始化条目，设置 __readable=true__、__currentEditor=null__，初始化内容长度；
- __DIRTY__：设置 __currentEditor__ 对象；
- __READ__：什么都不执行；

正常操作的 __DIRTY__ 配对 __CLEAN__ 或 __REMOVE__，否则就是脏数据且需要删除。

#### 5.4 processJournal

把计算初始大小和收集垃圾操作作为打开缓存的一部分。脏条目会假定为不一致要被删除。DIRTY操作的 __currentEditor__ 不为空，意味原内容已废弃，但新内容又没正确结束，所以新旧文件都要移除。

```java
private void processJournal() throws IOException {
  // 删除已经存在的临时日志文件
  deleteIfExists(journalFileTmp);
  // 逐个遍历lruEntries
  for (Iterator<Entry> i = lruEntries.values().iterator(); i.hasNext(); ) {
    Entry entry = i.next();
    if (entry.currentEditor == null) {
      // 统计可以读取条目的总长度
      for (int t = 0; t < valueCount; t++) {
        size += entry.lengths[t];
      }
    } else {
      // DIRTY操作的currentEditor不为空
      entry.currentEditor = null;
      for (int t = 0; t < valueCount; t++) {
        deleteIfExists(entry.getCleanFile(t));
        deleteIfExists(entry.getDirtyFile(t));
      }
      // 从lruEntries移除该记录
      i.remove();
    }
  }
}
```

#### 5.5 rebuildJournal

以下几种情况会重建日志：

1. 缓存是新建立的，没有日志，所以用重建操作完成新建日志文件；
2. 日志头部内容验证不通过，丢弃该文件并新建日志；
3. 启动缓存过程中读取已有日志内容出错，要用读取成功的内容重建文件替换已有日志；
4. 日志记录内容过多会影响io，所以通过重建去除冗余记录；

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
    // 第四行，条目可包含值的数量
    writer.write(Integer.toString(valueCount));
    writer.write("\n");
    // 第五行，空行
    writer.write("\n");

    // 把lruEntries有效记录写入日志
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
  
  // 如果已有一份日志文件存在，就把文件备份起来
  if (journalFile.exists()) {
    renameTo(journalFile, journalFileBackup, true);
  }
  // 临时文件变为正式文件
  renameTo(journalFileTmp, journalFile, false);
  // 删除备份文件
  journalFileBackup.delete();

  // 更换journalWriter到新日志文件
  journalWriter = new BufferedWriter(
      new OutputStreamWriter(new FileOutputStream(journalFile, true), Util.US_ASCII));
}
```

#### 5.6 deleteIfExists

删除已存在文件，删除失败会抛出IOException

```java
private static void deleteIfExists(File file) throws IOException {
  if (file.exists() && !file.delete()) {
    throw new IOException();
  }
}
```

#### 5.7 renameTo

文件重命名，并根据 __deleteDestination__ 决定是否删除已存在的 __to__ 文件。

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

返回键名为 __key__ 条目的快照，若文件不存在或不可读时返回null。如果有值被返回，则该值会被移到LRU队列的头部上。

```java
public synchronized Value get(String key) throws IOException {
  checkNotClosed();
  Entry entry = lruEntries.get(key);
  // 实体不存在，返回null
  if (entry == null) {
    return null;
  }

  // 文件不可读，返回null
  if (!entry.readable) {
    return null;
  }

  // 如果文件是可以读取的
  for (File file : entry.cleanFiles) {
      // 但检查时发现文件不存在，那文件肯定是刚刚被手动删除了
      if (!file.exists()) {
          return null;
      }
  }

  // 记录读取操作的日志
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

返回 __key__ 对应条目的编辑器，如果其他编辑操作正在进行则返回null。

```java
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
    // 为不存在的键创建新条目
    entry = new Entry(key);
    lruEntries.put(key, entry);
  } else if (entry.currentEditor != null) {
    // 另一个编辑正在进行
    return null;
  }
  
  // 为编辑的文件创建Editor
  Editor editor = new Editor(entry);
  entry.currentEditor = editor;

  // 在创建文件之前刷新日志以防止文件修改记录出现遗漏
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

// 返回缓存可用于存储数据的最大字节数
public synchronized long getMaxSize() {
  return maxSize;
}

// 返回当前已用于存储缓存的字节数
// 如果后台删除操作处于待处理中，则该值可能大于最大大小
public synchronized long size() {
  return size;
}
```

#### 5.11 setter

如有必要，更改缓存可以存储的最大字节数，把安排后台任务减少已存在存储内容的占用空间。

```java
public synchronized void setMaxSize(long maxSize) {
  this.maxSize = maxSize;
  executorService.submit(cleanupCallable);
}
```

#### 5.12 completeEdit

总结为以下流程：

- 没有处于编辑状态的条目不能调用此方法，否则抛出IllegalStateException；
- __success__ 为 __true__，用 __DirtyFile__ 更新 __CleanFile__，否则删除 __DirtyFile__ ；
- 最后向日志文件写入相关最新操作；
- 检查日志文件是否太大，有没有必要进行清理；

```java
private synchronized void completeEdit(Editor editor, boolean success) throws IOException {
  Entry entry = editor.entry;
  if (entry.currentEditor != editor) {
    throw new IllegalStateException();
  }

  // 首次创建实体的话，entry.readable为false，且每个索引必须要有一个值
  if (success && !entry.readable) {
    for (int i = 0; i < valueCount; i++) {
      // 没有写入内容就抛出IllegalStateException
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
        // 把dirtyFile重命名为cleanFile
        File clean = entry.getCleanFile(i);
        dirty.renameTo(clean);
        long oldLength = entry.lengths[i];
        long newLength = clean.length();
        entry.lengths[i] = newLength;
        // 更新保存内容的总大小
        size = size - oldLength + newLength;
      }
    } else {
      // 写入出现异常，删除脏文件
      deleteIfExists(dirty);
    }
  }

  redundantOpCount++;
  entry.currentEditor = null;
  // 编辑的内容已经成功写入文件系统，再添加日志记录
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
    // 不成功则增加移除记录
    lruEntries.remove(entry.key);
    journalWriter.append(REMOVE);
    journalWriter.append(' ');
    journalWriter.append(entry.key);
    journalWriter.append('\n');
  }
  // 刷新日志文件内容
  journalWriter.flush();

  if (size > maxSize || journalRebuildRequired()) {
    executorService.submit(cleanupCallable);
  }
}
```
#### 5.13 journalRebuildRequired

仅当冗余操作数超过2000，且该值大于 __lruEntries__ 键值对的数量时重建日志

```java
private boolean journalRebuildRequired() {
  final int redundantOpCompactThreshold = 2000;
  return redundantOpCount >= redundantOpCompactThreshold
      && redundantOpCount >= lruEntries.size();
}
```

#### 5.14 remove

通过 __key__ 删除存在的条目，但正在编辑的条目不能被移除。若条目移除成功，方法返回值为true。

```java
public synchronized boolean remove(String key) throws IOException {
  checkNotClosed();
  Entry entry = lruEntries.get(key);
  // 条目不存在，正在编辑不能删除
  if (entry == null || entry.currentEditor != null) {
    return false;
  }

  for (int i = 0; i < valueCount; i++) {
    // 删除文件
    File file = entry.getCleanFile(i);
    if (file.exists() && !file.delete()) {
      throw new IOException("failed to delete " + file);
    }
    size -= entry.lengths[i];
    entry.lengths[i] = 0;
  }

  // 为移除操作写入日志
  redundantOpCount++;
  journalWriter.append(REMOVE);
  journalWriter.append(' ');
  journalWriter.append(key);
  journalWriter.append('\n');

  // 从lruEntries移除该条目记录
  lruEntries.remove(key);

  // 检查是否需要重建日志文件
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
    return;
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

移除条目直到缓存占用没有超过限制

```java
private void trimToSize() throws IOException {
  while (size > maxSize) {
    Map.Entry<String, Entry> toEvict = lruEntries.entrySet().iterator().next();
    remove(toEvict.getKey());
  }
}
```

关闭缓存并删除所有已保存的值。同时，所有保存在缓存目录的文件也会删除。

```java
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

条目所含值的快照

```java
public final class Value {
  // 条目的Key
  private final String key;
  // 快照序列号
  private final long sequenceNumber;
  private final long[] lengths;
  private final File[] files;

    private Value(String key, long sequenceNumber, File[] files, long[] lengths) {
    this.key = key;
    this.sequenceNumber = sequenceNumber;
    this.files = files;
    this.lengths = lengths;
  }

  // 返回此快照中实体的编辑器
  // 若条目从快照创建开始已改变，或另一个编辑操作正在进行，方法返回null
  public Editor edit() throws IOException {
    return DiskLruCache.this.edit(key, sequenceNumber);
  }

  public File getFile(int index) {
      return files[index];
  }

  // 返回指定索引下文件的String值
  public String getString(int index) throws IOException {
    InputStream is = new FileInputStream(files[index]);
    return inputStreamToString(is);
  }

  // 返回指定索引下字节的长度
  public long getLength(int index) {
    return lengths[index];
  }
}
```

## 七、Editor

编辑一个条目的(多个)值

```java
public final class Editor {
  private final Entry entry;
  private final boolean[] written;
  private boolean committed;

  private Editor(Entry entry) {
    this.entry = entry;
    this.written = (entry.readable) ? null : new boolean[valueCount];
  }

  // 返回无缓冲输入流以便读取最后一次提交的值。若没有提交值，则方法返回null
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

  // 把最后一次提交的值以String返回，如果没有值提交过就返回null
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

  // 提交所有编辑操作所做修改，以便新数据对所有读取者可见
  // 提交操作后编辑锁会释放，让其他编辑操作可在同一个key上开始
  public void commit() throws IOException {
    // The object using this Editor must catch and handle any errors
    // during the write. If there is an error and they call commit
    // anyway, we will assume whatever they managed to write was valid.
    // Normally they should call abort.
    completeEdit(this, true);
    committed = true;
  }

  // 终止此次编辑。此操作会释放条目的编辑锁，并允许其他编辑操作在同一个key上开展编辑工作
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

  File[] cleanFiles;
  File[] dirtyFiles;

  // 条目已经发布的话此值为True
  private boolean readable;

  // 正在进行的编辑，如果未编辑此条目，则返回null
  private Editor currentEditor;

  // 此条目最近提交的编辑的序列号。
  private long sequenceNumber;

  private Entry(String key) {
    this.key = key;
    this.lengths = new long[valueCount];
    cleanFiles = new File[valueCount];
    dirtyFiles = new File[valueCount];

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

  // 根据索引从File[]获取File对象
  public File getCleanFile(int i) {
    return cleanFiles[i];
  }
    
  // 根据索引从File[]获取File对象
  public File getDirtyFile(int i) {
    return dirtyFiles[i];
  }
}
```
