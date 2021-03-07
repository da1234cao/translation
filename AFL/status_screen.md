原文地址：:point_right: [AFL -- status_screen.txt](https://github.com/google/AFL/blob/master/docs/status_screen.txt)

`Understanding the status screen`

本文档提供了状态屏幕的概述，以及有关对任何警告和UI中显示的红色文本进行故障排除的提示。 有关一般说明手册，请参见自述文件。

##  0. A note about colors

状态屏幕和错误消息使用颜色使内容保持可读性，并引起您对最重要细节的注意。 例如，红色几乎总是表示“consult this doc” :-)

不幸的是，只有在您的终端使用传统的un * x调色板（黑色背景上的白色文本）或接近该调色板的情况下，UI才能正确呈现。

如果您使用的是inverse video，则可能需要更改设置，例如：

* 对于GNOME终端，请转到“编辑”>“配置文件”首选项，选择“颜色”选项卡，然后从内置方案列表中选择“white on black”。
* 对于MacOS X Terminal应用程序，通过“外壳程序”>“新建窗口”菜单使用“专业”方案打开一个新窗口（或将“专业”设置为默认设置）。

或者，如果您确实喜欢当前的颜色，则可以编辑config.h以注释掉USE_COLORS，然后执行“make clean all”。

我不知道有任何其他简单的方法使这工作不引起其他副作用-抱歉。(>_<)

顺便说一句，让我们谈谈屏幕上的实际内容...

## 1. Process timing

|    run time     | 0 days, 8 hrs, 32 min, 43 sec |
| :-------------: | :---------------------------: |
|  last new path  | 0 days, 0 hrs, 6 min, 40 sec  |
| last uniq crash |         none seen yet         |
| last uniq hang  | 0 days, 1 hrs, 24 min, 32 sec |

本部分的内容很不言自明：它告诉您模糊器已经运行了多长时间，以及自从最近一次发现以来已经花费了多少时间。 这分为“路径”（触发新执行模式的测试用例的简写），崩溃和挂起。

**在计时方面：没有硬性规定，但是大多数测试工作要持续数天或数周。 实际上，对于一个中等复杂的项目，第一阶段可能需要一天左右的时间。 时不时地，一些工作将被允许运行数月**。

**有一件重要的事情要提防：如果该工具在启动后的几分钟内没有找到新路径，则可能是您没有正确调用目标二进制文件，并且它永远也无法解析我们要扔给它的输入文件； 另一个可能的解释是，默认内存限制（-m）过于严格，并且程序未能尽早分配缓冲区后退出。 或者输入文件显然是无效的，始终无法通过基本标头检查。**

如果一段时间内没有新路径出现，您最终也会在本节中看到一个红色的大警告：-)

## 2. Overall results

| cycles done  |  0   |
| :----------: | :--: |
| total paths  | 2095 |
| uniq crashes |  0   |
|  uniq hangs  |  19  |

本节的第一个字段为您提供到目前为止已完成的队列通过次数-即模糊器遍历到目前为止发现的所有有趣测试用例，对其进行模糊处理并循环回到最开始的次数。每个模糊测试会话都应至少完成一个周期。 理想情况下，运行时间应该比这更长

如前所述，第一次通过可能需要一天或更长的时间，因此请坐下来放松。 如果您想立即获得更广泛但更浅的覆盖范围，请尝试使用-d选项-通过跳过确定性的模糊化步骤，它可以为您提供更熟悉的体验。 但是，它在一些微妙的方面不如标准模式。

为了帮助您在何时按下Ctrl-C，cycle counter使用颜色编码。它在第一次通过时以洋红色显示，如果在随后的回合中仍在发现新发现，则变为黄色，然后在结束时变为蓝色-最终，在较长时间没有看到任何动作后，变成绿色。

屏幕的此部分中的其余字段应该非常明显：到目前为止，已经发现了许多测试用例（“路径”）以及独特故障的数量。 如自述文件中所述，可以通过浏览输出目录来实时探索测试用例，崩溃和挂起。

## 3. Cycle progress

| now processing  | 1296 (61.86%) |
| :-------------: | :-----------: |
| paths timed out |   0 (0.00%)   |

