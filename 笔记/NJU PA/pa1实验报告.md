## PA1实验报告

### 1、查阅ISA手册回答问题

#### 	riscv32有哪几种指令格式?

​	6中指令，分别是I型，R型，S型，U型，B型，J型。

#### 	LUI指令的行为是什么?

​	`lui rd,imm`将立即数的高20位加载到rd寄存器中，低12位补0。

#### 	mstatus寄存器的结构是怎么样的?

![image-20231024143237411](C:\Users\xuliz\AppData\Roaming\Typora\typora-user-images\image-20231024143237411.png)

### 2、shell命令

- **完成PA1的内容之后, `nemu/`目录下的所有.c和.h和文件总共有多少行代码? 你是使用什么命令得到这个结果的?**

```shell
find nemu/ -name "*.c" -o -name "*.h" | xargs cat | wc -l
```

![image-20231016124705284](C:\Users\xuliz\AppData\Roaming\Typora\typora-user-images\image-20231016124705284.png)



- **和框架代码相比, 你在PA1中编写了多少行代码? (Hint: 目前`pa0`分支中记录的正好是做PA1之前的状态, 思考一下应该如何回到"过去"?)**

![image-20231016124721425](C:\Users\xuliz\AppData\Roaming\Typora\typora-user-images\image-20231016124721425.png)



- **你可以把这条命令写入`Makefile`中, 随着实验进度的推进, 你可以很方便地统计工程的代码行数, 例如敲入`make count`就会自动运行统计代码行数的命令**

```makefile
count:
	find . -name "*.c" -o -name "*.h" -exec cat {} \; | grep -v "^$$" | wc -    l  
```



- **除去空行之外, `nemu/`目录下的所有`.c`和`.h`文件总共有多少行代码?**

```shell
find nemu/ -name "*.c" -o -name "*.h" -exec cat {} \; | grep -v "^$" | wc -l
```



### 3、从状态机的视角理解程序的运行

```c
(0, x, x) -> (1, 0, x) -> (2, 0, 0) -> (3, 0, 1) -> (4, 1, 1) -> (5, 1, 1)
-> (3, 1, 2) -> (4, 3, 2) -> (5, 3, 2) -> ... -> (4, 5050, 100) -> (5, 5050, 100)
```

### 4、PA1阶段1

#### 单步执行

```c
static int cmd_si(char *args) {
  int n = 0;
  char *param = strtok(args, " ");

  if (param == NULL) {
    n = 1;
  } else {
    sscanf(param, "%d", &n);
  }

  cpu_exec(n);
  return 0;
}
```

#### 打印寄存器

```c
static int cmd_info(char *args) {
  char *parma = strtok(args, " ");

  if (strcmp(parma, "r") == 0) {
    isa_reg_display();
  } else if (strcmp(parma, "w") == 0) {
    show_watchpoint();
  } else {
    printf("Unknow parma\n");
  }

  return 0;
}

// reg.c
void isa_reg_display() {
  for (int i = 0; i < 32; i++) {
    printf("%s : 0x%08x\n", regs[i], gpr(i));
  }
}
```

#### 扫描内存

```c
static int cmd_x(char *args) {
  char *parma1 = strtok(args, " ");
  args = parma1 + strlen(parma1) + 1;
  char *parma2 = strtok(args, " ");

  if (parma1 == NULL || parma2 == NULL) {
    printf("Unknow parma\n");
  } else {
    int n = 0;
    uint32_t addr = 0;
    bool success = false; 
    // 解析参数
    sscanf(parma1, "%d", &n);
    addr = expr(parma2, &success);
    // 扫描内存
    for (int i = 0; i < 4 * n; i++) {
      uint8_t val= vaddr_read(addr + i, 1);
      printf("%02x ",val);
    }
    printf("\n");
  }

  return 0;
}

```

我的代码上显示`vaddr_read`就是直接调用`paddr_read`,所以现在应该是还没有区分虚拟内存和物理内存，我只能访问`[0x80000000,0x8fffffff]`的物理地址，无法访问`0x1000000`。

### 5、PA1阶段2

#### 表达式求值

暂时是实现一个数字表达式的求值，认真读完`make_tokens`之后就明白了怎么存token。

```c
// make_token()
if (rules[i].token_type == TK_NOTYPE)
          break;
        
        // put all token to tokens
        tokens[nr_token].type = rules[i].token_type;
        switch (rules[i].token_type) {
          case TK_HEXNUM :
            strncpy(tokens[nr_token].str, substr_start, substr_len);
            tokens[nr_token].str[substr_len] = '\0';
            break;
          case TK_NUMBER :
            strncpy(tokens[nr_token].str, substr_start, substr_len);
            tokens[nr_token].str[substr_len] = '\0';
            break;
          case TK_REG :
            strncpy(tokens[nr_token].str, substr_start + 1, substr_len - 1);
            tokens[nr_token].str[substr_len - 1] = '\0';
            break;
          default: break;
        }
        nr_token++;

        break;
```

递归求值中，`check_parentheses(p, q)`函数的判断为：是否有括号在两边，且所有括号匹配无多余，且第一个`'('`匹配最后一个`')'`。

```c
// 检查括号是否全部匹配成功，且判断第一个'('是否匹配的是最后一个')'
bool check_pt_match(int p, int q) {
  int st[20], sp = 0;
  for (int i = p; i <= q; i++) {
    if (tokens[i].type == '(')
      st[sp++] = i;
    else if (tokens[i].type == ')') {
      if (sp == 0)
        return false;
      sp--;
    }
  }
  if (sp == 0 && st[0] == p)
      return true;
  return false;
}

// 判断括号是否在左右两边，且是对应的括号，且括号匹配正确
bool check_parentheses(int p, int q) {
  if (tokens[p].type == '(' && tokens[q].type == ')') {
    if (check_pt_match(p, q))
      return true;
  }
  return false;
}
```

