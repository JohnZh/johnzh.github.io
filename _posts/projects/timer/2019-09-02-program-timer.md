---
layout: post
title:  "Program Timer:  训练计划计时秒表"
date:   2019-09-02 00:00:00 +0800
categories: projects
permalink: /timer/about
download_url: /download/program-timer.apk
---

# 前言
早在 2017 年，我还在上班的时候，一个想法便一直困扰着我，从事 App 开发这么多年，参与过的项目近 20 个，完成度比较高的也有六七个，然而最后能留于我手机上的却一个也没有，我真的有做一些对自己，对其他人，对社会有价值的项目吗？而到了 2018 年，想做一些能让自己会一直使用的 app 的想法愈发的强烈。


# 启发
从 2014 年开始接触健身，最开始只是为了避免久坐变胖，毕竟在中国程序员一天工作 10+ 个小时的情况并不少见。到后面发现健身可以让自己的精神状态变好，以至于最后喜欢上健身。这其中，也伴随着健身知识的不断积累，健身方式的升级，训练计划的改变。而在一次训练中，我发现在健身的时候查看健身计划，切换计时软件执行休息或有氧，记录当下训练重量或者训练心得，最少得用到两个 App 来回切换，这其实十分的不便。而后，在试用了多款计时 App 后，发现都没有十分符合我需求的 App，于是便有了自己做一个的想法。	


# 产品介绍
{{page.title}} 是结合了自定义训练计划与计时跑秒的一款 App。专门为了健身爱好者或者资深健身人士用于制作自己每日，每周，甚至更久的训练计划而设计开发。提供了脚本编写和用户界面两种方式帮助你快速生成每日不同的健身计划，方便实用。


# 功能说明
- 支持 HIIT 和常规力量训练两种训练类型
- 支持复合动作计时模式
- 支持训练计时完成后直接进入休息计时（自动执行）
- 支持给训练里面每个动作添加单行备注
- 支持训练脚本方式，快速导入每日，每周，甚至更久的训练计划
- 支持以脚本形式快速导出所有的训练计划，方便备份
- 支持用户界面形式编写单个训练计划
- 支持后台运行
- 支持音效和震动的提示

# 应用截图
![应用截图](/assets/images/img_menu.jpeg){: width="180" float="left"}
![应用截图](/assets/images/img_samples.jpeg){: width="180" float="left"}
![应用截图](/assets/images/img_my_programs.jpeg){: width="180" float="left"}
![应用截图](/assets/images/img_new_program.jpeg){: width="180" float="left"}
![应用截图](/assets/images/img_new_action.jpeg){: width="180" float="left"}
![应用截图](/assets/images/img_workout.jpeg){: width="180" float="left"}
![应用截图](/assets/images/img_lock.jpeg){: width="180" float="left"}

# 帮助文档
[《训练脚本帮助文档》]({% post_url /projects/timer/2019-09-03-script-help %})  
[《Program Timer Help Doc.》]({% post_url /projects/timer/2019-09-19-script-help-en %})


# 下载使用
{% assign updateInfo = site.data.timer-update-info %}
[{{page.title}}]({{ updateInfo.downloadUrl }})


# 版本变化
2019-07-25 v1.0.0:
- 训练添加：脚本/用户界面模式
- 训练预览，开始训练前可以先预览内容
- 训练执行
- 训练编辑

2019-07-31 v1.5.0
- 修改计时机制为后台计时
- 优化训练编辑操作
- 全部训练脚本导出
- 批量脚本导入
- 预览改版

2019-08-23 v2.0.0
- 添加可以具有计时控制的通知
- 训练执行列表用户界面整体改版
- 添加设置
