## zshrc

1. 普通终端开代理，在 `.zshrc` 最后添加述代理设置

```
export ALL_PROXY="http://127.0.0.1:1087"
```

检查ip是否连通

``` shell 
curl cip.cc
```


2. 安装 __Homebrew__

``` shell
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
```

4. 安装 Python `brew install python`，并重启终端

5. 创建文件 **～/.pip/pip.conf**，添加以下内容走国内加速源

```
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
[install]
trusted-host = pypi.tuna.tsinghua.edu.cn
```
6. 用 __pip__  安装：`pip3 install powerline-status --user`

7. 新建并进入 `zshrc`，执行

```shell
git clone https://github.com/powerline/fonts.git --depth=1
cd fonts
./install.sh
```

9. 安装好字体库后设置iTerm2字体，具体的操作：iTerm2 -> Preferences -> Profiles -> Text，在Font区域选中Change Font，然后找到Meslo LG字体

10. 下载 solarized，双击两个配置文件加载
```
git clone https://github.com/altercation/solarized
cd solarized/iterm2-colors-solarized/
open .
```

11. 再次进入iTerm2 -> Preferences -> Profiles -> Colors -> Color Presets中根据个人喜好选择这两种配色

12. 下载

```shell
git clone https://github.com/fcamblor/oh-my-zsh-agnoster-fcamblor.git
cd oh-my-zsh-agnoster-fcamblor/
./install
```

13. 执行命令打开zshrc配置文件，将ZSH_THEME后面的字段改为 __agnoster__

14. 下载以下两个库
```shell
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting

git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```

打开 __zshrc__ 文件进行编辑。找到 __plugins__，此时 __plugins__ 中应该已经有了git，我们需要把高亮插件也加上：

```
plugins=(
  git
  zsh-autosuggestions
  zsh-syntax-highlighting
)
```





## Gem加速源

```shell
# 移除gem默认源，改成ruby-china源
$ gem sources -r https://rubygems.org/ -a https://gems.ruby-china.com/
# 使用Gemfile和Bundle的项目，可以做下面修改，就不用修改Gemfile的source
$ bundle config mirror.https://rubygems.org https://gems.ruby-china.com
# 删除Bundle的一个镜像源
$ bundle config --delete 'mirror.https://rubygems.org'
```