这个框告诉您fuzzer与当前队列周期的距离，它显示了当前正在处理的测试用例的ID，以及它决定丢弃的输入数量，因为它们持续超时。

有时在第一行中显示的“ *”后缀表示当前处理的路径不是“偏爱”的（稍后在第6节中讨论的属性）。

如果您认为模糊器的进度太慢，请参阅本文档第2节中有关-d选项的注释。

## 4. Map coverage

|  map density   | 10.15% / 29.07% |
| :------------: | :-------------: |
| count coverage | 4.03 bits/tuple |

本节提供了一些有关目标二进制文件中嵌入的instrumentation所观察到的覆盖范围的琐事。

框中的第一行告诉您，我们已经命中了多少个分支元组，与位图可以容纳的比例成比例。 左边的数字描述了当前输入； 右边的是整个输入语料库的值。

注意下极端情况：

* 低于200的绝对数字（？？）表明以下三点之一：该程序非常简单； 它没有被正确instrumentation（例如，由于与目标库的非instrumentation副本链接而导致）; 或者它在您的输入测试用例上过时了。 模糊器将尝试将其标记为粉红色，以提醒您注意。

* 过度使用模板生成的代码的非常复杂的程序可能很少会发生超过70％的百分比。

  因为高的位图密度使模糊器更难可靠地识别新程序状态，所以我建议使用AFL_INST_RATIO = 10左右重新编译二进制文件，然后重试（请参阅env_variables.txt）。

  模糊器将以红色标记高百分比。 很有可能，除非您测试extremely hairy software（例如v8，perl和ffmpeg），否则您永远不会看到它。

另一行处理二进制文件中元组命中计数的变化。 本质上，如果对于我们尝试的所有输入，每个采用的分支始终采用固定的次数，则该读数将为“ 1.00”。 当我们设法触发每个分支的其他命中计数时，指针将开始向“ 8.00”移动（8位映射命中的每个位），但可能永远不会达到极限。

在一起使用时，这些值可用于比较依赖同一instrumentation二进制文件的几个不同的模糊测试作业的覆盖范围。

[<font color=blue>译者注：不大明白map coverage</font>]

## 5. Stage progress

| now trying  |    interest 32/8    |
| :---------: | :-----------------: |
| stage execs | 3996/34.4k (11.62%) |
| total execs |        27.4M        |
| exec speed  |      891.7/sec      |

这部分让你深入了解模糊器现在正在做什么。它告诉你当前的阶段，可以是:

* calibration：在预模糊阶段，在该阶段检查执行路径以检测异常，确定基线执行速度等。 只要有新发现，就会非常简短地执行。
* trim L/S：另一个模糊测试前阶段，将测试用例修剪为最短的形式，仍然产生相同的执行路径。 长度（L）和步长（S）通常根据文件大小来选择。
* bitflip L/S：确定性位翻转。 在任何给定时间切换L位，使输入文件以S位递增。 当前的L / S变体为：1 / 1、2 / 1、4 / 1、8 / 8、16 / 8、32 / 8。
* arith L/8：确定性算法。模糊器尝试将小整数减去或加上8-、16-和32位值。切换总是8位
* interest L/8：确定的值覆盖。fuzzer有一个已知的“有趣的”8-、16-和32-位值的列表来尝试。切换是8位。
* extras：确定性注入字典术语。 可以将其显示为“用户”或“自动”，这取决于模糊器是使用用户提供的词典（-x）还是自动创建的词典。 您还会看到“ over”或“ insert”，具体取决于字典单词是覆盖现有数据还是通过偏移剩余数据以适应其长度来插入。
* havoc：一种固定长度的周期，具有堆叠的随机调整。 在此阶段尝试的操作包括位翻转，使用随机和“有趣的”整数进行覆盖，块删除，块复制以及各种与字典相关的操作（如果首先提供字典）。
* splice：最后一个策略，该策略在第一个完整队列周期后开始，没有新路径。 它等效于“破坏”，除了它首先在任意选择的中点将来自队列的两个随机输入拼接在一起。
* sync：仅在设置-M或-S时使用的阶段（请参见parallel_fuzzing.txt）。不涉及真正的模糊测试，但是该工具会扫描其他模糊测试器的输出，并在必要时导入测试用例。 第一次执行此操作可能需要几分钟左右。

