# Capabilities
## Capability
What's a capability
A capability is a unique, unforgeable token that gives the possessor permission to access an entity or object in system.

按照教程定义，可以理解为 capability 是访问系统中一切实体或对象的令牌，这个令牌包含了某些访问权力，只有持有这个令牌，才能按照特定的方式访问系统中的某个对象。教程中还提到了：

One way to think of a capability is as a pointer with access rights

可以将 capability 视为一个拥有访问权限的指针，而在C语言中，指针常常可以共享，因此个人理解这里隐含着一层 capability 可以传递和分享的意思。

seL4 中将 capability 分做三种：

内核对象（如 线程控制块）的访问控制的 capability。

抽象资源（如 IRQControl ）的访问控制的 capability 。（中断控制块是干啥的，以后会提到）

untyped capabilities ，负责内存分配的 capabilities 。seL4 不支持动态内存分配，在内核加载阶段将所有的空闲内存绑定到 untyped capabilities 上，后续的内核对象申请新的内存，都通过系统调用和 untyped caability 来申请。

在 seL4 内核初始化时，指向内核控制资源的所有的 capability 全部被授权给特殊的任务 root task ，用户代码向修改某个资源的状态，必须使用 kernel API 在指定的 capability 上请求操作。

关于 Root Task，有几个概念上的疑惑问答，参考：About Root Task in seL4

例如，root task 拥有控制它自己的进程控制块（TCB）的 capability ，这个 capability 被定义为一个常量 seL4_CapInitThreadTCB ，下面的一个例子就是通过这个 capability 来改变当前当前进程任务的栈顶指针（当需要一个很大的用户栈时，这个操作非常有用）：
## CNodes and CSlots
CNodes and CSlots
教程给出的定义：

A CNode (capability-node) is an object full of capabilities: you can think of a CNode as an array of capabilities.

简单来说， CNode 也是一个对象，但这个对象存的是一个数组，数组里的元素是 capability 。数组中的各个位置我们称为 CSlots ，在上面的例子中， seL4_CapInitThreadTCB 实际上就是一个 CSlot ，这个 CSlot 中存的是控制 TCB 的 capability 。

每个 Slot 有两种状态：

empty：没有存 capability。

full：存了 capability。

出于习惯，0号 Slot 一直为 empty。

一个 CNode 有 1 << CNodeSizeBits 个 Slot，一个 Slot 占 1 << seL4_SlotBits 字节。
## CSpace
CSpaces
教程给出的定义：

A CSpace (capability-space) is the full range of capabilities accessible to a thread, which may be formed of one or more CNodes.

值得注意的是， CSpace 是一个 线程 的能力空间，教程里只讨论了一个 CSpace 只有一个 CNode 的普遍情况。

# 内存
## Physical memory
除了一小部分静态的内核内存，seL4 所有的物理内存都在用户态被管理。在启动时由 seL4 创建的对象的 capability ，以及由 seL4 管理的其他的物理资源，在启动后都会传递给 root task。
## Untyped memory and capabilities
除了用于创建 root task 的对象之外，所有可用的物理内存的 capability 都作为 untyped memory 的 capability 传递给 root task。 Untyped memory 是一块特定大小的连续物理内存块，而 untyped capabilities 则是 untyped memory 的 capability。 untyped capabilities 可以被 retyped 。  
untyped capabilities 有一个 bool 的属性，指明对应的内存对象是否可以被内核写：内存对象可能不是 RAM 而是其他 Device，或者它可能位于内核无法寻址到的 RAM 区域。对于 Device untyped capabilities 只能 retyped 到 frame objects 中，不能被内核写入（？）。
seL4_BootInfo 对 root task 描述了所有的 untyped capabilities，包括他们的大小，是否是 device untyped，以及对应的物理地址。下面的代码打印出了初始的 untyped capabilities ：
## Retyping
只要untyped capability对应的memory大小允许，memory可以被拆分成许多新的object。使用seL4_Untyped_Retype函数对untyped capability进行创建新的capability（同时也是会对应的untyped object进行retype）。新的capability是原来untyped capability的children，children按顺序获得有效内存，且内存是对齐的。例如，创建4k object以后创建16k object，这样有12k就被浪费了。
seL4_Untyped_Retype

# Thread
## Thread Control Blocks
seL4 内核提供线程的抽象，用来管理任务的执行。线程在 seL4 中被表示为 Thread Control Block 对象（TCB）。

TCB 中包含了如下对象：

优先级和 maximum control priority。

寄存器状态和上下文。

CSpace capability。

VSpace capability。

endpoint capability 用于发送错误信息。

reply capability 。

## Scheduling model
在 seL4 内核中，调度器是一个基于优先级的轮询调度器，会选取最高优先级的线程在特定的处理器核心上执行。

## Priorities

调度器选取最高优先级的可运行线程进行调度，内部提供的优先级范围是 0～255 。255是被编码为 seL4_MaxPrio 的常量。

而 TCB 内部还有一个 maximum control priority （MCP），当设置优先级时，必须提供显示的 TCB 对应的 capability 来获取设置权限。如果设置的优先级大于 MCP，则设置操作失败。 root task 的 优先级和 MCP 都被设置为 seL4_MaxPrio 。

## Round robin

当有多个可运行的相同优先级的 TCB 时，调度器会采用 FIFO 的策略。而调度的时机则是使用时间片，每个 TCB 都有一个时间片字段，标识 TCB 有资格执行的周期数，而内核的计时器则会进行一个周期性的时钟中断来抢占时间片，当时间片用尽后则会撤销 TCB，调度其他有时间片的 TCB 。当然，线程也可以使用 seL4_Yield 系统调用来主动交出他们当前的时间片。

## Domain scheduling

In order to provide confidentiality seL4 provides a top-level hierarchical scheduler which provides static, cyclical scheduling of scheduling partitions known as domains.

意思大概是 seL4 提供了一个静态的分层调度器，每一层称为一个 domain （域），用于隔离不相关的子系统。调度域划分是在编译期就静态确定好的，而且是不可抢占的。线程可以绑定在一个域上，只有它们绑定的域被是激活状态时，线程才会被调度。跨域的 IPC 回等到域切换时才会被响应，而 seL4_Yied 是无法跨域的。每个调度域有一个空闲线程，当调度域没有可以运行的线程时，就会运行空闲线程。