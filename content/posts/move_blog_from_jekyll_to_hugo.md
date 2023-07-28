+++ 
date = 2023-07-28T16:49:57+08:00
title = "将个人博客从Jekyll迁移到Hugo"
description = "介绍如何将个人博客从Jekyll迁移到Hugo，以及修改github配置"
slug = "move blog from Jekyll to Hugo"
authors = ["木章永"]
tags = ["Blog", "Hugo"]
categories = ["Blog"]
+++

# 将个人博客从Jekyll迁移到Hugo
Github Page 默认使用Jekyll搭建个人博客，最近希望改用Hugo搭建个人博客。幸运的是，Github也支持使用Hugo来搭建博客，并可以通过创建`workflows`来自动发布博客

# Hugo安装
参考资料：["hugo installaction"](https://gohugo.io/installation/)


在Linux上，Hugo最简单的安装方式当然是通过包管理进行安装，以Ubuntu为例，安装Hugo只需要执行以下命令：
``` bash
apt-get install hugo
```
安装成功后，可以通过`hugo version`查看安装的版本，本人安装的，安装的版本是`v0.68.3`
```
$ hugo version
Hugo Static Site Generator v0.68.3/extended linux/amd64 BuildDate: 2020-03-25T06:15:45Z
```

apt安装的hugo版本并不是最新的，甚至可以说是陈旧的，在2023年安装的版本居然是2020年的。
新的版本和旧的版本的配置文件不同，官方文档使用的是新版本的格式的配置文件
> In Hugo 0.110.0 we changed the default config base filename to hugo, e.g. hugo.toml. We will still look for config.toml etc., but we recommend you eventually rename it (but you need to wait if you want to support older Hugo versions). The main reason we’re doing this is to make it easier for code editors and build tools to identify this as a Hugo configuration file and project.

个人建议是使用新的版本，如果要安装最新的版本，可以通过源码进行安装，也可以从Github上下载最新的安装包进行安装：https://github.com/gohugoio/hugo/releases

在Linux下，只需要下载后解压，然后把可执行文件路径添加到环境变量里面即可。
执行`hugo version`检查是否安装成功
``` bash
$ hugo version
hugo v0.115.4-dc9524521270f81d1c038ebbb200f0cfa3427cc5 linux/amd64 BuildDate=2023-07-20T06:49:57Z VendorInfo=gohugoio
```

# Hugo 目录结构

执行命令`hugo new site quickstart`可以创建一个博客网站， `quickstart`是要创建的文件夹名称，执行改命令后，`hugo`会创建一个文件夹，文件夹内目录结构如下
``` bash
├── archetypes
│   └── default.md
├── assets
├── content
│   └── posts
├── data
├── hugo.toml
├── layouts
├── resources
│   └── _gen
├── static
│   └── images
└── themes
```

下面介绍常用到的几个文件/文件夹：

content : 存放博客内容，以后编写的博客都放在此目录下，通常在此文件夹下根据博客的不同分类再创建不同的文件夹存放不同的博客，然后通过配置文件将不同的菜单目录映射不同的路径。

hugo.toml : 配置文件，用于配置网站的不同参数，采用不同的theme时，theme的配置参数也在此文件进行配置

static : 存在静态资源，如图片。 

themes : 主题，用到的主题放到此目录下，一个主题使用一个文件夹，通过在hugo.toml的配置设置想要用的主题。采用在github上开源的主题时，通过sub module的方式将仓库添加在themes目录下。可在 https://themes.gohugo.io/ 查找需要用的主题。

# 将Jekyll的博客迁移到Hugo 
由于`Jekyll`和`Hugo`都是使用Markdown编写博客，所以只需要将`Posts`文件夹的内容迁移即可，然后再根据`Hugo`的头部选择做适应的调整即可。
如果有图片，则修改图片放到`static/images`目录下，然后根据情况修改图片路径，可以通过`sed`命令进行批量的替换

修改后，建议现在本地执行`hugo server`查看具体效果，调整后将整个文件夹复制到旧的git仓库，再推送到github即可。

此时github还是会执行旧的脚本，尝试使用`Jekyll`的方式发布，需要添加workflows，具体参考：https://gohugo.io/hosting-and-deployment/hosting-on-github/

需要注意修改所给的`.yaml`文件中的`on`项，因为我用的主分支是`master`，所以将其修改为：
``` yaml
on:
  # Runs on pushes targeting the default branch
  push:
    branches:
      - master
```
若你的主分支是`main`则不用修改

之后只要推送博客到Github上就能自动发布的到Github Page上了。

# 新建文章
执行`ugo new posts posts/<post_name.md>`会在`posts`目录下创建新的文章，默认创建的文件是草稿`draft = true`，此时使用`hugo server`是不会渲染这些文章的，使用`hugo server -D`就会渲染这些文章。

可以在编写完成后再将`draft = true`去掉，就可以正式发布文章了。