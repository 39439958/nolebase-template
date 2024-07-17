## Cachelab Summary

### part A

仔细阅读一下writeup，大概可以知道partA的任务是：

在cism.c中写代码模拟实现一下cache，要求的格式中参数[h，v]为可选，我们可以不管，最后实现成以下这样的操纵即可

```shell
linux> ./csim -s 4 -E 1 -b 4 -t traces/yi.trace

hits:4 misses:5 evictions:3
```

代码如下：

```c
#include "cachelab.h"
#include <stdio.h>
#include <stdlib.h>
#include <getopt.h>
#include <string.h>
#define DEBUG 0

static int s;//组号
static int E;//多少路，即每组有多少行
static int B;//块内偏移
static int hit;
static int miss;
static int eviction;
static int cur_time = 0;//执行LRU算法时使用，记录当前时间，每次执行一次操作加一次时间

static struct cache{
    unsigned vaild;
    unsigned tag;
    int accessTime;   //使用LRU算法时用到
}cache[32][32];
//我们直接设计成32组，每组32行，实际执行时用不到这么多

void parseOption(int argc, char** argv, char *fileName);//1.读取命令行参数并保存
void parseTraceAndExecute(char *fileName);//2.读取trace文件并将模拟执行每行
void exec(unsigned address);//3.执行流程

void parseOption(int argc, char** argv, char *fileName) {
    int option;
    while ((option = getopt(argc, argv, "s:E:b:t:")) != -1) {
        switch (option) {
            case 's':
                s = atoi(optarg);
                break;
            case 'E':
                E = atoi(optarg);
                break;
            case 'b':
                B = atoi(optarg);
                break;
            case 't':
                strcpy(fileName, optarg);
                break;
        }
    }
}

void parseTraceAndExecute(char *fileName) {
    freopen(fileName, "r", stdin);
    char cmd;
    unsigned address;
    int blocksize;
    while (~scanf(" %c %x,%d", &cmd, &address, &blocksize)) {
        if (DEBUG) printf("%c, %x\n", cmd, address);
        switch (cmd) {
            case 'L':
                exec(address);
                break;
            case 'M':
                exec(address);
            case 'S':
                exec(address);
                break;
        }
    }
}

void exec(unsigned address) {
    cur_time++;
	//获取当前地址的组号和tag
    unsigned mask = 0xffffffff >> (32 - s);
    unsigned cur_set = (address >> B) & mask;
    unsigned cur_tag = address >> (s + B);

   if (DEBUG) printf ("%x, %x\n", cur_set, cur_tag);

    for (int i = 0; i < E; i++) {
        if (cache[cur_set][i].vaild && cache[cur_set][i].tag == cur_tag) {
            hit++;
            if (DEBUG) printf("hit ++ !\n");
            cache[cur_set][i].accessTime = cur_time;
            return;
        }
    }
    miss++;
    if (DEBUG) printf("miss ++ !\n");
    for (int i = 0; i < E; i++) {
        if (!cache[cur_set][i].vaild) {
            cache[cur_set][i].accessTime = cur_time;
            cache[cur_set][i].vaild = 1;
            cache[cur_set][i].tag = cur_tag;
            return;
        }
    }
    eviction++;
    if (DEBUG) printf("eviction ++ !\n");
    int min_time = cache[cur_set][0].accessTime;
    int min_time_id = 0;
    for (int i = 1; i < E; i++) {
        if (cache[cur_set][i].accessTime < min_time) {
            min_time = cache[cur_set][i].accessTime;
            min_time_id = i;
        }
    }
    cache[cur_set][min_time_id].tag = cur_tag;
    cache[cur_set][min_time_id].accessTime = cur_time;
}

int main(int argc, char** argv) {
    // 读入命令行参数并分解参数
    char *fileName = malloc(30 * sizeof(char));
    parseOption(argc, argv, fileName);

    // 分解trace文件并模拟执行每行
    parseTraceAndExecute(fileName);    

    printSummary(hit, miss, eviction);
    return 0;
}

```

### part B

没有优化到满分，32x32和61x67用的8分组分块，64x64用的4分组分块

```c
void transpose_submit(int M, int N, int A[N][M], int B[M][N])
{
	if (M == 32) {
		for (int i = 0; i < 32; i+=8) {
			for (int j = 0; j < 32; j += 8) {
				for (int k = i; k < i+8; k++) {
					int t1 = A[k][j];
					int t2 = A[k][j + 1];
					int t3 = A[k][j + 2];
					int t4 = A[k][j + 3];
					int t5 = A[k][j + 4];
					int t6 = A[k][j + 5];
					int t7 = A[k][j + 6];
					int t8 = A[k][j + 7];
					B[j][k] = t1;
					B[j + 1][k] = t2;
					B[j + 2][k] = t3;
					B[j + 3][k] = t4;
					B[j + 4][k] = t5;
					B[j + 5][k] = t6;
					B[j + 6][k] = t7;
					B[j + 7][k] = t8;
				}
			}
		}
	} else if (M == 64) {
		for (int i = 0; i < 64; i+=4) {
			for (int j = 0; j < 64; j += 4) {
				for (int k = i; k < i+4; k++) {
					int t1 = A[k][j];
					int t2 = A[k][j + 1];
					int t3 = A[k][j + 2];
					int t4 = A[k][j + 3];
					B[j][k] = t1;
					B[j + 1][k] = t2;
					B[j + 2][k] = t3;
					B[j + 3][k] = t4;
				}
			}
		}
	} else if (M == 61) {	
		for (int i = 0; i < N; i += 8)
		{
		for (int j = 0; j < M; j += 8)
		{
			for (int k = i; k < i + 8 && k < N; k++)
			{
			for (int l = j; l < j + 8 && l < M; l++)
			{
				B[l][k] = A[k][l];
			}
			}
		}
		}
	}

}
```

![image-20230603231450002](C:\Users\xuliz\AppData\Roaming\Typora\typora-user-images\image-20230603231450002.png)

![image-20230603231512960](C:\Users\xuliz\AppData\Roaming\Typora\typora-user-images\image-20230603231512960.png)

![image-20230603231541933](C:\Users\xuliz\AppData\Roaming\Typora\typora-user-images\image-20230603231541933.png)
