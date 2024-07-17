# PA3.1

在这一节中，我们通过`yield test`这个测试触发自陷操作，来梳理整个过程并在其中实现异常响应的机制。

### 设置异常入口地址

首先，`yield test`会调用`cte_init()`，这个函数会设置异常处理的入口地址，即把`mtvec`寄存器的值设置成`__am_asm_trap`,然后注册一个事件处理回调函数，这个回调函数由`yield test`提供。

### 触发自陷操作

从`cte_init()`函数返回后, `yield test`将会调用测试主体函数`hello_intr()`, 首先输出一些信息, 然后通过`io_read(AM_INPUT_CONFIG)`启动输入设备, 不过在NEMU中, 这一启动并无实质性操作. 接下来`hello_intr()`将通过`iset(1)`打开中断, 不过我们目前还没有实现中断相关的功能, 因此同样可以忽略这部分的代码. 最后`hello_intr()`将进入测试主循环: 代码将不断调用`yield()`进行自陷操作, 为了防止调用频率过高导致输出过快, 测试主循环中还添加了一个空循环用于空转。

现在我们看`yield()`中发生了什么。

```c
void yield() {
#ifdef __riscv_e
  asm volatile("li a5, -1; ecall");
#else
  asm volatile("li a7, 11; ecall"); 
#endif
}
```

这里原本是-1，我改成了11，不改的话,后面`mcause`的`difftest`无法通过，我们的机器应该是默认一直运行在`M-mode`模式下。

![image-20231224163949056](C:\Users\xuliz\AppData\Roaming\Typora\typora-user-images\image-20231224163949056.png)

然后，我们需要的是实现一下`ecall`指令。

首先，实现一些`nemu`中暂时没有的系统寄存器(SR)，分别是`mtvec`,`mepc`,`mstatus`,`mcause`。

`cpu state`在`isa-def.h`，在其中添加即可。

```c
typedef struct {
  word_t mtvec;
  word_t mepc;
  word_t mstatus;
  word_t mcause;
} CSRS;

typedef struct {
  word_t gpr[MUXDEF(CONFIG_RVE, 16, 32)];
  vaddr_t pc;
  CSRS csrs;
} MUXDEF(CONFIG_RV64, riscv64_CPU_state, riscv32_CPU_state);
```

然后，实现一下`isa_raise_intr()`函数，关于mstatus的设置，查阅手册即可。

![image-20231224164406440](C:\Users\xuliz\AppData\Roaming\Typora\typora-user-images\image-20231224164406440.png)

```c
word_t isa_raise_intr(word_t NO, vaddr_t epc) {
  // mstatus set
  cpu.csrs.mstatus &= ~(1<<7);
  cpu.csrs.mstatus |= ((cpu.csrs.mstatus&(1<<3))<<4);
  cpu.csrs.mstatus &= ~(1<<3);
  cpu.csrs.mstatus |= ((1<<11)+(1<<12));

  // store pc in mepc
  cpu.csrs.mepc = epc; 
  // set err number in mcause
  cpu.csrs.mcause = NO;
  // get the address of the interrupt/exception vector and set pc to it
  return cpu.csrs.mtvec;
}

```

然后，根据提示实现指令，分别是`ecall`,`csrrw`,`csrrs`

> 你需要在自陷指令的实现中调用`isa_raise_intr()`, 而不要把异常响应机制的代码放在自陷指令的helper函数中实现, 因为在后面我们会再次用到`isa_raise_intr()`函数.

```c
INSTPAT("??????? ????? ????? 010 ????? 11100 11", csrrs  , I, R(rd) = CSR(imm); CSR(imm) |= src1);
INSTPAT("??????? ????? ????? 001 ????? 11100 11", csrrw  , I, R(rd) = CSR(imm); CSR(imm) = src1);
INSTPAT("0000000 00000 00000 000 00000 11100 11", ecall  , N, ECALL(s->dnpc));
```

实现一下上面用到的两个宏，分别是CSR，ECALL。

```c
static word_t *csr_reg(word_t imm) {
  switch (imm) {
    case 0x300 :  return &(cpu.csrs.mstatus);
    case 0x305 :  return &(cpu.csrs.mtvec);
    case 0x341 :  return &(cpu.csrs.mepc);
    case 0x342 :  return &(cpu.csrs.mcause);
    default : Log("csr error");
  }
  return NULL;
}

#define CSR(i) *csr_reg(i)
#define ECALL(dnpc) { bool success; dnpc = (isa_raise_intr(isa_reg_str2val("a7", &success), s->pc)); }
```

运行`yield test`，可以发现程序可以跳入设置的异常入口地址`__am_asm_trap`。

接下来实现`difftest`。

> 针对riscv32, 你需要将`mstatus`初始化为`0x1800`.

在isa初始化工作中加入即可。

```c
// init.c
static void restart() {
  /* Set the initial program counter. */
  cpu.pc = RESET_VECTOR;

  /* The zero register is always 0. */
  cpu.gpr[0] = 0;

  /* initialize mstatus */
  cpu.csrs.mstatus = 0x00001800;
}
```

这里我们还需要修改`spike-diff`的内容，分别是`diff_context_t`结构体的内容、`diff_get_regs`和`diff_set_regs`函数。

```cpp
struct diff_context_t {
  word_t gpr[MUXDEF(CONFIG_RVE, 16, 32)];
  word_t pc;
  word_t mtvec;
  vaddr_t mepc;
  word_t mstatus;
  word_t mcause;
};

void sim_t::diff_get_regs(void* diff_context) {
  struct diff_context_t* ctx = (struct diff_context_t*)diff_context;
  for (int i = 0; i < NR_GPR; i++) {
    ctx->gpr[i] = state->XPR[i];
  }
  ctx->pc = state->pc;
  // CSR
  ctx->mtvec = state->mtvec->read();
  ctx->mepc = state->mepc->read();
  ctx->mstatus = state->mstatus->read();
  ctx->mcause = state->mcause->read();
}

void sim_t::diff_set_regs(void* diff_context) {
  struct diff_context_t* ctx = (struct diff_context_t*)diff_context;
  for (int i = 0; i < NR_GPR; i++) {
    state->XPR.write(i, (sword_t)ctx->gpr[i]);
  }
  state->pc = ctx->pc;
  // CSR
  state->mtvec->write(ctx->mtvec);
  state->mepc->write(ctx->mepc);
  state->mstatus->write(ctx->mstatus);
  state->mcause->write(ctx->mcause);
}
```

