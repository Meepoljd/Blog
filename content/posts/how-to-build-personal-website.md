+++
title = '使用Hugo快速搭建一个免费的静态博客'
date = 2024-01-23T20:52:53+08:00
+++

> 为了方便迁移，不被各种笔记软件绑架，所以最近将笔记内容都迁移到一个Markdown仓库进行管理。顺便的使用Hugo将这些笔记搭建一个个人博客。
> 
> 本文用于记录搭建过程中的一些操作，面向有开发基础的同学，不会再讲解什么是Git。

# 环境搭建

1. Git

2. 一个顺手的编辑器（为什么不试试Emacs呢）
   
   ## hugo
   
   安装Hugo，Mac用户可以直接
   
   ```
   brew install hugo
   ```
   
   为了方便使用，在创建站点时加入```--format yaml```，来指定配置文件的格式
   
   ```
   hugo new site MyWebsite --format yaml
   ```
   
   Hugo 会在目录查找一个 config.toml 的配置文件。如果这个文件不存在，将会查找 config.yaml，然后是 config.json 。
   
   ## 主题配置
   
   审美问题，实在不知道选什么，所以选择了一个人气较高的主题PaperMod。参考[官方文档](https://github.com/adityatelange/hugo-PaperMod/wiki/Installation)来进行安装。

Hugo的主题被存放在 MyWebsite/themes 目录下，可使用git clone 将代码拉下来

```
git clone https://github.com/adityatelange/hugo-PaperMod themes/PaperMod --depth=1
```

在 config.yml 加入如下的内容来指定使用的主题

```
theme: ["PaperMod"]
```

## 文章模板

hugo的模板文件位于archetypes文件夹中，其中有一个default.md文件，修改内容如下

```yml
---
title: "{{ replace .Name "-" " " | title }}"
date: {{ .Date }}
lastmod: {{ .Date }}
author: ["jiandong.liu93"]

categories:
- category 1
- category 2

tags:
- tag 1
- tag 2

keywords:
- word 1
- word 2

description: "" # 文章描述，与搜索优化相关
summary: "" # 文章简单描述，会展示在主页
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
draft: true # 是否为草稿
comments: false
showToc: true # 显示目录
TocOpen: true # 自动展开目录
autonumbering: true # 目录自动编号
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
searchHidden: true # 该页面可以被搜索到
showbreadcrumbs: true #顶部显示当前路径
mermaid: true
cover:
    image: ""
    caption: ""
    alt: ""
    relative: false
---
```

## 自定义

为网站进行一些个性化的配置，比如上几个要饭的二维码。可以使用主题官方提供的[配置文件模板](https://github.com/adityatelange/hugo-PaperMod/wiki/Installation#sample-configyml)。

```yaml
baseURL: "https://meepoljd.github.io/"
title: 老东叔写代码
paginate: 5
theme: PaperMod

enableRobotsTXT: true
buildDrafts: false
buildFuture: false
buildExpired: false

googleAnalytics: UA-123-45

minify:
  disableXML: true
  minifyOutput: true

params:
  env: production # to enable google analytics, opengraph, twitter-cards and schema.
  title: 老东叔写代码
  description: "埋藏无处安放的思绪"
  keywords: [Blog, Golang, Develop]
  author: jiandong.liu93
  # author: ["Me", "You"] # multiple authors
  images: ["<link or path of image for opengraph, twitter-cards>"]
  DateFormat: "January 2, 2006"
  defaultTheme: auto # dark, light
  disableThemeToggle: false

  ShowReadingTime: true
  ShowShareButtons: false
  ShowPostNavLinks: true
  ShowBreadCrumbs: true
  ShowCodeCopyButtons: true
  ShowWordCount: true
  ShowRssButtonInSectionTermList: true
  ShowFullTextinRSS: true
  UseHugoToc: true
  disableSpecial1stPost: false
  disableScrollToTop: false
  comments: false
  hidemeta: false
  hideSummary: false
  showtoc: true
  tocopen: false

  assets:
    # disableHLJS: true # to disable highlight.js
    # disableFingerprinting: true
    favicon: "<link / abs url>"
    favicon16x16: "<link / abs url>"
    favicon32x32: "<link / abs url>"
    apple_touch_icon: "<link / abs url>"
    safari_pinned_tab: "<link / abs url>"

  label:
    text: "Home"
    icon: /apple-touch-icon.png
    iconHeight: 35

  # home-info mode
  homeInfoParams:
    Title: "Hi there \U0001F44B"
    Content: 玄都观里花千树，尽是刘郎去后栽

  socialIcons:
    - name: github
      url: "https://github.com/Meepoljd"

  analytics:
    google:
      SiteVerificationTag: "XYZabc"
    bing:
      SiteVerificationTag: "XYZabc"
    yandex:
      SiteVerificationTag: "XYZabc"

  cover:
    hidden: true # hide everywhere but not in structured data
    hiddenInList: true # hide on list pages and home
    hiddenInSingle: true # hide on single page

  # for search
  # https://fusejs.io/api/options.html
  fuseOpts:
    isCaseSensitive: false
    shouldSort: true
    location: 0
    distance: 1000
    threshold: 0.4
    minMatchCharLength: 0
    limit: 10 # refer: https://www.fusejs.io/api/methods.html#search
    keys: ["title", "permalink", "summary", "content"]
menu:
  main:
    - identifier: search
      name: Search
      url: /search/
      weight: 10
    - identifier: archives
      name: Archive
      url: /archives/
      weight: 10
    - identifier: tags
      name: Tags
      url: /tags/
      weight: 20

# Read: https://github.com/adityatelange/hugo-PaperMod/wiki/FAQs#using-hugos-syntax-highlighter-chroma
pygmentsUseClasses: true
markup:
  highlight:
    noClasses: false
    # anchorLineNos: true
    # codeFences: true
    # guessSyntax: true
    # lineNos: true
    # style: monokai
```

## 搜索功能

根据[官网文档](https://github.com/adityatelange/hugo-PaperMod/wiki/Features#search-page)，需要插入一个页面进行配置。

向config.yml中添加如下内容

```yml
outputs:
  home:
    - HTML
    - RSS
    - JSON # necessary for search
```

之后在content中新建search.md文件，并插入如下内容

```yml
---
title: "Search" # in any language you want
layout: "search" # necessary for search
# url: "/archive"
# description: "Description for Search"
summary: "search"
placeholder: "placeholder text in search input box"
---
```

## 归档页

根据[官方文档](https://github.com/adityatelange/hugo-PaperMod/wiki/Features#archives-layout)，向content目录下插入archives.md文件。插入内容

```yml
---
title: "Archive"
layout: "archives"
url: "/archives/"
summary: archives
---
```

# 更新部署

## GihubPages

在跟老婆申请换了电脑之后，就没有预算购买服务器了。因此选择将网站部署在免费的[Github pages](https://docs.github.com/zh/pages/getting-started-with-github-pages)。

> GitHub Pages 是一项静态站点托管服务，它直接从 GitHub 上的仓库获取 HTML、CSS 和 JavaScript 文件，（可选）通过构建过程运行文件，然后发布网站。 可以在 GitHub Pages 示例集合中看到 GitHub Pages 站点的示例。

## Github Action

为了实现推送代码后，自动部署更新网站。参考Hugo官网推荐的方案，配置Github Action。

在网站根目录内创建```.github/workflows/hugo.yaml```文件，复制如下内容。

```yml
# Sample workflow for building and deploying a Hugo site to GitHub Pages
name: Deploy Hugo site to Pages

on:
  # Runs on pushes targeting the default branch
  push:
    branches:
      - main

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# Sets permissions of the GITHUB_TOKEN to allow deployment to GitHub Pages
permissions:
  contents: read
  pages: write
  id-token: write

# Allow only one concurrent deployment, skipping runs queued between the run in-progress and latest queued.
# However, do NOT cancel in-progress runs as we want to allow these production deployments to complete.
concurrency:
  group: "pages"
  cancel-in-progress: false

# Default to bash
defaults:
  run:
    shell: bash

jobs:
  # Build job
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.122.0
    steps:
      - name: Install Hugo CLI
        run: |
          wget -O ${{ runner.temp }}/hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb \
          && sudo dpkg -i ${{ runner.temp }}/hugo.deb          
      - name: Install Dart Sass
        run: sudo snap install dart-sass
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0
      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v4
      - name: Install Node.js dependencies
        run: "[[ -f package-lock.json || -f npm-shrinkwrap.json ]] && npm ci || true"
      - name: Build with Hugo
        env:
          # For maximum backward compatibility with Hugo modules
          HUGO_ENVIRONMENT: production
          HUGO_ENV: production
        run: |
          hugo \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/"          
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v2
        with:
          path: ./public

  # Deployment job
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v3
```

# Finally

一个简易的个人博客就搭建完成了