#1. 安装依赖

* 安装git
```shell
$ brew install git
```

* 安装node.js
```shell
$ wget -qO- https://raw.github.com/creationix/nvm/master/install.sh | sh
```
退出并重新登录
```shell
$ nvm install stable
```

#2. 安装hexo
```shell
$ npm install -g hexo-cli
$ npm install hexo --save

```

#3. 初始化
```shell
$ mkdir hexo-root
$ cd hexo-root
$ hexo init .
```
这里创建了一个空目录hex-root，并进行hexo初始化。后续的操作都在hexo-root目录下执行。

注意：最后一个命令(hexo init .)要求当前目录(hexo-root)是一个空目录。否则，会失败。


成功之后，可以看见当前目录下有这些内容：
```shell
$ ls
_config.yml  node_modules  package-lock.json  package.json  scaffolds  source  themes
```

#4. 本地测试

在本地起hexo server
```shell
$ hexo server
```
然后，在浏览器上可以测试初始化是否成功：
http://localhost:4000/

这时候应该能看见初始化自带的一篇博客：hello-world

测试完成后，ctrl+C停止hexo server。

#5. 设置
##5.1 设置主题为next
初始化后，默认自带了一个landscape主题：
```shell
$ ls themes/
landscape
```

下载next主题：
```shell
$ git clone https://github.com/idhyt/hexo-theme-next.git themes/next
$
$ ls themes/
landscape  next
```

把主题设置为next：
```shell
$ vim _config.yml
......
#theme: landscape
theme: next
```

重新本地测试（见第4节），发现已经变为简约的next主题了。

##5.2 设置new_post_name
当我们使用命令：
```
hexo new {title}
```
创建一篇新博客的时候，hexo会为我们生成一个{title}.md文件。为了日后方便看博客的创建时间，我们希望.md文件名中带上当前日期。为此，修改_config.yml:
```shell
$ vim _config.yml
......
#new_post_name: :title.md # File name of new posts
new_post_name: :title-:year-:month-:day.md # File name of new posts
$
$ hexo new testpage
INFO  Created: ~/hexo-root/source/_posts/testpage-2018-05-12.md
$
$ ls source/_posts/
hello-world.md  testpage-2018-05-12.md

```

##5.3 设置categories和tags

现在执行本地测试（见第4节），点击主页上方的Categories和Tags，会得到错误：

* Cannot GET /categories/
* Cannot GET /tags/

这是因为source/下没有categories和tags信息。创建它们：
```shell
$ hexo new page categories
INFO  Created: ~/hexo-root/source/categories/index.md
$
$ hexo new page tags
INFO  Created: ~/hexo-root/source/tags/index.md
```
可见，在source/下创建了两个目录：categories和tags，并且各创建了一个index.md文件。

重新执行本地测试，点击Categories和Tags时，已经不报错了。我们尝试在testpage-2018-05-12.md中加入category和tag，看看能不能成功建立索引：
```shell
$ vim source/_posts/testpage-2018-05-12.md
---
title: testpage
date: 2018-05-12 23:19:34
tags: [filesystem,ext4]
categories: linux
---
```
重新执行本地测试，点击Categories和Tags，发现里面都是空的。索引建立失败。解决很简单：
* 在source/categories/index.md中加入type: "categories

```shell
$ vim source/categories/index.md
---
title: categories
date: 2018-05-12 23:29:23
type: "categories"
---
```

* 在source/tags/index.md中加入type: "tags"

```shell
$ vim source/tags/index.md
---
title: tags
date: 2018-05-12 23:29:35
type: "tags"
---
```


重新执行本地测试，点击Categories和Tags，发现索引建立已经成功。


#6. 创建github repository

创建git repository:  https://github.com/<your-name>/<your-name>.github.io

注意：repository名一定要为github的名字。


#7. 拷贝public key到github

若还没有生成key pair，请先生成。然后：
```shell
$ cat ~/.ssh/id_rsa.pub
```
并拷贝其输出内容。
到github上，点击：右上的头像 -> Settings -> SSH and GPG keys -> New SSH key; 填写Title(名称自取)和Key (粘贴前面拷贝的id_rsa.pub的内容)。


#8. 配置deploy

```shell
$ vim _config.yml
......
deploy:
  type: git
  repo: git@github.com:<your-name>/<your-name>.github.io.git
  branch: master
```

#9. deploy

第一次deploy，需要先安装hexo-deployer-git：
```shell
$ npm install hexo-deployer-git --save
```
只第一次deploy之前执行！

下面就是deploy了：

```shell
$ hexo clean
$ hexo generate
$ hexo deploy
```

* 第一条命令：删除本地生成的静态页面；
* 第二条命令：根据source/\_posts下面的.md文件，以及主题等，重新生成静态页面；
* 第三条命令：把第二步生成的静态页面上传到github；

现在，博客就生成了：http://{your-name}.github.io/


#10. 创建markdown repository
到此为止，博客就搭建起来了。以后写博客，就是创建.md文件。为了保存曾经的博客(.md文件)，我另外创建一个github repository。通过这个repository，在公司和家里的不同电脑上，都可以写博客了。最后，在配置了hexo的电脑发布。
