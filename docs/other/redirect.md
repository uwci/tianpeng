---
title: 2>&1
date: 2018-09-28 09:44:06
tags: [Linux, Bash]
description: 重定向符解析
---

初识这个命令的时候，还是山川dalao帮我在开发机上部署java程序时，打印日志的时候使用的

当时觉得 这 2 呀 1 呀 & 什么的 怕不是位运算？
怎么就变成打印日志了

**内心毫无波澜的仰慕**

后来自己在ECS上打印日志的时候也用到过这个命令，想还是深究一下这个问题

懒癌晚期的我就cope了一个例子`nohup command >> /dev/null 2>&1 &`

其中
* `nohup`代表 持久化 命令，使得在退出ssh之后，还能持续运行命令 = `no hang up`
* 最后一个`&` 代表以job的方式不挂起运行
* `/dev/null` 代表空设备地址

所以等价于分析`command >> /dev/null 2>&1`

## 重定向符`>`

这个命令想必大家多少都用过，
比如说两个二进制文件合成一个文件的时候
```bash
cat a >> b
# 其中 >  表示覆盖写入
#     >> 表示连续写入
```

上述命令其实可以分为两个部分，
* 先做了一个`cat a`命令
* 然后把`cat a`得到的输出，重定向到b中

看起来是显而易见的过程，组成了我们后面的`2>&1`命令

## 2 1 0

重定向符号中的数字有特殊的含义

* 0 表示`stdin`标准输入
* 1 表示`stdout`标准输出，系统默认值是1，所以`>b`等价于 `1>b`
* 2 表示`stderr`标准错误

那么`2>&1`的意识就应该是把命令的标准错误 重定向 到标准输出中

好像又不是 这里多了一个`&`

## `>` and `>&`

* `>name`    表示将输出重定向到文件 name
* `>&number` 表示将输出重定向到文件描述符 number

所以这里`2>&1`的意识便是，把标准错误，重定向到文件描述符`&1`中，即把标准错误重定向到标准输出中

那看我们最初的那个例子
`command >> /dev/null 2>&1` = `command 1>> /dev/null 2>&1`

1. 把`command`的`标准输出`，重定向 至 `/dev/null`中
2. 把`标准错误`重定向到标准输出，即 把 `标准错误` 也 重定向 写入 `/dev/null`中

这个时候 `重定向` 就相当于 `指针`的概念
![图片.png | center | 556x200](https://cdn.nlark.com/yuque/0/2018/png/104214/1538123222101-0129a050-0d43-4dc6-bbef-5f74e21dcb8e.png "")

其实多想一会就会发现这个看起来很骚的写法确实有它的好处

我们为啥不写成`command >> /dev/null 2> /dev/null`

很明显，把重定向分成两个单独的操作之后，也意味着两次文件打开
这样的操作代价更大

很多时候看起来很漂亮的代码，可能也对应着性能上的提升

## `3>&1 1>&2 2>&3` :100:

经过上面的讲解 好像 明白了重定向 文件描述符的原理

但 看到`3>&1 1>&2 2>&3`的时候 我还是一脸懵逼的

蛤 不是 说好 只有 0 1 2, 3又是啥子

其实 这里的 3 是一个我们自己定义的文件描述符

所以这句code的意识是
1. 先定义个fd 叫做 3, 把它重定向 &1-stdout
2. 把fd 1 重定向到 &2-stdeer
3. 把fd 2 重定向到 &3

有人说，实际上做的是swap(stdout, stdeer)的工作
那么 我这个时候
* 执行`make 3>a 3>&1 1>&2 2>&3` a中应该为`stdout`
* 执行`make 2>a 3>&1 1>&2 2>&3` a中应该为`stdout`
* 执行`make 1>a 3>&1 1>&2 2>&3` a中应该为`stdeer`

但实际上测试的时候发现运行的时候并不是这样的
* 执行`make 3>a 3>&1 1>&2 2>&3` a中为`stdeer`
* 执行`make 2>a 3>&1 1>&2 2>&3` a中应该为`stdout + stdeer`
* 执行`make 1>a 3>&1 1>&2 2>&3` a中应该为`stdout + stdeer`

![图片.png | center | 556x200](https://cdn.nlark.com/yuque/0/2018/png/104214/1538127583777-edadb61f-f6e9-49ce-a657-cee8dbe7944e.png "")

我是这么来理解的Linux执行命令的时候是从左至右
* `make 3>a 3>&1 1>&2 2>&3`
  - 一开始遇到 fd 3的时候，不认识它，不输出。然后往右遍历，发现 3被2重定向，于是`fd3`=`stdeer`, 输出;
* `make 2>a 3>&1 1>&2 2>&3`
  - 一开始遇到 fd 2的时候，认识它，输出`stdeer`。然后往右遍历，发现 2被1重定向, 于是`fd2`=`stdout`, 输出;
* `make 1>a 3>&1 1>&2 2>&3`
  - 一开始遇到 fd 1的时候，认识它，输出`stdout`。然后往右遍历，发现 1被3重定向，3被2重定向, 于是`fd1`=`stdeer`, 输出;

遍历的原则是只考虑需要输出的fd，然后遍历到已知fd为止。

再回来看`make >a 2>&1`
- 一开始遇到 fd 1的时候，认识它，输出`stdout`。然后往右遍历，发现 1被2重定向，于是`fd1`=`stdeer`, 输出;

## 参考
. [Linux里的2>&1究竟是什么？](https://mp.weixin.qq.com/s/k-QHnuIg7VnSTY9qyBZueQ)
. [What does “3>&1 1>&2 2>&3” do in a script?](https://unix.stackexchange.com/questions/42728/what-does-31-12-23-do-in-a-script)