之后就是怎么判断主运算符的问题了，根据讲义可以写出如下代码。

```c
// eval()
// find the host operator
    int op = -1;
    for (int i = p; i <= q; i++) {
      if (tokens[i].type != '+' && tokens[i].type != '-' &&
          tokens[i].type != '*' && tokens[i].type != '/' &&
          tokens[i].type != TK_EQ && tokens[i].type != TK_NOEQ &&
          tokens[i].type != TK_AND && tokens[i].type != TK_OR )
          continue;
      if (check_left(p, i - 1) && check_right(i + 1, q))
        continue;
      if (op == -1 || check_priority(op, i)) {
        op = i;
      } 
    }
```

实现负数的方法在后面实现解引用中有提到，给`'-'`添加一个新的`token`类型`TK_NEG`，在`expr()`函数中将类型修改一下，然后在`eval()`分裂时使用单侧分裂。

```c
// expr()
  for (int i = 0; i < nr_token; i++) {
    if (tokens[i].type == '*' && (i == 0 || check_certain_type(tokens[i - 1].type))) {
      tokens[i].type = TK_DEREF;
    }
    if (tokens[i].type == '-' && (i == 0 || check_certain_type(tokens[i - 1].type))) {
      tokens[i].type = TK_NEG;
    }
  }
// eval()
if (op == -1) {
      if (tokens[p].type == TK_DEREF) {
        word_t tmp_val = vaddr_read(eval(p + 1, q), 4);
        return tmp_val;
      }
      else if (tokens[p].type == TK_NEG) {
        return -1 * eval(p + 1, q);
      }
    }
```

#### 表达式生成器

```c
int numlen(int num)
{
    if(num == 0) return 1;
    int len = 0;        
    while (num) {
        len++;
        num /= 10;
    }	         
    return len;              
}

void gen_num() {
  int num = choose(1000);
  sprintf(buf+buf_index, "%d", num);
  buf_index += numlen(num);
}

void gen(char c){
    buf[buf_index++] = c;
}

void gen_rand_op() {
    char op[4] = {'+','-','*','/'};
    int op_index = choose(4);
    buf[buf_index++] = op[op_index];
}

void gen_rand_expr() {
    if (buf_index > 60000)
        return;
    switch(choose(3)) {
        case 0: 
            gen_num(); 
            break;
        case 1: 
            gen('(');
            gen_rand_expr();
            gen(')');
            break;
        default:
            gen_rand_expr(); 
            gen_rand_op(); 
            gen_rand_expr(); 
            break;
    }
}
```

修改`main`函数进行测试

```c
int main(int argc, char *argv[]) {
  /* Initialize the monitor. */
#ifdef CONFIG_TARGET_AM
  am_init_monitor();
#else
  init_monitor(argc, argv);
#endif

  /* Start engine. */
  //engine_start();

  //test expr
    char buf[65535];
    FILE *fptr;
    if ((fptr = fopen("/home/silly/ysyx-workbench/nemu/tools/gen-expr/input", "r")) == NULL)
    {
        printf("Error! opening file");
        exit(1);         
    }
    int cnt = 1;
    bool flag = false;
     while (fgets(buf, sizeof(buf), fptr) != NULL) {
         bool *success = false;
         char *param = strtok(buf, " ");
         word_t res = expr(param, success);
         word_t res1 = atoi(buf);
         if (res != res1) {
         printf("%d: %dfalse\n",cnt, res1);
            flag = true;
            break;
        }
        cnt++;
    }
    if (!flag) printf("ALL test accept![%d]\n",cnt - 1);
    fclose(fptr);
	exit(0);
  return is_exit_status_bad();
}

```

![image-20231024161954269](C:\Users\xuliz\AppData\Roaming\Typora\typora-user-images\image-20231024161954269.png)

### 6、PA1阶段3

#### 扩展表达式求值

实现一下识别十六进制数，寄存器，`'=='`，`'!='`，`'&&'`，`''||'`，指针解引用按照讲义给的框架实现一下就行了。

#### 实现监测点

先复习一下链表操作，实现`new_wp`和`free_wp`。

```c
// get a new wp in free wp_pool
WP* new_wp(char *expr, word_t val) {
  if (!free_)
    assert(0);
  // get free wp
  WP *wp = free_;
  free_ = free_->next;
  // insert in head
  wp->next = head;
  head = wp;
  wp->val = val;
  memmove(wp->expr, expr, strlen(expr));
  wp->expr[strlen(expr)] = '\0';
  return wp;
}

void free_wp(WP *wp) {
  WP *p = head, *q = head;
  while (p != NULL) {
    if (p->NO == wp->NO)
      break;
    q = p;
    p = p->next;
    q = q->next;
  }
  if (p->NO == head->NO) {
    p->val = 0;
    head = head->next;
    p->next = free_;
    free_ = p;
    wp->expr[0] = '\0';
  } else {
    p->val = 0;
    q->next = p->next;
    p->next = free_;
    free_ = p;
    wp->expr[0] = '\0';
  }
}
```

### 7.请解释gcc中的`-Wall`和`-Werror`有什么作用? 为什么要使用`-Wall`和`-Werror`?

```
-Wall，打开gcc的所有警告。
-Werror，它要求gcc将所有的警告当成错误进行处理。
```

使用它们可以让编译器帮你找到一些错误。
