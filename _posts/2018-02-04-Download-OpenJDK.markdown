---
layout:     post
title:      "下载OpenJDK源码"
date:       2018-02-04
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - JDK
---

## 安装mercurial

在MacOS下示例，先通过`easy_install`安装`mercurial`

```bash
$ sudo easy_install mercurial
```

安装过程提示

```bash
Searching for mercurial
Reading https://pypi.python.org/simple/mercurial/
Best match: mercurial 4.5
Downloading https://pypi.python.org/packages/d5/31/513699382639ceb525f0fd3989ba060674ed4e7b1745d5939979eb6d4d8a/mercurial-4.5.tar.gz#md5=5ca07ebb0c7f7eeb7b5a8ca9822cb8f1
Processing mercurial-4.5.tar.gz
Writing /tmp/easy_install-f5dkOr/mercurial-4.5/setup.cfg
Running mercurial-4.5/setup.py -q bdist_egg --dist-dir /tmp/easy_install-f5dkOr/mercurial-4.5/egg-dist-tmp-YDY1bx
zip_safe flag not set; analyzing archive contents...
hgdemandimport.demandimportpy2: module references __path__
hgext3rd.__init__: module references __path__
mercurial.lsprof: module references __file__
mercurial.sslutil: module references __file__
mercurial.debugcommands: module references __file__
mercurial.i18n: module references __file__
mercurial.chgserver: module MAY be using inspect.getabsfile
mercurial.extensions: module references __file__
mercurial.ui: module MAY be using inspect.getouterframes
mercurial.statprof: module references __file__
mercurial.statprof: module MAY be using inspect.getsource
mercurial.statprof: module MAY be using inspect.stack
mercurial.util: module references __file__
mercurial.cffi.mpatchbuild: module references __file__
mercurial.cffi.bdiffbuild: module references __file__
hgext.mq: module references __file__
hgext.__init__: module references __path__
creating /Library/Python/2.7/site-packages/mercurial-4.5-py2.7-macosx-10.13-intel.egg
Extracting mercurial-4.5-py2.7-macosx-10.13-intel.egg to /Library/Python/2.7/site-packages
Adding mercurial 4.5 to easy-install.pth file
Installing hg script to /usr/local/bin

Installed /Library/Python/2.7/site-packages/mercurial-4.5-py2.7-macosx-10.13-intel.egg
Processing dependencies for mercurial
Finished processing dependencies for mercurial
```

检查`mercurial`安装是否成功

```bash
$ hg --version

Mercurial Distributed SCM (version 4.5)
(see https://mercurial-scm.org for more information)

Copyright (C) 2005-2018 Matt Mackall and others
This is free software; see the source for copying conditions. There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

## 克隆源

```bash
$ hg clone http://hg.openjdk.java.net/jdk9/jdk9

