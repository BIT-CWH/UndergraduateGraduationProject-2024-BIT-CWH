# 同步IPC
总耗时：10825088  

## 代码路径
初始化-调用测试函数-生成辅助线程-测试  
### 主线程：
申请ipcbuffer-申请endpoint-申请创建辅助线程-循环接受并回复（不断调用reply_recv）  
reply_recv调用:trap（切换栈-保存寄存器-切换内核页表-分发中断至fastpath_reply_recv）-fastpath_reply_recv（合法性检查-消息收发-切换进程（切换页表-设置进程）-保存内容）
### 辅助线程：
获取ipcbuffer-获取endpoint-循环发送接收(不断调用call)  
call调用：trap（切换栈-保存寄存器-切换内核页表-分发中断至fastpath_call）-fastpath_call（合法性检查-消息收发-切换进程（切换页表-设置进程）-保存内容）
## 汇编部分时间读取设计方案
位置：trap.S  
获取：在被测部分前后调用rdtime指令读取时间至寄存器，然后相减得到时间，并将结果写入某个寄存器  
读取并输出：通过rust内嵌汇编函数读取寄存器的内容，并用debug宏输出
