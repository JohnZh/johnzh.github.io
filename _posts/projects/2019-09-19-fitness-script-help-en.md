---
layout: post
title:  "Fitness plan & timer doc"
date:   2019-09-03 00:00:00 +0800
categories: docs
permalink: /fitnessAppHelp/en
lang: en
---

# First
[**Samples**]({% post_url /projects/2019-09-04-fitness-plan-samples %}) in App has showed the most of how to write script.  
You can long click samples item and **check** the script content.

<a id="complete_training_structure"/>

# Completed training program
{% highlight go linenos %}
# Program name

R60 

begin 

Action name 4#12 

end
{% endhighlight %}

- Line 1: Program name, **Required**
- Line 3: R60, the global rest time, the rest time bewteen and actions, **Required**
- Line 5, 9: the bengin and the end sign of training actions，**Required**
- Line 7: action script sample, **Required**, see [Action script](#action)


# Completed HIIT training program
{% highlight go linenos %}
# HIIT 名称

R20 
HIIT 8#20s

begin 

Action 1 {Notes of action 1}
Action 2
Action 3

end
{% endhighlight %}

- Different with [Completed training program](#complete_training_structure)，HIIT training program has HIIT sign, HIIT sets and exercise time of single action，see line 4
- For action, name is **Required**
- For action, notes are optional

<a id="action" />

# Action script
```
Action name sets#repetition {notes} rest-time
```
- Action name, **Required**
- Sets, integers, **Required**
- Repetition, integers or time(eg. 20s), optional
- Notes, single line, optional
- Rest-time, optional

> The above elements are not fixed positions


#### General sample
```
Incline bench press 3#12 r30 {15kg}
```
Explanation: Incline bench press. 3 sets, 12 reps each set, overwrite the rest between sets to 30 secs, notes: 15kg.


#### Cardio sample
```
Jump jack 6#20s r20
```
Explanation：Jump jack. 6 sets, 20 secs each set, overwrite the rest between sets to 20 secs.


#### 复合动作举例
```
引体向上/俯卧撑 4#10/20 

上斜仰卧起坐/开合跳 6#7/20s r20

波比/卷腹 4#20s/20s

仰卧触脚尖/卷腹/旋转 3#20s/20s/20s
```

#### 不同组不同重复次数举例
```
大重量杠铃卧推 6#12.10.8.5.5.5

或

大重量杠铃卧推 6#12.10.8.5
```

> 未写明的组的次数，会直接用最后一组次数补全

同样也支持复合组，例如：`上斜仰卧起坐/开合跳 6#20/40s.10/30s.7/20s`
