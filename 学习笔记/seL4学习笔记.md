# Capabilities
## Capability
Capability可以被理解为访问内核对象的令牌，这个令牌包含了内核对象的位置和访问权限。只能通过Capability按照特定方式访问内核对象。因此Capability 视为一个拥有访问权限的指针，便于共享。

seL4 中将 Capability 分做三种：

- 内核对象（如 线程控制块）的访问控制的 Capability。

- 抽象资源（如 IRQControl ）的访问控制的 Capability 。

- Untyped Capabilities ，负责内存分配的 capabilities 。seL4 不支持动态内存分配，在内核加载阶段将所有的空闲内存绑定到 untyped capabilities 上，后续的内核对象申请新的内存，都通过系统调用和 untyped caability 来申请。

在 seL4 内核初始化时，指向内核控制资源的所有的 capability 全部被授权给特殊的任务 root task（用户态进程） ，用户代码向修改某个资源的状态，必须使用 kernel API 在指定的 capability 上请求操作。
## CNodes and CSlots
CNode 是一个内核对象，指向一个数组，数组里的元素是 Capability 。数组中的各个位置称为 CSlots，例如seL4_CapInitThreadTCB 就是一个 CSlot ，这个 CSlot 中存的是控制 TCB 的 Capability 。每个 Slot 有Empty和Full两种状态，出于习惯，0号 Slot 一直为 empty。

一个 CNode 有 1 << CNodeSizeBits 个 Slot，一个 Slot 占 1 << seL4_SlotBits 字节。
## CSpace
CSpace代表线程的Capability空间，包含进程的所有Capability，由一个或多个CNode组成。普遍情况下CSpace 只有一个 CNode。

# 内存
## 物理内存
除了一小部分静态的内核内存，seL4 所有的物理内存都在用户态被管理。在启动时由 seL4 创建的对象的 Capability ，以及由 seL4 管理的其他的物理资源，在启动后都会传递给 root task。
## Untyped memory and capabilities
除了用于创建 root task 的对象之外，所有可用的物理内存的 Capability 都作为 Untyped Memory 的 Capability 传递给 root task。 Untyped Memory 是一块特定大小的连续物理内存块，而 Untyped Capabilities 则是 Untyped Memory 的 Capability。 Untyped Capabilities 可以被 Retyped 。  
Untyped Capabilities 有一个 bool 的属性，指明对应的内存对象是否可以被内核写：内存对象可能不是 RAM 而是其他 Device，或者它可能位于内核无法寻址到的 RAM 区域。  
seL4_BootInfo 对 root task 描述了所有的 untyped capabilities，包括他们的大小，是否是 device untyped，以及对应的物理地址。
## Retyping
只要Untyped Capability对应的Memory大小允许，Memory可以被拆分成许多新的内核对象。使用seL4_Untyped_Retype函数对Untyped Capability进行创建新的Capability（同时也是会对应的Untyped Object进行Retype）。新的Capability是原来Untyped Capability的children，children按顺序获得有效内存，且内存是对齐的。例如，创建4k object以后创建16k object，这样有12k就被浪费了。
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