---
title: 建设博客站点 - 5. 使用GitHub Actions工作流自动部署站点
description: 使用GitHub Actions工作流自动部署站点，减少复杂度
slug: build-blog-5
date: 2026-01-07 06:00:00+0000
categories:
    - 建设博客站点
    -
tags:
    - GitHub Actions
    -
# 参考：
#
---

## 1. 当前部署问题
1. 当前的`blog`库有两个分支的`main`和`gh-pages`，其中`main`中存放的是`hugo`站点，是源库，`gh-pages`中存放的是`hugo`生成的静态站点，是目标库。

2. 现在每次发布文章后要经过如下几步：
   1. 进入`main`分支，`commit`到本地`git`库，`push`到`Github`服务器端的`main`分支。

   2. 运行`hugo`命令生成的静态站点到`public`目录。

   3. 进入`gh-pages`分支，将`main`分支的`public`目录内容覆盖到`gh-pages`分支。

   4. `commit`到本地`git`库，`push`到`Github`服务器端的`gh-pages`分支。

3. 可以看到，上面的步骤手工处理起来还是有些复杂，但通过`GitHub Actions`的工作流机制可以减少复杂度。

## 2. 使用GitHub Actions工作流
1. 进入到`main`分支，创建`.github\workflows\`目录。

2. 进入到`.github\workflows\`目录下，创建工作流文件`deploy.yml`，其内容如下：
   ```
   # 参考: https://github.com/peaceiris/actions-gh-pages

   name: Build and Deploy Hugo Site

   on:
     push:
       branches:
         - main  # 当push到main分支时触发

   jobs:
     deploy:
       runs-on: ubuntu-latest

       permissions:
         contents: write  # 添加此行以授予写入权限，未加此行时会出错

       steps:
         - name: Checkout repository
           uses: actions/checkout@v6
           with:
             submodules: true  # 如果您的主题使用子模块，请启用

         - name: Setup Hugo
           uses: peaceiris/actions-hugo@v3
           with:
             hugo-version: 'latest'  # 或指定版本，如'0.154.2'
             extended: true  # 如果使用Hugo扩展版

         - name: Build site
           run: hugo --minify  # 构建静态文件到public文件夹

         - name: Deploy to gh-pages
           uses: peaceiris/actions-gh-pages@v4
           with:
             github_token: ${{ secrets.GITHUB_TOKEN }}  # 使用内置令牌
             publish_dir: ./public  # 源目录
             publish_branch: gh-pages  # 目标分支
   ```

3. `commit`到本地`git`库，并`push`到`Github`服务器端的`main`分支。

4. 现在`GitHub Actions`工作流就可以自动运行了，以后每次发布文章后只需经过如下1个步：
   1. 进入`main`分支，`commit`到本地`git`库，`push`到`Github`服务器端的`main`分支。
   
5. 现在发布一篇文章后，只需将`main`分支修改提交到`Github`服务器端的`main`分支，然后等待Github库的[actions](https://github.com/lbgitjp/blog/actions "actions")工作流完成即可查看部署好的站点，与使用工作流之前相比，大幅减少了复杂度。