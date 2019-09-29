---
layout: post
title:  "Program Timer Help Doc"
date:   2019-09-03 00:00:00 +0800
categories: docs
permalink: /timer/help/en
lang: en
---

# First
[**Samples**]({% post_url /projects/timer/2019-09-04-program-samples %}) in App has showed the most of how to write script.  
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
- Line 5, 9: the bengin and the end sign of training actions, **Required**
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

- Different with [Completed training program](#complete_training_structure), HIIT training program has HIIT sign, HIIT sets and exercise time of single action, see line 4
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


#### General action sample
```
Incline bench press 3#12 r30 {15kg}
```
Explanation: Incline bench press. 3 sets, 12 reps each set, overwrite the rest between sets to 30 secs, notes: 15kg.


#### Cardio action sample
```
Jump jack 6#20s r20
```
Explanation：Jump jack. 6 sets, 20 secs each set, overwrite the rest between sets to 20 secs.


#### Complex action sample
```
Pull up/Push up 4#10/20 

Decline sit ups/Jump jack 6#7/20s r20

Burpees/Reverse crunch 4#20s/20s

Lying toe touches/Reverse crunch/Circles 3#20s/20s/20s
```

#### Different repetition for different set sample
```
Heavy bench press 6#12.10.8.5.5.5

or

Heavy bench press 6#12.10.8.5
```

> The set without specific repetition will be auto filled by the last repetition

Complex sets are also supported. eg. `Decline sit ups/Jump jack 6#20/40s.10/30s.7/20s`
