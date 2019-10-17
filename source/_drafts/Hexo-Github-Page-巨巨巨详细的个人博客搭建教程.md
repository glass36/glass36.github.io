---
title: Hexo+Github Page 巨巨巨详细的个人博客搭建教程
tags:
- Hexo
- Git
categories:
---
　　这几天经过各种花式探索，终于偷偷利用上班时间搭建好了这个博客网站。那我的第一篇正式博文就来记录下个人博客的搭建流程吧。(免得以后又忘记= =)
# 必备知识点
## 什么是Hexo？
根据[Hexo官网](https://hexo.io/)的信息可以了解到Hexo是一个使用简单，快速的博客框架。
我们只需要修改部分配置文件，就可以立马生成自己想要的HTML静态资源文件。

它的特点如下：
- <font size=4>搭建迅速</font>

Hexo本身基于Node.js的静态博客框架，利用强大的Node.js,可以在数秒内建立上百个静态文件


- <font size=4>支持Markdown</font>

Markdown 本身也是一种轻量级的标记语言,只要记住几个特殊的语法就能轻松的撰写博文，加之本身可以内嵌Html，因此很受欢迎。至少我前天刚学会这个语言，今天就拿来写博客了，也不知道要踩多少坑。。。
- <font size=4>一键部署</font>

仅仅几条命令，就能直接把博客运行起来，这一点官网甚至直接甩出了简单的几条命令。
```bash
npm install hexo-cli -g
hexo init blog
cd blog
npm install
hexo server
```
- <font size=4>强大的插件支持</font>

可以支持EJS, Pug, Nunjucks等常用模版引擎。

## 如何实现利用Hexo搭建个人博客？
既然了解到了Hexo是一款静态博客框架，那么为了让别人能访问到自己的博客，我们需要把它发布在互联网上。这里直接选择托管在GitHub Page上，简单粗暴，省去了租云服务器的麻烦。

# 搭建步骤
