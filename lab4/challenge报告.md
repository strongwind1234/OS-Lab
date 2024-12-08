#challenge实验报告

local_intr_save函数其实是__intr_save函数，
首先检查当前的状态寄存器（sstatus）中的 SSTATUS_SIE 位。
如果 SSTATUS_SIE 位被设置（即中断是允许的），那么函数会调用 intr_disable函数 来禁用中断，并将 SSTATUS_SIE 位清除。
函数返回一个标志（1或0），表示在调用之前中断是否被允许。这个标志用于后续恢复中断状态。

local_intr_restore这个函数接受一个标志参数(local_intr_save)，如果这个标志为真（即 __intr_save 返回的值），则调用 intr_enable 来重新启用中断。