最后还需要修改一下`isa_difftest_checkregs`

```c
bool isa_difftest_checkregs(CPU_state *ref_r, vaddr_t pc) {
    int reg_num = ARRLEN(cpu.gpr);
    for (int i = 0; i < reg_num; i++) {
        if (ref_r->gpr[i] != cpu.gpr[i]) {
            printf("reg[%d] is different! ref: 0x%08x, current: 0x%08x\n", i, ref_r->gpr[i], cpu.gpr[i]);
            return false;
        }
    }
    if (ref_r->pc != cpu.pc) {
        printf("pc is different! ref: 0x%08x, current: 0x%08x\n", ref_r->pc, cpu.pc);
        return false;
    }
    if (ref_r->csrs.mstatus != cpu.csrs.mstatus) {
        printf("mstatus is different! ref: 0x%08x, current: 0x%08x\n", ref_r->csrs.mstatus, cpu.csrs.mstatus);
        return false;
    }
    if (ref_r->csrs.mcause != cpu.csrs.mcause) {
        printf("mcause is different! ref: 0x%08x, current: 0x%08x\n", ref_r->csrs.mcause, cpu.csrs.mcause);
        return false;
    }
    if (ref_r->csrs.mepc != cpu.csrs.mepc) {
        printf("mepc is different! ref: 0x%08x, current: 0x%08x\n", ref_r->csrs.mepc, cpu.csrs.mepc);
        return false;
    }
    if (ref_r->csrs.mtvec != cpu.csrs.mtvec) {
        printf("mtvec is different! ref: 0x%08x, current: 0x%08x\n", ref_r->csrs.mtvec, cpu.csrs.mtvec);
        return false;
    }
    return true;
}
```

### 保存上下文

接下来程序的执行流应该到了`__am_asm_trap`这个汇编函数，它的任务是保存上下文，跳转至异常处理分发函数`__am_irq_handle`并在其中跳转至异常处理函数`user_handler`，恢复上下文，跳转至发生异常指令或下一条（视情况而定）。

我们先看保存上下文，我们需要做的事只有重新组织`abstract-machine/am/include/arch/$ISA-nemu.h` 中定义的`Context`结构体的成员, 使得这些成员的定义顺序和 `abstract-machine/am/src/$ISA/nemu/trap.S`中构造的上下文保持一致。

先给出组织好的代码。

```c
struct Context {
  // TODO: fix the order of these members to match trap.S
  uintptr_t gpr[NR_REGS], mcause, mstatus, mepc;
  void *pdir;
};

#ifdef __riscv_e
#define GPR1 gpr[15] // a5
#else
#define GPR1 gpr[17] // a7
#endif

#define GPR2 gpr[10] // a0
#define GPR3 gpr[11] // a1
#define GPR4 gpr[12] // a2
#define GPRx gpr[10] // a0
```

接下来我们预处理一下汇编文件，看看它干了什么。

```asm
# 0 "trap.S"
# 0 "<built-in>"
# 0 "<command-line>"
# 1 "/usr/riscv64-linux-gnu/include/stdc-predef.h" 1 3
# 0 "<command-line>" 2
# 1 "trap.S"
# 40 "trap.S"
.align 3
.globl __am_asm_trap
__am_asm_trap:
	addi sp, sp, -((32 + 3 + 1) * 8)
  	
    # 保存寄存器至栈上
    sd x1, (1 * 8)(sp)
    sd x3, (3 * 8)(sp)
    sd x4, (4 * 8)(sp)
    sd x5, (5 * 8)(sp)
    sd x6, (6 * 8)(sp)
    sd x7, (7 * 8)(sp)
    sd x8, (8 * 8)(sp)
    sd x9, (9 * 8)(sp)
    sd x10, (10 * 8)(sp)
    sd x11, (11 * 8)(sp)
    sd x12, (12 * 8)(sp)
    sd x13, (13 * 8)(sp)
    sd x14, (14 * 8)(sp)
    sd x15, (15 * 8)(sp)
    sd x16, (16 * 8)(sp)
    sd x17, (17 * 8)(sp)
    sd x18, (18 * 8)(sp)
    sd x19, (19 * 8)(sp)
    sd x20, (20 * 8)(sp)
    sd x21, (21 * 8)(sp)
    sd x22, (22 * 8)(sp)
    sd x23, (23 * 8)(sp)
    sd x24, (24 * 8)(sp)
    sd x25, (25 * 8)(sp)
    sd x26, (26 * 8)(sp)
    sd x27, (27 * 8)(sp)
    sd x28, (28 * 8)(sp)
    sd x29, (29 * 8)(sp)
    sd x30, (30 * 8)(sp)
    sd x31, (31 * 8)(sp)

	# 取出CSR并保存至栈上
    csrr t0, mcause
    csrr t1, mstatus
    csrr t2, mepc
    sd t0, ((32 + 0) * 8)(sp)
    sd t1, ((32 + 1) * 8)(sp)
    sd t2, ((32 + 2) * 8)(sp)

    # set mstatus.MPRV to pass difftest
    li a0, (1 << 17)
    or t1, t1, a0
    csrw mstatus, t1

	# a0用于传递__am_irq_handle需要的参数，也就是Context *c,也就是刚刚保存在栈上的上下文
    mv a0, sp
    jal __am_irq_handle

	# 恢复寄存器和CSR
    ld t1, ((32 + 1) * 8)(sp)
    ld t2, ((32 + 2) * 8)(sp)
    csrw mstatus, t1
    csrw mepc, t2

    ld x1, (1 * 8)(sp)
    ld x3, (3 * 8)(sp)
    ld x4, (4 * 8)(sp)
    ld x5, (5 * 8)(sp)
    ld x6, (6 * 8)(sp)
    ld x7, (7 * 8)(sp)
    ld x8, (8 * 8)(sp)
    ld x9, (9 * 8)(sp)
    ld x10, (10 * 8)(sp)
    ld x11, (11 * 8)(sp)
    ld x12, (12 * 8)(sp)
    ld x13, (13 * 8)(sp)
    ld x14, (14 * 8)(sp)
    ld x15, (15 * 8)(sp)
    ld x16, (16 * 8)(sp)
    ld x17, (17 * 8)(sp)
    ld x18, (18 * 8)(sp)
    ld x19, (19 * 8)(sp)
    ld x20, (20 * 8)(sp)
    ld x21, (21 * 8)(sp)
    ld x22, (22 * 8)(sp)
    ld x23, (23 * 8)(sp)
    ld x24, (24 * 8)(sp)
    ld x25, (25 * 8)(sp)
    ld x26, (26 * 8)(sp)
    ld x27, (27 * 8)(sp)
    ld x28, (28 * 8)(sp)
    ld x29, (29 * 8)(sp)
    ld x30, (30 * 8)(sp)
    ld x31, (31 * 8)(sp)
    
	# 跳转回发生异常指令或下一条
    addi sp, sp, ((32 + 3 + 1) * 8)
    mret

```

