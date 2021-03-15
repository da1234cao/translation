原文地址：:point_right: [AFL -- llvm_mode](https://github.com/google/AFL/tree/master/llvm_mode)

```
Fast LLVM-based instrumentation for afl-fuzz
```

[<font color=blue>译者注：afl-gcc和afl-clang可以开箱即用，非常方便。但是afl-clang-fast同样如此，且这三个有点非常非常诱人。</font>]

## 1. 介绍

此目录中的代码使您可以使用真正的编译器级 instrumentation 来检测AFL程序，而不是afl-gcc和afl-clang所采用的更粗糙的汇编级重写方法。 这具有几个有趣的属性：

1. 编译器可以进行很多优化，这些优化在手动插入程序集时很难实现。 结果，某些受CPU约束的速度较慢的程序将以大约2倍的速度运行。

   快速二进制文件的收益并不那么明显，因为二进制文件的速度主要受到创建新进程的成本的限制。 在这种情况下，增益可能会保持在10％以内。

2.  instrumentation 是独立于CPU的。 至少原则上，您应该能够依靠它来在非x86体系结构上对程序进行模糊处理（在使用AFL_NO_X86 = 1构建afl-fuzz之后）。

3. 该instrumentation 可以更好地处理多线程目标。
4. 由于该功能依赖于LLVM的内部，因此它是Clang特定的，并且*不适用于* GCC。

**一旦证明此实现足够健壮和可移植，它可能会取代afl-clang**。 目前，它可以单独构建并与原始代码共存。

这个想法和大部分实现都来自Laszlo Szekeres。

## 2. 使用

为了利用此机制，您需要在系统上安装clang。 您还应该确保**llvm-config**工具在路径中（或在环境中通过LLVM_CONFIG指向）。

不幸的是，有些带有clang的系统没有llvm-config或LLVM开发标头。  FreeBSD就是一个例子。  FreeBSD用户还会遇到静态构建clang以及无法加载模块的问题（加载afl-llvm-pass.so时，您会看到“服务不可用”）。

要解决所有问题，您可以从以下位置获取操作系统的**预构建二进制文件**：  http://llvm.org/releases/download.html

编译C++代码的时候，确保还包括将CXX设置为afl-clang-fast++。

...然后在编译功能并稍后构建软件包时，将压缩包中的bin/目录放在$PATH的开头。 您不需要为此root。

**要构建工具本身，请键入“ make”。 这将在父目录中生成称为afl-clang-fast和afl-clang-fast ++的二进制文件**。 完成此操作后，您可以采用类似于AFL的标准操作模式的方式来检测第三方代码，例如：

```shell
# Be sure to also include CXX set to afl-clang-fast++ for C++ code.
CC=/path/to/afl/afl-clang-fast ./configure [...options...]
make
```

该工具使用与afl-gcc大致相同的环境变量（请参阅../docs/env_variables.txt）。 这包括AFL_INST_RATIO，AFL_USE_ASAN，AFL_HARDEN和AFL_DONT_OPTIMIZE。

注意：如果要为所有用户在系统上安装LLVM帮助器，则需要在父目录中发出“ make install”之前对其进行构建。

## 3. Gotchas, feedback, bugs

这是一种早期机制，所以欢迎现场报告。您可以将错误报告发送到<afl-users@googlegroups.com>。

## 4. 优点特性一：deferred instrumentation

**AFL试图通过仅执行一次目标二进制文件，仅在main()之前停止目标二进制文件，然后克隆此“主”进程以稳定地提供目标来进行模糊处理，来优化性能。**

Although this approach eliminates much of the OS-, linker- and libc-level costs of executing the program，但它并不总是对执行其他耗时的初始化步骤的二进制文件有所帮助-例如，解析大型配置文件数据。

在这种情况下，一旦大部分初始化工作已经完成，但是在二进制文件尝试读取模糊输入并解析之前，稍后将forkserver初始化是有益的。 在某些情况下，这可以提供10倍以上的性能提升。 您可以以相当简单的方式在LLVM模式下实现延迟初始化。

**首先，在代码中找到可以进行延迟克隆的合适位置**。 这需要格外小心，以免破坏二进制文件。 特别是，如果您在以下情况下选择一个位置，则该程序可能会出现故障：

1. 任何重要线程或子进程的创建-因为forkserver无法轻松克隆它们。
2. 通过setitimer()或等效调用初始化计时器。
3. 临时文件，网络套接字，对偏移量敏感的文件描述符以及类似的共享状态资源的创建-但前提是它们的状态在以后有意义地影响程序的行为。
4. 对模糊输入的任何访问，包括读取有关其大小的元数据。

**选定位置后，将此代码添加到适当的位置**。

```shell
#ifdef __AFL_HAVE_MANUAL_CONTROL
  __AFL_INIT();
#endif
```

您不需要#ifdef防护，但包括它们可以确保使用afl-clang-fast以外的工具进行编译时，程序将继续正常运行。

最后，使用afl-clang-fast重新编译程序（afl-gcc或afl-clang *不会*生成延迟初始化二进制文件）-您应该一切就绪！

## 5. 优点特性二：persistent mode

**一些库提供的API是无状态的，或者可以在处理不同的输入文件之间重置其状态。 当执行这样的重置时，可以重复使用一个长期存在的过程来测试多个测试用例，从而消除了对重复的fork()调用和相关的OS开销的需求**。

该程序的基本结构是：

```shell
  while (__AFL_LOOP(1000)) {

    /* Read input data. */
    /* Call library code to be fuzzed. */
    /* Reset state. */

  }

  /* Exit normally */
```

循环内指定的数值控制AFL从头重新启动过程之前的最大迭代次数。 这样可以最大程度地减少内存泄漏和类似故障的影响；  1000是一个很好的起点，而更高的值会需要占有更多的cpu，而不会给您带来任何实际的性能优势。

../experimental/persistent_demo/中显示了更详细的模板。与以前的模式类似，此功能仅适用于afl-clang-fast； 使用其他编译器时，可以使用#ifdef防护禁止它。

请注意，与以前的模式一样，该功能易于滥用。 如果您没有完全重置临界状态，则可能会导致误报或浪费大量CPU能力，而无济于事。 特别注意内存泄漏和文件描述符的状态。

**PS：由于仍然涉及任务切换，因此该模式的速度不如LLVM的LibFuzzer提供的“纯”进程内模糊检测那么快。 但是它比普通的fork()模型快很多，并且与进程内模糊测试相比，它应该更健壮**。

## 6. 优点特性三： new 'trace-pc-guard' mode

**LLVM的最新版本附带了内置的执行跟踪功能，该功能可以为AFL提供必要的跟踪数据，而无需对程序集进行后处理或安装任何编译器插件**。 看：  http://clang.llvm.org/docs/SanitizerCoverage.html#tracing-pcs-with-guards

```shell
# If you have a sufficiently recent compiler and want to give it a try, build afl-clang-fast this way:
# 这是个新特性，编译的时候查看当前版本是否支持
AFL_TRACE_PC=1 make clean all
```

请注意，此模式目前比“vanilla” afl-clang-fast慢20％，比afl-clang慢5-10％。 这可能是因为instrumentation 没有进行inlined，而是直接进行函数调用。 在支持它的系统上，使用-flto(链接时候优化)编译目标应该会有所帮助。

