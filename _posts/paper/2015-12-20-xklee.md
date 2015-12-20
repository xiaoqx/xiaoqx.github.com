---
layout: post
title: 堆分配大小可控检测分析
category: paper
description: 堆分配大小可控检测分析
---

## 介绍

不当内存操作一直是引发软件漏洞的主要原因之一。堆分配大小可控（CMA）是指当动态内存分配的关键参数可以被外界输入控制，恶意用户可以通过精心构造输入数据导致非预期的内存分配。本文讨论了CMA可能引发的相关安全问题以及CMA的检测方法。论文提出了一种新颖的CMA检测方法，主要通过结合静态路径分析和路径导向符号执行技术的优势，系统的检测目标代码中的CMA问题。论文在经典的符号执行引擎KLEE的基础上，实现了CMA检测原型系统SCAD；通过对Linux系统常用的工具程序Coreutils进行测试，SCAD发现了10个CMA相关的问题，其中3个属于未公开漏洞,其中3个是未公开的。三个包含bug的工具分别是split、cut和shuf，其中split和cut的CMA bug将导致内存耗尽并自动退出，[shuf]中的bug可以直接导致程序崩溃，在我们报告给开发者24小时内就得到了修复。

* split －CT
* cut -c3333333333333
* [shuf] -er

实验结果表明，SCAD的导向路径搜索算法与KLEE提供的8个路径搜索算法相比具有明显优势；针对内存分配相关的代码，SCAD的导向符号执行相比传统的符号执行引擎具有更高的代码覆盖率。

## 背景
在多数编程语言中，程序员可以静态、自动、动态的管理内存。例如，在C语言中，通过库函数malloc来申请堆内存空间，程序可以通过内存指针来访问堆中的数据，当内存不再被使用时，可以调用free函数来释放。
	动态内存分配的灵活性给程序带来了很多便利和优势，同时也导致了不少可能存在的问题。这些问题可能导致程序崩溃甚至安全漏洞，通常表现为段错误。动态内存分配常常导致的错误包括：
*	1．没有检测失败的分配而导致错误。
*	2．内存泄露。
*	3．逻辑错误。动态内存分配的使用有着自己的逻辑：分配-使用-释放。如果违反了这个逻辑，将可能导致错误。比如悬挂指针（dangling pointer），野指针（wild pointer），多次释放（double free）等。

CMA可以引发上述大部分内存错误，导致程序出现bug。

## 主要内容

请参考论文。

## 使用说明

待续。

## 漏洞分析

### BugAnalysis - shuf
       2014-02-23 15:02   Qixue Xiao <xiaoqixue_1@163.com>
=========================================================================


### Bug overview

	shuf -er or shuf -eer [ segment fault]
	impact [coreutils 8.22 ]

```
[15:03:59]xqx@server:~/data/xqx/projects/coreutils-8.22$ ./obj-gcov/src/shuf -er
Segmentation fault (core dumped)

```

### Analysis

when shuf execute -e without give the expected input lines, it will assign n_lines to 0 in "write_random_lines" while the "repeat" (-r) be set. and this var will be as the genmax parameter when "randint_genmax" function called. the code as follows in shuf.c:

```
369   for (i = 0; i < count; i++)
370     {
371       const randint j = randint_choose (s, n_lines);
372       char *const *p = lines + j;
373       size_t len = p[1] - p[0];
374       if (fwrite (p[0], sizeof *p[0], len, stdout) != len)
375         return -1;
376     }
377

```

'j' will be a random number between 0-0xffffffffffffffff in my 64bit ubuntu, and 'p' will be a unexpected point which will be access next. when p point to an ilegal memory, it will be error when access it, which may be result in a Segmentation fault.

if an attacker could control the random which gened by randint_choose, it may be get the infomation without an legal authority.



[shuf]: http://debbugs.gnu.org/cgi/bugreport.cgi?bug=16855
