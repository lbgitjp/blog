---
title: 建设博客站点 - 3. 用Git库分支来管理Hugo站点和生成的静态站点
description: 用同一个Git库的不同分支来管理本地Hugo站点和生成的静态站点
slug: build-blog-3
date: 2026-01-06 02:00:00+0000
categories:
    - 建设博客站点
    -
tags:
    - Github Pages
    -
# 参考：
#
---

## 1. 目标
1. 在[建设博客站点 - 1. 在本地使用Hugo创建站点](../build-blog-1 "建设博客站点 - 1. 在本地使用Hugo创建站点")中生成了Hugo本地站点`blog-src`。

2. 在[建设博客站点 - 2. 将本地Hugo站点上传到Github Pages](../build-blog-2 "建设博客站点 - 2. 将本地Hugo站点上传到Github Pages")中从Github克隆了`blog`库到本地，并将Hugo本地站点`blog-src`生成的静态站点-`public`目录拷贝到本地`blog`库中，提交到Github后将其`main`分支作为`GitHub Pages`。

3. 我们现在要将上面的两个库合并成一个Git库，将`blog-src`库作为`blog`库的`main`分支，将`blog-src`库生成的静态站点-`public`目录拷贝到`blog`库的`gh-pages`分支，将其`gh-pages`分支作为`GitHub Pages`，将两个库合并到同一个Git库的不同分支便于管理。

## 2. 创建Git分支
1. 打开命令行，进入到运行`blog`库所在目录。

2. 运行命令`git branch gh-pages`，这会创建一个新的分支`gh-pages`，该分支包含`main`分支同样内容-也就是`blog-src`库生成的静态站点-`public`目录中的内容。

3. 删除`main`分支中除了`.git`外的所有内容，并将`blog-src`库中的所有内容拷贝到`main`分支中。

4. 添加`.gitignore`文件，在其中添加如下内容：
   ```
   public
   resources
   .hugo_build.lock
   ```
   这将动态生成的内容排除在git版本管理之外，其中`public`目录的内容会被拷贝到`gh-pages`分支进行发布。

5. 在命令行中运行如下命令，将本地站点内容提交到Github上`blog`库的`main`分支中。
   ```bash
   git add .
   git commit -m "Hugo站点初始版本"
   git push
   ```
6. 运行命令`git checkout gh-pages`切换到`gh-pages`分支。

7. 在命令行中运行如下命令，将本地站点内容提交到的`gh-pages`分支。
   ```bash
   git add .
   git commit -m "发布站点初始版本"
   ```

8. 运行命令`git push --set-upstream origin gh-pages`，该命令会做三件事：
   1. 在Github服务器的`blog`库中创建`gh-pages`分支.
   2. 将本地`gh-pages`分支与Github上的`gh-pages`分支关联，关联后下次上传时只需要运行`git push`命令即可。
   3. 并将本地`gh-pages`分支内容提交到Github服务器上`blog`库的`gh-pages`分支中。

## 3. 设置Github库的GitHub Pages
1. 打开https://lbgitjp.github.io/blog/ 会出现错误，因为其指向的是`blog`库的`main`分支，该分支是Hugo原始站点，而不是编译完成后的静态站点。

2. 在Github库的[Settings](https://github.com/lbgitjp/blog/settings "Settings")导航中进入到[Pages](https://github.com/lbgitjp/blog/settings/pages "Pages")设置页，将**Build and deployment**下的**Branch**选择为`gh-pages`分支并保存设置。

3. 这时候如果打开Github库的[actions](https://github.com/lbgitjp/blog/actions "actions")页就会发现**pages build and deployment**工作流正在运行。

4. 等上面的工作流运行完成，打开https://lbgitjp.github.io/blog/ 就可以看到部署完成的站点。

5. 现在站点的编辑和发布就可以在同一个目录的同一个库完成了，在`main`分支编辑并测试本地站点，完成后发布到`gh-pages`分支中作为GitHub Pages，这两个分支都需要提交到Github的`blog`库中。
