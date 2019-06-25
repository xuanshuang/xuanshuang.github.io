---
title: 用github.io搭建一个博客
date: 2019-02-14 00:13:14
tags: 
- Hexo
- next
- Github Page
---
迫于生计，弄一个博客润色润色苍白的简历。
<!--more-->

## Why
哎，工作不好找，日子不好过啊。从上学开始，就看身边的同学各种捣鼓个人网站。因为懒，一直也就看看，平时拿个记事本记录记录就算了。浏览招聘要求时发现，有个人博客加分啊，好的坏的至少还有印象分啊，趁着空，还是搞一个吧。

## How
* 博客要好看啊，但是只能“执着像素级还原”，没有原创审美啊，只好上网找了找开源的框架，毕竟内涵是博文不是^_^。于是使用了`hexo`和`hexo-theme-next`。
* 国内的域名都要上实名认证，还要各种持证摆拍，还是懒，所以觉得github.io托管挺好的。

## Steps

### 创建工程目录
* 安装hexo和hexo-theme-next。跟着[hexo](https://hexo.io/docs/)文档做，很容易就在本地创建懒博客目录。
* 照着[hexo-theme-next](https://github.com/theme-next/hexo-theme-next)上的配置，简单调整了一下色调。
* 增加了本文和一篇以前记录的文章。
* 上传到github。习惯了从github上下载仓库，好久没有本地上传了，也顺道记录一下。首先在git新建一个仓库，然后本地运行下面命令。
```
  git init
  git add README.md
  git commit -m "first commit"
  git remote add origin https://github.com/wsy5555621/blog.git
  git push -u origin master
```

### GitHub Pages
Github 有两种形式的 page：

* 个人或组织的page：只能存在一个，master分支，地址为 xxx.github.io
* 项目page：每个项目可以生成一个，gh-pages分支，地址为 xxx.github.io/projectname

为了简单，就直接使用第一种形式了，让master分支作为静态资源分支，develop分支保存blog源文件。
