---
layout: post
title:   "angr.concolic"
date:   2019-07-05 09:55:01 +0800
categories: 基于动态的恶意软件分析
tag: 符号执行

---

* content
{:toc}




# angr-concolic testing

这篇文章目的是利用angr的符号执行引擎，执行有趣路径，并求出路径条件。

`angr.surveyors.Explorer`类可以完成目标。

* 初始状态，初始地址，即API调用地址`addr_call`
* 目标地址`addr_des`
* 过滤地址`addr_pass`

### SM方法

利用模拟管理器simulation_manager（旧版本的path_group）求解。

```python
>>> initial_state = p.factory.blank_state(addr=addr_call)
>>> sm = b.factory.simulation_manager(initial_state)
>>> sm.explore(find=addr_des, avoid=addr_pass)
>>> sm.found[0].state.se._solver.result.model

>>> sm.found[0].state.posix.dump(0) #程序所有输入
>>> sm.found[1].state.posix.dump(0) #程序所有输出
```

simulation_manager初始化运行之后，有以下几种状态

* `step()`: 向下执行一个block，`step()`函数产生active状态，表示该分支在执行中。
* `run()`: 表示运行到结束，`run()`函数产生deadended状态，表示分支结束。存储所有路径状态。
* `explore()`: 对地址进行限制以减少执行遍历的路径。只存储found状态。

`result`,`files`方法已经没有了，

```python
>>> sm.found[0].state.se._solver.result.model # se,之前的slover
AttributeError: 'SolverComposite' object has no attribute 'result'
>>> inputdata = s.posix.files[0].all_bytes()
AttributeError: 'SimSystemPosix' object has no attribute 'files'
```



### surveyer方法

```python
>>> initial_state = p.factory.blank_state(addr=addr_call, 
                                      remove_options={simuvex.o.LAZY_SOLVES})
>>> ex = angr.surveyors.Explorer(p, start=initial_path, find=(addr_des,), avoid=(addr_pass,), enable_veritesting=True)

>>> concolic_run = ex.run()
>>> concolic_run
Out: <Explorer with paths: 1 active, 3 spilled, 1 deadended, 0 errored, 0 unconstrained, 1 found, 1 avoided, 0 deviating, 0 looping, 0 lost>
```

从`found[].state`查找路径条件信息。

```python
>>> final_state = concolic_run.found[0].state

>>> final_state.se.any_str(final_state.memory.load(0x402159, PW_LEN))
```



