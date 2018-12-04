---
title: Bash学习之——数据流重导向
categories:
  - 脚本
date: 2018-03-08 15:16:05
tags:
 - Bash
 - 脚本
---
## stdout、stderr、/dev/null

* 标准输入　　  (stdin)：代码为 0 ，使用 < 或 << ；
* 标准输出　 　(stdout)：代码为 1 ，使用 > 或 >> ；
* 标准错误输出 (stderr)：代码为 2 ，使用 2> 或 2>> ；

#### 覆盖与累加
*  1> ：以覆盖的方法将『正确的数据』输出到指定的文件或装置上；
*  1>>：以累加的方法将『正确的数据』输出到指定的文件或装置上；
*  2> ：以覆盖的方法将『错误的数据』输出到指定的文件或装置上；
*  2>>：以累加的方法将『错误的数据』输出到指定的文件或装置上；

例：将 stdout 与 stderr 分别存入不同的文件中：
``` Bash
find /home -name .bashrc > list_right 2> list_error
```

#### /dev/null 垃圾桶黑洞装置与特殊写法
如果我知道错误信息会发生，所以要将错误信息忽略掉而不显示或储存呢？黑洞装置 /dev/null 可以吃掉任何导向这个装置的信息喔！

将错误的数据丢弃，屏幕上显示正确的数据
``` Bash
[dmtsai@www ~]$ find /home -name .bashrc 2> /dev/null
/home/dmtsai/.bashrc  <==只有 stdout 会显示到屏幕上， stderr 被丢弃了
```

再想象一下，如果我要将正确与错误数据通通写入同一个文件去呢？这个时候就得要使用特殊的写法了！ 

我们同样用底下的案例来说明：

将命令的数据全部写入名为 list 的文件中
``` Bash
[dmtsai@www ~]$ find /home -name .bashrc > list 2> list  <==错误
[dmtsai@www ~]$ find /home -name .bashrc > list 2>&1     <==正确
[dmtsai@www ~]$ find /home -name .bashrc &> list         <==正确
```

#### standard input ： < 与 <<
``` Bash
范例六：利用 cat 命令来创建一个文件的简单流程
[root@www ~]# cat > catfile
testing
cat file test
<==这里按下 [ctrl]+d 来离开
[root@www ~]# cat catfile
testing
cat file test

范例七：用 stdin 取代键盘的输入以创建新文件的简单流程
[root@www ~]# cat > catfile < ~/.bashrc
[root@www ~]# ll catfile ~/.bashrc
-rw-r--r-- 1 root root 194 Sep 26 13:36 /root/.bashrc
-rw-r--r-- 1 root root 194 Feb  6 18:29 catfile
# 注意看，这两个文件的大小会一模一样！几乎像是使用 cp 来复制一般！
```

理解 < 之后，再来则是怪可怕的 << 这个连续两个小于的符号了。 他代表的是『结束的输入字符』的意思！举例来讲：『我要用 cat 直接将输入的信息输出到 catfile 中， 且当由键盘输入 eof 时，该次输入就结束』，那我可以这样做：
``` Bash
[root@www ~]# cat > catfile << "eof"
> This is a test.
> OK now stop
> eof  <==输入这关键词，立刻就结束而不需要输入 [ctrl]+d
[root@www ~]# cat catfile
This is a test.
OK now stop     <==只有这两行，不会存在关键词那一行！
```

## 命令运行的判断依据： ; , &&, ||
![&&与||](bash1.png)

例：
``` Bash
不清楚 /tmp/abc 是否存在，但就是要创建 /tmp/abc/hehe 文件
[root@www ~]# ls /tmp/abc || mkdir /tmp/abc && touch /tmp/abc/hehe
```
由于Linux 底下的命令都是由左往右运行的，所以范例三有几种结果我们来分析一下：

* 若 /tmp/abc 不存在故回传 $?≠0，则 (2)因为 || 遇到非为 0 的 $? 故开始 mkdir /tmp/abc，由于 mkdir /tmp/abc 会成功进行，所以回传 $?=0 (3)因为 && 遇到 $?=0 故会运行 touch /tmp/abc/hehe，最终 hehe 就被创建了；
* 若 /tmp/abc 存在故回传 $?=0，则 (2)因为 || 遇到 0 的 $? 不会进行，此时 $?=0 继续向后传，故 (3)因为 && 遇到 $?=0 就开始创建 /tmp/abc/hehe 了！最终 /tmp/abc/hehe 被创建起来。

![流程](bash2.png)

例题：
以 ls 测试 /tmp/vbirding 是否存在，若存在则显示 "exist" ，若不存在，则显示 "not exist"！
``` Bash
ls /tmp/vbirding && echo "exist" || echo "not exist"
```