### 事件分发

修改一下`__am_irq_handle`函数。

```c
Context* __am_irq_handle(Context *c) {
  if (user_handler) {
    Event ev = {0};
    switch (c->mcause) {
      case 0: ev.event = EVENT_SYSCALL; break;
      case 11: ev.event = EVENT_YIELD; break;
      default: ev.event = EVENT_ERROR; break;
    }
    c = user_handler(ev, c);
    assert(c != NULL);
  }
  return c;
}
```

### 恢复上下文

具体过程在上面的汇编代码注释中已经说明了，现在需要在`nemu`中实现一下`mret`指令。

```c
#define MRET() { \
  s->dnpc = CSR(0x341); \
  cpu.csrs.mstatus &= ~(1<<3); \
  cpu.csrs.mstatus |= ((cpu.csrs.mstatus&(1<<7))>>4); \
  cpu.csrs.mstatus |= (1<<7); \
  cpu.csrs.mstatus &= ~((1<<11)+(1<<12)); \
}
```

> 对于mips32的`syscall`和riscv32的`ecall`, 保存的是自陷指令的PC, 因此软件需要在适当的地方对保存的PC加上4, 使得将来返回到自陷指令的下一条指令.

根据上述提示，我们可以在`simple_trap()`中实现mepc+4的操作。

```c
Context *simple_trap(Event ev, Context *ctx) {
  switch(ev.event) {
    case EVENT_IRQ_TIMER:
      putch('t'); break;
    case EVENT_IRQ_IODEV:
      putch('d'); break;
    case EVENT_YIELD:
      putch('y'); 
      ctx->mepc +=4;
      break;
    default:
      panic("Unhandled event"); break;
  }
  return ctx;
}
```

### 梳理整个执行流

![image-20231224171300176](C:\Users\xuliz\AppData\Roaming\Typora\typora-user-images\image-20231224171300176.png)

### 实现etrace

在`ecall`指令中调用即可。

```c
static void etrace() {
  IFDEF(CONFIG_ETRACE, {
    printf("\n" 
      ANSI_FMT("[ETRACE]", ANSI_FG_YELLOW) 
      "ecall in mepc = " FMT_WORD ", mcause = " FMT_WORD "\n",
      cpu.csrs.mepc, cpu.csrs.mcause);
  });
}
```



# PA3.2

这节通过分析一个最简单的操作系统`nanos-lite`来理解用户程序和系统调用。

先简单梳理一下Nanos-lite的行为:

1. 打印Project-N的logo, 并通过`Log()`输出hello信息和编译时间. 需要说明的是, Nanos-lite中定义的`Log()`宏并不是NEMU中定义的`Log()`宏. Nanos-lite和NEMU是两个独立的项目, 它们的代码不会相互影响, 你在阅读代码的时候需要注意这一点. 在Nanos-lite中, `Log()`宏通过你在`klib`中编写的`printf()`输出, 最终会调用TRM的`putch()`.
2. 调用`init_device()`对设备进行一些初始化操作. 目前`init_device()`会直接调用`ioe_init()`.
3. 初始化`ramdisk`. 一般来说, 程序应该存放在永久存储的介质中(比如磁盘). 但要在NEMU中对磁盘进行模拟是一个略显复杂工作, 因此先让Nanos-lite把其中的一段内存作为磁盘来使用. 这样的磁盘有一个专门的名字, 叫`ramdisk`.
4. `init_fs()`和`init_proc()`, 分别用于初始化文件系统和创建进程, 目前它们均未进行有意义的操作, 可以忽略它们.
5. 调用`panic()`结束Nanos-lite的运行.

### 为Nanos-lite实现正确的事件分发

识别出`EVENT_YIELD`，打印即可。

```c
static Context* do_event(Event e, Context* c) {
  switch (e.event) {
    case EVENT_YIELD : Log("yield succeed!"); c->mepc += 4; break;
    default: panic("irq:Unhandled event ID = %d", e.event);
  }
  return c;
}
```

### 实现loader

> - 可执行文件在哪里?
> - 代码和数据在可执行文件的哪个位置?
> - 代码和数据有多少?
> - "正确的内存位置"在哪里?

1. 位于`ramdisk.img`中，目前被放再地址0x83000000处。
2. 通过了解elf文件格式可知，在`program header table`中带有`Type=PT_LOAD`的项就是需要加载的可执行文件。
3. 每个segment的属性有标注。
4. 放到`[VirtAddr, VirtAddr + MemSiz)`这一连续区间

搞清楚上述问题后开始实现`loader`。

`loader`的实现借助`ramdisk_read`函数即可，之前在`nemu`中实现`ftrace`时已经知道了elf文件格式，这次实现起来比较轻松，重点有以下两个。

1.  `ramdisk_read((void *)ph[i].p_vaddr, ph[i].p_offset, ph[i].p_memsz);`"，上文中问了正确的内存位置"在哪里，直接使用`(void *)ph[i].p_vaddr`，我暂时没搞清楚为什么直接使用它
2. 返回`ehdr.e_entry`

