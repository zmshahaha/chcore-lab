## 思考题1

> 阅读 `_start` 函数的开头，尝试说明 ChCore 是如何让其中一个核首先进入初始化流程，并让其他核暂停执行的。

通过 `mrs x8, mpidr_el1` 读出mpidr_el1内容到寄存器 `x8` 中，用 `and x8, x8, #0xff` 取其低8位地址获得cpu核id， `cbz x8, primary` 则是如果核号为0，才会跳转至 `primary` 进行初始化。否则（其他核）运行 `b .` （跳转到当前地址重复运行）进入死循环

## 练习题 2

> 在 `arm64_elX_to_el1` 函数的 `LAB 1 TODO 1` 处填写一行汇编代码，获取 CPU 当前异常级别。

`mrs x9, CurrentEL`

## 练习题 3

> 在 `arm64_elX_to_el1` 函数的 `LAB 1 TODO 2` 处填写大约 4 行汇编代码，设置从 EL3 跳转到 EL1 所需的 `elr_el3` 和 `spsr_el3` 寄存器值。具体地，我们需要在跳转到 EL1 时暂时屏蔽所有中断、并使用内核栈（`sp_el1` 寄存器指定的栈指针）。

模仿124-127行得到
```
adr x9, .Ltarget
msr elr_el3, x9
mov x9, SPSR_ELX_DAIF | SPSR_ELX_EL1H
msr spsr_el3, x9
```
elr_el3是控制异常返回（即eret？）后执行地址，SPSR_ELX_DAIF | SPSR_ELX_EL1H见代码新增注释