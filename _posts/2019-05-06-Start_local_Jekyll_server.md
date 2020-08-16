---
layout:     post
title:      "安装Jekyll本地环境"
date:       2019-05-06
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    false
tags:
    - Website
---

以下环境以 __Ubuntu__ 为例，如果环境为 __Mac OS__ 则把 __apt install__ 换成 __brew install__

__Jekyll__ 依赖 __Ruby__ 语言，需要先安装

```bash
$ sudo apt install ruby
```

类似 __pip__ 于 __Python__，__RubyGems__ 是 __Ruby__ 的包管理器，其终端命令为 __gem__。安装 __Ruby__ 之后用 __gem__ 直接安装 __Jekyll__ 可能会报以下错误：

```bash
$ sudo gem install jekyll 
Fetching: public_suffix-3.1.1.gem (100%)
Successfully installed public_suffix-3.1.1
Fetching: addressable-2.6.0.gem (100%)
Successfully installed addressable-2.6.0
Fetching: colorator-1.1.0.gem (100%)
Successfully installed colorator-1.1.0
Fetching: http_parser.rb-0.6.0.gem (100%)
Building native extensions. This could take a while...
ERROR:  Error installing jekyll:
	ERROR: Failed to build gem native extension.

    current directory: /var/lib/gems/2.5.0/gems/http_parser.rb-0.6.0/ext/ruby_http_parser
/usr/bin/ruby2.5 -r ./siteconf20190706-17669-1vutbo1.rb extconf.rb
mkmf.rb can't find header files for ruby at /usr/lib/ruby/include/ruby.h

extconf failed, exit code 1

Gem files will remain installed in /var/lib/gems/2.5.0/gems/http_parser.rb-0.6.0 for inspection.
Results logged to /var/lib/gems/2.5.0/extensions/x86_64-linux/2.5.0/http_parser.rb-0.6.0/gem_make.out
```

因此需安装 __Ruby__ 的开发插件

```bash
$ sudo apt-get install ruby`ruby -e 'puts RUBY_VERSION[/\d+\.\d+/]'`-dev
```

配置 __Gem__ 国内加速源

```shell
$ sudo gem sources -r https://rubygems.org/ -a https://gems.ruby-china.com/
```

安装 __Jekyll__

```bash
$ sudo gem install jekyll
```

其次 __Jekyll__ 还依赖 __jekyll-paginate__

```bash
$ sudo gem install jekyll-paginate
```

所有配置完成后，终端移动到目标文件夹下启动服务即可

```bash
$ jekyll serve
```

参考链接：

- [Error while installing json gem 'mkmf.rb can't find header files for ruby'](https://stackoverflow.com/q/20559255/8750399)