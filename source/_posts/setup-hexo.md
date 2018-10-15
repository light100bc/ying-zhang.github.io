title: Hello Hexo！
category: setup
date: 2016-09-15
tags:
---

欢迎访问。
本博客使用[Hexo](http://hexo.io/)生成。本文记录了设置过程。

<!--more-->

---

# 简介

[Hexo](https://hexo.io/)是一个基于[Node.js](https://nodejs.org/)的博客生成工具。它把 [Markdown 格式](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet)的博客文章(Post)转换成一个静态的html博客网站，除了生成文章正文的html页面，还生成了博客列表，分类，tag等辅助页面.
hexo有多种博客主题可选，这里使用了 [jane 主题](https://github.com/hejianxian/hexo-theme-jane)。
[Github](https://github.com)提供了`https://[github-userId].github.io` 这样的二级域名的静态html站点的托管服务，还可以通过`CNAME`设置自己的域名（需向域名服务商购买）。

下面的设置过程参考了[hexo简易教程](http://mclspace.com/2014/10/19/about-hexo/) 和 [hexo_install_config](http://methor.github.io/%E5%B7%A5%E5%85%B7/hexo/hexo-install-config/)。


# 在 Ubuntu 上安装所需程序

在终端执行下面的命令。

## 安装git

```
sudo apt install git
```

设置 git

```
git config --global user.name  "ZHANG Ying"
git config --global user.email "your@email.com"
git config --global core.autocrlf input  # true会将LF转换为CRLF，false则不做任何处理
```

git默认使用`~/.ssh`下的 key，需要在Github上注册对应的public key才能提交代码。

## 安装node.js

a. 从Node.js官网 https://nodejs.org/en/download/ 下载Linux-x64系统的二进制安装包，解压到`/usr/local/`。

```
cd ~
ver=v8.12.0
wget https://nodejs.org/dist/${ver}/node-${ver}-linux-x64.tar.xz

tar axf node-${ver}-linux-x64.tar.xz
cd node-${ver}-linux-x64/
sudo cp -r ./ /usr/local/
cd ..
rm -rf node-${ver}-linux-x64.tar.xz
```

b. 通过添加软件源的方式
参考[Installing Node.js via package manager](https://nodejs.org/en/download/package-manager/)，

```
curl -sL https://deb.nodesource.com/setup_8.x | sudo -E bash -
sudo apt install -y nodejs

sudo apt install -y build-essential #如果npm安装的包需要在本地编译，则需要安装编译工具如gcc等
```

# 在Windows 10上安装所需程序

## 安装git

从git官网 https://git-scm.com/downloads 下载Windows 系统的 Git客户端安装程序。
执行安装程序，建议
+ 将其安装到`C:\git`这样比较短的路径下，方便以后敲命令，
+ 并且选择 Use Git and optional Unix tools from the Windows Command Prompt ，
+ 还要在 How should Git treat line endings in text files? （处理换行符）选项中选择 Checkout as-is, commit Unix-style line endings 。

>安装git的同时会安装[MinGW(Minimalist GNU for Windows)](http://www.mingw.org/)，上面的选项会把git和MinGW的可执行程序添加到`PATH`环境变量中，这样就可以方便地在Windows命令行窗口直接使用git和其它常用的Unix命令了。
>安装git后，`C:\git\usr\bin`下面的`.exe`程序就是MinGW支持的Unix命令，如bash, ls, mv, cp, grep, cat, sort, head, tail, wc, vim, tar, curl, ssh, scp等。

打开命令窗口（快捷键为Win+X，C或A），参考上面Ubuntu系统的命令设置git 的用户名，Email。
Windows系统上git默认使用`C:\Users\[UserName]\.ssh`下的 key。
> 以`.`开头的文件夹需要在命令窗口执行`mkdir`来创建。

## 安装node.js
从Node.js https://nodejs.org/en/download/ 下载[Windows x64系统的二进制安装包](https://nodejs.org/dist/v8.12.0/node-v8.12.0-x64.msi) 。
执行安装程序，建议将其安装到`C:\nodejs`，并选择将其添加到`PATH`路径中。


# 修改 npm 源
```
npm config set registry http://registry.npm.taobao.org/
```

# 安装hexo
打开命令窗口，执行下面的命令
```
npm install hexo-cli -g
```

# 初次设置

## 初始化博客文件夹
```
hexo init blog
```

在Github创建一个名为`[github-userId].github.io`的项目。

## 安装rss插件，git插件，math插件
```
npm install hexo-generator-feed --save
npm install hexo-deployer-git   --save
npm install hexo-math           --save
```

## 设置 `_config.yml`

在 `_config.yml` 修改博客名，固定链接格式，主题，deploy repo 等。部分设置如下：

```
## Themes: https://hexo.io/themes/
theme: jane

# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repository: git@github.com:ying-zhang/ying-zhang.github.io.git
  branch: master

math:
  engine: 'mathjax'
  mathjax:
    src: https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.5/MathJax.js?config=TeX-MML-AM_CHTML
```

## 使用[jane主题](https://github.com/hejianxian/hexo-theme-jane)
这里没有使用git clone来下载 jane 主题的代码库，而是直接下载代码库的zip文件，解压到`themes/`文件夹。
直接在主题文件中修改了等宽字体，及正文的宽度。
顺便删除默认主题landscape。

> 对jane主题的修改：将侧栏移到左侧，这个在项目repo已经[有人提过issue](https://github.com/hejianxian/hexo-theme-jane/issues/5)，作者的方案需要手动修改`style.sty`，跟我摸索发现的一样，我还修改了滚动条显示的样式。


## 文件结构

| 文件/文件夹     |       说明     |
| --------------- | -------------- |
| `_config.yml`   | 博客的主配置文件 |
| `.deploy_git/`  | 用于push到Github、heroku等的git仓库，里面有与`public/`相同的内容及`.git`的记录 |
| `public/`       | 生成的完整的静态html网站，执行 `hexo clean` 会清除此文件夹 |
| `scaffolds/`    | 里面有三个`.md`文件，其中`post.md`是博客文章的默认模板 |
| `node_modules/` | 每个blog的文件夹中安装的hexo插件 |
| `source/`       | 撰写的博客文章要放在`source/_post`下，`source/`文件夹下的.md文件都会被生成为.html文件，其它类型的文件和文件夹则保持原样复制到 `public/`。可以在这里新建一个 `img/` 文件夹用于保存图片，文章中可以用`![图片说明](/img/image.png)`这样的代码来插入图片。 |
| `themes/`       | 存放主题相关文件，各主题也有自己的`_config.yml。` |


## 修改`scaffold/post.md`模板
增加catagory，tags，及摘要和正文的分割线`<!--more-->`。
> 注意，Markdown对空格和空行敏感。

```
---
title: {{ title }}
date: {{ date }}
category: misc
tags: [tag1, tag2]
---
摘要部分
<!--more-->
---
正文部分
```

## 修改 hexo-server默认的4000端口号为80
如果本机没有跑web服务器占用80端口的话，可以让hexo-server使用80端口而不是默认的4000端口，这样在浏览器直接输入机器名就可以预览blog了。
需修改`node_modules/hexo-server/index.js` 第8行 `port`。
比如我的机器名是z，在chrome地址栏直接输入 `z/` 即可，ie则需要输入 http://z/

# hexo 基本操作

## 常用命令

{% codeblock line_number:false%}
hexo s     # server   本地预览，http://localhost:4000 。
hexo d     # deploy   部署整个网站到heroku
hexo g     # generate 生成静态html

hexo s -g  # 本地预览，编辑了博客文章后，不必重新运行该命令，刷新页面即可，直到按 `Ctrl + C`会退出。
hexo d -g  # 依次执行生成和部署

hexo clean # 删除 public/ 文件夹下所有内容
{% endcodeblock %}

## 撰写文章
执行下面的命令，在`source/_posts/`下生成名为 `New-Post.md` 的文件。
也可以通过常规的文件操作在`source/_posts/`下新建一个 `New-Post.md` 文件。

{% codeblock line_number:false%}
hexo n "New Post"
{% endcodeblock %}

更多详情可参考[使用hexo写作](http://hexo.io/docs/writing.html)的文档。

## 发布静态html的博客网站到 Github
{% codeblock line_number:false%}
hexo d -g
{% endcodeblock %}

## 为整个blog建立repo

Hexo可以将生成的静态html博客网站发布到git仓库，但不会同步`source/`下源文件，`_config.yml`设置等。
如果需要在别的机器上写blog，还要重新设置，并拷贝这些文件。
为此，为已经设置好的博客建立一个新的git repo。

{% codeblock line_number:false%}
cd blog/
git init
{% endcodeblock %}

编辑`.gitignore`如下。
{% codeblock line_number:false%}
.DS_Store
Thumbs.db
node_modules/
themes/
public/
.deploy_git/
.npmignore
{% endcodeblock %}
> 注意：把`node_modules/`和`themes/`排除，但把这两个文件夹分别压缩后，这里使用7zip压缩，生成`node_modules.7z` 和 `themes.7z` 会包括在git repo中。


在Github创建一个名为`blog`的项目，
{% codeblock line_number:false%}
git add .
git commit -m "Init"
git remote add github git@github.com:ying-zhang/blog.git
git push -u github master
{% endcodeblock %}

## 克隆已有的blog项目
需要已经安装好了 git，nodejs 和 hexo。
{% codeblock line_number:false%}
git clone git@github.com:ying-zhang/blog.git
cd blog

git remote rename origin github

npm install --save
7z x themes.7z
{% endcodeblock %}

# Tips

Windows 下编辑文本需要注意编码，应使用 utf8 无bom 格式的编码，建议使用 VS Code 编辑器。

## 编辑数学公式
注意，需要`hexo-math`插件，但这个插件的作者已经停止维护了，不知有什么替代的。
```
{% math %}
\begin{align*}
\begin{align*}
-\sum_i{P_i\log_2\frac{P_i}{Q_i}} =& \sum_i{P_i\log_2\frac{Q_i}{P_i}} \\
                                \le& \log_2\sum_i{P_i\frac{Q_i}{P_i}} \tag{Jensen 不等式} \\
                                  =& \log_2\sum_i{Q_i} \\
                                  =& 0
\end{align*}
{% endmath %}
```

{% math %}
\begin{align*}
-\sum_i{P_i\log_2\frac{P_i}{Q_i}} =& \sum_i{P_i\log_2\frac{Q_i}{P_i}} \\
                                \le& \log_2\sum_i{P_i\frac{Q_i}{P_i}} \tag{Jensen 不等式} \\
                                  =& \log_2\sum_i{Q_i} \\
                                  =& 0
\end{align*}
{% endmath %}

## 设置代码块不显示行号
```
{% codeblock line_number:false%}
echo "Hello Hexo!"
{% endcodeblock %}
```

---

飞鸟集 第128

>如果你不等待着要说出完全的真理，那末把话说出来是很容易的。

所以，就随便写吧。
