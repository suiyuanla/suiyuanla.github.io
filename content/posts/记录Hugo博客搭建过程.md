---
title: "记录Hugo博客搭建过程"
date: "2024-08-09T10:23:47+08:00"
draft: false
categories:
  - fun
tags:
  - blog
series:
  - "没什么用的记录"
---

## 前言

这是我个人搭建博客的一个记录，主要作用是记录自己的搭建过程留作备份，或许有一定的参考价值，但请根据使用组建的官方连接获取最新消息，以防文章过期。

## 一、 整体架构

1. 使用[Markdown](https://zh.wikipedia.org/wiki/Markdown)编写博客文章。
2. 使用[Hugp](https://gohugo.io/) 将Markdown 文章转换为HTML 网页代码。
3. 使用[PaperMod](https://github.com/adityatelange/hugo-PaperMod) 作为**Hugo** 的网页主题，提供美观的界面。
4. 使用[Disqus](https://gohugo.io/content-management/comments/) 基于**Github Discuss** 提供评论功能。
5. 利用[Github Pages](https://pages.github.com/) 部署和访问博客(Github提供的 github.io 域名)。
6. 利用[Github Actions](https://github.com/features/actions) 达成自动推送Markdown 到仓库，便会更新网站。

## 二、效果

如前所述，配置好后就可以专注于使用Markdown写文章，当写好文章使用`git push` 推送到Github仓库后，Github Actions 就会自动更新博客。

而且由于使用Github Pages 访问博客（如我的博客是: [suiyuanla.github.io](https://suiyuanla.github.io)），并没有实际使用服务器与购买域名，可以实现零成本部署博客。

当然，由于github.io 域名并不稳定，实际上可能需要自行购买域名。当然，如果你是学生的话，Github Education 教育优惠也提供免费域名，请自行挖掘。

## 三、搭建过程

### 安装Hugo

[Hugo](hugohttps://gohugo.io/)：世界上最快的网站构建框架。（官网自述）：

```bash
# 我使用arch linux 以及paru aur helper
paru -S hugo  # 安装hugo
```

本机部署Hugo 的一个原因是，初始化项目以及新建文章和本地编辑预览需要使用。

### 创建项目

此处应首先选择一个[Hugo 主题](https://themes.gohugo.io/)，查看主题是否需要特殊的创建选项。

我选用的是[PaperMod](https://github.com/adityatelange/hugo-PaperMod)，去年第一次搭建博客的时候它并不是Hugo 官网主题的第一个，但现在已经放到HUgo官网主题的第一个了。（_笑，并不是我随大流，但是确实好用_）

Hugo的 config文件支持`yaml`、`toml`等格式，而且默认使用`toml`格式。但是PaperMod主题提供的是`yaml`，因此创建网站时要指定使用`yaml`格式：

```bash
# 参考：https://github.com/adityatelange/hugo-PaperMod/wiki/Installation
# 我的网站使用suiyuan-blog，可自行替换
hugo new site suiyuan-blog --format yaml

# 使用tree查看创建好的目录结构（非必做）
tree suiyuan-blog

suiyuan-blog
├── archetypes  # 存放模板
├── assets
├── content     # 文件下的posts文件夹存放博客内容
├── data
├── hugo.yaml   # Hugo的配置文件
├── i18n
├── layouts     # 默认没有，添加评论功能时需要使用
├── public      # 默认没有，是我自行设置用于存放生成html文件的文件夹，不上传到Github仓库
├── static      # 用于存放静态资源，比如网站图标
└── themes      # 用于存放主题

# hugo.yaml 配置
baseURL: https://suiyuanla.github.io/   # 我的博客网址
languageCode: zh-CN                     # 网站语言
title: Suiyuan's Blog                   # 网站标题
publishDir: "public"                    # 使用hugo server 命令时生成html的位置，在相对路径public下
```

### 安装主题

使用[PaperMod](https://github.com/adityatelange/hugo-PaperMod/wiki/Installation)推荐的方法：

```bash
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
git submodule update --init --recursive # needed when you reclone your repo (submodules may not get cloned automatically)

# 修改hugo.yaml，添加：
theme: PaperMod
```

### 配置主题

由于我使用的是PaperMod，因此很多为[PaperMod的自定义选项](https://github.com/adityatelange/hugo-PaperMod/wiki/Features)，还有部分[Hugo Template配置](https://gohugo.io/templates/)，具体配置参见：[我的配置](https://github.com/suiyuanla/suiyuanla.github.io/blob/main/hugo.yaml)。

### 创建和预览文章

创建文章默认使用的是`archetypes/default.md` 模板，由于默认生成的是`toml`格式的头，需要修改为：

```yaml
# archetypes/default.md
# 注意的是draft为true，说明生成的文章默认是草稿，不会生成html，如果需要生成需要在server上指定--buildDrafts选项
# 如果文章正式写好了要发布，则将draft: true 改为flase
---
title: '{{ replace .File.ContentBaseName "-" " " | title }}'
date: "{{ .Date }}"
draft: true
---
```

可以使用下面命令创建和预览文章：

```bash
# 会在content/posts/ 创建test.md，可以用vscode打开suiyuan-blog这个文件夹作为项目，编写Markdown
hugo new content content/posts/test.md
hugo server --buildDrafts   # --buildDrafts 会build我们新建的草稿，在浏览器访问localhost:1313可以看到搭建的网站
```

### 评论功能

这部分由于是一年前部署的，已经没有什么印象了（开摆），请参考[PaperMod Features Comments](https://github.com/adityatelange/hugo-PaperMod/wiki/Features#comments)，以及[Hugo Comments](https://gohugo.io/content-management/comments/)。

### 部署过程

此部分参考[Github Pages](https://pages.github.com/)、[Github Actions](https://github.com/features/actions)。

1. 如果想通过`username.github.io`直接访问博客，需要建立一个名称为`username.github.io`的仓库。
2. 然后在**Actions** 里搜索名称为**hugo** 的Github Action 并部署，全默认配置不需要修改。
3. 在仓库的**settings->Pages** 开启Pages功能，**Build and deployment -> Source** 选择**Github Actions**
4. 将创建的网站项目推送到创建的仓库，待Action执行完成就可以访问**username.github.io** 看到自己搭建的博客。