```c
static uintptr_t loader(PCB *pcb, const char *filename) {
  // open elf
  Elf_Ehdr ehdr;
  ramdisk_read(&ehdr, 0, sizeof(Elf_Ehdr));
  assert(*(uint32_t *)ehdr.e_ident == 0x464c457f);

  // check elf type
  assert(ehdr.e_machine == EXPECT_TYPE);

  // read phdr
  Elf_Phdr ph[ehdr.e_phnum];
  ramdisk_read(ph, ehdr.e_phoff, sizeof(Elf_Phdr)*ehdr.e_phnum);
  for (int i = 0; i < ehdr.e_phnum; i++) {
    if (ph[i].p_type == PT_LOAD) {
      ramdisk_read((void *)ph[i].p_vaddr, ph[i].p_offset, ph[i].p_memsz);
      memset((void *)(ph[i].p_vaddr + ph[i].p_filesz), 0, ph[i].p_memsz - ph[i].p_filesz);
    }
  }
  return ehdr.e_entry;
}
```

### 识别系统调用

在AM中实现识别出各种系统调用和yield即可。

```c
Context* __am_irq_handle(Context *c) {
  if (user_handler) {
    Event ev = {0};
    switch (c->mcause) {
      case 0: case 1: case 2: case 3: case 4: case 7: case 8: case 9: ev.event = EVENT_SYSCALL; break;
      case 11: ev.event = EVENT_YIELD; break;
      default: ev.event = EVENT_ERROR; break;
    }
    c = user_handler(ev, c);
    assert(c != NULL);
  }
  return c;
}
```

### 实现SYS_yield系统调用

接下来就是在`nanos-lite`中添加系统调用。

先将`am/include/arch/riscv.h`中的寄存器修改好。

```c
#define GPR2 gpr[10] // a0
#define GPR3 gpr[11] // a1
#define GPR4 gpr[12] // a2
#define GPRx gpr[10] // a0
```

然后在`nanos-lite`中识别系统调用并实现系统调用的内容。

```c
static Context* do_event(Event e, Context* c) {
  switch (e.event) {
    case EVENT_YIELD : Log("yield succeed!"); c->mepc += 4; break;
    case EVENT_SYSCALL : do_syscall(c); c->mepc += 4; break;
    default: panic("irq:Unhandled event ID = %d", e.event);
  }

  return c;
}
```

此处的代码包括了后续部分的内容。

```c
#include <common.h>
#include "syscall.h"

#define strace(); Log("[strace] : %s , a0 : %d, a1 : %d, a2 : %d, ret : %d\n", syscall_name[a[0]], a0, a[2], a[3], c->GPRx);

static char* syscall_name[] = {"exit", "yield", "open", "read",
                               "write", "kill", "getpid", "close",
                               "lseek", "brk", "fstat", "time",
                               "signal", "execve", "fork", "link",
                               "unlink", "wait", "times", "gettimeofday"};

void yield();
void halt(int code);
void putch(char ch);

// file system call
int fs_open(const char *pathname, int flags, int mode);
size_t fs_read(int fd, void *buf, size_t len);
size_t fs_write(int fd, const void *buf, size_t len);
size_t fs_lseek(int fd, size_t offset, int whence);
int fs_close(int fd);

int sys_yield() {
  yield();
  return 0;
}

void sys_exit(int code) {
  halt(code);
}

void do_syscall(Context *c) {
  uintptr_t a[4];
  a[0] = c->GPR1; // a7
  a[1] = c->GPR2; // a0
  a[2] = c->GPR3; // a1
  a[3] = c->GPR4; // a2

  uintptr_t a0 = a[1];

  switch (a[0]) {
    case SYS_exit : 
      strace(); 
      sys_exit(a[0]); 
      break;
    case SYS_yield : 
      c->GPRx = sys_yield(); 
      break;
    case SYS_write : 
      c->GPRx = fs_write(a[1], (char *)a[2], a[3]);
      break;
    case SYS_brk :
      c->GPRx = 0;
      break;
    case SYS_open :
      c->GPRx = fs_open((char *)a[1], a[2], a[3]);
      break;
    case SYS_read :
      c->GPRx = fs_read(a[1], (char *)a[2], a[3]);
      break;
    case SYS_lseek :
      c->GPRx = fs_lseek(a[1], a[2], a[3]);
      break;
    case SYS_close :
      c->GPRx = fs_close(a[1]);
      break;
    default : panic("syscall:Unhandled syscall ID = %d", a[0]);
  }
  strace();
}

```

### 操作系统上TRM

这里要求我们实现`write`系统调用和`brk`系统调用。

主要是libos那边的实现。

```c
void *_sbrk(intptr_t increment) {
  uintptr_t addr = program_break + increment;
  if (addr >= (intptr_t)&_end) {
    if (_syscall_(SYS_brk, addr, 0, 0) == 0) {
      program_break = addr;
      return (void *)(addr - increment);
    } 
  }
  return (void *)-1;
}
```

### 梳理hello执行的整个过程

首先，`hello.c`在`navy-apps`项目下被编译成一个elf可执行文件。

之后这个可执行文件重命名放入`nanos-lite`的指定文件夹，被`nanos-lite`放置在`[ramdisk_start, ramdisk_end]`处。

然后，`nanos-lite`会被和`am`一起被编译成elf可执行文件并将其发送被`nemu`，然后执行nemu。

最后我们看一下`hello.c`的执行，它的执行很简单，就是通过调用`nanos-lite`提供的系统调用`write`和`brk`来进行输出字符串。

# PA3.3

### 简易文件系统

这一节让我们为`nanos-lite`实现一个简易的文件系统，使它可以根据传入文件名的参数不同读入不同的文件到对应内存位置。

> 另外, 我们也不希望每次读写操作都需要从头开始. 于是我们需要为每一个已经打开的文件引入偏移量属性`open_offset`, 来记录目前文件操作的位置. 每次对文件读写了多少个字节, 偏移量就前进多少.

根据上述提示，我们应该为文件记录表添加一个`open_offset`选项。

```c
typedef struct {
  char *name;
  size_t size;
  size_t disk_offset;
  ReadFn read;
  WriteFn write;
  size_t open_offset;
} Finfo;
```

接下来我们实现讲义中提到的五个系统调用使用到的函数并将对应系统调用也实现，讲义也给了我们几个注意点。

首先是`fs_open`，遍历文件记录表并找到所需文件即可，没找到则直接assert。

```c
int fs_open(const char *pathname, int flags, int mode) {
  int ft_len = sizeof(file_table) / sizeof(Finfo);
  for (int i = 3; i < ft_len; i++) {
    if (strcmp(pathname, file_table[i].name) == 0) {
      file_table[i].open_offset = 0;
      return i;
    }
  }
  panic("Should not reach here");
  return 0;
}
```

