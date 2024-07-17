# PA4-虚实交错的魔法：分时多任务

## 多道程序

### 上下文切换

#### 内核线程

##### 实现上下文切换（1）

首先是`kcontext()`，理解讲义之后我们会发现其实很简单，就是让我们创建一个`Context *cp`指向所给的栈底位置，然后把entry填入`Context`的`mepc`中，为了后续在`__am_asm_trap`中`mret`时会返回到f函数中，这里的减4是为了后面方便统一加4。

```c
Context *kcontext(Area kstack, void (*entry)(void *), void *arg) {
  Context *cp = (Context *)(kstack.end - sizeof(Context));
  cp->mepc = (uintptr_t)entry - 4;
  return cp;
}
```

然后我们修改CTE中`__am_asm_trap()`的实现，这个也很简单，修改一下sp寄存器即可。

```asm
__am_asm_trap:
  ...
  mv a0, sp
  jal __am_irq_handle

  mv sp, a0

 ...

```

然后修改一下`yield-os`。

```c
static Context *schedule(Event ev, Context *prev) {
  current->cp = prev;
  current = (current == &pcb[0] ? &pcb[1] : &pcb[0]);
  current->cp->mepc += 4;
  return current->cp;
}
```

##### 实现上下文切换(2)

用a0传参即可。

```c
Context *kcontext(Area kstack, void (*entry)(void *), void *arg) {
  Context *cp = (Context *)(kstack.end - sizeof(Context));
  cp->mepc = (uintptr_t)entry - 4;
  cp->gpr[10] = (uintptr_t)(arg);
  return cp;
}
```

### OS中的上下文切换

#### RT-Thread(选做)

这部分主要让我们实现`context.c`中的几个函数。

`rt_hw_stack_init`需要我们创建一个上下文结构，使得在进行内核线程切换时可以利用上下文结构恢复现场，我们需要创建的结构如下图所示。

```
|               |
+---------------+ <---- stack.end(stack_addr)
|               |
|    context    |
|               |
+---------------+ <---- pcb->sp
|   tentry      | 
|---------------|
|   texit       |
|---------------|
|   parameter   |
|---------------|
|               |    
+---------------+ <---- stack,start
|               |
```

```c
void wrap(void *arg) {
  // parse paramater
  rt_ubase_t *stack_bottom = (rt_ubase_t *)arg;
  rt_ubase_t tentry = *stack_bottom; 
  stack_bottom--;
  rt_ubase_t texit = *stack_bottom; 
  stack_bottom--;
  rt_ubase_t parameter = *stack_bottom; 
  // function call
  ((void(*)())tentry) (parameter);
  ((void(*)())texit) ();
}

rt_uint8_t *rt_hw_stack_init(void *tentry, void *parameter, rt_uint8_t *stack_addr, void *texit) {
  // align
  stack_addr = (rt_uint8_t*)((uintptr_t)stack_addr & ~(sizeof(uintptr_t) - 1));
  // create context
  Area stack;
  stack.start = stack_addr - STACK_SIZE;
  stack.end = stack_addr;
  Context *cp = kcontext(stack, wrap, (void *)(stack.end - sizeof(Context) - 1));
  // set parameter
  rt_ubase_t *stack_bottom = (rt_ubase_t *)(stack.end - sizeof(Context) - 1);
  *stack_bottom = (rt_ubase_t)tentry;
  stack_bottom--;
  *stack_bottom = (rt_ubase_t)texit;
  stack_bottom--;
  *stack_bottom = (rt_ubase_t)parameter;

  return (rt_uint8_t *)cp;
}
```

然后就是实现上下文的切换，需要注意的是`to`和`from`是二级指针，需要强转后再解引用。

