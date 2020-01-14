---
layout: post
title:  "Program Timer"
date:   2020-01-14 00:00:00 +0800
categories: projects
permalink: /programTimer/en
---
![Top image](/assets/images/img_top_big_en.png)

# Introduction
{{page.title}} is an App that combine custom training program with timer. It is designed and developed for fitness enthusiasts or senior fitness people to make their own daily training program. It supports two methods to help people make training program quickly, script and GUI, meanwhile,  workout time and rest time in action will be recognized and executed by timer. For convenience, notes for each action is developed for recording weights, tips and so on.


# Features
- Support HIIT and general strength training
- Support timing for superset
- Support rest timing after action timing(auto run)
- Support notes for each action
- Support batch import training programs by scripts
- Support batch export all training program scripts
- Support GUI to create training program
- Support background timing
- Support sound and vibration

# Screenshots
![Screenshot](/assets/images/img_menu_en.jpeg){: width="180" float="left"}
![Screenshot](/assets/images/img_samples_en.jpeg){: width="180" float="left"}
![Screenshot](/assets/images/img_my_programs_en.jpeg){: width="180" float="left"}
![Screenshot](/assets/images/img_new_program_en.jpeg){: width="180" float="left"}
![Screenshot](/assets/images/img_new_action_en.jpeg){: width="180" float="left"}
![Screenshot](/assets/images/img_workout_en.jpeg){: width="180" float="left"}
![Screenshot](/assets/images/img_lock_en.jpeg){: width="180" float="left"}

# Docs
[《Program Timer Help Doc.》]({% post_url /projects/timer/2019-09-19-script-help-en %})  
[《训练脚本帮助文档》]({% post_url /projects/timer/2019-09-03-script-help %})


# Download
[Download: {{page.title}}]({{ site.url }}/programTimer/download)
<div id="code"></div><br/>


# Versions
2019-07-25 v1.0.0:
- Create training program: script/GUI
- Training preview before workout
- Training program timing
- Edit Training program

2019-07-31 v1.5.0
- Background timing
- Optimise edit trining feature
- Batch export training program
- Batch import training program
- Update training preivew feature

2019-08-23 v2.0.0
- Add notification for timing operation
- Redesgin training workout list
- Add settings feature

2019-12-02 v2.0.4
- Fix i18n bugs
- Redesgin training program list
- Add Admob

2020-01-10 v2.0.7
- Fix batch import bug
- Optimise error tips for batch import


<script src="/assets/js/jquery.min-1.5.2.js"></script>
<script src="/assets/js/jquery.qrcode.min.js"></script>
<script type="text/javascript">
  $("#code").qrcode({
    width: 200,
    height: 200,
    correctLevel:0,
    text: "{{ site.url }}/programTimer/download"
  });
</script>