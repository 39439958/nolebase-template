### 前言

这部分代码我写了很久，倒不是没思路那种，是写出来了很难调试，我后面没接difftest了，因为感觉涉及nemu的硬件部分比较少，所以不想花时间折腾了，在4.2的讲义末尾也说了4.2是难度最大的一部分，那下面我来梳理一下4.2要做什么以及怎么做



![image-20240505232113340](C:\Users\xuliz\AppData\Roaming\Typora\typora-user-images\image-20240505232113340.png)

### 思路分析

4.2的内容其实很好理解，就是实现一个分页的虚拟内存的机制，学过os的同学应该很熟悉这个工作机制了，具体任务呢如下：

1.硬件模拟层面的实现：`isa_mmu_check`和`isa_mmu_translate`

2.AM层中VME的实现：`map()`

3.Nanos-lite中loader的分页加载、AM中`__am_irq_handle()`和`ucontext()`的调整

4.Nanos-lite中`mm_brk()`的实现

接下来我一个一个讲。



#### 1.nemu中实现分页机制

讲义中提出了两个问题：

> 如何判断CPU当前是否处于分页模式?

查看手册后可以发现根据 satp 寄存器的MODE位即可判断，所以我们在`isa-def.h`中将 satp 寄存器的定义添加上，并且将其中的`isa_mmu_check`宏函数修改一下。

```c
typedef struct {
  word_t mtvec;
  vaddr_t mepc;
  word_t mstatus;
  word_t mcause;
  word_t satp;
} CSRS;

typedef struct {
  word_t gpr[MUXDEF(CONFIG_RVE, 16, 32)];
  vaddr_t pc;
  CSRS csrs;
} MUXDEF(CONFIG_RV64, riscv64_CPU_state, riscv32_CPU_state);
```

```c
#define isa_mmu_check(vaddr, len, type) (cpu.csrs.satp >> 31)
```

![image-20240505233737522](C:\Users\xuliz\AppData\Roaming\Typora\typora-user-images\image-20240505233737522.png)

> 分页地址转换的具体过程应该如何实现?

```cpp
#define PT1_ID(addr) (addr >> 22)
#define PT2_ID(addr) ((addr & 0x3fffff) >> 12)
#define OFFSET(addr) (addr & 0xfff)
paddr_t isa_mmu_translate(vaddr_t vaddr, int len, int type) {
  // L1
  paddr_t *pt_1 = (paddr_t *)guest_to_host((paddr_t)(cpu.csrs.satp << 12));
  assert(pt_1 != NULL);
  // L2
  word_t *pt_2 = (word_t *)guest_to_host(pt_1[PT1_ID(vaddr)]);
  assert(pt_2 != NULL);
  // paddr
  paddr_t paddr = (paddr_t)((pt_2[PT2_ID(vaddr)] & (~0xfff)) | OFFSET(vaddr));
  return paddr;
}
```

还需要修改一些东西：

- 指令读取系统寄存器时，将satp寄存器加上，它的编号在手册中为0x180
- vaddr.c中使用`isa_mmu_check`和`isa_mmu_translate`



#### 2.AM层中VME的实现

