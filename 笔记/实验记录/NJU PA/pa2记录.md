# PA2

## 2.1 不停计算的机器

### 理解YEMU如何执行程序

> 画出在YEMU上执行的加法程序的状态机

`R[0]=16,R[1]=33 -> R[0]=49,R[1]=33`

## 2.2 RTFM

### RTFSC理解指令执行的过程

> 请整理一条指令在NEMU中的执行过程

1. `execute()`的循环中执行的`exec_once()`内，将`pc`赋值给`s->pc`和`s->npc`，然后执行`isa_exec_once()`，然后让`cpu.pc = s->dpc`，`isa_exec_once()`这个函数中有一条执行的执行过程。

2. 首先取指，`s->isa.inst.val = inst_fetch(&s->snpc, 4);`这条语句取出指令并将`s->npc`增加该条指令的长度。

3. 然后是译码和执行，`decode_exec(s)`中通过下面这段代码来对指令进行译码和执行

```c
INSTPAT_START();
  INSTPAT("??????? ????? ????? ??? ????? 00101 11", auipc  , U, R(rd) = s->pc + imm);
  INSTPAT("??????? ????? ????? 100 ????? 00000 11", lbu    , I, R(rd) = Mr(src1 + imm, 1));
  INSTPAT("??????? ????? ????? 000 ????? 01000 11", sb     , S, Mw(src1 + imm, 1, src2));

  INSTPAT("0000000 00001 00000 000 00000 11100 11", ebreak , N, NEMUTRAP(s->pc, R(10))); // R(10) is $a0
  INSTPAT("??????? ????? ????? ??? ????? ????? ??", inv    , N, INV(s->pc));
  INSTPAT_END();
```

这段代码的工作有识别出指令对应的语句，如果识别成功，则对指令的寄存器和立即数进行读取，并执行相对应的操作。

### 实现更多的指令

2.补充指令

完成所有测试后的代码如下：