接下来是`fs_read`、`fs_write`和`fs_lseek`，`fs_close`直接返回0即可。

```c
size_t fs_read(int fd, void *buf, size_t len) {
  // 处理对stdin, stdout和stderr的读操作
  if (fd < 3) {
    return 0;
  }
  
  size_t size = file_table[fd].size;
  size_t open_offset = file_table[fd].open_offset;
  
  // 处理长度越界
  int real_len = len;
  if (open_offset + len > size) {
    real_len = size - open_offset;
  }

  // 进行实际读取操作
  int ret = ramdisk_read(buf, file_table[fd].disk_offset + open_offset, real_len);
  file_table[fd].open_offset = open_offset + real_len;

  return ret;
}
```

```c
size_t fs_write(int fd, const char *buf, size_t len) {
  size_t ret = 0;
  if (fd == 1 || fd == 2) {
    for (int i = 0; i < len; i++) {
      putch(buf[i]);
    }
    ret = len;
  }
  else if (fd == 0) {

  }
  else {
    assert(file_table[fd].open_offset + len <= file_table[fd].size);
    ret = ramdisk_write(buf, file_table[fd].disk_offset + file_table[fd].open_offset, len);
    file_table[fd].open_offset += len;
  }
  return ret;
}
```

```c
size_t fs_lseek(int fd, size_t offset, int whence) {
  size_t cur_offset = file_table[fd].open_offset;

  switch (whence) {
    case SEEK_SET:
      assert(offset <= file_table[fd].size);
      file_table[fd].open_offset = offset;
      break;
    case SEEK_CUR:
      assert(cur_offset + offset <= file_table[fd].size);
      file_table[fd].open_offset = cur_offset + offset;
      break;
    case SEEK_END:
      assert(file_table[fd].size + offset <= file_table[fd].size);
      file_table[fd].open_offset =  file_table[fd].size + offset;
      break;
    default:
      assert("Invalid whence parameter\n");
    }
    return file_table[fd].open_offset;
}
```

接着实现这五个系统调用并修改`libos`中对系统调用的调用。

```c
case SYS_write : 
    c->GPRx = fs_write(a[1], (char *)a[2], a[3]);
    break;
case SYS_open :
    c->GPRx = fs_open((char *)a[1], a[2], a[3]);
    break;
case SYS_read :
    c->GPRx = fs_read(a[1], (char *)a[2], a[3]);
    break;
case SYS_lseek :
    c->GPRx = fs_lseek(a[1], a[2], a[3]);
    break;
case SYS_close :
    c->GPRx = fs_close(a[1]);
    break;
```

还有就是对loader的修改。

```c
static uintptr_t loader(PCB *pcb, const char *filename) {
  int fd = fs_open(filename, 0, 0);

  // open elf
  Elf_Ehdr ehdr;
  fs_read(fd, &ehdr, sizeof(Elf_Ehdr));
  assert(*(uint32_t *)ehdr.e_ident == 0x464c457f);

  // check elf type
  assert(ehdr.e_machine == EXPECT_TYPE);

  // read phdr
  Elf_Phdr ph[ehdr.e_phnum];
  fs_lseek(fd, ehdr.e_phoff, SEEK_SET);
  fs_read(fd, ph, sizeof(Elf_Phdr)*ehdr.e_phnum);

  for (int i = 0; i < ehdr.e_phnum; i++) {
    if (ph[i].p_type == PT_LOAD) {
      fs_lseek(fd, ph[i].p_offset, SEEK_SET);
      fs_read(fd, (void *)ph[i].p_vaddr, ph[i].p_memsz);
      memset((void *)(ph[i].p_vaddr + ph[i].p_filesz), 0, ph[i].p_memsz - ph[i].p_filesz);
    }
  }

  fs_close(fd);
  return ehdr.e_entry;
}
```

### 串口

首先实现串口，这里我们需要重构一下之前的`fs_read()`和`fs_write()`两个函数，并实现一下`serial_write()`，然后修改一下对应的文件记录表结构体即可。

```c
size_t fs_read(int fd, void *buf, size_t len) {
  assert(fd >= 0 && fd <= sizeof(file_table) / sizeof(Finfo));
  size_t ret = 0;
  Finfo *f = &file_table[fd];

  if (f->read != NULL) {
    return f->read(buf, 0, len);
  } else {
    // 处理长度越界
    size_t real_len = len;
    if (f->open_offset + len > f->size) {
      real_len = f->size - f->open_offset;
    }
    // 进行实际读取操作
    ret = ramdisk_read(buf, f->disk_offset + f->open_offset, real_len);
    f->open_offset +=  real_len;
  }
  return ret;
}

size_t fs_write(int fd, const char *buf, size_t len) {
  assert(fd >= 0 && fd <= sizeof(file_table) / sizeof(Finfo));
  size_t ret = 0;
  Finfo *f = &file_table[fd];

  if (f->write != NULL) {
    return f->write(buf, 0, len);
  } else {
    // 处理长度越界
    size_t real_len = len;
    if (f->open_offset + len > f->size) {
      real_len = f->size - f->open_offset;
    }
    // 进行实际写入操作
    ret = ramdisk_write(buf, f->disk_offset + f->open_offset, real_len);
    f->open_offset += real_len;
  }
  return ret;
}
```

```c
size_t serial_write(const void *buf, size_t offset, size_t len) {
  char *cbuf = (char *)buf;
  for (int i = 0; i < len; i++) {
    putch(cbuf[i]);
  }
  return len;
}
```

### gettimeofday

接下来是实现gettimeofday系统调用，根据pa2的IOE我们知道am把时间信息存在AM_TIMER_UPTIME寄存器中，我们使用`ioe_read`获取即可。

查阅一下`timeval`的信息，`struct timeval` 是一个用于表示时间间隔的结构体，在许多UNIX系统上被广泛使用。它的定义通常如下：

```c
struct timeval {
    time_t tv_sec;      // 秒数
    suseconds_t tv_usec; // 微秒数
};
```

- `tv_sec` 表示自 Epoch（1970年1月1日午夜）以来的秒数。这是一个整数类型的字段，通常是 `time_t`，用于存储秒部分的时间。

- `tv_usec` 表示秒的小数部分，以微秒为单位。这是一个整数类型的字段，通常是 `suseconds_t`。微秒是秒的百万分之一。

