原文地址：:point_right: [AFL -- README.dislocator](https://github.com/google/AFL/tree/master/libdislocator)

`libdislocator, an abusive allocator`

这是一个附带的库，**可以作为模糊二进制文件中libc分配器的临时替代品**。它通过几种方式提高了遇到与堆相关的安全漏洞的几率。

* 它分配所有的缓冲区，所以它们与后续的PROT NONE页相邻，导致大多数off-by-one的读写操作都立即执行segfault（大概意思是，缓冲区占满，缓冲区之后是PROT NONE，导致那种越位一个的读写操作，也能segfault）（分配的堆块的最后一个字节以页对齐，然后会在这后面分配一个页，这个页的权限是不可访问的，这样一旦溢出就会崩溃了？）
* 它在已分配的缓冲区的正下方添加canary，以捕获对负偏移量的写操作(但是不会捕获读操作)（方向写反情况）
* 它将malloc()返回的内存设置为garbage values，提高了目标访问未初始化数据时崩溃的几率（malloc是不初始化分配的内存，这里给放入一些garbage values是什么值，为什么可以提高崩溃率）
* 它将已释放的内存设置为PROT NONE，实际上并没有重用它，这导致大多数“use- afterfree”bug立即跳转到segfault。（使用释放指针的内存处理）
* 它强制所有realloc()调用返回一个新的地址，并在原始块上设置PROT NONE。这捕获了realloc之后使用的bug
* 它检查calloc()溢出，并可能导致alloc请求的软或硬失败(通过传递超过了可配置的内存限制(AFL_LD_LIMIT_MB,AFL_LD_HARD_FAIL).)

基本上，**它是受一些可用于OpenBSD分配器的非默认选项的启发**-请参阅该平台上的malloc.conf（5）以供参考。 它在某种程度上类似于gmalloc和DUMA等其他调试库，**但是它简单，即插即用，并且专门为模糊测试而设计**。

**请注意，它对于基于栈的内存处理错误不起作用**。 使用AFL_HARDEN时启用的GCC/clang的**-fstack-protector-all**设置可以捕获其中的一部分。

分配器速度很慢且占用大量内存（即使最小的分配也会占用4 kB的物理内存和8 kB的虚拟内存），使其完全不适合“生产”用途。 但是**在模糊小型独立的二进制文件时，它比ASAN/MSAN更快，更轻松。**

要使用这个库，请像这样运行AFL

```shell
AFL_PRELOAD=/path/to/libdislocator.so ./afl-fuzz [...other params...]
```

我们不得不指定路径，即使`./libdislocator.so`或者`$PWD/libdislocator.so`

与afl-tmin相似，该库不是“专有”的，**可以与其他模糊测试器或测试工具一起使用**，而无需进行任何代码调整。 它不需要AFL指令的二进制文件即可工作。

请注意，只有在目标二进制文件是动态链接的情况下，AFL_PRELOAD方法（AFL在内部映射到LD_PRELOAD或DYLD_INSERT_LIBRARIES，取决于操作系统）才有效。 否则，尝试使用该库将无效。