```c
INSTPAT("??????? ????? ????? ??? ????? 00101 11", auipc  , U, R(rd) = s->pc + imm);
INSTPAT("??????? ????? ????? 100 ????? 00000 11", lbu    , I, R(rd) = Mr(src1 + imm, 1));
INSTPAT("??????? ????? ????? 000 ????? 01000 11", sb     , S, Mw(src1 + imm, 1, src2));

INSTPAT("??????? ????? ????? 010 ????? 01000 11", sw     , S, Mw(src1 + imm, 4, src2));
INSTPAT("??????? ????? ????? 000 ????? 00100 11", addi   , I, R(rd) = src1 + imm);
INSTPAT("??????? ????? ????? ??? ????? 11011 11", jal    , J, R(rd) = s->pc + 4; s->dnpc = s->pc + imm);
INSTPAT("??????? ????? ????? 000 ????? 11001 11", jalr   , I, R(rd) = s->pc + 4; s->dnpc = (src1 + imm) & (~1));
INSTPAT("??????? ????? ????? 010 ????? 00000 11", lw     , I, R(rd) = SEXT(Mr(src1 + imm, 4), 32));
INSTPAT("0000000 ????? ????? 000 ????? 01100 11", add    , R, R(rd) = src1 + src2);
INSTPAT("0100000 ????? ????? 000 ????? 01100 11", sub    , R, R(rd) = src1 - src2);
INSTPAT("??????? ????? ????? 011 ????? 00100 11", sltiu  , I, R(rd) = ((word_t)src1 < (word_t)imm ? 1 : 0));
INSTPAT("??????? ????? ????? 010 ????? 00100 11", slti   , I, R(rd) = ((sword_t)src1 < (sword_t)imm ? 1 : 0));
INSTPAT("??????? ????? ????? 000 ????? 11000 11", beq    , B, if(src1 == src2) s->dnpc = s->pc + imm);
INSTPAT("??????? ????? ????? 001 ????? 11000 11", bne    , B, if(src1 != src2) s->dnpc = s->pc + imm);
INSTPAT("0000000 ????? ????? 011 ????? 01100 11", sltu   , R, R(rd) = ((word_t)src1 < (word_t)src2 ? 1 : 0));
INSTPAT("0000000 ????? ????? 100 ????? 01100 11", xor    , R, R(rd) = src1 ^ src2);
INSTPAT("0000000 ????? ????? 110 ????? 01100 11", or     , R, R(rd) = src1 | src2);
INSTPAT("??????? ????? ????? 001 ????? 01000 11", sh     , S, Mw(src1 + imm, 2, src2));
INSTPAT("0100000 ????? ????? 101 ????? 00100 11", srai   , I, if((imm - 1024) < 32) R(rd) = ((sword_t)src1 >> (imm - 1024)));
INSTPAT("??????? ????? ????? 111 ????? 00100 11", andi   , I, R(rd) = src1 & imm);
INSTPAT("0000000 ????? ????? 001 ????? 01100 11", sll    , R, R(rd) = src1 << (src2 % 32));
INSTPAT("0000000 ????? ????? 111 ????? 01100 11", and    , R, R(rd) = src1 & src2);
INSTPAT("??????? ????? ????? 100 ????? 00100 11", xori   , I, R(rd) = src1 ^ imm);
INSTPAT("??????? ????? ????? 110 ????? 00100 11", ori    , I, R(rd) = src1 | imm);
INSTPAT("??????? ????? ????? 101 ????? 11000 11", bge    , B, if((sword_t)src1 >= (sword_t)src2) s->dnpc = s->pc + imm);
INSTPAT("??????? ????? ????? ??? ????? 01101 11", lui    , U, R(rd) = imm);
INSTPAT("000000? ????? ????? 101 ????? 00100 11", srli   , I, if(imm < 32) R(rd) = (src1 >> imm));
INSTPAT("??????? ????? ????? 111 ????? 11000 11", bgeu   , B, if(src1 >= src2) s->dnpc = s->pc + imm);
INSTPAT("000000? ????? ????? 001 ????? 00100 11", slli   , I, if(imm < 32) R(rd) = (src1 << imm ));
INSTPAT("0000001 ????? ????? 000 ????? 01100 11", mul    , R, R(rd) = src1 * src2);
INSTPAT("0000001 ????? ????? 100 ????? 01100 11", div    , R, R(rd) = src1 / src2);
INSTPAT("0000001 ????? ????? 110 ????? 01100 11", rem    , R, R(rd) = (sword_t)src1 % (sword_t)src2);
INSTPAT("??????? ????? ????? 100 ????? 11000 11", blt    , B, if((sword_t)src1 < (sword_t)src2) s->dnpc = s->pc + imm);
INSTPAT("0000000 ????? ????? 010 ????? 01100 11", slt    , R, R(rd) = (sword_t)src1 < (sword_t)src2);
INSTPAT("??????? ????? ????? 001 ????? 00000 11", lh     , I, R(rd) = SEXT(Mr(src1 + imm, 2), 16));
INSTPAT("??????? ????? ????? 000 ????? 00000 11", lb     , I, R(rd) = SEXT(Mr(src1 + imm, 2), 8));
INSTPAT("??????? ????? ????? 101 ????? 00000 11", lhu    , I, R(rd) = Mr(src1 + imm, 2));
INSTPAT("0000001 ????? ????? 001 ????? 01100 11", mulh   , R, R(rd) = (((int64_t)(sword_t)src1 * (int64_t)(sword_t)src2) >> 32));
INSTPAT("0000001 ????? ????? 011 ????? 01100 11", mulhu  , R, R(rd) = (((uint64_t)src1 * (uint64_t)src2) >> 32));
INSTPAT("0000001 ????? ????? 111 ????? 01100 11", remu   , R, R(rd) = (src1 % src2));
INSTPAT("0000001 ????? ????? 101 ????? 01100 11", divu   , R, R(rd) = (src1 / src2));
INSTPAT("0100000 ????? ????? 101 ????? 01100 11", sra    , R, R(rd) = ((sword_t)src1 >> ((sword_t)src2 - 32)));
INSTPAT("0000000 ????? ????? 101 ????? 01100 11", srl    , R, R(rd) = (src1 >> (src2 - 32)));
INSTPAT("??????? ????? ????? 110 ????? 11000 11", bltu   , B, if(src1 < src2) s->dnpc = s->pc + imm);

INSTPAT("0000000 00001 00000 000 00000 11100 11", ebreak , N, NEMUTRAP(s->pc, R(10))); // R(10) is $a0
INSTPAT("??????? ????? ????? ??? ????? ????? ??", inv    , N, INV(s->pc));
```

