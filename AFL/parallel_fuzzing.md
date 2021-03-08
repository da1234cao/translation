原文地址：:point_right: [AFL -- parallel_fuzzing.txt](https://github.com/google/AFL/blob/master/docs/parallel_fuzzing.txt)

> Tips for parallel fuzzing

该文档讨论在同一台机器或者不同机器上，同步afl-fuzz的模糊测试工作。

## 1. 介绍

afl-fuzz的每个副本将占用一个CPU内核。 这意味着在n核系统上，您几乎总是可以在n个并发的模糊作业中运行，而几乎不会对性能造成任何影响（您可以使用afl-gotcpu工具来确保）。

实际上，如果您仅依赖多核系统上的一项工作，那么您将无法充分利用硬件。 因此，并行化通常是正确的方法。

当针对多个不相关的二进制文件或以“哑”（-n）模式使用该工具时，只需启动几个完全独立的afl-fuzz实例就可以了。 当您想让多个模糊器锤击一个共同的目标时，情况就变得更加复杂：如果一个模糊器合成了一个有趣的测试用例，则其余实例将无法使用该输入来指导其工作。

为了解决这个问题，afl-fuzz提供了一种简单的方式来实时同步测试用例。

## 2. 在一个系统上并行

如果要在本地系统上的多个内核上并行执行单个作业，只需创建一个新的空输出目录（“ sync dir”），该目录将由afl-fuzz的所有实例共享； 然后为每个实例提出一个命名方案-例如“ fuzzer01”，“ fuzzer02”等。

像这样运行第一个（“ master”，-M）：

```shell
$ ./afl-fuzz -i testcase_dir -o sync_dir -M fuzzer01 [...other stuff...]
```

…然后，像这样启动secondary(-S)实例

```shell
$ ./afl-fuzz -i testcase_dir -o sync_dir -S fuzzer02 [...other stuff...]
$ ./afl-fuzz -i testcase_dir -o sync_dir -S fuzzer03 [...other stuff...]
```

每个fuzzer将把它的状态保存在一个单独的子目录中，就像这样` /path/to/sync_dir/fuzzer01/`

每个实例还将定期重新扫描顶级同步目录，以查找其他模糊测试人员发现的任何测试用例-并在它们被认为足够有趣时将其合并到自己的模糊测试中。

**-M和-S模式之间的区别在于，主实例仍将执行确定性检查**。 而辅助实例将直接进行随机调整。 如果您根本不想进行确定性的模糊测试，则可以使用-S运行所有实例。 对于非常慢或复杂的目标，或者在运行高度并行化的作业时，这通常是一个不错的计划。

请注意，尽管有实验支持并行化确定性检查，但运行多个-M实例非常浪费。 要利用它，您需要创建-M实例，如下所示：

```shell
$ ./afl-fuzz -i testcase_dir -o sync_dir -M masterA:1/3 [...]
$ ./afl-fuzz -i testcase_dir -o sync_dir -M masterB:2/3 [...]
$ ./afl-fuzz -i testcase_dir -o sync_dir -M masterC:3/3 [...]
```

...，其中“：”后的第一个值是特定主实例的顺序ID（从1开始），第二个值是用于分布确定性模糊的模糊器总数。 请注意，如果启动的模糊器数量少于传递给-M的第二个数字所指示的数量，则最终覆盖范围可能会很差。

您还可以使用提供的afl-whatsup工具从命令行监视作业进度。 当实例不再找到新路径时，可能是时候停止了。

警告：在显式指定-f选项时要格外小心。 每个模糊器都必须使用单独的临时文件； 否则，事情会往南走。 一个安全的例子可能是：

```shell
$ ./afl-fuzz [...] -S fuzzer10 -f file10.txt ./fuzzed/binary @@
$ ./afl-fuzz [...] -S fuzzer11 -f file11.txt ./fuzzed/binary @@
$ ./afl-fuzz [...] -S fuzzer12 -f file12.txt ./fuzzed/binary @@
```

This is not a concern if you use @@ without -f and let afl-fuzz come up with the file name.

## Multi-system parallelization

多系统并行化的基本操作原理与第2节中介绍的机制相似。主要区别在于，您需要编写一个简单的脚本来执行两个操作：

将SSH与authorized_keys结合使用以连接到每台计算机，并为该计算机本地的每个<fuzzer_id>检索/ path / to / sync_dir / <fuzzer_id> / queue /目录的tar存档。 最好使用在主机ID中包含主机名的命名方案，以便您可以执行以下操作：

```shell
    for s in {1..10}; do
      ssh user@host${s} "tar -czf - sync/host${s}_fuzzid*/[qf]*" >host${s}.tgz
    done
```

在所有剩余的机器上分发和解包这些文件，例如:

```shell
    for s in {1..10}; do
      for d in {1..10}; do
        test "$s" = "$d" && continue
        ssh user@host${d} 'tar -kxzf -' <host${s}.tgz
      done
    done
```

实验/ distributed_fuzzing /中有一个这样的脚本示例； 您还可以在以下位置找到由Martijn Bogaard开发的功能更强大的实验工具： https://github.com/MartijnB/disfuzz-afl；Richo Healey的另一个客户机-服务器实现是：https://github.com/richo/roving

请注意，这些第三方工具在运行Internet或不受信任的用户的系统上运行是不安全的。

在开发自定义测试用例同步代码时，请牢记一些优化

* 同步不必经常发生； 每30分钟左右运行一次任务可能很好。
* 无需同步崩溃/或挂起/； 您只需要通过queue / *复制（理想情况下还需要fuzzer_stats）。
* 不必（也不建议！）覆盖现有文件；  tar中的-k选项是避免这种情况的好方法。
* 无需为不在特定计算机上本地运行的Fuzzer提取目录，而只需在较早的运行期间将其复制到该系统上即可。
* 对于大型机队，您将需要为每个主机合并压缩包，因为这将使您使用n个SSH连接进行同步，而不是n *（n-1）。
* 您可能还希望实现分段同步。 例如，您可能有10组系统，第1组仅将测试用例推送到第2组；而第1组将测试用例推送到第2组。 第2组仅将他们推入第3组； 依此类推，最终第10组反馈到第1组。这种安排将使测试有趣的案例在整个机群中传播，而不必将每个模糊测试队列都复制到每个主机。
* 您不希望每个系统上都有afl-fuzz的“主”实例； 您应该使用-S来全部运行它们，而只需在车队中的某个位置指定一个进程来使用-M即可。

建议不要跳过同步脚本并直接在网络文件系统上运行模糊器。  I / O等待状态中的意外延迟和无法杀死的进程可能使事情变得混乱。

## 4. Remote monitoring and data collection

您可以使用**screen**，nohup，tmux或等效的工具来运行afl-fuzz的远程实例。 如果将程序的输出重定向到文件，它将自动从精美的UI切换到更受限的状态报告。 在输出目录中的fuzzer_stats文件中也始终写入有基本的机器可读信息。 在本地，可以使用afl-whatsup解释该信息。

原则上，您可以使用主实例（-M）的状态屏幕来监视总体模糊处理进度并决定何时停止。 在这种模式下，最重要的信号只是很长时间没有找到新路径。 如果您没有主实例，则只需选择任何一个辅助实例来进行观察即可。

您还可以依赖该实例的输出目录来收集综合语料库，该语料库涵盖了舰队中任何地方发现的所有值得注意的路径。 辅助（-S）实例仅需要确保已启动就不需要任何特殊监视。

请记住，崩溃输入不会自动传播到主实例，因此您可能仍想从同步脚本或运行状况检查脚本中监控整个队列的崩溃（请参阅afl-whatsup）。

## 5. Asymmetric setups

也许值得注意的是，以下一切都是允许的:

* **与其他可导工具一起运行afl-fuzz（例如，通过concilolic执行）。 第三方工具只需要遵循上述协议，即可从out_dir / <fuzzer_id> / queue / *中提取新的测试用例，并将自己的发现写入到out_dir / <ext_tool_id> / queue / *中顺序编号的id：nnnnnn文件中**。

* **使用不同的（但相关的）目标二进制文件运行一些同步的模糊器**。 例如，在共享发现的测试用例的同时，对几个不同的JPEG解析器（例如IJG jpeg和libjpeg-turbo）进行压力测试可以产生协同作用，并提高总体覆盖率。

  （在这种情况下，每个二进制文件运行一个-M实例是一个不错的计划。）

* 让某些个模糊测试器以不同的方式调用二进制文件。例如，“ djpeg”支持多种DCT模式，可通过命令行标志进行配置，而“ dwebp”则支持增量解码和单次解码。 在某些情况下，采用多种不同的模式然后合并测试用例将提高覆盖率。
  
* 更令人信服的是，使用不同的起始测试用例（e.g., progressive and standard JPEG）或字典来运行同步模糊器。同步机制可确保测试集随时间推移变得相当均匀，但会引入一些初始可变性。

[<font color=blue>译者注：Asymmetric setups很好</font>]