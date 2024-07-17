## GDB使用

```shell
run:运行
stop:停止
finish:完成当前函数并给出返回值
test:测试
stepi:一步一步执行（pc）
q：退出

b 23:添加断点到某行
b *3291312321:添加断点到某位置
delete:删除断点

p *(int *)($rsp + 8):显示某个位置的值
x/5i $pc:显示接下来5条pc指令
x/30 0x4032d0:显示0x4032d0位置后的数据
layout asm:显示汇编代码方便阅读

disas fountion：反汇编某个函数

```

```shell
gdb-multiarch kernel/kernel

# (gdb) 进入gdb后执行
set confirm off
set architecture riscv:rv64
target remote localhost:31000
set riscv use-compressed-breakpoints yes

layout split

layout asm

layout src

```