```c
int sys_gettimeofday(struct timeval *tv, struct timezone* tz) {
  assert(tv != NULL);
  int us = 0;
  ioe_read(AM_TIMER_UPTIME, &us);
  tv->tv_sec = us / (1000*1000);
  tv->tv_usec = us % (1000*1000);
  return 0;
}
```

接下来实现测试，模仿其他测试即可，我们让程序每过0.5秒输出当前的秒数和毫秒数。

```c
#include <stdio.h>

int main() {
  struct timeval tv;
  gettimeofday(&tv, NULL);

  int ms = 500;
  while (1) {
    while ((tv.tv_sec * 1000 + tv.tv_usec / 1000) < ms) {
      gettimeofday(&tv, NULL);
    }
    printf("time %d ", tv.tv_sec);
    printf("ms = %d\n", ms);
    ms += 500;
  }
}
```

### NDL实现时钟

接下来实现NDL的时钟，使`NDL_GetTicks`返回当前时间的毫秒数，由于我们的`gettimeofday`获取的时间实际上是`AM_TIMER_UPTIME`寄存器里的时间，此时间代表的是系统开机后的时间，所以我们需要在`NDL_init`中先初始化一下第一次获取的时间，之后用每次获取的时间减去开始时间来获取启动NDL库之后的当前时间。

```c
int NDL_Init(uint32_t flags) {
  if (getenv("NWM_APP")) {
    evtdev = 3;
  }
  // 初始化时间
  struct timeval tv;
  gettimeofday(&tv, NULL);
  init_time = (uint32_t)(tv.tv_sec * 1000 + tv.tv_usec / 1000);
  return 0;
}

uint32_t NDL_GetTicks() {
  struct timeval tv;
  gettimeofday(&tv, NULL);
  // 返回毫秒数
  uint32_t now_time = (uint32_t)(tv.tv_sec * 1000 + tv.tv_usec / 1000) - init_time;
  return now_time;
}
```

之后修改一下测试，并将`libndl`库加入navy项目的库链接编译选项中。

### 把按键输入抽象成文件

我们可以看一下`am-test`中的`keyboard test`是怎么实现的，可以得到一些启发。

```c
size_t events_read(void *buf, size_t offset, size_t len) {
  AM_INPUT_KEYBRD_T kbd;
  ioe_read(AM_INPUT_KEYBRD, &kbd);
  int ret = 0;
  if (kbd.keycode == AM_KEY_NONE) {
    *(char *)buf = '\0';
  } else {
    ret = sprintf((char *)buf, "%s %s\n\0", kbd.keydown ? "kd" : "ku", keyname[kbd.keycode]);
  }
  return ret;
}
```

修改一下文件记录表数据结构，即VFS。

```c
[FD_EVENT] = {"/dev/event", 0 ,0, events_read,invalid_write},
```

最后实现一下`NDL_PollEvent()`，由于字符型设备所以不需要带缓冲的`fread()`，用`read()`即可。

```c
int NDL_PollEvent(char *buf, int len) {
  return read(event_fd, buf, len);
}
```

### VGA实现

首先实现`dispinfo_read()`并在`NDL_Init()`中解析出屏幕大小，然后在`NDL_OpenCanvas()`中设置画布大小。

这里我们在`nanos-lite`中看似是读取`dispinfo`文件的信息，其实底层实现是将AM中的`AM_GPU_CONFIG`抽象寄存器的值读取出来。

```c
size_t dispinfo_read(void *buf, size_t offset, size_t len) {
  AM_GPU_CONFIG_T cfg;
  ioe_read(AM_GPU_CONFIG, &cfg);
  sprintf((char *)buf, "WIDTH:%d\nHEIGHT:%d\n", cfg.width, cfg.height);
  screen_h = cfg.height;
  screen_w = cfg.width;
  return 0;
}
```

这里是上层的实现，通过读`dispinfo`文件获取屏幕大小。

```c
// 打开vga的dispinfo文件并解析出屏幕大小
  vga_fd = open("proc/dispinfo", "r");
  char buf[64];
  read(vga_fd, buf, sizeof(buf));
  sscanf(buf, "WIDTH:%d\nHEIGHT:%d\n", &screen_w, &screen_h);
```

```c
void NDL_OpenCanvas(int *w, int *h) {
  ...
  // 画布小于屏幕
  assert(screen_h >= *h && screen_w >= *w);
  // 设置canvas大小
  if (*w == 0 && *h == 0) {
    canvas_h = screen_h;
    canvas_w = screen_w;
  } else {
    canvas_h = *h;
    canvas_w = *w;
  }
}
```

接下来在`init_fs()`中初始化`/dev/fb`的大小，并实现`fb_write()`。

这里也是看似是向fb文件中写入像素信息，但实际上底层（`nanos-lite`操作系统层）实现是将像素信息写入到AM的`AM_GPU_FBDRAW`寄存器，这里因为函数形参的限制，我们没法获取上层的w和h，所以我们选择每次写入一行，在下面的测试中就是一次写入128个像素。

```c
size_t fb_write(const void *buf, size_t offset, size_t len) {
  //printf("offset : %d, len : %d\n", offset, len);
  AM_GPU_FBDRAW_T fb_ctl;
  fb_ctl.pixels = (uint32_t *)buf;
  fb_ctl.x = offset % screen_w;
  fb_ctl.y = offset / screen_w;
  fb_ctl.w = len, fb_ctl.h = 1;
  fb_ctl.sync = true;
 // printf("x : %d, y : %d, w : %d, h: %d\n",fb_ctl.x, fb_ctl.y, fb_ctl.w, fb_ctl.h);
  ioe_write(AM_GPU_FBDRAW, &fb_ctl);
  return 0;
}
```

最后实现一下`NDL_DrawRect()`，也就是上层的实现，可以对x和y进行居中化的效果，`write`没法带`offset`参数，所以我们通过`lseek`来调整每次写入的位置，写入的`pixels`也要记得往后遍历增加。

```c
void NDL_DrawRect(uint32_t *pixels, int x, int y, int w, int h) {
  x += (screen_w - canvas_w) / 2;
  y += (screen_h - canvas_h) / 2;
  for (int i = 0; i < h; i++) {
    int offset = (y + i)* screen_w + x;
    lseek(fb_fd, offset, SEEK_SET);
    write(fb_fd, pixels + (i * w), w);
  }
}
```