```c
static Context* ev_handler(Event e, Context *c) {
  switch (e.event) {
    case EVENT_YIELD:  {
      rt_thread_t cp = rt_thread_self(); 
      rt_ubase_t to = cp->user_data;
      c = *(Context**)to;
      c->mepc += 4;
      break;
    }
    default: printf("Unhandled event ID = %d\n", e.event); assert(0);
  }
  return c;
}

void rt_hw_context_switch_to(rt_ubase_t to) {
  rt_thread_t pcb = rt_thread_self();
  rt_ubase_t user_data_replicator = pcb->user_data;
  pcb->user_data = to;
  yield();
  pcb->user_data = user_data_replicator;
}

void rt_hw_context_switch(rt_ubase_t from, rt_ubase_t to) {
  rt_thread_t pcb = rt_thread_self();
  rt_ubase_t user_data_replicator = pcb->user_data;
  pcb->user_data = to;
  *(Context**)from = pcb->sp;
  yield();
  pcb->user_data = user_data_replicator;
}
```

#### Nanos-lite

##### 在Nanos-lite中实现上下文切换

在实现了`yield-os`之后，实现这个就不难了。

```c
void context_kload(PCB *p, void (*entry)(void *), void *arg) {
  p->cp = kcontext((Area) { p->stack, p + 1 }, hello_fun, arg);
}
```

我们把`mepc+4`的操作放到`do_event`中实现，让代码格式统一一些。

```c
Context* schedule(Context *prev) {
  current->cp = prev;
  current = (current == &pcb[0] ? &pcb[1] : &pcb[0]);
  return current->cp;
}
```

```c
static Context* do_event(Event e, Context* c) {
  switch (e.event) {
    case EVENT_YIELD : c = schedule(c); c->mepc += 4; break;
    case EVENT_SYSCALL : do_syscall(c); c->mepc += 4; break;
    default: panic("irq:Unhandled event ID = %d", e.event);
  }

  return c;
}
```

`mian`函数中已经有了一个`yield()`，所以我们没必要在`init_proc`中添加了。

### 用户进程

#### 创建用户进程上下文

##### 实现多道程序系统

首先是`ucontext()`，和`kcontext()`基本一致。

```c
Context *ucontext(AddrSpace *as, Area kstack, void *entry) {
  Context *cp = (Context *)(kstack.end - sizeof(Context));
  cp->mepc = (uintptr_t)entry - 4;
  return cp;
}
```

接下来是`context_uload()`函数，这里我本来使用的是loader函数，也定义了，但是编译还是报未定义错，搞不清楚什么原因，就修改了下`naive_uload()`函数，把函数入口直接返回。

```c
void context_uload(PCB *p, const char *filename) {
  uintptr_t entry = naive_uload(p, filename);
  p->cp = ucontext(&p->as, (Area) { p->stack, p + 1 }, (void *)entry);
  p->cp->GPRx = (uintptr_t)heap.end;
}
```

然后是在Navy的`_start`中设置正确的栈指针。

#### 用户进程的参数

##### 给用户进程传递参数

此处的指针操作比较麻烦，需要耐心根据图示写。

```c
void context_uload(PCB *p, const char *filename, char *const argv[], char *const envp[]) {
  uintptr_t entry = naive_uload(p, filename);

  int argc = 0, envc = 0;
  while (argv[argc] != NULL) argc++;
  while (envp[envc] != NULL) envc++;

  char* us1 = (char*)heap.end;
  // clone argv
  for (int i = 0; i < argc; i++) {
    size_t len = strlen(argv[i]) + 1; // include null character
    us1 -= len;
    strncpy(us1, argv[i], len);
  }
  // clone envp
  for (int i = 0; i < envc; i++) {
    size_t len = strlen(envp[i]) + 1; // include null character
    us1 -= len;
    strncpy(us1, envp[i], len);
  }

  uintptr_t* us2 = (uintptr_t *)us1;
  us2 -= (argc + envc + 3);

  us2[0] = argc;

  char* us_tmp = (char*)heap.end;
  for (int i = 0; i < argc; i++) {
    size_t len = strlen(argv[i]) + 1; 
    us_tmp -= len;
    us2[i + 1] = (uintptr_t)us_tmp;
  }
  us2[argc + 1] = 0;
  for (int i = 0; i < envc; i++) {
    size_t len = strlen(envp[i]) + 1; // include null character
    us_tmp -= len;
    us2[argc + i + 2] = (uintptr_t)us_tmp;
  }
  us2[argc + 2 + envc] = 0;

  p->cp = ucontext(&p->as, (Area) { p->stack, p + 1 }, (void *)entry);
  p->cp->GPRx = (uintptr_t)us2;
}
```

