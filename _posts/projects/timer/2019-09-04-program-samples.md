---
layout: post_no_cmt
title:  "Program Timer Samples"
date:   2019-09-03 00:00:00 +0800
categories: docs
---

# 默认示例（中文）
{% highlight go %}
# 默认示例

r60

begin

平板卧推 3#12 {一行一个动作}

曲臂伸 3#12 {花括号里面是备注。如 15kg}

仰卧起坐，备注可以不写 4#15s r15

引体向上 7# R30 {组数必须有，组间休息可覆盖}

提踵/原地小跑 1#25/90s {超组表达：12/12，12/20s，20s/20s/20s}

杠铃弯举 r20 {名字，休息，备注，组次可任意位置，但需空格分隔} 2#12

脚后跟触碰/跳绳/跳弓箭步 3#20s/20s/20s {三个连续动作为一组，重复次数以 s 结尾意味着这里需要跑秒}

多组不同次动作 7#12.8.6.4 {总共 7 组，次数分别为 12，8，6，4，4，4，4}

奇怪的动作示例 4#12/20s.8/15s.8/10s

end
{% endhighlight %}


# Default Samples（English）
{% highlight go %}
# Default Samples

r60

begin

Bench press 3#12 {One action one line}

Dip 3#12 {notes in curly braces. eg. 15kg}

Action without notes is ok 4#15s r15

Pull up 7# r30 {sets is necessary. rest between sets could be overwritten}

Calf raise/run in place 1#25/90s {supersets：12/12，12/20s，20s/20s/20s}

Barbell curl r20 {the order of name, rest, notes, sets#reps could be changed} 2#12

Side to side heel touch/jump rope/jump lungs 3#20s/20s/20s {3 movements without rest is 1 set, totally 3 sets}

Action with different reps for different sets 7#12.8.6.4 {totally 7 sets, 12, 8, 6, 4, 4, 4, 4}

Wired action 4#12/20s.8/15s.8/10s

end
{% endhighlight %}


# 大重量训练 Heavy Weight Training
{% highlight go %}
# 大重量训练 Heavy Weight Training

r90

begin

Dead lifts 屈腿硬拉 6#12.8.5.5.5.5
Wide chin ups 引体向上 6# {每组做到力竭 as much as you can}
Upright rows 直立划船 5#12.6
Decline sit ups 下斜仰卧起坐 4#15s r15s

end
{% endhighlight %}


# HIIT 示例（HIIT Sample）
{% highlight go %}
# HIIT 示例

r90

HIIT 5#20s

begin

Burpees 波比
Jumping jacks 开合跳
Mountain climbers 平地登山
Side to side jump 边对边跳跃

end
{% endhighlight %}