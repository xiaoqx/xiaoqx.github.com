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


### 工作流程

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


