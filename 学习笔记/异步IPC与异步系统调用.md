# 异步IPC代码路径
## async_helper_thread(client)
1. 获取AsyncArgs并等待AsyncArgs中某些参数就绪
2. 生成recv_reply_coroutine
3. 将recv_reply_coroutine注册为用户态中断接收协程
4. 生成reply_notification
5. 从AsyncArgs中获取本线程tcb并将tcb绑定并注册为reply_notification的接收端
6. 将自己注册为req_notification的发送端，获取sender_id
7. 将sender_id和reply_notification设置到AsyncArgs中
8. 等待服务器就绪后开始通信
## async_ipc_test(server)
1. 生成req_notification
2. 生成recv_req_coroutine
3. 将recv_req_coroutine注册为用户态中断接收协程
4. 初始化本线程tcb并将tcb绑定为req_notification的接收端
5. 生成测试所用的client以及其tcb，并设置AsyncArgs有关参数
6. 等待client生成reply_notification并将自己绑定并注册为reply_notification的发送端,获取sender_id
7. 将sender_id设置到AsyncArgs中并设置服务器就绪标志
## 规律
notification都是在接收端生成的，便于与tcb绑定

# 异步系统调用代码路径
## 用户态
1. 生成reply_notification
2. 生成recv_reply_coroutine
3. 将recv_reply_coroutine注册为用户态中断接收协程
4. 获得new_buffer的capability并在reply_notification上注册异步系统调用
5. 生成syscall_test协程测试系统调用
## 内核态(async_syscall_handler)
1. 从syscall_buffer中取出系统调用条目
2. 处理后写回syscall_buffer
3. 若用户态接收协程不在运行则发送用户态中断通知唤醒
## 流程
1. 用户态注册异步系统调用
2. 用户态启动异步系统调用
3. 内核态在async_syscall_handler中处理系统调用

# 其他
## AsyncArgs(用于client与server共享部分数据结构)
1. req_ntfn：客户端向服务端发送信号，由服务端生成
2. reply_ntfn：服务端向客户端发送信号，由客户端生成
3. server_sender_id：服务端注册自己为reply_notification的发送端，通过此id向客户端发送用户态中断
4. client_sender_id：客户端注册自己为req_notification的发送端，通过此id向服务端发送用户态中断
5. child_tcb：客户端tcb，由服务端创建客户线程时生成
6. ipc_new_buffer：维护IPC请求和回复内容
7. server_ready：供客户端查询服务端是否准备完成