## 2.3  程序，运行时环境与AM

### 阅读Makefile和通过批处理模式运行NEMU

主要需要阅读的makefile有`abstract-machine/scripts/platform/nemu.mk`，`abstract-machine/scripts/isa/riscv.mk`，`abstract-machine/scripts/riscv32-nemu.mk`，`abstract-machine/Makefile`这四个。

批处理模式通过阅读`nemu/src/monitor/monitor.c`中的`parse_args`函数可知道需要传递-b参数给nemu，在`abstract-machine/scripts/platform/nemu.mk`中添加上`NEMUFLAGS += -b`即可。

### 实现字符串处理函数

通过man查询一下函数的行为，实现后可以找个glibc的标准实现看看，不过现在glibc的实现有些太复杂了，很多宏的使用，不容易读懂。

这里贴出我的代码。

```c
size_t strlen(const char *s) {
  size_t len = 0;
  while (*s++ != '\0') {
    len++;
  }
  return len;
}

char *strcpy(char *dst, const char *src) {
  char *d = dst;
  
  while ((*d++ = *src++) != 0);

  return dst;
}

char *strncpy(char *dst, const char *src, size_t n) {
  char *d = dst;

  while (n) {
    if ((*d = *src) != 0) src++;
    ++d;
    --n;
  }

  return dst;
}

char *strcat(char *dst, const char *src) {
    char *d = dst;

    while (*d++);
    --d;
    while ((*d++ = *src++) != 0);

    return dst;
}

int strcmp(const char *s1, const char *s2) {
  while (*((char *)s1) == *((char *)s2)) {
    if (!*s1++) {
        return 0;
    }
    ++s2;
  }

  return (*((char *)s1) < *((char *)s2)) ? -1 : 1;
}

int strncmp(const char *s1, const char *s2, size_t n) {
  while (n && (*((char *)s1) == *((char *)s2))) {
		if (!*s1++) {
			return 0;
		}
		++s2;
		--n;
	}

	return (n == 0) ? 0 : (*((char *)s1) - *((char *)s2));
}

void *memset(void *s, int c, size_t n) {
  char *tmp = (char *)s;

  for (size_t i = 0; i < n; i++) {
    tmp[i] = c;
  }

  return s;
}


/**
 * 此函数处理内存重叠情况
*/
void *memmove(void *dst, const void *src, size_t n) {
    char *d = (char *)dst;
    const char *s = (const char *)src;

    if (s >= d) {
        while (n) {
            *d++ = *s++;
            --n;
        }
    } else {
        while (n) {
            --n;
            d[n] = s[n];
        }
    }

    return dst;
}

/**
 * 此函数不处理内存重叠，涉及内存重叠使用memmove
*/
void *memcpy(void *out, const void *in, size_t n) {
  char *o = (char *)out;
  const char *i = (const char *)in;

  while (n--) {
    *o++ = *i++;
  }

  return out;
}

/**
 * 返回-1表示s1 < s2, 返回1表示s1 > s2, 返回0表示相等
*/
int memcmp(const void *s1, const void *s2, size_t n) {
    const char *p1 = s1;
    const char *p2 = s2;

    for (size_t i = 0; i < n; i++) {
        if (p1[i] < p2[i]) {
            return -1;
        } else if (p1[i] > p2[i]) {
            return 1;
        }
    }
    return 0;
}

#endif
```

### 实现sprintf函数

这个函数的实现需要了解`stdarg.h`，记得给`out`加上`'\0'`即可。

在学习`stdarg.h`时，我看到一篇很好的文章，讲解了这几个宏的简易实现，注意栈的地址由高到底，形参从右到左压栈。

```c
#define va_list char*
#define va_start(ap,arg) (ap=(va_list)&arg+sizeof(arg))
#define va_arg(ap,t) (*(t*)((ap+=sizeof(t))-sizeof(t)))
#define va_end(ap) (ap=(va_list)0)
```

- va_list是一个指针，选择void*或char*;
- va_start将va_list指向最后一个具名参数后面的位置，即第一个不确定参数的位置；
- va_arg获取当前参数的值，同时将指针指向下一个参数；
- va_end将指针置为0。

我的实现结果如下，实现了一下`vsnprintf`，`stdio.h`里的其他几个函数都可以调用它。