### 定点运算

两个`fixedpt`进行除法，`(a * 2^8) / (b * 2^8) = a / b`，所以结果应该再乘`2^8`。

向上取整时检测一下有没有小数部分，有则加1即可。

```c
static inline fixedpt fixedpt_muli(fixedpt A, int B) {
	return A * B;
}

/* Divides a fixedpt number with an integer, returns the result. */
static inline fixedpt fixedpt_divi(fixedpt A, int B) {
	return A / B;
}

/* Multiplies two fixedpt numbers, returns the result. */
static inline fixedpt fixedpt_mul(fixedpt A, fixedpt B) {
	return (A * B) >> 8;
}

/* Divides two fixedpt numbers, returns the result. */
static inline fixedpt fixedpt_div(fixedpt A, fixedpt B) {
	return (A / B) << 8;
}

static inline fixedpt fixedpt_abs(fixedpt A) {
	return A > 0 ? A : -A;
}

static inline fixedpt fixedpt_floor(fixedpt A) {
	return A & 0xffffff00;
}

static inline fixedpt fixedpt_ceil(fixedpt A) {
	fixedpt integer_part = A & 0xffffff00;
    if (A & 0x000000ff) {
        integer_part += 0x00000100;
    }
    return integer_part;
}
```

写个测试检测一下有没有问题。

```c
#include "fixedptc.h"
#include <stdio.h>

int main() {
  fixedpt a = fixedpt_rconst(1.2);
  fixedpt b = fixedpt_rconst(-1.2);
  int c = 3;
  printf("%d\n",fixedpt_floor(a) >> 8);
  printf("%d\n",fixedpt_ceil(a) >> 8);
  printf("%d\n",fixedpt_floor(b) >> 8);
  printf("%d\n",fixedpt_ceil(b) >> 8);
  printf("%d %d\n", b, fixedpt_abs(b));
  printf("%d %d\n", fixedpt_muli(a, c), fixedpt_divi(a, c));
  printf("%d %d\n", fixedpt_muli(b, c), fixedpt_divi(b, c));
  printf("%d %d\n", fixedpt_mul(a, b), fixedpt_div(a, b));
  printf("%d\n", 0xfffffe00);
}
```

### NSlider

这部分要求我们实现`mini-SDL`的一些函数，这些函数使用我们在前面实现的NDL库即可。

首先，我们需要实现`SDL_UpdateRect()`，这个函数需要使用我们在上面实现过的`NDL_DrawRect()`，这部分无非是查阅一下`SDL_Surface`的结构就可以了。

```c
void SDL_UpdateRect(SDL_Surface *s, int x, int y, int w, int h) {
  NDL_DrawRect((uint32_t *)s->pixels, x, y, s->w, s->h);
}
```

实现完后再调一些参数并更新`nanos-lite`即可。

然后就是让我们实现翻页的功能，我们打开`NSlider`的`main`函数查看发现只使用到了`SDL_Event`的`e.type`和`e.key.keysym.sym`，查阅代码后发现`e.key.keysym.sym`其实是一个数字值，这个值和`events_read()`函数中的keycode应该是一样的，但是我们在`events_read()`只把`keyname`放入了`buf`，所以此时我们需要修改一下代码，让它把`keycode`也存入`buf`中方便我们获取，这样`SDL_WaitEvent`的实现就很简单了。

```c
int SDL_WaitEvent(SDL_Event *event) {
  char buf[64];
  char type[8], key_name[8];
  int keycode;

  while (1) {
    if (NDL_PollEvent(buf, sizeof(buf)) == 0) {
      continue;
    }
    sscanf(buf, "%s %s %d\n", type, key_name, &keycode);
    event->type = buf[1] == 'u' ? SDL_KEYUP : SDL_KEYDOWN;
    event->key.keysym.sym = keycode;
    return 1;
  }
  return 0;
}
```

接着修改一下main函数的参数并将多张ppt放入脚本中转换（修改一下脚本，脚本默认只转一张）即可。

![image-20240101140339286](C:\Users\xuliz\AppData\Roaming\Typora\typora-user-images\image-20240101140339286.png)

### MENU

这部分主要是实现`SDL_FillRect()`和`SDL_BlitSurface()`，不需要调用NDL库的函数，理清楚逻辑就能很快实现。

`SDL_BlitSurface`实现中，要注意指针的使用，例如在`*((uint32_t *)(src->pixels) + (src_y * src_w + src_x) + (i * src_w + j))`中，先将`(src->pixels)`转换成`(uint32_t *)`，再对它进行加法操作，才能一次加4个字节，如果是直接加，会只加1个字节，因为`SDL_Surface`实现时用到的是`uint8_t *pixels;`，还有就是两个画布间的关系要理清楚。

```c
void SDL_BlitSurface(SDL_Surface *src, SDL_Rect *srcrect, SDL_Surface *dst, SDL_Rect *dstrect) {
  	assert(dst && src);
  	assert(dst->format->BitsPerPixel == src->format->BitsPerPixel);

  	int w = 0, h = 0;
    int src_x = 0, src_y = 0;
    int dst_x = 0, dst_y = 0;
    if (srcrect) {
      w = srcrect->w; h = srcrect->h;
      src_x = srcrect->x; src_y = srcrect->y;
    } else {
      w = src->w; h = src->h;
    }
    if (dstrect) {
      dst_x = dstrect->x; dst_y = dstrect->y;
    } 
    
    uint32_t *tmp_src = (uint32_t *)src->pixels;
    uint32_t *tmp_dst = (uint32_t *)dst->pixels;
    for (int i = 0; i < h; i++) {
      for (int j = 0; j < w; j++) {
        tmp_dst[(dst_y + i) * dst->w + dst_x + j] = tmp_src[(src_y + i) * src->w + src_x + j];
      }
    }
}

void SDL_FillRect(SDL_Surface *dst, SDL_Rect *dstrect, uint32_t color) {
  int x, y, w, h;
  if (dstrect) {
    x = dstrect->x; y = dstrect->y;
    w = dstrect->w; h = dstrect->h;
  } else {
    x = y = 0;
    w = dst->w; h = dst->h;
  }

  uint32_t *pixels = (uint32_t *)(dst->pixels);
  for(int i = 0; i < h; i++) {
    for (int j = 0; j < w; j++) {
      pixels[(y + i) * w + x + j] = color;
    }
  }
}
```