其余字段应该是不言而喻的：存在当前阶段的exec计数进度指示器，全局exec计数器以及当前程序执行速度的基准。 这可能会从一个测试用例到另一个测试用例波动，但是理想情况下，**基准在大多数情况下应该超过500 execs / sec-如果它保持在100以下，则该工作可能会花费很长时间**。

模糊测试器还将明确警告您有关慢速目标的信息。 如果发生这种情况，请参阅模糊器随附的perf_tips.txt文件，以获取有关如何加快速度的想法。

## 6. Findings in depth

| favored paths |  879 (41.96%)  |
| :-----------: | :------------: |
| new edges on  |  423 (20.19%)  |
| total crashes |  0 (0 unique)  |
| total tmouts  | 24 (19 unique) |

这为您提供了几个完成度的指标。本节包括基于代码中包含的最小化算法（该方法将获得更多的播放时间），模糊器最喜欢的路径数，以及实际上导致更好的边缘覆盖率的测试用例的数量（与仅推入测试路径相比）。 分支命中计数器）。 还有其他更详细的崩溃和超时计数器。

请注意，超时计数器与挂起计数器有所不同。 这包括超过超时的所有测试用例，即使它们没有超出超时的程度也足以将其分类为挂起。

## 7. Fuzzing strategy yields

|  bit flips  | 57/289k, 18/289k, 18/288k  |
| :---------: | :------------------------: |
| byte flips  | 0/36.2k, 4/35.7k, 7/34.6k  |
| arithmetics | 53/2.54M, 0/537k, 0/55.2k  |
| known ints  | 8/322k, 12/1.32M, 10/1.70M |
| dictionary  |    9/52k, 1/53k, 1/24k     |
|    havoc    |      1903/20.0M, 0/0       |
|    trim     |    20.31%/9201, 17.05%     |

这只是另一个nerd-targeted的部分，跟踪我们在前面讨论的每一种模糊策略中，网出了多少条路径，与执行人员尝试的数量成比例。这有助于令人信服地验证关于afl-fuzz所采取的各种方法的有效性的假设。

本节中的trim strategy stats 统计信息与其余统计信息略有不同。该行的第一个数字显示从输入文件中删除的字节数的比率； 第二个对应于实现此目标所需的执行人员数量。 最后，第三个数字表示虽然无法删除但被认为没有效果并且被从某些更昂贵的确定性模糊处理步骤中排除的字节所占的比例。

## Path geometry

|  levels   |    5    |
| :-------: | :-----: |
|  pending  |  1570   |
| pend fav  |   583   |
| own finds |    0    |
| imported  |    0    |
| stability | 100.00% |

本节中的第一个字段跟踪通过引导的模糊过程达到的路径深度。 本质上：用户提供的初始测试用例被认为是“ 1级”。 通过传统的模糊测试可以从中得出的测试用例被认为是“ 2级”。 通过将它们用作后续模糊测试回合的输入而得出的结果为“第3级”； 等等。因此，最大深度可以大致代表您从afl-fuzz采取的仪器指导方法中获得的价值。

下一个字段显示尚未经过任何模糊测试的输入数量。 对于模糊器真正希望在此队列周期中到达的“最喜欢”条目，也提供了相同的统计信息（非最喜欢条目可能必须等待几个周期才能获得机会）。

接下来，在并行化模糊测试中，我们获得了在此模糊测试部分中发现并从其他模糊测试器实例导入的新路径的数量； 以及相同输入的出现程度有时会在测试的二进制文件中产生可变的行为。

最后一点实际上很有趣：它可以测量观察到的迹线的一致性。 如果程序对于相同的输入数据始终表现相同，则它将获得100％的分数。 当该值较低但仍显示为紫色时，模糊过程不太可能受到负面影响。 如果变成红色，则可能会遇到麻烦，因为AFL难以区分调整输入文件的有意义和“幻像”效果。

现在，大多数目标只会得到100％的分数，**但是当您看到较低的数字时，有几件事情要注意**：

