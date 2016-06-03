---
layout: post
title: afl-fuzzer 分析
category: blog
description: afl-fuzzer学习与分析
---



dwarfdump任意写漏洞分析
=======


### 漏洞现象

该漏洞存在当前最新的dwarf－20151114中。该漏洞可导致任意地址写。
漏洞发生在dwarfdump/print_abbrevs.c中，代码如下：

```
243                 if (abbrev_code > 0) {
244                     if (abbrev_code > abbrev_array_size) {
245                         /* Resize abbreviation array */
246                         abbrev_array_size *= 2;
247                         abbrev_array = (Dwarf_Signed *)
248                             realloc(abbrev_array,
249                             (abbrev_array_size+1) * sizeof(Dwarf_Signed));
250                     }
251                     abbrev_array[abbrev_code] = abbrev_entry_count;
252                     ++CU_abbrev_count;
253                     offset += length;
254                 } else {

```

第251行abbrev_code的值可以大于abbrev_array的大小，从而导致越界内存写入。
运行**dwarfdump -ka aw.elf**可触发漏洞。
使用valgrind检测结果如下：

```
==76920== Memcheck, a memory error detector        
==76920== Copyright (C) 2002-2013, and GNU GPL'd, by Julian Seward et al.
==76920== Using Valgrind-3.10.1 and LibVEX; rerun with -h for copyright info
==76920== Command: ./dwarfdump/dwarfdump -ka /tmp/aw.elf
==76920==
==76920== Invalid write of size 8
==76920==    at 0x40BE28: get_abbrev_array_info (print_abbrevs.c:251)
==76920==    by 0x40DDD9: print_one_die_section (print_die.c:706)
==76920==    by 0x40CF23: print_infos (print_die.c:319)
==76920==    by 0x4046D6: process_one_file (dwarfdump.c:1280)
==76920==    by 0x403529: main (dwarfdump.c:630)
==76920==  Address 0x541fc00 is 18,352 bytes inside an unallocated block of size 4,156,304 in arena "client"
```


### 漏洞分析
通过对触发漏洞的约束和路径进行分析，猜测问题应该出在_dwarf_decode_u_leb128函数中。

其中，触发漏洞的约束如下：

```
13     address: (Add w64 147851088                                                                                              
14          (Mul w64 8
15                   (Or w64 (And w64 (ZExt w64 (Read w8 11 aw.elf2880))                                                                                                             
16                                    127)                                                                                                                                           
17                           2432)))

```

路径约束如下：

```
18         (Eq false                                                                                                                                                                 
19             (Sle 0 N1:(Read w8 0 aw.elf2880)))
20         (Eq false                                                                                                                                                                 
21             (Sle 0 N2:(Read w8 1 aw.elf2880)))                                                                                                                                    
22         (Eq false
23             (Sle 0 N3:(Read w8 2 aw.elf2880)))                                                                                                                                    
24         (Eq false                                                                                                                                                                 
25             (Sle 0 N4:(Read w8 3 aw.elf2880)))                                                                                                                                    
26         (Eq false
27             (Sle 0 N5:(Read w8 4 aw.elf2880)))
28         (Eq false
29             (Sle 0 N6:(Read w8 5 aw.elf2880)))
30         (Eq false
31             (Sle 0 N7:(Read w8 6 aw.elf2880)))
 32         (Eq false
 33             (Sle 0 N8:(Read w8 7 aw.elf2880)))
 34         (Eq false
 35             (Sle 0 N9:(Read w8 8 aw.elf2880)))
 36         (Eq false
 37             (Sle 0 N10:(Read w8 9 aw.elf2880)))
 38         (Sle 0 N11:(Read w8 10 aw.elf2880))
 39         (Eq 0
 40             (Extract w16 0 (Or w64 (Shl w64 (And w64 (ZExt w64 N10) 127)
 41                                             63)
 42                                    (Or w64 (Shl w64 (And w64 (ZExt w64 N9) 127)
 43                                                     56)
 44                                            (Or w64 (Shl w64 (And w64 (ZExt w64 N8) 127)
 45                                                             49)
 46                                                    (Or w64 (Shl w64 (And w64 (ZExt w64 N7) 127)
 47                                                                     42)
 48                                                            (Or w64 (Shl w64 (And w64 (ZExt w64 N6) 127)
 49                                                                             35)
 50                                                                    (Or w64 (Shl w64 (And w64 (ZExt w64 N5) 127)
 51                                                                                     28)
 52                                                                            (Or w64 (Shl w64 (And w64 (ZExt w64 N4) 127)
 53                                                                                             21)
 54                                                                                    (Or w64 (Shl w64 (And w64 (ZExt w64 N3) 127)
 55                                                                                                     14)
 56                                                                                            (Or w64 (Shl w64 (And w64 (ZExt w64 N2) 127)
 57                                                                                                             7)
 58                                                                                                    (Shl w64 (And w64 (ZExt w64 N1) 127)
 59                                                                                                             0))))))))))))
 60         (Eq 0
 61             (Extract w16 0 (ZExt w64 N11)))
 62         (Eq false
 63             (Sle 0 (Read w8 11 aw.elf2880)))]
 64        false)

```

猜测在_dwarf_decode_u_leb128函数中存在一条路径，可使得返回的值为非预期的值。



