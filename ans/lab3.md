# 修复了lab2启动页表设置中的bug，由于原来的高地址非0级页表被设为低地址的页表出错，但由于低地址页表事实上也能用所以未发现，本实验需访问仅高地址页表有映射的外设空间故发现错误，详情见git diff

## 第一部分：用户进程和线程

> 思考题 1: 内核从完成必要的初始化到用户态程序的过程是怎么样的？尝试描述一下调用关系。

在Linux中:
 - 内核完成初始化后, 创建并运行idle进程(`pid = 0`)
 - idle进程通过`kernel_thread`创建init进程(`pid = 1`), init进程在kernel态初始化后通过`execve`运行可执行文件init(`kernel_init`函数), init进程之后将完成设备驱动程序的初始化等工作
 - 之后init进程从内核态转入用户态, 使用`fork`产生其他用户态进程

在本lab对应的chcore中:
 - 内核完成初始化后执行`main`函数, `main`函数对内核做进一步初始化(如打开MMU和配置中断表)后调用`create_root_thread`创建第一个用户态线程, 之后使用`switch_context`将vmspace切换为该用户态线程的vmspace, 并使用`eret_to_thread`将控制权转交到该线程(主要是将sp设置为该用户态线程的`thread_ctx`的`arch_exec_cont_t`的地址, 之后通过`exception_exit`将存储在内存中的用户态线程各寄存器状态载入寄存器中)
