---
title: 【Bash】Bash学习之——变量内容的删除、取代与替换
categories:
  - 脚本
date: 2018-01-31 10:21:44
tags:
 - 脚本
 - Bash
---
## 删除

从左到右：

- '#' ：符合取代文字的『最短的』那一个；
- '##'：符合取代文字的『最长的』那一个

从右到左：

- % ：符合取代文字的『最短的』那一个；
- %%：符合取代文字的『最长的』那一个

![变量内容的删除](bash1.png)

## 取代

- ${变量/旧字符串/新字符串}     若变量内容符合『旧字符串』则『第一个旧字符串会被新字符串取代』
- ${变量//旧字符串/新字符串}    若变量内容符合『旧字符串』则『全部的旧字符串会被新字符串取代』

![变量内容的取代](bash2.png)

## 变量的测试与内容替换

![变量的测试与内容替换](bash3.png)

``` bash
范例一：测试一下是否存在 username 这个变量，若不存在则给予 username 内容为 root
[root@www ~]# echo $username
           <==由于出现空白，所以 username 可能不存在，也可能是空字符串
[root@www ~]# username=${username-root}
[root@www ~]# echo $username
root       <==因为 username 没有配置，所以主动给予名为 root 的内容。
[root@www ~]# username="vbird tsai" <==主动配置 username 的内容
[root@www ~]# username=${username-root}
[root@www ~]# echo $username
vbird tsai <==因为 username 已经配置了，所以使用旧有的配置而不以 root 取代

范例二：若 username 未配置或为空字符串，则将 username 内容配置为 root
[root@www ~]# username=""
[root@www ~]# username=${username-root}
[root@www ~]# echo $username
      <==因为 username 被配置为空字符串了！所以当然还是保留为空字符串！
[root@www ~]# username=${username:-root}
[root@www ~]# echo $username
root  <==加上『 : 』后若变量内容为空或者是未配置，都能够以后面的内容替换！

测试：若 str 不存在时，则 var 的测试结果直接显示 "无此变量"
[root@www ~]# unset str; var=${str?无此变量}
-bash: str: 无此变量    <==因为 str 不存在，所以输出错误信息 

测试：若 str 存在时，则 var 的内容会与 str 相同！
[root@www ~]# str="oldvar"; var=${str?novar}
[root@www ~]# echo var="$var", str="$str"
var=oldvar, str=oldvar  <==因为 str 存在，所以 var 等于 str 的内容
```
