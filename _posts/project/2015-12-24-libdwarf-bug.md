---
layout: post
title: libdwarf 0day 分析
category: project
description: libdwarf漏洞的分析与利用
---





libdwarf任意写漏洞分析与利用
=======


### 漏洞原因分析

在libdwarf一些历史版本中，存在任意写内存漏洞。可导致攻击者在任意地址写入任意值。
可影响的版本包括libdwarf-20140805及以前的版本。本文的分析的版本为libdwarf-20140805。

漏洞发生在\_dwarf_setup中，代码如下：

```
漏洞位置：
        if (doas.type == SHT_RELA && sections[doas.info]) {
            add_rela_data(sections[doas.info],&doas,
                  obj_section_index);
         }
          
```

漏洞分析：
在上面的代码中，sections数组的大小为31。而doas.info的值是来自被分析的二进制文件，可以为任意值。
在此处未检测doas.info的大小，当doas.info较大时，存在数组越界。



### 漏洞利用思路

对于任意地址写的漏洞很容易想到改写GOT表来修改函数地址。为什么GOT表可写，可以参考关于ELF和动态链接相关的知识。
通过查看代码，发现在触发上面的漏洞之后会调用一个memset函数。

思路比较简单，在利用过程中仍解决一些问题。

* sections是一个堆上的数组，每次分配的地址不一样，通过这样的一个任意地址写准确的写入到固定的地址。
* /bin/sh 放到文件中，但如何确定其地址；

解决上述问题后，可稳定的执行代码，如下图所示：
![成功执行shell](/images/libdwarf-exploit.png)

