---
layout:     post
title:      "解决 mac OSX pip OSError: [Errno 1] Operation not permitted"
date:       2016-10-26
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    false
tags:
    - Python
---


这段时间研究机器学习，然后发现Mac OS的Python库有点旧，就用pip更新一下：

```bash
$ pip install --upgrade numpy
```

结果抛出下面异常：

```
Exception:
Traceback (most recent call last):
  File "/Library/Python/2.7/site-packages/pip-7.1.0-py2.7.egg/pip/basecommand.py", line 223, in main
    status = self.run(options, args)
  File "/Library/Python/2.7/site-packages/pip-7.1.0-py2.7.egg/pip/commands/install.py", line 299, in run
    root=options.root_path,
  File "/Library/Python/2.7/site-packages/pip-7.1.0-py2.7.egg/pip/req/req_set.py", line 640, in install
    requirement.uninstall(auto_confirm=True)
  File "/Library/Python/2.7/site-packages/pip-7.1.0-py2.7.egg/pip/req/req_install.py", line 726, in uninstall
    paths_to_remove.remove(auto_confirm)
  File "/Library/Python/2.7/site-packages/pip-7.1.0-py2.7.egg/pip/req/req_uninstall.py", line 125, in remove
    renames(path, new_path)
  File "/Library/Python/2.7/site-packages/pip-7.1.0-py2.7.egg/pip/utils/__init__.py", line 314, in renames
    shutil.move(old, new)
  File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/shutil.py", line 302, in move
    copy2(src, real_dst)
  File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/shutil.py", line 131, in copy2
    copystat(src, dst)
  File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/shutil.py", line 103, in copystat
    os.chflags(dst, st.st_flags)
OSError: [Errno 1] Operation not permitted: '/tmp/pip-1_nHcH-uninstall/System/Library/Frameworks/Python.framework/Versions/2.7/Extras/lib/python/numpy-1.8.0rc1-py2.7.egg-info'
```

Google后找到解决方案：

```shell
$ pip install --upgrade pip

$ sudo pip install numpy --upgrade --ignore-installed
$ sudo pip install scipy --upgrade --ignore-installed
$ sudo pip install scikit-learn --upgrade --ignore-installed
```