```c
int sprintf(char *out, const char *fmt, ...) {
  va_list ap;
  va_start(ap, fmt);
  
  int val = vsnprintf(out, 1024, fmt, ap);
  va_end(ap);

  return val;
}

int vsnprintf(char *out, size_t n, const char *fmt, va_list ap) {
    char *start = out;
    while (n-- && *fmt != '\0') {
        if (*fmt == '%') {
            fmt++;
            if (*fmt == 's') {
                char *tmp_s = va_arg(ap, char*);
                while (*tmp_s != '\0') {
                    *out++ = *tmp_s++;
                }
            }
            else if (*fmt == 'd') {
                int tmp_int = va_arg(ap, int);
                if (tmp_int < 0) {
                    *out++ = '-';
                    tmp_int = -1 * tmp_int;
                }
                int number = tmp_int;
                int len  = 0;
                do {
                    number /= 10;
                    len++;
                } while (number);
                out = out + len - 1;
                int tmp_len = len;
                while (tmp_len--) {
                    int tmp = tmp_int % 10;
                    *out-- = tmp + 48;
                    tmp_int /= 10;
                }
                out += (len+1);
            }
            else if (*fmt == '%') {
                *out++ = '%';
            }
            else if (*fmt == 'c') {
                char tmp_char = va_arg(ap, int);
                *out++ = tmp_char;
            }
            else {
                return -1;
            }
        }
        else {
            *out++ = *fmt;
        }
        fmt++;
    }
    *out = '\0';
    return out - start;
}
```

## 2.4 基础设施

### 实现iringbuf

模仿nemu相关代码即可。

```c
#include <common.h>
#include <utils.h>

typedef struct iringbuf
{
  vaddr_t pcs[20];
  uint32_t insts[20];
  uint32_t iring_rf;
  uint32_t iring_wf;
}iringbuf;

iringbuf irb;

void iringbuf_write_inst(vaddr_t pc, uint32_t inst) {
  irb.pcs[irb.iring_wf] = pc;
  irb.insts[irb.iring_wf] = inst;
  irb.iring_wf = (irb.iring_wf + 1) % 20;
  if (irb.iring_wf == irb.iring_rf)
    irb.iring_rf = (irb.iring_rf + 1) % 20;
}

void iringbuf_display() {
#ifdef CONFIG_ITRACE
  char logbuf[64];
  while (irb.iring_rf != irb.iring_wf) {
    // 存储pc和指令内容到缓冲区
    char *p = logbuf;
    if(irb.iring_rf + 1 == irb.iring_wf) {
      p += snprintf(p, 8, "--->");
    } else {
      memset(p, ' ', 4);
      p += 4;
    }
    p += snprintf(p, sizeof(logbuf), FMT_WORD ":", irb.pcs[irb.iring_rf]);
    uint8_t *inst = (uint8_t *)&irb.insts[irb.iring_rf];
    for (int j = 3; j >= 0; j--) {
      p += snprintf(p, 4, " %02x", inst[j]);
    }
    memset(p, ' ', 4);
    p += 4;
    // 解析指令对应的汇编语句
    void disassemble(char *str, int size, uint64_t pc, uint8_t *code, int nbyte);
    disassemble(p, logbuf + sizeof(logbuf) - p,
      irb.pcs[irb.iring_rf], (uint8_t *)&irb.insts[irb.iring_rf], 4);
    Log("%s\n", logbuf); 
    irb.iring_rf = (irb.iring_rf + 1) % 20; 
  }
#endif
}
```

我把存iringbuf放在了取指令之后，把显示iringbuf放在了内存访问越界之后。

### 实现mtrace

在`paddr_read()`和`paddr_write()`中实现相关代码，并在`nemu/Kconfig`中实现一个打开CONFIG_MTRACE=1的选项

```c
word_t paddr_read(paddr_t addr, int len) {
  IFDEF(CONFIG_MTRACE, Log("read in address = " FMT_PADDR ", len = %d\n", addr, len));
  //
}

void paddr_write(paddr_t addr, int len, word_t data) {
  IFDEF(CONFIG_MTRACE, Log("write in address = " FMT_PADDR ", len = %d, data = " FMT_WORD "\n", addr, len, data));
  //
}

```

