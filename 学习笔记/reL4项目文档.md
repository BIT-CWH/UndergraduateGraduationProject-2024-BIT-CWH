# 结构
reL4的项目结构如下所示：

- boot：此模块包含内核启动所需要的函数和数据结构，如堆分配器、内存初始化器等。
- common：此模块包含内核的通用函数、错误处理函数以及用户态内核态交互有关的数据结构。
- cspace：此模块包含Capability机制的具体实现，包括capability的声明、类型和一系列权限控制方法等。
- interrupt：此模块包含中断的识别和处理方法。
- kernel：此模块包含内核代码的连接方法、内核陷入的处理和返回时所需要调用的方法。而在现有reL4中，微内核以同步的方式进行IPC，本模块也包含了具有特定特征消息通信可使用的fastpath通信方法，可以大大提升效率。
- syscall：此模块包含内核系统调用解码和处理所需的函数，以及获取系统调用参数和返回系统调用结果所需的设置寄存器和tcb等的辅助函数。
- task_manager：此模块包含了reL4的task相关内容。首先是reL4的IPC机制，如endpoint和notification对象以及相关通信方法的实现。其次是reL4的线程管理和调度内容，如tcb系列方法的定义以及线程调度器的实现。
- uint：此模块包含了用户态中断的具体实现，内含两个子模块：uintc和uintr。uintc包括了用户态中断的软件实现，包括用户态中断所需的数据结构定义以及发送端和接收端的注册流程。uintr实现了对下层硬件的封装，为用户态中断发送和接收提供了接口。
- vspace：此模块包含reL4的虚存管理系统的实现。
# 启动方式

# 基于用户态中断的Notification
## 注册
reL4下的通知机制允许一个线程通过一个内核对象的句柄（cap）向另一个线程非阻塞地发送信号。其注册过程在在用户态一共涉及到三个API：
- `seL4_Untyped_Retype`: 将一片未被使用的内存初始化为一个Notification内核对象。
- `seL4_CNode_Mint`: 从原始的notification cap中派生一个新的被赋予了标记的cap作为发送端标记（badged_ntfn_cap）。
- `seL4_TCB_BindNotification`: 将notification对象绑定到某个线程，只有绑定线程可以通过cap来接收这个notification对象的信号，调用`seL4_Uint_Notification_register_receiver`系统调用，来将TCB注册为接收端。
由于原始的通知机制发送端无需注册，只需要拿到对应的ntfn cap即可进行发送，因此我们将用户态中断的发送端注册后移到第一次发送中。

## 通信
发送：
- 我们在用户态的发送线程运行时中维护一个ntfn_cap的注册状态表，在 `seL4_Signal` 中将首先检查入参的cap是否注册了发送端，如果没有注册，则会调用 `seL4_Uint_Notification_register_sender` 注册发送端，然后将从内核态拿到的 `send id` 用于更新状态表，最后通过 `uipi_send(send_id)` 来发送信号。如果之前已经注册过了，则直接从状态表中拿到 `send id` 并调用 `uipi_send(send_id)` 。  

接收：  
- 当设置了中断处理函数时，我们不需要主动调用上面两个接口来询问信号。为了兼容性，我们对上述两个接口做了适配：
	-  `seL4_Wait(ntfn_cap)`: 由于用户态中断的相互独占的性质，入参将被忽略，调用 `uipi_read()` 来读取中断寄存器，如果没有信号，则调用 `yield_now().await` 切换当前协程。 
	-  `seL4_Poll(ntfn_cap)`：同样忽略入参，调用 `uipi_read()`，直接返回读取结果。

## 通信权限控制
在用户态中断改造之后的通知机制，发送端仍然可以维护 ntfn cap 到 send id 的映射来将ntfn cap看作发送端顶点，而接收端的顶点则必须是绑定了内核对象的TCB。
由于用户态中断的内核对象对接受线程的独占性，接收线程无法再通过不同的内核对象来区分不同的接收方，以至于接收线程在接收时不需要指定参数。当然我们用类似处理发送端的方法来在接收方的用户态做兼容性处理，但考虑到大多数情况下一个接收端通过一个内核对象和不同的badge来区分不同的发送方已经足够，因此我们这里没有进行过多的兼容性处理。