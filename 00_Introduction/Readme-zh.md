# 章节 0: 介绍

我决定做一个名为《手写编译器旅程》的项目。在这之前，我编写过[assemblers（汇编器）](https://github.com/DoctorWkt/pdp7-unix/blob/master/tools/as7)以及用一个无类型语言编写的 [simple compiler（简易编译器）](https://github.com/DoctorWkt/h-compiler)。但我从来没有写过一个能“自己编译”的编译器。所以这就是为什么我做出文章开头这个决定的原因。在这一整个过程当中，我会持续的把我的工作记录下来以便其他人能够一起学习。这么做也有助于我理清自己的思路和想法。

希望这些东西对你我都有用！



## 旅程的目标

以下是我在这段旅程中的所有目标：

 + 写一个能自编译的编译器，我认为如果一个编译器能够自我编译，那才是真正的编译器。
 + 要完成这个目标至少得在一个真实的硬件平台运行，我见过一些可以为假想的机器生成代码的编译器。我想要我的编译器可以运行在一个真实的硬件上。另外，如果可能的话，我的这个编译器是一个能支持不同后端并且能够运行在不同硬件平台上的编译器。
 + 实践在研究之前，整个编译器领域有大量的研究成果。我想要进行一场从零开始的旅程，所以我倾向于实践方式方法而不是着重于理论的方式方法。我虽然嘴上是这么说的，但是有时我还是需要引入一些基础理论的东西。
 + 遵循`KISS`原则：保持简单！我一定会使用Ken Thompson's原则：“当遇到疑惑，请用蛮力！”
 + 积跬步以至千里，我将这次旅程打碎成了许多小步骤以此来替代“步子迈大容易扯到跨”的情况。这将使得编译器中每个新书写的小部分都是“小小”的并且容易“被一口吃掉”（理解）的东西。

## 目标语言

选择一个旅程中的目标语言是一个十分困难的事情。如果选了类似于 Python，Go etc 等高级编程语言，我就不得不得去实现一大堆这个语言内置的类库。我可以用类似Lisp的等语言去写这个编译器呢又会很[容易做到（done easily）](ftp://publications.ai.mit.edu/ai-publications/pdf/AIM-039.pdf)。所以我选择从操旧业，我要用C语言的一个子集来写这个编译器而让其能够自己编译自己。C语言可以看作是汇编语言的升级版（这句话针对C语言的某些子集而不是指[C18](https://en.wikipedia.org/wiki/C18_(C_standard_revision))），并且利用这个角度有助于将C语言代码编译成汇编语言会容易一些。爽！所以我依旧喜欢C语言。



## 编译器的基本工作原理

编译器的工作就是将不同的输入语言（通常都是高级语言）转化为另一种语言（转换后的语言相对于输入语言来说等级得的多），他的主要工作步骤如下：

![](D:/Work/Env/Git/GitRepo/acwj/00_Introduction/Figs/parsing_steps.png)

 + [lexical analysis（词法分析）](https://en.wikipedia.org/wiki/Lexical_analysis)

   通过词法分析来确认词语元素，在一些语言中，`=` 与 `==` 不是一回事，所以你不能只读取 `=` ，我们称之为词语元素标记（lexical elements tokens）。

 + [Parse（解析）](https://en.wikipedia.org/wiki/Parsing) 

   对输入内容做解析, i.e. 确保输入的内容能够满足结构以及语法上的准确性。例如你的代码也许会有以下的分支决定结构：

```c
  if (x < 23) {
    print("x is smaller than 23\n");
  }
```

> 但是你在别的语言中可能会是这样写这个分支结构：

```python
  if (x < 23):
    print("x is smaller than 23\n")
```

> 比如你在语句结尾没有加分号，这就是编译器检查语法错误的地方（以上案例就是这样的情况示例）。

 +  [semantic analysis（语义分析）](https://en.wikipedia.org/wiki/Semantic_analysis_(compilers))

   对输入内容进行语义分析，i.e. 它本身并不明白你输入的东西是什么意思，这实际上不同于确认语法与结构。举个例子🌰，在英语中，一个句子的组成可以看作`<主语> <副词> <形容词> <宾语>`。以下两个句子拥有相同的结构但是他们表达的意思却完全不同：

```
  David ate lovely bananas.
  Jennifer hates green tomatoes.
```

 + [Translate（转换）](https://en.wikipedia.org/wiki/Code_generation_(compiler))

   将输入内容的意思翻译成另一个语言，在这个步骤我们将转换输出部分时间，the meaning of the input into a different language. Here we
   convert the input, parts at a time, into a lower-level language.

## Resources

There's a lot of compiler resources out on the Internet. Here are the ones
I'll be looking at.

### Learning Resources

If you want to start with some books, papers and tools on compilers,
I'd highly recommend this list:

  + [Curated list of awesome resources on Compilers, Interpreters and Runtimes](https://github.com/aalhour/awesome-compilers) by Ahmad Alhour

### Existing Compilers

While I'm going to build my own compiler, I plan on looking at other compilers
for ideas and probably also borrow some of their code. Here are the ones
I'm looking at:

  + [SubC](http://www.t3x.org/subc/) by Nils M Holm
  + [Swieros C Compiler](https://github.com/rswier/swieros/blob/master/root/bin/c.c) by Robert Swierczek
  + [fbcc](https://github.com/DoctorWkt/fbcc) by Fabrice Bellard
  + [tcc](https://bellard.org/tcc/), also by Fabrice Bellard and others
  + [catc](https://github.com/yui0/catc) by Yuichiro Nakada
  + [amacc](https://github.com/jserv/amacc) by Jim Huang
  + [Small C](https://en.wikipedia.org/wiki/Small-C) by Ron Cain,
    James E. Hendrix, derivatives by others

In particular, I'll be using a lot of the ideas, and some of the code,
from the SubC compiler.

## Setting Up the Development Environment

Assuming that you want to come along on this journey, here's what you'll
need. I'm going to use a Linux development environment, so download and
set up your favourite Linux system: I'm using Lubuntu 18.04.

I'm going to target two hardware platforms: Intel x86-64 and 32-bit ARM.
I'll use a PC running Lubuntu 18.04 as the Intel target, and a Raspberry
Pi running Raspbian as the ARM target.

On the Intel platform, we are going to need an existing C compiler.
So, install this package (I give the Ubuntu/Debian commands):

```
  $ sudo apt-get install build-essential
```

If there are any more tools required for a vanilla Linux
system, let me know.

Finally, clone a copy of this Github repository.

## The Next Step

In the next part of our compiler writing journey, we will start with
the code to scan our input file and find the *tokens* that are the
lexical elements of our language. [Next step](../01_Scanner/Readme.md)