```
config MTRACE
depends on TRACE && TARGET_NATIVE_ELF && ENGINE_INTERPRETER
bool "Enable memory tracer"
default y
```

### 消失的符号

> 我们在`am-kernels/tests/cpu-tests/tests/add.c`中定义了宏`NR_DATA`, 同时也在`add()`函数中定义了局部变量`c`和形参`a`, `b`, 但你会发现在符号表中找不到和它们对应的表项, 为什么会这样? 思考一下, 什么才算是一个符号(symbol)?

宏不是变量，会在预处理阶段展开。局部变量和行参会在`.stack`段。

### 寻找"Hello World!"

> 在Linux下编写一个Hello World程序, 编译后通过上述方法找到ELF文件的字符串表, 你发现"Hello World!"字符串在字符串表中的什么位置? 为什么会这样?

![image-20231110141933844](C:\Users\xuliz\AppData\Roaming\Typora\typora-user-images\image-20231110141933844.png)

在`.rodata`段，因为是常量。

### 实现ftrace

主要难点在于解析elf文件，原因是对elf文件不熟悉，手册内容太长了，所以明白整体结构后，需要用什么字段直接上搜索引擎查，查到之后再去手册里搜索验证即可。

```c
/**
 * 解析elf文件并存入symbol_tables
*/
void parse_elf(const char *elf_file) {
    if (elf_file == NULL)
        return;
    // 打开ELF文件
    FILE *fp = fopen(elf_file, "rb");
    Assert(fp, "Can not open '%s'", elf_file);

    // 读取ELF header
    Elf32_Ehdr elf_header;
    if (fread(&elf_header, sizeof(Elf32_Ehdr), 1, fp) <= 0) {
        fclose(fp);
        exit(EXIT_FAILURE);
    }

    // 检查文件是否为ELF文件
    if (memcmp(elf_header.e_ident, ELFMAG, SELFMAG) != 0) {
        fprintf(stderr, "Not an ELF file\n");
        fclose(fp);
        exit(EXIT_FAILURE);
    }

    // 移动到Section header table,寻找字符表节
    fseek(fp, elf_header.e_shoff, SEEK_SET);
    Elf32_Shdr strtab_header;
    while (1) {
        if (fread(&strtab_header, sizeof(Elf32_Shdr), 1, fp) <= 0) {
            fclose(fp);
            exit(EXIT_FAILURE);
        }
        if (strtab_header.sh_type == SHT_STRTAB) {
            break;
        }
    }

    // 读取字符串表内容
    char *string_table = malloc(strtab_header.sh_size);
    fseek(fp, strtab_header.sh_offset, SEEK_SET);
    if (fread(string_table, strtab_header.sh_size, 1, fp) <= 0) {
        fclose(fp);
        exit(EXIT_FAILURE);
    }

    // 寻找符号表节
    Elf32_Shdr symtab_header;
    fseek(fp, elf_header.e_shoff, SEEK_SET);
    while (1) {
        if (fread(&symtab_header, sizeof(Elf32_Shdr), 1, fp) <= 0) {
            fclose(fp);
            exit(EXIT_FAILURE);
        }
        if (symtab_header.sh_type == SHT_SYMTAB) {
            break;
        }
    }

    /* 读取符号表中的每个符号项 */ 

    fseek(fp, symtab_header.sh_offset, SEEK_SET);
    Elf32_Sym symbol;
    // 确定符号表的条数
    size_t num_symbols = symtab_header.sh_size / symtab_header.sh_entsize;
    // 分配内存用于存储符号表
    symbol_tables = malloc(num_symbols * sizeof(symbol_table));

    for (size_t i = 0; i < num_symbols; ++i) {
        if (fread(&symbol, sizeof(Elf32_Sym), 1, fp) <= 0 ) {
            fclose(fp);
            exit(EXIT_FAILURE);
        }

        // 判断符号是否为函数，并且函数的大小不为零
        if (ELF64_ST_TYPE(symbol.st_info) == STT_FUNC && symbol.st_size != 0) {
            // 从字符串表中获取符号名称
            const char *name = string_table  + symbol.st_name;
            // 存储符号信息到 symbol_table 结构体数组
            strncpy(symbol_tables[i].name, name, sizeof(symbol_tables[i].name) - 1);
            symbol_tables[i].addr = symbol.st_value;
            symbol_tables[i].info = symbol.st_info;
            symbol_tables[i].size = symbol.st_size;
        }
        symbol_tables_size = num_symbols;
    }

    // 关闭文件并释放内存
    fclose(fp);
    free(string_table);
}
```