```c
void call_main(uintptr_t *args) {
  int argc = (int)args[0];
  char **argv = (char **)(args + 1);
  char **envp = (char **)(args + argc + 2);
  environ = envp;
  exit(main(argc, argv, envp));
  assert(0);
}

```

在前面已经把Navy中`_start`的代码修改过了，就是把a0放到sp上。

仙剑的商标打印就在mian函数中，非常简单就可以修改成功。

##### 实现带参数的execve()

接下来这里我尝试只跑一个用户进程总是会出错，查了半天，发现原来是之前为了模拟设备的等待来实现切换进程，写了很多yield在device中，去掉就可以了。

好多天没看代码了，记忆不清楚导致这个问题卡了很久，每次隔段时间写pa就会需要整体回顾一遍，这样太浪费时间了，不过对代码的理解似乎更深刻了，也算是螺旋上升的过程了。

```c
// context_uload()
char* us1 = (char*)new_page(8);
char* us_tmp = us1;

...

// char* us_tmp = (char*)heap.end;
```

```c
int sys_execve(const char *pathname, char *const argv[], char *const envp[]) {
  if (pathname == NULL)
    return -1;
  //naive_uload(NULL, pathname);
  context_uload(current, pathname, argv, envp);
  switch_boot_pcb();
  yield();
  return 0;
}
```

```c
void* new_page(size_t nr_page) {
  pf += nr_page * PGSIZE;
  return pf;
}
```

为了理解深刻进程的用户栈和进程pcb中的内核栈的不同，我打印了一个进程的用户栈和内核栈，可以看到`kernel stack:0x82260000, user stack:0x82299000`

```shell
[/home/silly/ysyx-workbench/nanos-lite/src/main.c,13,main] 'Hello World!' from Nanos-lite
[/home/silly/ysyx-workbench/nanos-lite/src/main.c,14,main] Build time: 23:03:44, Feb  8 2024
[/home/silly/ysyx-workbench/nanos-lite/src/mm.c,27,init_mm] free physical pages starting from 0x82291000
[/home/silly/ysyx-workbench/nanos-lite/src/device.c,74,init_device] Initializing devices...
[/home/silly/ysyx-workbench/nanos-lite/src/ramdisk.c,27,init_ramdisk] ramdisk info: start = 0x8000315d, end = 0x8225e55d, size = 36025344 bytes
[/home/silly/ysyx-workbench/nanos-lite/src/irq.c,17,init_irq] Initializing interrupt/exception handler...
kernel stack:0x82260000, user stack:0x82299000
```







## 4.2

之前的代码都是在链接时刻确定的位置



切换到内核线程时需要更新页目录表吗。

为此, 我们需要思考内核线程的调度会对分页机制造成什么样的影响. 内核线程和用户进程最大的不同, 就是它没有用户态的地址空间: 内核线程的代码, 数据和栈都是位于内核的地址空间. 那在启动分页机制之后, 如果`__am_irq_handle()`要返回一个内核线程的现场, 我们是否需要考虑通过`__am_switch()`切换到内核线程的虚拟地址空间呢?

答案是, 不需要. 这是因为AM创建的所有虚拟地址空间都会包含内核映射, 无论在切换之前是位于哪一个虚拟地址空间, 内核线程都可以在这个虚拟地址空间上正确运行. 因此我们只要在`kcontext()`中将上下文的地址空间描述符指针设置为`NULL`, 来进行特殊的标记, 等到将来在`__am_irq_handle()`中调用`__am_switch()`时, 如果发现地址空间描述符指针为`NULL`, 就不进行虚拟地址空间的切换.





1.用户进程yield时，保存的上下文在内核栈中还是用户栈中？



2.两个进程请求分配物理堆空间，为什么不会重叠？





目前任务：

4.2

**loader分页加载**



mm_brk函数修改



设置内核线程



4.3

添加时钟中断



实现用户栈与内核栈的切换



展示4个用户进程并发执行