### NTerm

这里要求我们实现`SDL_GetTicks()`和`SDL_PollEvent()`这两个函数，注意文档里规定`returns 1 if there are any pending events, or 0 if there are none available.`

```c
int SDL_PollEvent(SDL_Event *ev) {
  char buf[64];
  char type[8], key_name[8];
  int keycode;

  if (NDL_PollEvent(buf, 0) == 0) {
    return 0;
  }
  sscanf(buf, "%s %s %d\n", type, key_name, &keycode);
  ev->type = buf[1] == 'u' ? SDL_KEYUP : SDL_KEYDOWN;
  ev->key.keysym.sym = keycode;
  return 1; 
}
```

### Flappy Bird

突然发现前面我一直开着itrace，导致nemu的性能下降太多了，所以我把itrace关了，后面的bird和pal也能流畅运行了。

我在`nanos-lite`里`update`时发现`bird`的`makefile`报软链接问题，查询后发现是`navy`里的`fsing`文件夹没有`games`这个目录，导致bird缺失上级目录，加上后就好了。

接下是实现`IMG_Load()`。

```c
SDL_Surface* IMG_Load(const char *filename) {
  int fd = open(filename, 0, 0);

  int file_size = lseek(fd, 0, SEEK_END);
  lseek(fd, 0, SEEK_SET);
  void *buf = malloc(file_size);
  read(fd, buf, file_size);
  SDL_Surface *ret = STBIMG_LoadFromMemory((char *)buf, file_size);

  free(buf);
  close(fd);
  return ret;
}
```

### 仙剑

文件我在网上找到了一堆大写的MKF的文件，还有一些其他的，看了下pal的源码发现打开文件都是小写的，写了个脚本把文件全转成小写。

再按讲义写一个`sdlpal.cfg`放到游戏文件里面就可以了。

然后是调整`SDL_BlitSurface`和`SDL_UpdateRect`的代码，复制函数的代码添加一部分即可。

```c
void SDL_BlitSurface(SDL_Surface *src, SDL_Rect *srcrect, SDL_Surface *dst, SDL_Rect *dstrect) {
  assert(dst && src);
  assert(dst->format->BitsPerPixel == src->format->BitsPerPixel);

  if (src->format->BytesPerPixel == 4) {
    int w = 0, h = 0;
    int src_x = 0, src_y = 0;
    int dst_x = 0, dst_y = 0;
    if (srcrect) {
      w = srcrect->w; h = srcrect->h;
      src_x = srcrect->x; src_y = srcrect->y;
    } else {
      w = src->w; h = src->h;
    }
    if (dstrect) {
      dst_x = dstrect->x; dst_y = dstrect->y;
    } 
    
    uint32_t *tmp_src = (uint32_t *)src->pixels;
    uint32_t *tmp_dst = (uint32_t *)dst->pixels;
    for (int i = 0; i < h; i++) {
      for (int j = 0; j < w; j++) {
        tmp_dst[(dst_y + i) * dst->w + dst_x + j] = tmp_src[(src_y + i) * src->w + src_x + j];
      }
    }
  } else if (src->format->BytesPerPixel == 1) {
    
    int w = 0, h = 0;
    int src_x = 0, src_y = 0;
    int dst_x = 0, dst_y = 0;
    if (srcrect) {
      w = srcrect->w; h = srcrect->h;
      src_x = srcrect->x; src_y = srcrect->y;
    } else {
      w = src->w; h = src->h;
    }
    if (dstrect) {
      dst_x = dstrect->x; dst_y = dstrect->y;
    } 
    
    uint8_t *tmp_src = (uint8_t *)src->pixels;
    uint8_t *tmp_dst = (uint8_t *)dst->pixels;
    for (int i = 0; i < h; i++) {
      for (int j = 0; j < w; j++) {
        tmp_dst[(dst_y + i) * dst->w + dst_x + j] = tmp_src[(src_y + i) * src->w + src_x + j];
      }
    }
  }
}
```

画图的代码要稍微修改下，之前图方便直接把画布s全部画上去，这回需要判断一下是否为调色盘类型的surface，

```c
void SDL_UpdateRect(SDL_Surface *s, int x, int y, int w, int h) {
  if (w == 0 && h == 0) { 
    w = s->w;
    h = s->h;
  }  
  if (s->format->BytesPerPixel == 4) {
    NDL_DrawRect((uint32_t *)s->pixels, x, y, w, h);
  } else if (s->format->BytesPerPixel == 1) {
    uint32_t *pixels = malloc(w * h * 4);
    uint8_t *src = (uint8_t *)s->pixels;
    for (int i = 0; i < h; i++) {
      for (int j = 0; j < w; j++) {
        pixels[i * w + j] = color_translater(&s->format->palette->colors[src[i * w + j]]);
      }
    }
    NDL_DrawRect(pixels, x, y, w, h);
    free(pixels);
  }
}
```

### 可以运行其它程序的开机菜单

这里要求我们实现`SYS_execve`，已经实现很多次了。

### 为NTerm中的內建Shell添加环境变量的支持

只需要修改一下`sh_handle_cmd`函数即可，查询一下`setenv`和`execvp`函数。

```c
static void sh_handle_cmd(const char *cmd) {
  char tmp_cmd[64];
  strcpy(tmp_cmd, cmd);
  tmp_cmd[strlen(cmd) - 1] = '\0';
  setenv("PATH", "/bin", 0);
  execvp(tmp_cmd, NULL);
}
```

### 必答题 - 理解计算机系统

Navy：`mgo.kf`文件被读取后，会被设置成`Palette`模式的`surface`，之后会被`SDL_BlitSurface`复制到屏幕画布中，接着在更新画面时调用`SDL_UpdateRect`，`SDL_UpdateRect`调用`NDL_DrawRect`，通过向`fb_fd`写入像素实现所需操作，之后便转入`Nanos-lite`中。

Nanos-lite：接收到write的系统调用，调用`fs_write`，`fs_write`调用对应的`fb_write`，`fb_write`调用am中的`ioe_write`向抽象寄存器`AM_GPU_FBDRAW`中写入数据，之后便是am层和nemu层的任务。
