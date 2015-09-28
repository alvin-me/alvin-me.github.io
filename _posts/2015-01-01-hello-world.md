---
layout: post
title: "起源"
tagline: "博客部署"
category : tech
tags : [Jekyll]
---

闲暇之余，便想着搭建个人博客。现在，你看到的这个网站就是我的成果啦。网站是使用 Jekyll 生成，Bootstrap 主题风格，使用 Markdown 撰写博客，并托管于 Github Pages。

林林总总的查阅了很多资料，将其整理在下面。

## 零、知识储备
 
想要完全的定制一个自己的博客，需要了解这些知识： 

* `Github` 管理网页源码
* `Jekyll` 静态站点生成器，它会根据网页源码生成静态文件
* `Liquid` 一种标记语言，Jekyll使用它来输出变量和处理逻辑判断
* `Markdown` 一种轻量级标记语言，写博客就用它
* `HTML`
* `CSS`
* `JavaScript` 网络脚本语言，用来改进设计、验证表单、检测浏览器、创建cookies，以及更多的应用
* `jQuery` 一个JavaScript库，简化 JavaScript 编程
* `Bootstrap` 来自Twitter，目前最受欢迎的前端框架,基于HTML、CSS、JavaScript
 
## 一、创建项目文件

以下是一个获取最简单 Jekyll 模板并生成静态页面的方法。它会创建 myblog 项目文件，通过在浏览器中输入http://localhost:4000便可在本地访问新生成的博客站点。

```bash
~ $ gem install jekyll
~ $ jekyll new myblog
~ $ cd myblog
~/myblog $ jekyll serve
# => Now browse to http://localhost:4000
```

或者可以[手动创建所有项目文件](http://www.ruanyifeng.com/blog/2012/08/blogging_with_jekyll.html)，这种方式有利于深入了解 Jekyll 目录结构。

下面是一个最基本的 Jekyll 项目目录结构。

    .
    |-- _config.yml
    |-- _layouts
    |   |-- default.html
    |-- _posts
    |   |-- 2012-10-25-hello-wrold.md
    |-- index.html
    
## 二、解析 Jekyll

两个地方深入了解 Jekyll : [Jekyll 主页](http://jekyllcn.com/) 和 [Jekyll-Bootstrap](http://jekyllbootstrap.com/) 。

* Jekyll-Bootstrap 是 Jekyll 原作者推荐的学习示例，包括了博客的多个常见模块，如归档、分类、标签、评论、搜索、统计等。 首先需要了解 [HTML](http://www.w3school.com.cn/h.asp)、[CSS](http://www.w3school.com.cn/css/index.asp)、[JavaScript](http://www.w3school.com.cn/js/index.asp）)、[jQuery](http://www.w3school.com.cn/jquery/index.asp) 。
* 使用 [Liquid](https://github.com/shopify/liquid/wiki/liquid-for-designers) 标记语言存储变量和处理逻辑， 在 Jekyll 中预定义了一些[模板变量](http://jekyllrb.com/docs/variables/)。
* 使用 [Markdown](http://wowubuntu.com/markdown/index.html) 标记语言撰写博客。
  
## 三、定制博客
    
确实仔细的阅读了以上的链接信息以后，应该会对整个博客组成原理有了清晰的认识，下面我们就可以轻松地定制自己想要的博客样式了。
  
* [添加博客评论](http://liberize.me/tech/jekyll-use-duoshuo-comment-system.html)，采用的是目前国内最受欢迎的`多说`评论，就是下面的那个评论。 Jekyll-Bootstrap 默认博客评论系统是 `Disqus`，然而 Disqus 在国内访问速度和稳定性并不理想，而且无法和国内的各种社交网站耦合。
* [更多的 Jekyll 主题](http://jekyllthemes.org/)。
* [Twitter Bootstrap](http://www.bootcss.com/) 是一个不错的前端开发构架，提供美观的界面元素，可以学习王皓的[Bootstrap用户界面架构视频教程](http://www.icoolxue.com/album/show/78)，以深入了解。

## 四、可能会遇到这些问题

__Windows 下运行 Jekyll Serve 命令没有错误，但无法启动服务站点？__

高版本的 Jekyll 有查看代码改变，并且自动再生成的功能，该功能在 Windows 下支持不是很好。解决方法是使用低版本的 Jekyll ，可以运行以下命令安装([link](https://github.com/jekyll/jekyll/issues/3221))。 

```bash
Jekyll –v
Gem uninstall Jekyll
Gem install Jekyll –version “=2.1.1”
```