修改指令。

```c
INSTPAT("??????? ????? ????? ??? ????? 11011 11", jal    , J, R(rd) = s->pc + 4;
   s->dnpc = s->pc + imm;
   IFDEF(CONFIG_FTRACE, {
    if (rd == 1) {
        call_trace(s->pc, s->dnpc);
    }
   }
   )
   );
  INSTPAT("??????? ????? ????? 000 ????? 11001 11", jalr   , I, R(rd) = s->pc + 4;
   s->dnpc = (src1 + imm) & (~1);
   IFDEF(CONFIG_FTRACE,{
    if (s->isa.inst.val == 0x00008067)
        ret_trace(s->pc);
    else if (rd == 1)//跳转到某个寄存器的位置时，因为rd默认为1
        call_trace(s->pc, s->dnpc);
   })
   );
```

在`parse_args`中添加一个参数，在Kconfig中添加一个FTRACE参数。

### 如何生成native的可执行文件

> 阅读相关Makefile, 尝试理解`abstract-machine`是如何生成`native`的可执行文件的.

```shell
ARCH_H := arch/$(ARCH).h
CFLAGS   += -O0 -MMD -Wall -Werror $(INCFLAGS) \
            -D__ISA__=\"$(ISA)\" -D__ISA_$(shell echo $(ISA) | tr a-z A-Z)__ \
            -D__ARCH__=$(ARCH) -D__ARCH_$(shell echo $(ARCH) | tr a-z A-Z | tr - _) \
            -D__PLATFORM__=$(PLATFORM) -D__PLATFORM_$(shell echo $(PLATFORM) | tr a-z A-Z | tr - _) \
            -DARCH_H=\"$(ARCH_H)\" \
            -fno-asynchronous-unwind-tables -fno-builtin -fno-stack-protector \
            -Wno-main -U_FORTIFY_SOURCE
```

传入不同的ARCH参数会调用不同的编译器。

### 奇怪的错误码

> 为什么错误码是`1`呢? 你知道`make`程序是如何得到这个错误码的吗?

1. **1**: 一般性错误。通常表示构建过程中发生了未指定的错误。
2. **2**: 语法错误。Makefile文件中存在语法错误。
3. **127**: 找不到要执行的命令。这通常是由于系统无法找到指定的命令或脚本。
4. **126**: 执行的命令无法执行。通常是由于权限问题或者脚本没有执行权限。
5. **139**: 与段错误有关的错误。这可能是由于程序在执行期间引起了段错误，可能是由于访问无效的内存地址等原因。

### 这是如何实现的?

> 为什么定义宏`__NATIVE_USE_KLIB__`之后就可以把`native`上的这些库函数链接到klib? 这具体是如何发生的? 尝试根据你在课堂上学习的链接相关的知识解释这一现象.

cc 编译时 使用 -fno-builtin 选项就可以使用自己编写的库函数同名函数

### 实现Difftest

```c
bool isa_difftest_checkregs(CPU_state *ref_r, vaddr_t pc) {
  int reg_num = ARRLEN(cpu.gpr);
  for (int i = 0; i < reg_num; i++) {
    if (ref_r->gpr[i] != cpu.gpr[i]) {
      return false;
    }
  }
  if (ref_r->pc != cpu.pc) {
    return false;
  }
  return true;
}
```

## 2.5输入输出

### 实现printf

写一个缓冲区保存临时字符串，调用一下`vnsprintf()`。

```c
char buf[1024];
void putch(char ch);

int printf(const char *fmt, ...) {
    va_list ap;
    va_start(ap, fmt);
    
    int val = vsnprintf(buf, 1024, fmt, ap);
    char *tmp = buf;
    while (*tmp != 0) {
        putch(*tmp);
        tmp++;
    }

    va_end(ap);
    return val;
}
```

### 实现IOE

> 提示： i8253计时器初始化时会分别注册`0x48`处长度为8个字节的端口, 以及`0xa0000048`处长度为8字节的MMIO空间, 它们都会映射到两个32位的RTC寄存器. CPU可以访问这两个寄存器来获得用64位表示的当前时间.

