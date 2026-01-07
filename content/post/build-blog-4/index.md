---
title: 建设博客站点 - 4. 设置域名
description: 在cloudflare.com设置CNAME域名来访问博客站点
slug: build-blog-4
date: 2026-01-06 07:00:00+0000
categories:
    - 建设博客站点
    -
tags:
    - 域名
    -
# 参考：
#
---

## 1. 域名问题
1. 每个Github用户都有一个独有的二级域名和其账号相关联，如用户为`lbgitjp`的二级域名就是[https://lbgitjp.github.io/](https://lbgitjp.github.io/ "https://lbgitjp.github.io/")，其中主域名为[https://github.io/](https://github.io/ "https://github.io/")，属于Github官方。

2. 目前所建的站点是通过[https://lbgitjp.github.io/](https://lbgitjp.github.io/ "https://lbgitjp.github.io/")下一级子目录[https://lbgitjp.github.io/blog/](https://lbgitjp.github.io/blog/ "https://lbgitjp.github.io/blog/")来托管和访问的。

3. 每个Github用户可以通过在创建一个和其独有的独有的二级域名相同名称的库-如`lbgitjp.github.io`来托管静态站点，这样就可以通过二级域名来直接访问该站点，而不需要通过下一级子目录`blog`来访问站点，但每个Github账号只能创建一个这样的库，无法做到托管多个站点。

4. 而且域名`lbgitjp.github.io`是`github.io`的子域名，对自己的品牌不太友好，因此最好能改成用户自己的域名。

## 2. 设置域名
1. 用户可以在[https://www.cloudflare.com/](https://www.cloudflare.com/ "https://www.cloudflare.com/")站点或其它域名管理站点注册一个属于自己的域名。

2. 在域名的DNS管理中添加一个`CNAME`记录，将该记录指向用户在Github的二级域名[https://lbgitjp.github.io/](https://lbgitjp.github.io/ "https://lbgitjp.github.io/")。例如，添加一条`test.xxx.com`的域名，其指向`lbgitjp.github.io`。

3. 如果添加的是主域名`CNAME`记录，可能需要等待一段时间才能全网生效，如果是添加的子域名`CNAME`记录，应该会即使生效。

## 3. 发布站点
1. 进入到本地`blog`库的`main`分支。

2. 修改其中的`hugo.toml`文件，将`baseurl`项改为
`baseurl = "https://test.xxx.com/"`，其中`test.xxx.com`就是新添加的`CNAME`记录。

3. 在命令行中运行如下命令，将本地`main`分支中的内容提交到Github服务器`blog`库的`main`分支中。
   ```bash
   git add .
   git commit -m "修改baseurl"
   git push
   ```

4. 在命令行运行`hugo`命令编译站点，拷贝`public`中的内容。

5. 运行命令`git checkout gh-pages`切换到`gh-pages`分支。

6. 将拷贝的`public`中的内容覆盖到`gh-pages`分支。

7. 在`gh-pages`分支添加一个文件名为`CNAME`的文件，其中只有一行内容`test.xxx.com`，就是前面新添加的域名。

7. 在命令行中运行如下命令，将本地`gh-pages`分支中的内容提交到Github上`blog`库的`gh-pages`分支中。
   ```bash
   git add .
   git commit -m "添加CNAME"
   git push
   ```

8. 等Github服务端的actions工作流运行完成，就可以通过新域名`https://test.xxx.com/`来访问部署完成的站点。

## 3. 注意事项
1. 在`CNAME`中设置域名`test.xxx.com`会导致通过原来的`https://lbgitjp.github.io/blog/`子目录访问会自动重定向到域名地址`https://test.xxx.com/`，因此如果域名`test.xxx.com`失效了，通过`https://lbgitjp.github.io/blog/`或`https://test.xxx.com/`都无法访问该站点，此时需要删除`CNAME`文件才能恢复原来的`https://lbgitjp.github.io/blog/`子目录访问。


2. 如果没有自己的域名，想通过在`blog`库的`CNAME`中设置每个Github用户独有的二级域名`lbgitjp.github.io`是不能达到通过二级域名`https://lbgitjp.github.io/`来访问`blog`站点的，仍然需要通过`https://lbgitjp.github.io/blog/`来访问，通过`https://lbgitjp.github.io/blog/`访问时也不会自动重定向到`https://lbgitjp.github.io/`，此时的`CNAME`设置和没有设置效果一样。