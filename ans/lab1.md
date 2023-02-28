## 思考题1

> 阅读 `_start` 函数的开头，尝试说明 ChCore 是如何让其中一个核首先进入初始化流程，并让其他核暂停执行的。

通过 `mrs x8, mpidr_el1` 读出mpidr_el1内容到寄存器 `x8` 中，用 `and x8, x8, #0xff` 取其低8位地址获得cpu核id， `cbz x8, primary` 则是如果核号为0，才会跳转至 `primary` 进行初始化。否则（其他核）运行 `b .` （跳转到当前地址重复运行）进入死循环

## 练习题 2

> 在 `arm64_elX_to_el1` 函数的 `LAB 1 TODO 1` 处填写一行汇编代码，获取 CPU 当前异常级别。

`mrs x9, CurrentEL`