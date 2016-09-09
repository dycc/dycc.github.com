---
title: 'GitHub+Ruby+DevKit+Jekyll搭建博客'
layout: post
guid: urn:uuid:2016-09-07-ruby-jekyll-blog
tags:
    - GitHub
    - Ruby
    - Jekyll
---

![乖，摸摸头](/../media/files/2016/9/07-1.jpg "乖，摸摸头")

# `Git` -> `Ruby` -> `Devkit` -> `Jekyll`

## 1. Git

### 1.1 Git简介
Git是一个开源的分布式版本控制系统，用以有效、高速的处理从很小到非常大的项目版本管理。
GitHub可以托管各种git库的站点。
GitHub Pages免费的静态站点，三个特点：免费托管、自带主题、支持自制页面和Jekyll。

### 1.2 Git常用命令

    $ git clone git@github.com:username/username.github.com.git //本地如果无远程代码，先做这步
    $ cd .ssh/username.github.com //定位到你blog的目录下
    $ git pull origin master //先同步远程文件，后面的参数会自动连接你远程的文件
    $ git status //查看本地自己修改了多少文件
    $ git add . //添加远程不存在的git文件
    $ git commit * -m "what I want told to someone"
    $ git push origin master //更新到远程服务器上

## 2. Ruby

### 2.1 Ruby简介
Ruby，一种简单快捷的面向对象（面向对象程序设计）脚本语言，在20世纪90年代由日本人松本行弘(Yukihiro Matsumoto)开发，遵守GPL协议和Ruby License。
它的灵感与特性来自于 `Perl`、`Smalltalk`、`Eiffel`、`Ada`以及 `Lisp` 语言。由 Ruby 语言本身还发展出了JRuby（Java平台）、IronRuby（.NET平台）等其他平台的 Ruby 语言替代品。
Ruby的作者于1993年2月24日开始编写Ruby，直至1995年12月才正式公开发布于fj（新闻组）。因为Perl发音与6月诞生石pearl（珍珠）相同，因此Ruby以7月诞生石ruby（红宝石）命名。

### 2.2 Ruby安装
下载并安装 RubyInstaller for Windows。
版本可以选择2.0或者1.9.3都可以。
双击安装，安装时选中“Add Ruby executables to your PATH”前的框将ruby添加到环境变量中。
安装完成后，在 Windows 命令行窗口中执行以下命令，检查ruby是否已经加到PATH中： `ruby --version`。

### 2.3 Ruby Devkit安装
下载最新的DevKit，DevKit是windows平台下编译和使用本地C/C++扩展包的工具。它就是用来模拟Linux平台下的make,gcc,sh来进行编译。
但是这个方法目前仅支持通过RubyInstaller安装的Ruby，并双击运行解压到C:\DevKit。
然后打开终端cmd，输入下列命令进行安装：

    cd C:\DevKit
    ruby dk.rb init
    ruby dk.rb install

## 3. Jekyll
完成上面的准备就可以安装Jekyll了,因为Jekyll是用Ruby编写的,最好的安装方式是通过RubyGems(gem):

    gem install Jekyll

并使用命令检验是否安装成功

    jekyll --version

安装Rdiscount，这个用来解析Markdown标记的包，使用如下命令：

    gem install rdiscount

cd 到工程目录，启动服务：

    jekyll --server
