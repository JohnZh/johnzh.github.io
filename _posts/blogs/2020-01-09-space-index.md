---
layout: post
title:  "Java：明明有空格 indexOf 却是 -1"
date:   2019-09-09 00:00:00 +0800
categories: [blogs, projects]
---

# 问题描述
项目中 bug 引出的问题，一个字符串中明明有空格，然而使用 `String.indexOf(" ")` 却得到了 -1

- 出现问题的字符串：推 5


# 问题原因
原因大概是由于 Unicode 中存在 3 种空格：(\u00A0,\u0020,\u3000) 分别为不间断空格，半角，全角空格，而代码中使用的一般是半角空格，indexOf 其他两个都会得到 -1

# 问题修复
统一空格后再进行字符串处理，或者更甚者除了回车和换行，其他空白字符都转变为半角空格，下为代码举例：

```
/**
 * 统一三种类型的空格
 *
 * @param str
 * @return
 */
private String unitizeSpace(String str) {
    str = str.replaceAll("[^\\S\\r\\n]+", " ");
    return str;
}
```
