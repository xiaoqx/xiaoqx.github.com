---
layout: post
title: afl-fuzzer 分析
category: blog
description: afl-fuzzer学习与分析
---



afl-fuzzer 分析(1)
=======


### 总体思路

参考technical_details.txt，先不写。


#### 插桩记录边覆盖基本流程

以llvm-mode为例，结合代码进行分析。

afl-clang-fast.c主要是一个wraper，完成如下命令的封装：

```
clang
-Xclang
-load
-Xclang
../afl-llvm-pass.so
-Qunused-arguments
-O3
-funroll-loops
-Wall
-D_FORTIFY_SOURCE=2
-g
-Wno-pointer-sign
-DAFL_PATH="/usr/local/lib/afl"
-DBIN_PATH="/usr/local/bin"
-DVERSION="1.74b"
../test-instr.c
-o
test-instr
-g
-O3
-funroll-loops
../afl-llvm-rt.o
```

这个命令主要是调用clang来编译目标文件，并通过afl-llvm-pass.so来完成插桩。
这里也可以通过opt来调用。（这样可以在llvm IR中来观察插桩的代码，比看汇编方便）

插桩后的代码与未插桩的代码比较如下：
![IR比较](/images/afl-fuzzer/afl-diff.png)

alf-llvm-pass主要在每个basic block（bb）前插入代码记录执行情况，代码功能主要是在
一个64k的共享内存中记录两个bb之间的边的执行次数。
通过编译时随机生成的一个数来作为该bb块的序号，并用当前bb块的序号和上一bb块的序号（右移1位）进行
异或来表示边在共享内存中的位置。

afl中的共享内存是利用系统V中的IPC来实现的，具体通过shmget，shmat等函数完成。

afl-llvm-rt.o.c主要是提供了一些初始化操作，相当于在目标程序执行main函数之前完成一些操作。
主要通过constructor来实现，在__afl_manual_init函数中完成了获取共享内存地址，以及start_forkserver_ 。

当一次具体执行（即一次fuzz测试完成）后，共享内存中将记录该次执行所覆盖到的所有边的次数。

### Fuzzing的异常检测
afl主要检测crash和hang两种异常。    
crash是基于linux的coredump机制来完成的（？），fuzzing执行的target程序结束后通过WIFSIGNALED来判断。
WIFSIGNALED这个函数的功能是检测进程的结束是否是因为捕获信号而终止，如果是则认为是crash。
hang主要是通过设计超时来检测，如果fuzzing执行的target程序在运行结束后，对child_timed_out标志进行判断，
如果被设置了，则认为是hang。


#### Fuzzing策略
afl使用了多种Fuzzing策略（数据变异方法），

值得注意的是afl实现了一种fork-server的fuzzing方法，使得其测试效率提升了至少2倍（a factor of two）。

##### fork server
大致介绍一下我理解的fork server（由于linux编程基础较差，也许理解的有误）。

由于linux启动新的进程通常使用fork+exec** 来实现，fork产生一个新的进程，exec将目标代码加载到这个新进程。
而当启动的新进程就是父进程本身的程序时，就可以不需要exec。afl利用这一点，在被测试的目标进程中插桩来完成fork，
从而省掉了exec这个步骤，详细介绍可参考
[1][fuzzing binaries without execve](http://lcamtuf.blogspot.com/2014/10/fuzzing-binaries-without-execve.html)

#### 选择fuzzing 种子（culling the corpus）


#### 优化fuzzing输入（trimming input files）

#### crash的分类

#### crash的可利用性识别


### 总结
（fuzzing只是afl的一小部分，其包括的东西真不少。）
