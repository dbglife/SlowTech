---
title: 《系统开发》之开始OS设计
date: 2017-03-02 12:30:32
tags:
- 操作系统
- 翻译
categories:
- 内核
- 翻译
---

原文地址：[http://www.osdever.net/tutorials/view/os-design](http://www.osdever.net/tutorials/view/os-design)

这篇文章来自于一个老的论坛alt.so.development的文章。有'>'符号的部分的文章是Ian Nottingham写的。接下来的部分是有Alistair J. R. Yong编写。

> 你好，我已经尝试开始编写一个内核了，但是我所查看到的所有的例子都不是为编写内核最初的阶段准备的。我不能够确认我改从哪里开始，以及要做些什么...，谁有一个这样内核开发的早期阶段的例子吗？或许一个指导原则或一个需要做什么的TO-DO列表也好？

我想说你第一件需要考虑的东西是，没有一个你可以从它开始的完整的子系统。我发现，内核最有意思的一个事情是几乎一半的东西都依赖于其他的东西，所以你需要同时开发一点这几乎一半的子系统。

如果你是从0开始，第一个你开发的东西是一堆小宏或函数，以便将一个漂亮的接口放在X86混乱的位操作之上，易于今后的编程。如果你有雄心壮志，你最好尝试使得它可移植或至少半可移植的界面。（当遇到了这么多烦热的细节时，我更相信作弊会更好。如果你并不是非常热衷于编写大量的低级 PC硬件的汇编代码，我建议你获取Utah OsKit，刚刚发布了v0.96版本。所有这些位操作细节已经做好。）

你也需要大量的二进制函数用于内核调用（没有strcmp(),没有strcpy()，没有malloc()，等等），还有调试函数，包括assert(),panic()。

同时，你确实需要做出你的设计决定，否则你会遇到我曾经遇到过的类似情景，不得不分析清楚所有东西，并且重新做一遍，这是很痛苦的事情。

首要重大的东西是内存模型。段模式还是分段？现在，似乎大多数人会选择分段基础上的分页，主要是因为平坦模式的内存的整洁，以及分页通常是可移植的，而分段不是。
<!-- more -->
你要将你的内核放到哪里？你是否允许每一个进程都有它的独立空间，或兼具两者？虚拟内存？

然后，具备多任务或多线程？如果你确认要添加这些功能，或许在开始时最重要的是你的代码是否会在SMP机器上运行？如果是这样，那么你需要从一开始就加入旋转锁和其他的同步内容。从新回头加入这些内容将是一场噩梦。

你的处理器模型是什么？仅仅是基本的两级的进程拥有线程的模型（此处不准确），或是更加复杂的模型？（我使用的是一个三级的互动会话/任务/设备拥有进程，进而进程又有线程的模型。）当你做任务切换时，你会自己编写代码来完成它，或使用Intel提供的TSS来完成？如果是后者，你会使用完整的方式，为每一个线程配备一个TSS，或只有两个TSS，并且在交换前和交换后是将适当的信息写入到TSS中？

系统调用，你将如何实现系统调用？有几种可能的实现方式 —— Linux等，似乎是使用软件中断，因此这种方式也很流行。其他的选择，调用门可以让用户模式程序直接调用适当的内核函数。或者仅仅有一个调用向IPC端口发送消息，或其他类似功能，可以执行所有通信，甚至是通过IPC与kernel通信。

讲一下IPC，消息端口？命名管道？队列？本地sockets？所有上述的一切？最好是现在考虑一下它，因为它可能会影响到接下来的设计。你或许想要一个基本的功能，然后其他的功能都按照它来实现（例如在Laura，每一个特别的功能都是基于基础的通信端口设计，这会影响到TCP/IP栈的设计（网络是另外需要考虑的东西））

Ich. 设备驱动，你如何和他们通信？以及文件系统。这真的完全是一个混合的话题。简单来讲，像Unix模型一样，你可以将所有的东西当作文件，并且通过文件系统来访问他们。否则会有更复杂的问题等在哪里。再一次用我自己的内核作为例子，Laura模型有一个命名空间管理器，它会例举出系统中的每一个对象，设备，文件系统和他们的文件，等等。访问任何东西，设备，文件等，你需要请求命名空间管理器，然后它会给你返回一个IPC端口，用于对象访问。每一个注册的东西能够支持完全自定义的一组动作。而非是标准的Open(),close(),read(),write(),ioctl()...

安全！如果你想包括万物，那你需要考虑一下安全问题。它和SMP一样，返工添加将非常麻烦。如果你想要避免到处都是安全漏洞的问题。同样有很多不同的方法来实现它，我的内核使用了对象ACL，一句相关的线程，任务和会话，来验证可能的三种不同的令牌。

好啦，这就是所有的设计材料，但是这是最重要的内容了。我的意思是首先要解决掉这些问题。否则，你会花掉大量时间来分析和重构。

一旦你开始了真正的代码编写，好吧（除非你的工具为你提供了），否则首要的事情是一个粗鲁的控制台驱动。当然它不会成为你最终版本中使用的，但是没有printf()，而想调试所有的东西简直就是个混蛋。

然后，我想想啊。很多依赖于你的OS如何设计，才能决定你依赖于那些东西。第一个需要的东西是能够引导你的内核的代码（如果你使用多启动项的引导加载器，或其他类似的东西，这会让你感觉到不那么痛苦。使用类似的引导加载器开始，或许是一个好主意，即使你后面想要自己编写自己的引导加载器。），它会完成大部分的基础设置（完成段设置，分配一个内核堆栈等等）

然后，我的第一步是写了一些东西，显示了启动信息，并停机。（这样我就可以看到有一些事情发生了）。然后编译完，第一步就是重写上面的代码，然后显示信息，然后进入系统空闲循环（这个循环最终会成为空闲线程）。这显然要比“for(;;)”复杂地多 —— 因为我想最好是让CPU停机，避免浪费过多电力，但是...

我觉得接下来的事情可能处理陷阱比较好，尽管这一步你所能做的就是显示消息，输出状态，让内核进入错误模式（这个应该成为你的默认处理）。这些消息会被后面的调试处理掉（比简单的崩溃要更有用），并且当你已经实现了框架了，这也是你实现真正的处理例程的前瞻。

中断处理要实现东西，我想也将是下一步要实现的东西。我不会再一个未知中断上慌乱。

这些之后，就可以做一些独立于内核的事情了。下一步就是处理定时器中断的处理函数。仅仅是一些使用系统定时器完成的事情，因此它是十分必要的。同样的，我也需要定时器正常工作来完成工作调度，因此...

By Andy @ 2017/03/02 12:30:37 