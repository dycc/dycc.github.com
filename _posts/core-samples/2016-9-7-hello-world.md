---
layout: post
category : lessons
tagline: "Supporting tagline"
tags : [intro, beginner, jekyll, tutorial]
---
{% include JB/setup %}

## Git简介

引用：_config.yml中的键值均引用到其他页面{{ site.title }}；

### What is Jekyll?

### 4.6 个性化博客

Github Page完成了博客的主要功能，写作、发布、修改，这些都是相对静态的东西，就是你自己可以控制的事情，还有一些动态的东西Github Pages无法支持，比如说文章浏览次数、文章的评论等，还有一些个性化的东西，像个性化页头页尾、代码高亮可以把博客整的更漂亮一点，其实这写都可以通过第三方应用来实现，个性化自己的博客。

### 5 使用Markdown

Markdown 的宗旨是实现「易读易写」。可读性，无论如何，都是最重要的。
Markdown 的语法全由一些符号所组成，这些符号经过精挑细选，其作用一目了然。格式撰写的文件可以直接以纯文本发布，并且看起来不会像是由许多标签或是格式指令所构成。

### 5.2 基本语法

使用一个或多个空行分隔内容段来生成段落 <p>。
标题（h1~h6）格式为使用相应个数的“#”作前缀，比如以下代码表示 h3：

** this is a level-3 header.**

- 使用 4 个以上 空格或 1 个以上 的 tab 来标记代码段落，它们将被<pre> 和 <code> 包裹，这意味着代码段内的字体会是 monospace家族的，并且特殊符号不会被转义。
- 使用 [test](http://example.net "optional title") 来标记普通链接。
- 使用 ![img](http://example.net/img.png "optional title") 来标记图片。

# 5.3 Notepad++支持Markdown语法高亮

- 1. 下载userDefineLang.xml
- 2. 将 userDefineLang.xml 放置到此目录：C:\Users\Administrator\AppData\Roaming\Notepad++
- 3. 重启 Notepad++，在语言菜单下可以看到自定义的 Markdown 高亮规则