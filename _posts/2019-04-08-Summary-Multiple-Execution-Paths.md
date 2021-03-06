---
layout: post
title:  "Summary-Multiple Execution Paths"
date:   2019-04-08 08:55:01 +0800
categories: 基于动态的恶意软件分析
tag: 论文调研
---
* content
{:toc}


# Exploring Multiple Execution Paths for Malware Analysis (SP'07)

---

## 问题

动态分析工具的问题是只能观察到单个程序的执行。然而，遗憾的是，某些恶意行为可能仅在特定情况下触发（例如，在某一特定日期，某个文件存在时或收到某个命令时）。

在本文中，我们提出了一个系统，它允许我们探索多个执行路径，并识别仅在满足某些条件时执行的恶意操作。这使我们能够自动提取正在分析的程序的更完整视图，并确定在何种情况下执行可疑行动。我们的实验结果表明，许多恶意软件样本根据从环境读取的输入显示不同的行为。因此，通过探索多个执行路径，我们可以获得更完整的行动图。

## 贡献

- 我们提出了一种动态分析技术，使我们能够创建有关恶意代码行为的综合报告。为此，我们的系统探索了多个程序路径，这些路径由程序处理的输入驱动。此外，我们的系统会报告触发特定操作的输入条件集。
- 我们开发了一种工具，通过在基于虚拟机的环境中执行它们来分析Microsoft Windows程序。我们的系统跟踪用户输入，并可在控制流决策点创建当前流程的快照。此外，我们可以将运行进程重新设置为先前存储的状态，并不断修改其内存，以便探索替代执行路径。
- 我们在大量现实恶意软件样本上评估了我们的系统，并证明我们能够识别在单路径执行跟踪中无法观察到的行为。

## 原理

为了能够探索多个程序路径，需要两个主要组件。 首先，我们需要一种机制来确定我们的系统应该何时分析两个程序路径。 为此，我们跟踪程序如何使用来自某些输入源的数据。 其次，当找到一个有趣的分支点时，我们需要一种机制来保存当前的程序状态，并在以后重新加载它以探索替代路径。

### Tracking Input (DTA)

* 污点源：主要是系统调用，它返回我们认为与恶意代码行为相关的信息。这包括访问文件系统的系统调用（例如，检查文件是否存在，读取文件内容），Windows注册表和网络。此外，返回当前时间或网络连接状态的系统调用很有趣。每当我们的程序调用相关函数（或系统调用）时，我们的系统会自动为接收此函数结果的每个内存位置分配一个新标签。有时，这意味着标记了一个整数。在其他情况下，例如，当程序从文件或网络读取时，标记完整的返回缓冲区，每个字节使用一个唯一标签。

* 逆映射：除了将内存位置映射到标签的影子内存之外，我们还需要逆映射。逆映射为每个标签存储当前保存该标签的所有存储器位置的地址。当进程重置为先前存储的状态并且必须重写某个输入变量时，需要此信息。原因是当修改具有特定标签的存储器位置时，必须同时更改具有相同标签的所有其他位置。否则，过程的状态变得不稳定。**（确保内存更新的一致性）**

* 线性依赖：初始输入值在被用作控制流决策中的参数之前被复制到新的存储器位置。在这种情况下，重写此参数意味着必须**使用相同的值更新共享相同标签的所有位置**。但是，到目前为止，我们还没有考虑过初始输入**不是简单复制**，而是在计算中用作操作数的情况。使用上面简述的直接污点传播机制，带有标记参数的操作的结果接收该参数的标签。当操作的结果具有与参数不同的值时，也会发生这种情况。不幸的是，这会在快照点重写变量时导致问题。特别是，**当不同的内存位置共享相同的标签但保持不同的值时，不能简单地用一个新的值覆盖这些内存位置。**

  **SOLUTION：**更新标签；新标签的值取决于旧标签的值，对于add操作， $label_{new}=label_{old}+increment$；目前只能模拟输入变量之间的线性关系，为了跟踪标签之间的线性相关性，必须扩展负责加法，减法和乘法的机器指令的污点传播机制。

* 非线性依赖：例如，按位操作符，不能一致更新状态且不能探索替代路径。维护N集合，跟踪所有属于非线性依赖关系的标签。无论何时应重写标签，都要确定所有相关标签。如果这些标签中的任何一个在N中，则不能一致地改变状态并且不能探索替代路径。

### Saving and Restoring Program State

监视使用一个（或两个）标记参数的条件操作的程序执行。当识别出这样的分支指令时，创建当前过程状态的快照。

* 当前执行状态的快照包含正在使用的完整虚拟地址空间的内容。另外，我们必须存储当前映射和约束系统。此外，必须确保考虑条件操作本身，即路径约束。当采用条件的if分支时（即，对于当前标记值，它的计算结果为true），条件直接用作路径约束。否则，当遵循else分支时，采取条件的否定。
* 当程序状态恢复时，我们系统的第一项任务是加载以前保存的程序地址空间内容，并用存储的内容覆盖当前值。然后，加载保存的约束系统。与采用第一个分支的情况类似，在跟踪替代分支时也需要添加适当的路径约束。为此，最初使用的路径约束是相反的（也就是说，其否定）。此新路径约束将添加到约束系统，并启动约束求解器。

## 优缺点

优：多路径探索提供更完整的视图。

缺：

* 不支持非线性依赖操作。
* DTA开销大，可能难以解决拖延代码。



## 启发

* 问题是回溯时的一致性更新问题，可以通过负载的约束求解器解决，例如SAT求解器可解决位运算依赖。`Save and Restore`的概念是蛮有借鉴意义的，不同于符号执行；而且可以在原型系统QEMU上执行。

