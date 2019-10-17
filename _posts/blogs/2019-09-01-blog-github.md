---
layout: post_no_cmt
title:  "使用 github pages & jekyll 搭建个人博客"
date:   2019-09-01 00:00:00 +0800
categories: blogs
---

# 技术
- [Git](https://www.git-scm.com)
- [Github pages]
- [Jekyll]：静态网站生成器
- [Jekyll 中文](https://jekyllcn.com)
- [gitalk](https://github.com/gitalk/gitalk)： 基于 github issus 和 preact 的评论系统


# 搭建流程
1. 注册 github，创建一个新的仓库，名称：**账号名称.github.io**（参考：[Github pages]）
2. [下载安装 Jekyll](#download_install_jekyll)
3. Clone 下 Github 上创建的仓库到本地，进入本地仓库路径下创建项目：jekyll new .
4. 代码全部提交后，把项目 push 到 github 上进行预览
5. 也可以本地使用：jekyll serve 然后复制命令行中提示的地址进行预览
6. 集成 gitalk 博客评论系统

<a id="download_install_jekyll"/>
# 下载安装 Jekyll
- Jekyll 需要 ruby 环境且版本大于 2.3.0，ruby 的版本管理使用 rvm
- 查看当前 ruby 版本：ruby -v
- 查看所有 ruby 下载：rvm list know
- 安装具体 ruby 版本：rvm install 2.4.6
- 当前环境使用具体版本：rvm use 2.4.6
- 安装Jekyll：gem install jekyll bundler


[Github pages]: https://pages.github.com
[Jekyll]: https://jekyllrb.com
[Jekyll 中文]: https://jekyllcn.com