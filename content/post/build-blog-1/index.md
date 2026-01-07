---
title: 建设博客站点 - 1. 在本地使用Hugo创建站点
description: 几分钟内快速创建一个可以在本地运行的Hugo站点, 并生成静态站点
slug: build-blog-1 # 需要设置slug，否则会使用上面的title作为地址，不便于帖子间的相互引用
date: 2026-01-05 04:00:00+0000 # 注意date值晚于当前时间的帖子不会显示，到了时间才显示
image:
categories:
    - 建设博客站点
    -
tags:
    - Hugo
    -
weight: # You can add weight to some posts to override the default sorting (date descending)

# 参考：
# 1. https://gohugo.io/getting-started/quick-start/
---

## 1. 安装Hugo
1. 从Hugo[下载地址](https://github.com/gohugoio/hugo/releases "下载地址")下载平台相关的Hugo安装包，
本次使用的是[hugo_extended_withdeploy_0.154.2_windows-amd64.zip](https://github.com/gohugoio/hugo/releases/download/v0.154.2/hugo_extended_withdeploy_0.154.2_windows-amd64.zip "hugo_extended_withdeploy_0.154.2_windows-amd64.zip")，
为了避免后续出现问题, 推荐使用extened版本的安装包。

2. 将安装波解压到安装目录，比如c:\hugo，然后将该目录添加到系统环境变量PATH中，如果不添加环境变量，需要在每次运行hugo命令前手工设置PATH变量, `set PATH=%PATH%;c:\hugo`。

3. 打开命令行，输入hugo version，显示版本号即为安装成功
`hugo v0.154.2-f66d0944461bf32c4e69588bc3e093f14e4e149d+extended+withdeploy windows/amd64 BuildDate=2026-01-02T16:08:44Z VendorInfo=gohugoio`

## 2. 创建站点
1. 打开命令行，运行`hugo new site blog`，生成了新的骨架站点`blog`，这个目录目录内有很多空目录和一个hugo配置文件`hugo.toml`，除了`content`目录，`themes`目录和hugo配置文件`hugo.toml`外，其它目录都没有用，删除后也没发行什么问题。

2. 运行`cd blog`进入该站点目录，然后运行`hugo server`，就启动了站点`http://localhost:1313/`，在浏览器中打开该站点会显示`Page Not Found`，因为目前还只是一个空站点，没有任何内容。

3. 查找主题，从hugo[主题站点](https://themes.gohugo.io/ "主题站点")查找合适的主题，本次使用的是`ananke`主题https://themes.gohugo.io/themes/gohugo-theme-ananke/。

4. 下载主题，运行`git init`，在该站点内初始化一个空的git库，然后运行`git submodule add https://github.com/theNewDynamic/gohugo-theme-ananke.git themes/ananke`，将`ananke`主题库下载到`themes`目录下.
**注意**: 该步不是必须使用`git`的，可以直接将`ananke`主题下载后解压到`themes`目录下也是可以的.

5. 修改配置文件`hugo.toml`，在其中添加一行`theme = 'ananke'`。

6. 运行`hugo server`，启动站点`http://localhost:1313/`，就会发现站点已经正常运行

7. 检查`blog`目录, 发现其中新增加了一个`public`子目录，该目录是hugo在运行时自动生成的静态站点。

8. 运行`hugo new content content/posts/hello-world.md`命令添加一篇新帖子, 生成的`hello-world.md`是一个Markdown格式的文件, 用户可以使用支持该格式的编辑器来进行编辑，其格式如下：
   ```markdown
   ---
   title: "Hello World"
   date: 2026-01-05T14:20:53+08:00
   tags: []
   featured_image: ""
   description: ""
   ---
   ```


9. 运行`hugo server`，启动站点`http://localhost:1313/`，就会发现新发的帖子已经正常显示，现在就基本上可以将`public`子目录中的内容发布到托管站点了。

10. 在发布前还需要对hugo配置文件`hugo.toml`进行配置, 将其中的`baseURL`设置为服务器站点的网址：
`baseURL = "https://yoursite.com/"`

11. 现在运行`hugo`命令就会使用配置文件`hugo.toml`中的配置来发布最新的内容到`public`子目录中，将该目录中的内容发布到支持静态站点的托管服务器上就可以了。