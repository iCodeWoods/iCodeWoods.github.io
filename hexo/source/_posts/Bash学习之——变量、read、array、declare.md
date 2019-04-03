---
title: Bash学习之——变量、read、array、declare
categories:
  - 脚本
date: 2018-01-26 11:33:26
tags:
 - 脚本
 - Bash
---
## 变量的配置守则

- 双引号内的特殊字符如 $ 等，可以保有原本的特性，如： “var="lang is $LANG"”则“echo $var”可得“lang is zh_TW.UTF-8”
- 单引号内的特殊字符则仅为一般字符 （纯文本），如： “var='lang is $LANG'”则“echo $var”可得“lang is $LANG”
- 在一串指令的执行中，还需要借由其他额外的指令所提供的信息时，可以使用反单引号 `指令` 或 “$（指令）”
- 若该变量为扩增变量内容时，则可用 "$变量名称" 或 ${变量} 累加内容，如： “PATH="$PATH":/home/bin”或“PATH=${PATH}:/home/bin”
- 若该变量需要在其他子程序执行，则需要以 export 来使变量变成环境变量： “export PATH”
- 通常大写字符为系统默认变量，自行设置变量可以使用小写字符，方便判断 （纯粹依照使用者兴趣与嗜好）
- 取消变量的方法为使用 unset：“unset 变量名称”。例如取消 myname 的设置：“unset myname”

<!-- more -->

## 环境变量

- $：目前这个 Shell 的线程代号，亦即是所谓的 PID
- ?：上一个执行的指令所回传的值
- RANDOM：产生随机数。目前大多数的 distributions 都会有随机数生成器，那就是 /dev/random 这个文件。 我们可以通过这个随机数文件相关的变量 ($RANDOM) 来随机取得随机数值。在 BASH 的环境下，这个 RANDOM 变量的内容，介于 0~32767 之间，所以，你只要 echo $RANDOM 时，系统就会主动的随机取出一个介于 0~32767 的数值。如果想要使用 0~9 之间的数值呢？利用 declare 宣告数值类型，然后这样做就可以了：
``` bash
bash-3.2$ declare -i number=$RANDOM%10; echo $number
8   <== 此时会随机取出 0~9 之间的数值喔！
```

## 变量键盘读取、数组与宣告： read, array, declare

### read

读取来自键盘输入的变量。
``` bash
[root@www ~]# read [-pt] variable
选项与参数：
-p ：后面可以接提示字符
-t ：后面可以接等待的秒数

例：提示使用者 30 秒内输入自己的大名，将该输入字符串作为名为 named 的变量内容
[root@www ~]# read -p "Please keyin your name: " -t 30 named
Please keyin your name: VBird Tsai   <==注意看，会有提示字符喔！
[root@www ~]# echo $named
VBird Tsai        <==输入的数据又变成一个变量的内容了！
```

### declare / typeset

declare 或 typeset 是一样的功能，就是在“宣告变量的类型”。
``` bash

[root@www ~]# declare [-aixr] variable
选项与参数：
-a ：将后面名为 variable 的变量定义成为数组 (array) 类型
-i ：将后面名为 variable 的变量定义成为整数数字 (integer) 类型
-x ：用法与 export 一样，就是将后面的 variable 变成环境变量；
-r ：将变量配置成为 readonly 类型，该变量不可被更改内容，也不能 unset

范例一：让变量 sum 进行 100+300+50 的加总结果
[root@www ~]# sum=100+300+50
[root@www ~]# echo $sum
100+300+50  <==咦！怎么没有计算加和？因为这是文字型态的变量属性啊！
[root@www ~]# declare -i sum=100+300+50
[root@www ~]# echo $sum
450
```

默认的情况下， bash 对于变量有几个基本的定义：

- 变量类型默认为“字串”，所以若不指定变量类型，则 1+2 为一个“字串”而不是“计算式”。
- bash 环境中的数值运算，默认最多仅能到达整数形态，所以 1/3 结果是 0；

### array

在 bash 里数组的配置方式是：var[index]=content
``` bash
范例：配置上面提到的 var[1] ～ var[3] 的变量。
[root@www ~]# var[1]="small min"
[root@www ~]# var[2]="big min"
[root@www ~]# var[3]="nice min"
[root@www ~]# echo "${var[1]}, ${var[2]}, ${var[3]}"
small min, big min, nice min
```

*参考文章：[鸟哥的私房菜](http://cn.linux.vbird.org/linux_basic/0320bash.php#variable)*
