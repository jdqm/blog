---
title: 利用oh-my-zsh打造漂亮的终端
date: 2017-08-28 21:19:45
updated: 2017-08-28 21:19:45
tags:
    -shell
categories: linux
---

前几天发现同事的终端窗口(terminal)比较漂亮，于是我就问了一下原来是安装了**[oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh)**，于是趁着今天周日我就开始折腾了，现在把这个过程记录下来分享一下。开始之前先上两张图来体现一下**目标**与**现状**
<!-- more -->
![目标](http://upload-images.jianshu.io/upload_images/3631399-af325fc01f055af4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这就是今天的目标，假设你不喜欢这个界面，那也没关系，oh-my-zsh有140多个主题，任君选择，当然你也可以定制自己的主题。

![现状](http://upload-images.jianshu.io/upload_images/3631399-064567e5aff0d8c6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个是我现在的终端窗口界面，好像也不难看，但是太单调了感觉。好了，进入正题吧。

## I. 安装oh-my-zsh
安装是有一些先决条件的：
- Unix-like操作系统（如 macOS, Linux）
- 确保Zsh已安装，且是v4.3.9及以上版本，可通过命令 zsh --version 确认，我现在的版本是 zsh 5.2 (x86_64-apple-darwin16.0) ，如果你没有安装或者版本过低，请点这里 [Installing ZSH](https://github.com/robbyrussell/oh-my-zsh/wiki/Installing-ZSH)
- 已安装curl或者wget（我的mac系统默认已安装了curl）
- 已安装git

根据你自己的情况在终端执行如下命令进行安装：

通过 curl 安装（我选择的方式）
```
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

通过 wget 安装

```
sh -c "$(wget https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh -O -)"
```

安装过程中有一个步骤需要输入密码（是切换默认shell解释器的命令需要授权 chsh -s /bin/zsh ，因为我的mac默认shell解释器是bash，所以要切换到zsh），你可以通过 cat /etc/shells 来查看你的机器安装了哪些shell解释器，到这oh-my-zsh安装完成，终端界面已经发生了改变，使用了默认的主题（robbyrussell）。


![默认主题效果图](http://upload-images.jianshu.io/upload_images/3631399-81f423a58c62caf4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## II. 切换主题

使用vi编辑器打开 ~/.zshrc 这个文件

```
vi ~/.zshrc
```
将

```
ZSH_THEME="robbyrussell"
```
改为

```
ZSH_THEME="agnoster"
```
后保存，然后执行

```
source ~/.zshrc
```

## III. 配置颜色方案

我采用的是 Solarized Dark 配色方案。
- 首先你需要Solarized下载配色方案，可以在执行如下命令通过git来下载（你也可以直接到这个网址下载你想要的文件：https://github.com/altercation/solarized）
```
git clone git://github.com/altercation/solarized.git
```
这样在你执行这个命令的目录就会创建一个solarized文件夹，我们要的配色方案就下载这里面，按照如下步骤导入 /solarized/osx-terminal.app-colors-solarized/Solarized Dark ansi.terminal 这个文件。

![import-color.png](http://upload-images.jianshu.io/upload_images/3631399-9b02921889f1b86c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

导入完成之后，设置为默认的描述文件。另外还需要设置一下启动时使用的的描述文件为“Solarized Dark ansi”，设置完成后关闭终端重新打开。

![设置启动终端是使用的描述文件](http://upload-images.jianshu.io/upload_images/3631399-767ba2a7b4670ac6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## IV. 安装Powerline字体
由于agnoster主题使用的是Powerline字体，所以我们要安装至少一个这样的字体库。依次执行下面的命令即可完成安装。
```
# clone
git clone https://github.com/powerline/fonts.git --depth=1
# install
cd fonts
./install.sh
# clean-up a bit
cd ..
rm -rf fonts
```
参考资料 [https://github.com/powerline/fonts](https://github.com/powerline/fonts)

## V. 更改shell的字体
- 选中终端窗口，点击左上角的 终端-->偏好设置-->描述文件-->文本-->字体，点击更改


![偏好设置面板](http://upload-images.jianshu.io/upload_images/3631399-d5989f07253e45e7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 选择一个powerline字体（我选择的是Anonymous Pro for Powerline ，常规体，14pt）

![选择字体](http://upload-images.jianshu.io/upload_images/3631399-a058884bf1f29802.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

到这里基本完成了，只是还有一些细节要修改，上一张效果图看看

![设置完配色方案、字体后的效果图](http://upload-images.jianshu.io/upload_images/3631399-8e100fb00d4f2963.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

还有两个细节需要修改
①yangsheng@yangshengdeMacBook-Pro 这个东西太长，并且没什么用，我想把它隐藏掉。这个就要修改agnoster主题的配置文件了。打开~/.oh-my-zsh/themes/agnoster.zsh-theme
```
vi ~/.oh-my-zsh/themes/agnoster.zsh-theme
```
找到下面的内容

```
prompt_context() {
  if [[ "$USER" != "$DEFAULT_USER" || -n "$SSH_CLIENT" ]]; then
    prompt_segment black default "%(!.%{%F{yellow}%}.)$USER@%m"
  fi
}
```
将 
```
prompt_segment black default "%(!.%{%F{yellow}%}.)$USER@%m" 
```
这一行用“#”注释掉即可。保存后执行一下 source ~/.zshrc 重新加载配置，你会发现那一串文字不见了。

②这个蓝色不好，连字都看不清楚。处理这个问题前，我推荐mac用户下载一个 iterm2(也许你会爱上它，进而抛弃terminal)，下载地址 [http://www.iterm2.com/downloads.html](http://www.iterm2.com/downloads.html)

- 下载安装完成之后，打开它，选择左上角的iterm2-->preferences-->Profiles-->color-->color Presets..，选择import我们下载的配色方案```/solarized/iterm2-colors-solarized/Solarized Dark.itermcolors```
![导入配色方案](http://upload-images.jianshu.io/upload_images/3631399-7769a8597e5a28f2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 选择Powerline字体(我选择的是Anonymous Pro for Powerline ，常规体，14pt)

![select-iterm2-font.png](http://upload-images.jianshu.io/upload_images/3631399-f09c74254ac6bfca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

打开一个新的iterm2，已经实现了我们今天的目标，很帅。如果你愿意使用iterm2替代terminal，你就不用往下看了。

![目标效果已实现](http://upload-images.jianshu.io/upload_images/3631399-6c8705070a2aff13.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

作为有一点强迫症的我，现在要把 terminal 的颜色改成与 iterm2 一致。


![terminal颜色面板](http://upload-images.jianshu.io/upload_images/3631399-4dda9fc0a7c09a5e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![iterm2颜色面板](http://upload-images.jianshu.io/upload_images/3631399-b0dabd03298fe598.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

拿着小笔尽情地吸吧，修改完成后，terminal的外观与iterm2就一致了。关于oh-my-zsh的内容实在太多，插件、主题都比较多，后续再慢慢体会。

oh-my-zsh开源地址: [https://github.com/robbyrussell/oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh)