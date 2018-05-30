---
title: 基于hexo+github搭建个人博客 
date: 2018-05-23 15:40:32
tags: [hexo]
categories: hexo
---

终于完成hexo+github博客搭建。作为第一篇博客，本来记录了整个搭建过程。

<!-- more -->

# 安装依赖

## 安装git
```shell
$ brew install git
```

## 安装node.js
```shell
$ wget -qO- https://raw.github.com/creationix/nvm/master/install.sh | sh
```
退出并重新登录

```shell
$ nvm install stable
```

# 安装hexo
```
$ npm install -g hexo-cli
$ npm install hexo --save

```

# 初始化
```shell
$ mkdir hexo-root
$ cd hexo-root
$ hexo init .
```
这里创建了一个空目录hex-root，并进行hexo初始化。**后续的操作都在hexo-root目录下执行**。

注意：最后一个命令(hexo init .)要求当前目录(hexo-root)是一个空目录。否则，会失败。


成功之后，可以看见当前目录下有这些内容：
```shell
$ ls
_config.yml  node_modules  package-lock.json  package.json  scaffolds  source  themes
```

# 本地测试

在本地起hexo server
```shell
$ hexo clean
$ hexo generate
$ hexo server
```
然后，在浏览器上可以测试初始化是否成功：
http://localhost:4000/

这时候应该能看见初始化自带的一篇博客：hello-world

测试完成后，ctrl+C停止hexo server。

# 设置
## 设置主题为next
初始化后，默认自带了一个landscape主题：
```shell
$ ls themes/
landscape
```

下载next主题：
```shell
$ git clone https://github.com/theme-next/hexo-theme-next themes/next

$ ls themes/
landscape  next
```

把主题设置为next：
```shell
$ vim _config.yml
theme: next
```

把next主题的scheme设置为Mist(纯属个人喜好)
```shell
$ vim themes/next/_config.yml  
scheme: Mist
```

把代码高亮设置为night eighties(纯属个人喜好)
```shell
$ vim themes/next/_config.yml
highlight_theme: night eighties
```

重新本地测试，发现已经变为简约的next主题了。

## 设置new_post_name
当我们使用命令：
```
hexo new {title}
```
创建一篇新博客的时候，hexo会为我们生成一个{title}.md文件。为了日后方便看博客的创建时间，我们希望.md文件名中带上当前日期。为此，修改_config.yml:
```shell
$ vim _config.yml
new_post_name: :title-:year-:month-:day.md # File name of new posts

$ hexo new testpage
INFO  Created: ~/hexo-root/source/_posts/testpage-2018-05-12.md

$ ls source/_posts/
hello-world.md  testpage-2018-05-12.md

```

## 菜单栏
默认情况下，菜单栏中只有Home和Archives两项。把categories, tags和about都打开，这样，菜单栏就会显示相应的菜单项了。

```shell
$ vim themes/next/_config.yml  
menu:
   home: / || home
   about: /about/ || user
   tags: /tags/ || tags
   categories: /categories/ || th
   archives: /archives/ || archive  
```
现在执行本地测试，点击主页上方的Categories,Tags和About按钮时，会得到错误：

* Cannot GET /categories/
* Cannot GET /tags/
* Cannot GET /about/

这是因为source/下没有categories, tags和about信息。创建它们：

```shell
$ hexo new page about
INFO  Created: ~/hexo-root/source/about/index.md

$ vim source/about/index.md
自我介绍

$ hexo new page categories
INFO  Created: ~/hexo-root/source/categories/index.md

$ hexo new page tags
INFO  Created: ~/hexo-root/source/tags/index.md
```

可见，在source/下创建了三个目录：categories, tags和about，并且各创建了一个index.md文件。

重新执行本地测试，发现点击Categories, Tags和About时不再报错了。现在我们尝试在testpage-2018-05-12.md中加入category和tag，看看能不能成功建立categories和tags索引：
```shell
$ vim source/_posts/testpage-2018-05-12.md
title: testpage
date: 2018-05-12 23:19:34
tags: [filesystem,ext4]
categories: linux
```
重新执行本地测试，点击Categories和Tags，发现里面都是空的。索引建立失败。解决：
* 在source/categories/index.md中加入type: "categories"

```shell
$ vim source/categories/index.md
title: categories
date: 2018-05-12 23:29:23
type: "categories"
```

* 在source/tags/index.md中加入type: "tags"

```shell
$ vim source/tags/index.md
title: tags
date: 2018-05-12 23:29:35
type: "tags"
```

重新执行本地测试，点击Categories和Tags，发现索引建立已经成功。

## 配置中文
```shell
$ vim themes/next/languages/zh-Hans.yml
menu:
  home: 主页
  categories: 分类
  archives: 归档
  tags: 标签
  message: 留言
  about: 关于
  search: 搜索
sidebar:
  overview: 本站概览
  toc: 文章目录
post:
  posted: 发表于
  edited: 更新于
  in: 所属分类
  views: 阅读次数
  comments_count: 评论数
  read_more: 阅读全文
  sticky: 置顶
  toc_empty: 此文章未包含目录
reward:
  wechatpay: 微信支付
  alipay: 支付宝
state:
  posts: 日志
  pages: 页面
  tags: 标签
  categories: 分类
counter:
  tag_cloud:
    zero: 暂无标签
    one: 目前共计 1 个标签
    other: "目前共计 %d 个标签"
  categories:
    zero: 暂无分类
    one: 目前共计 1 个分类
    other: "目前共计 %d 个分类"
  archive_posts:
    zero: 暂无日志。
    one: 目前共计 1 篇日志。
    other: "目前共计 %d 篇日志。"

$ vim _config.yml
language: zh-Hans
```

## 基本信息

```shell
$ vim _config.yml
title: <Your-name>的博客
description: 简短自我/博客描述
author: <Your-name>
```

## 头像设置

把头像（例如me.jpg）放到themes/next/source/images/目录下，然后修改next的配置文件：

```shell
$ vim themes/next/_config.yml
avatar:
  url: /images/me.jpg
```

## 打赏

把微信和支付宝的收款二维码图片例如wechatpay.jpg和alipay.jpg）放到themes/next/source/images/目录下，然后修改next的配置文件：

```shell
$ vim themes/next/_config.yml
reward_comment: 写的不错，有赏！
wechatpay: /images/wechatpay.jpg
alipay: /images/alipay.jpg
```

## 社交链接
```shell
$ vim themes/next/_config.yml
social:
  GitHub: https://github.com/<your-name> || github
  E-Mail: mailto:<your-email-address> || envelope
```

## 站内搜索功能
```shell
$ npm install hexo-generator-search --save
$ npm install hexo-generator-searchdb --save

$ vim _config.xml
search:
  path: search.xml
  field: post
  format: html
  limit: 10000

$ vim themes/next/_config.yml
local_search:
  enable: true
  trigger: auto
  top_n_per_article: 1
  unescape: false
```


## 生成摘要

当点击“Home”的时候，会显示博文列表。通过配置，可以同时显示一个摘要。有两种方式生成摘要：

### 自动生成
这种方式比较简单粗暴，就是认为你博文的“前N个字”就是摘要，所有博文都一样。例如，前300字自动作为摘要：
```shell
$ vim themes/next/_config.yml
auto_excerpt:
  enable: true
  length: 300
```

### 自定义
自动生成摘要的方式虽然简单，但不精确。博主可以为每篇博文精确定义摘要，方式是：

```shell
vim blog-title-{year}-{month}-{day}.md
摘要部分
<!-- more -->
正文部分
```
也就是通过<!-- more -->将摘要和正文分开。

**注意：显然，推荐使用自定义摘要。但是，可以把自动生成摘要打开，这样，若博客有自定义摘要，则使用自定义摘要；若没有，则自动生成。**

## leancloud访问量统计和评论功能

### Leancloud准备

* 登录https://leancloud.cn/并注册
* 创建应用: 访问控制台 --> 创建应用 --> 填入新应用名称(自取) --> 创建
* 创建Counter: 选择刚创建的应用，存储 --> 创建Class --> 填入Class名称(必须是"Counter") --> 创建Class

注意：Comment已经存在，所以我们没有创建Comment；


### 获取AppID和AppKey

选择刚创建的应用，设置 --> 应用Key，记下里面的"AppID"和"AppKey"。

### 配置next主题

```shell
$ vim themes/next/_config.yml
leancloud_visitors:
  enable: true
  app_id: 上一步获取的AppID
  app_key: 上一步获取的AppKey
  security: false

valine:
  enable: true
  appid: 上一步获取的AppID
  appkey: 上一步获取的AppKey
```


# 创建github repository

创建git repository:  https://github.com/<your-name>/<your-name>.github.io

注意：
* repository名一定要为github的名字。
* 把"Initialize this repository with a README"勾选上。

# 拷贝public key到github

若还没有生成key pair，请先生成。然后：
```shell
$ cat ~/.ssh/id_rsa.pub
```
并拷贝其输出内容。
到github上，点击：右上的头像 -> Settings -> SSH and GPG keys -> New SSH key; 填写Title(名称自取)和Key (粘贴前面拷贝的id_rsa.pub的内容)。


# 配置deploy

```shell
$ vim _config.yml
deploy:
  type: git
  repo: git@github.com:<your-name>/<your-name>.github.io.git
  branch: master
```

# deploy

第一次deploy，需要先安装hexo-deployer-git：
```shell
$ npm install hexo-deployer-git --save
```
只第一次deploy之前执行！

下面就是deploy了：

```shell
$ hexo clean
$ hexo generate
$ hexo deploy
```

* 第一条命令：删除本地生成的静态页面；
* 第二条命令：根据source/\_posts下面的.md文件，以及主题等，重新生成静态页面；
* 第三条命令：把第二步生成的静态页面上传到github；

现在，博客就生成了：http://{your-name}.github.io/

# 创建markdown repository
到此为止，博客就搭建起来了。以后写博客，就是创建.md文件。为了保存曾经的博客(.md文件)，我另外创建一个github repository。通过这个repository，在公司和家里的不同电脑上，都可以写博客了。最后，在配置了hexo的电脑上发布。


# 常见错误
博文.md文件中除了开头部分，若包含"---"，则很容易出一些奇怪的YAMLException，并且报错的位置可能相去甚远，很难定位。例如：
* YAMLException: end of the stream or a document separator is expected at line x, column y
* YAMLException: unidentified alias at line x, column y
* YAMLException: name of an alias node must contain at least one character at line x, column y

# 小结
之前使用CSDN，页面不够漂亮，也不能做一些个性化的设置，所以计划搭一个自己的博客，今天终于得以实现。第一次搭建博客，并且对前端网站不熟悉，踩了不少坑，所以本文记下这个过程，作为开篇。