* 在测试的二进制文件中，结合使用未初始化的内存和某些内在的熵源。 对AFL无害，但可能表示存在安全漏洞。
* 尝试操纵持久资源，例如遗留在临时文件或共享内存对象中。 通常，这是无害的，但是您可能需要仔细检查以确保程序不会过早退出。磁盘空间，SHM句柄或其他全局资源用完也会触发此情况。
* 达到某些实际上旨在随机表现的功能。通常无害。 例如，在对sqlite进行模糊处理时，会输入“ select random（）;”之类的输入 将触发变量执行路径。
* 多个线程以半随机顺序一次执行。 当“稳定性”指标保持在90％左右时，这是无害的，但如果不是这样，则可能成为问题。 这是尝试的方法：
* 使用llvm_mode /中的afl-clang-fast-它使用线程本地跟踪模型，该模型不太容易出现并发问题，
* 查看目标是否可以在没有线程的情况下进行编译或运行。 常见的./configure选项包括--without-threads，-disable-pthreads或--disable-openmp。
* 用GNU Pth（https://www.gnu.org/software/pth/）替换pthread，这使您可以使用**确定性调度程序**。
* 在持久模式下，“稳定”度量标准的细微下降可能是正常的，因为并非所有代码在重新输入时都表现出相同的行为。 但是主要的下降可能表示__AFL_LOOP（）中的代码在后续迭代中行为不正确（例如，由于状态的不完全清理或重新初始化），并且大部分的测试工作都浪费了。
* 检测到变量行为的路径在<out_dir> /queue/.state/variable_behavior/目录中用匹配的条目标记，因此您可以轻松地查找它们。

## 9. CPU load

[cpu: 25%]

这个小部件显示了本地系统上的明显CPU利用率。 通过获取处于“可运行”状态的进程数，然后将其与系统上的逻辑核心数进行比较来计算它。

如果该值显示为绿色，则说明您使用的CPU内核数量少于系统上可用的CPU内核数量，并且可能可以并行使用以提高性能。 有关如何执行此操作的提示，请参见parallel_fuzzing.txt。

如果该值显示为红色，则说明CPU *可能*被超额预订，并且运行其他模糊测试可能不会给您带来任何好处。

当然，此基准非常简单； 它告诉您准备运行多少个进程，而不告诉您它们可能需要多少资源。 它还没有区分物理内核，逻辑内核和虚拟CPU。 这些中的每一个的性能特征都会有很大的不同。

如果要更精确的测量，可以从命令行运行afl-gotcpu实用程序。

## Addendum: status and plot files

对于无人值守的操作，一些关键状态屏幕信息也可以在输出目录的fuzzer_stats文件中以机器可读的格式找到。 这包括：

  - start_time     - unix time indicating the start time of afl-fuzz
  - last_update    - unix time corresponding to the last update of this file
  - fuzzer_pid     - PID of the fuzzer process
  - cycles_done    - queue cycles completed so far
  - execs_done     - number of execve() calls attempted
  - execs_per_sec  - current number of execs per second
  - paths_total    - total number of entries in the queue
  - paths_found    - number of entries discovered through local fuzzing
  - paths_imported - number of entries imported from other instances
  - max_depth      - number of levels in the generated data set
  - cur_path       - currently processed entry number
  - pending_favs   - number of favored entries still waiting to be fuzzed
  - pending_total  - number of all entries waiting to be fuzzed
  - stability      - percentage of bitmap bytes that behave consistently
  - variable_paths - number of test cases showing variable behavior
  - unique_crashes - number of unique crashes recorded
  - unique_hangs   - number of unique hangs encountered
  - command_line   - full command line used for the fuzzing session
  - slowest_exec_ms- real time of the slowest execution in ms
  - peak_rss_mb    - max rss usage reached during fuzzing in mb

其中大多数直接映射到前面讨论的UI元素。

最重要的是，您还可以找到一个名为“ plot_data”的条目，其中包含大多数这些字段的绘图表历史记录。 如果您安装了gnuplot，则可以**使用随附的“ afl-plot”工具将其转换为不错的进度报告**。