```c
void __am_timer_uptime(AM_TIMER_UPTIME_T *uptime) {
  uint32_t h = inl(RTC_ADDR + 4);
  uint32_t l = inl(RTC_ADDR);
  uptime->us = (uint64_t)l + ((uint64_t)h << 32);
}
```

### 看看NEMU跑多块

121分。

### 设置的一处错误

```c
static void rtc_io_handler(uint32_t offset, int len, bool is_write) {
  assert(offset == 0 || offset == 4);
  if (!is_write && offset == 4) { // 此处应该为offset == 0
    uint64_t us = get_time();
    rtc_port_base[0] = (uint32_t)us;
    rtc_port_base[1] = us >> 32;
  }
}
```

分析原因：应该在访问偏移量为0的时候获取时间，不然`__am_timer_uptime`函数在获取低位地址时会出现获取到上一次的低位地址的问题（上面代码写成了先获取高位地址，所以我在测试时并没有出现问题，这里可以通过改成先获取高位地址，或者改offset == 0）。

### 实现dtrace

```c
word_t map_read(paddr_t addr, int len, IOMap *map) {
  IFDEF(CONFIG_DTRACE, Log("read device %s : address in  = " FMT_PADDR ", len = %d\n", map->name , addr, len));
  //
}

void map_write(paddr_t addr, int len, word_t data, IOMap *map) {
  IFDEF(CONFIG_DTRACE, Log("write device %s : address in = " FMT_PADDR ", len = %d\n", addr, len));
  //
}
```

### 实现键盘

```c
void __am_input_keybrd(AM_INPUT_KEYBRD_T *kbd) {
    uint32_t k = inl(KBD_ADDR);
    kbd->keydown = (k & KEYDOWN_MASK ? true : false);
    kbd->keycode = k & ~KEYDOWN_MASK;
}
```

### 实现VGA

```c
// abstract-machine/am/src/platform/nemu/ioe/gpu.c
void __am_gpu_config(AM_GPU_CONFIG_T *cfg) {
  uint32_t wh_data = inl(VGACTL_ADDR);
  uint32_t h = wh_data & 0xffff;
  uint32_t w = wh_data >> 16;
  *cfg = (AM_GPU_CONFIG_T) {
    .present = true, .has_accel = false,
    .width = w, .height = h,
    .vmemsz = 0
  };
}

void __am_gpu_fbdraw(AM_GPU_FBDRAW_T *ctl) {
  int x = ctl->x, y = ctl->y, w = ctl->w, h = ctl->h;
  if (!ctl->sync && (w == 0 || h == 0)) return;
  uint32_t *pixels = ctl->pixels;
  uint32_t *fb = (uint32_t *)(uintptr_t)FB_ADDR;
  uint32_t screen_w = inl(VGACTL_ADDR) >> 16;
  for (int i = y; i < y+h; i++) {
    for (int j = x; j < x+w; j++) {
      fb[screen_w*i+j] = pixels[w*(i-y)+(j-x)];
    }
  }
  if (ctl->sync) {
    outl(SYNC_ADDR, 1);
  }
}

// nemu/src/device/vga.c
void vga_update_screen() {
  // TODO: call `update_screen()` when the sync register is non-zero,
  // then zero out the sync register
  uint32_t sync = vgactl_port_base[1];
  if (sync) {
    update_screen();
    vgactl_port_base[1] = 0;
  }
}
```

## 代码回顾

我们在这里列出目前大家应该读过并理解的代码, 供大家进行自我检讨:

- NEMU中除了`fixdep`, `kconfig`, 以及没有选择的ISA之外的全部已有代码(包括Makefile)
- `abstract-machine/am/`下与$ISA-nemu相关的, 除去CTE和VME之外的代码
- `abstract-machine/klib/`中的所有代码
- `abstract-machine/Makefile`和`abstract-machine/scripts/`中的所有代码
- `am-kernels/tests/cpu-tests/`中的所有代码
- `am-kernels/tests/am-tests/`中运行过的测试代码
- `am-kernels/benchmarks/microbench/bench.c`
- `am-kernels/kernels/`中的`hello`, `slider`和`typing-game`的所有代码

如果你发现自己不能理解这些代码的行为, 就赶紧看看吧. 多看一个文件, bug少调几天, 到了PA3你就会领教到了.