destination directory: jdk9
requesting all changes
adding changesets
adding manifests
adding file changes
added 2628 changesets with 4461 changes to 468 files
new changesets cfeea66a3fa8:a08cbfc0e4ec
updating to branch default
322 files updated, 0 files merged, 0 files removed, 0 files unresolved
```

## 获取源码

```bash
$ cd jdk9
$ bash get_source.sh
```

安装过程如下示意，没有提示错误即完成

```bash
# Repositories:  corba jaxp jaxws langtools jdk hotspot nashorn
                corba:   hg clone http://hg.openjdk.java.net/jdk9/jdk9/corba corba
                 jaxp:   hg clone http://hg.openjdk.java.net/jdk9/jdk9/jaxp jaxp
                 jaxp:   requesting all changes
                corba:   requesting all changes
                 jaxp:   adding changesets
                corba:   adding changesets
                corba:   adding manifests
                 jaxp:   adding manifests
                corba:   adding file changes
                 jaxp:   adding file changes
                corba:   added 876 changesets with 5451 changes to 2597 files
                corba:   new changesets 55540e827aef:5666eba44ac6
                corba:   updating to branch default
                corba:   1201 files updated, 0 files merged, 0 files removed, 0 files unresolved
                jaxws:   hg clone http://hg.openjdk.java.net/jdk9/jdk9/jaxws jaxws
                jaxws:   requesting all changes
                jaxws:   adding changesets
                jaxws:   adding manifests
                jaxws:   adding file changes
                 jaxp:   added 1153 changesets with 14751 changes to 8449 files
                 jaxp:   new changesets 6ce5f4757bde:364631d8ff2e
                 jaxp:   updating to branch default
                 jaxp:   3352 files updated, 0 files merged, 0 files removed, 0 files unresolved
            langtools:   hg clone http://hg.openjdk.java.net/jdk9/jdk9/langtools langtools
            langtools:   requesting all changes
            langtools:   adding changesets
            langtools:   adding manifests
            langtools:   adding file changes
                jaxws:   added 801 changesets with 21839 changes to 10824 files
                jaxws:   new changesets 0961a4a21176:a1d64f45f9d5
                jaxws:   updating to branch default
                jaxws:   3760 files updated, 0 files merged, 0 files removed, 0 files unresolved
                  jdk:   hg clone http://hg.openjdk.java.net/jdk9/jdk9/jdk jdk
                  jdk:   requesting all changes
                  jdk:   adding changesets
                  jdk:   adding manifests
            langtools:   added 4174 changesets with 38097 changes to 11847 files
            langtools:   new changesets 9a66ca7c79fa:65bfdabaab9c
            langtools:   updating to branch default
            langtools:   9464 files updated, 0 files merged, 0 files removed, 0 files unresolved
              hotspot:   hg clone http://hg.openjdk.java.net/jdk9/jdk9/hotspot hotspot
              hotspot:   requesting all changes
              hotspot:   adding changesets
              hotspot:   adding manifests
              hotspot:   adding file changes
                  jdk:   adding file changes
              hotspot:   added 12824 changesets with 78616 changes to 15832 files
              hotspot:   new changesets a61af66fc99e:b756e7a2ec33
              hotspot:   updating to branch default
              hotspot:   9078 files updated, 0 files merged, 0 files removed, 0 files unresolved
              nashorn:   hg clone http://hg.openjdk.java.net/jdk9/jdk9/nashorn nashorn
              nashorn:   requesting all changes
              nashorn:   adding changesets
              nashorn:   adding manifests
              nashorn:   adding file changes
              nashorn:   added 1928 changesets with 14563 changes to 4181 files
              nashorn:   new changesets b8a1b238c77c:17cc754c8936
              nashorn:   updating to branch default
              nashorn:   3293 files updated, 0 files merged, 0 files removed, 0 files unresolved
                  jdk:   added 17287 changesets with 152446 changes to 50650 files
                  jdk:   new changesets 37a05a11f281:65464a307408
                  jdk:   updating to branch default
                  jdk:   27295 files updated, 0 files merged, 0 files removed, 0 files unresolved
# Repositories:  . corba jaxp jaxws langtools jdk hotspot nashorn
                    .:   cd . && hg pull -u
                corba:   cd corba && hg pull -u
                 jaxp:   cd jaxp && hg pull -u
                jaxws:   cd jaxws && hg pull -u
            langtools:   cd langtools && hg pull -u
                  jdk:   cd jdk && hg pull -u
              hotspot:   cd hotspot && hg pull -u
              nashorn:   cd nashorn && hg pull -u
                    .:   pulling from http://hg.openjdk.java.net/jdk9/jdk9
                corba:   pulling from http://hg.openjdk.java.net/jdk9/jdk9/corba
                 jaxp:   pulling from http://hg.openjdk.java.net/jdk9/jdk9/jaxp
                jaxws:   pulling from http://hg.openjdk.java.net/jdk9/jdk9/jaxws
                  jdk:   pulling from http://hg.openjdk.java.net/jdk9/jdk9/jdk
              hotspot:   pulling from http://hg.openjdk.java.net/jdk9/jdk9/hotspot
            langtools:   pulling from http://hg.openjdk.java.net/jdk9/jdk9/langtools
              nashorn:   pulling from http://hg.openjdk.java.net/jdk9/jdk9/nashorn
                    .:   searching for changes
                    .:   no changes found
                corba:   searching for changes
                corba:   no changes found
                 jaxp:   searching for changes
                 jaxp:   no changes found
                  jdk:   searching for changes
                  jdk:   no changes found
              nashorn:   searching for changes
              nashorn:   no changes found
                jaxws:   searching for changes
                jaxws:   no changes found
              hotspot:   searching for changes
              hotspot:   no changes found
            langtools:   searching for changes
            langtools:   no changes found
```

如果下载JDK8u而不是JDK9，自行替换上述终端命令即可。