下面是参考的[博文](http://0x0atang.github.io/reversing/2015/09/17/flareon2-concolic.html)翻译。

## Smashing Flare-on #2 with Concolic Testing

将concolic执行应用于解决`CTFs`和`crackmes`的想法并不新鲜。 [Cr4sh](http://blog.cr4.sh/2015/03/automated-algebraic-cryptanalysis-with.html)用OpenREIL和Z3解决了代数加密`crackme`问题。 [Felipe](https://feliam.wordpress.com/2010/10/07/the-symbolic-maze/)和[Artem](http://blog.trailofbits.com/2014/12/04/close-encounters-with-symbolic-execution-part-2/)演示了使用KLEE（和McSema）符号性地解决迷宫问题。 [Michele](http://doar-e.github.io/blog/2015/08/18/keygenning-with-klee/)最近创建了一个keygen工具，用于打破使用KLEE的程序的串行验证过程。

`CTFs`和`crackmes`有一个共同的问题解决模式：你需要给程序什么输入才能让它做某事。 例如，我们需要为`crackme`提供什么参数才能显示`成功！` 而不是`失败！`？ 或者输入`crackme`什么秘钥才能将格式正确的图像输出到磁盘？ 通常，我们希望知道哪些输入可以调用程序的成功执行路径，同时避免失败执行路径。

这种模式使得特别适合使用`concolic testing`来有效地解决它。 这通常涉及以下组合：

* IR提升：将程序二进制文件转换为无副作用的中间表示，用于程序分析。 IR的例子是`BAP BIL`，`OpenREIL`，`Valgrind VEX`和`LLVM IR`。
* 符号执行：求解对达到程序部分所需的输入的约束，并帮助进行路径探索。 符号执行引擎的示例可以在`KLEE`，`S2E`和`Triton`中找到。
* 约束求解：确定到达程序所需部分的可行输入范围。 约束求解器的示例是`Z3`和`STP`。
* 污点分析：跟踪数据和控制流以确定（部分）用户可控输入的约束。

原作者在2015年开始了Flare-On，目的是尝试应用复杂的执行来解决挑战。所以需要一个框架/工具，可以直接处理核心问题，理想情况下尽可能无忧无虑。在探索了一些工具之后，这里有一些想法。

首先，我喜欢OpenREIL的简单性和友好的Python绑定。但是，它没有一个完整的符号执行引擎。与Cr4sh一样，我们可以利用OpenREIL的内部IR仿真引擎pyopenreil.VM来实现Z3的简单符号执行引擎。但是，根据CTF二进制文件中使用的操作，我们可能需要使用操作扩展引擎以处理更复杂的符号执行，例如使用符号条件进行分支探索。

接下来，基于PIN的concolic测试框架，Triton有一个很好的污点分析引擎，可以跟踪每个程序点用户可控的内存或寄存器。与OpenREIL一样，它提供了用户友好的Python界面。但就目前而言，**它仅适用于64位程序**。在Triton中扩展对32位指令的支持非常有用。

我们还可以使用McSema将x86程序提升到LLVM IR，并使用KLEE符号性地执行LLVM bitcode中的程序。但是KLEE使用Linux二进制文件更好（仅限linux？），并且需要开发一个额外的KLEE线束程序来驱动符号执行。此外，**当前的开源KLEE适用于整个程序**，并且**缺乏在程序中较小的函数子集**上进行符号执行的能力。最近扩展了KLEE，UC-KLEE可以限制对所选函数的执行，但它尚不可用。

最后，我偶然发现了最近在Blackhat 2015上发布的angr。这是由UCSB的安全研究人员开发的优秀二进制分析工具，并且已被Shellphish广泛用于DARPA网络大挑战赛。

凭借其强大的引擎，angr为用户提供了制作脚本的能力，以解决CTF挑战中的以下典型目标，例如Flare-On中的那些：

```plain
What range of inputs is required to reach EIP_SUCCESS and avoid EIP_FAILURE?
达到EIP_SUCCESS需要的输入的范围
```

所以，angr就成了这个CTF的首选。

我将描述使用angr来有效地执行路径探索，使用concolic执行来解决＃2而无需手动逆向工程。

### 识别有趣代码

挑战＃2，very_success（[二进制文件](http://0x0atang.github.io/files/20150917/very_success)）是一个要求输入密码的Windows二进制文件。 如果验证密码不正确，则打印出`You are failure`; 如果密码正确，则打印`You are success`。

反汇编相对简单。 函数`sub_401084`显然需要一些用户输入，执行验证并返回一个结果，该结果确定程序是否打印“成功”。 目标是输入一个密码，允许我们将分支设置为`0x40106B`（“成功”）并避免分支在`0x401072`（“失败”）。

让我们放大并找到有关函数`sub_401084`的更多信息。 它需要三个参数，即

（1）地址到`0x4010E4`的字节缓冲区，这是用于验证的参考密钥，

（2）到`0x402159`的用户输入缓冲区的地址，以及

（3）提供给程序的用户密码的长度。

基本块（高亮显示为青色，这里应有图）可能执行用于基于参考密钥验证用户密码的主要计算。

通常在这一点上，我会开始手动反转函数`sub_401084`，或者如果我很懒，就在这个函数上运行Hex Ray反编译器。

### 无手工逆向解决#2

今年，我尝试使用angr象符号地解决这个问题。 换句话说，我想知道什么用户密码可以让我到达`0x40106B`的成功执行路径。 假设密码是有效的电子邮件地址，我们可以选择使用python脚本强制密码，该脚本遍历电子邮件地址的有效字符以增加密码长度，并在程序执行到达所需代码时提醒我们地址。 但这效率低下。

在这种情况下，Concoic执行在我们的路径探索中可以很好地和优雅地工作，以达到我们想要的代码分支。

我们首先在Python中初始化angr环境，然后加载挑战二进制文件。

```python
In [1]: import angr

In [2]: p = angr.Project('very_success')
```

我们将分析本地化为特定功能`sub_401084`。 在初始化引擎以在初始代码地址`0x401084`处开始执行concoic之后，我们需要相应地设置此函数的参数。 我们从`0x401093`的指令知道该函数正在寻找长度为0x25字节的密码。 我们对此进行硬编码，以及堆栈上各种参数的地址偏移量。

```python
In [3]: initial_state = p.factory.blank_state(addr=0x401084)

In [4]: initial_state.stack_push(0x25)                  # arg_8__len_password

In [5]: initial_state.stack_push(0x402159)              # arg_4__user_password

In [6]: initial_state.stack_push(0x4010e4)              # arg_0__ref_key

In [7]: initial_state.stack_push(0x401064)              # return addr
```

为了引导concolic执行，我们需要将包含用户输入密码的缓冲区符号性地表示为由`8 * 0x25`位组成的BitVector表达式。 我们使用state.BV类来做到这一点。

```python
In [8]: PW_LEN = 0x25

In [9]: initial_state.mem[0x402159:] = initial_state.BV('password', 8 * PW_LEN)
```

更进一步，因为密码只包含可以打印的字符，所以应该添加这些限制到初始条件。在路径探索时，这些限制条件会引导符号执行。

```python
In [10]: for i in xrange(PW_LEN):
   ....:     char = initial_state.memory.load(0x402159 + i, 1)
   ....:     initial_state.add_constraints(char >= 0x21, char <= 0x7e)
```

在配置约束和初始状态的情况下，我们距离concolic执行只有一步之遥。 我们使用初始状态初始化路径。 要开始使用concolic测试进行路径探索，我们依靠`angr.surveyors.Explorer`类。 我们指定要达到的所需代码地址以及我们要避免的代码地址。 请注意，`concolic_run`的显示输出表示在路径探索期间，一条路径（“1找到”）满足了我们的要求。

```python
In [11]: initial_path = p.factory.path(state=initial_state)

In [12]: ex = angr.surveyors.Explorer(p, start=initial_path, find=(0x40106b,), avoid=(0x401072,), enable_veritesting=True)

In [13]: concolic_run = ex.run()

In [14]: concolic_run
Out[14]: <Explorer with paths: 1 active, 3 spilled, 1 deadended, 0 errored, 0 unconstrained, 1 found, 1 avoided, 0 deviating, 0 looping, 0 lost>
```

最后，从`found.state`提取密码。

```python
In [15]: final_state = concolic_run.found[0].state

In [16]: final_state.se.any_str(final_state.memory.load(0x402159, PW_LEN))
Out[16]: 'a_Little_b1t_harder_plez@flare-on.com'
```

### 优化

**关于angr API和各种类的文档缺乏**。 因此，上面的说明可以通过github站点上的现有文档和一些示例脚本找到。 顺便说一句，在我努力拼凑出各种模块来解决这个挑战之后，一个解决挑战＃2的示例angr脚本随后被添加到github repo中。

事实证明，我们可以在创建`blank_state`时删除选项`simuvex.o.LAZY_SOLVES`。 这通过确保我们不遍历不可用的路径来使路径探索更有效。

此外，代替我使用的`angr.surveyors.Explorer`，示例angr脚本使用PathGroup类。 它似乎是一种更优雅，更好的方法来管理探索的路径。