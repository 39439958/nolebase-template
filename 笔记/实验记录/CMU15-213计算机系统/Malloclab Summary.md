## Malloclab Summary

### 1、writeup

mm_init : 调用 mm_init 来执行任何必要的初始化，比如分配初始堆区域。如果在执行初始化时出现问题，返回值应为 -1，否则为0。

mm_malloc : malloc总是返回8字节对齐的指针。

### 2、隐式空闲链表+firstfit

```c
/*
 * mm-naive.c - The fastest, least memory-efficient malloc package.
 * 
 * In this naive approach, a block is allocated by simply incrementing
 * the brk pointer.  A block is pure payload. There are no headers or
 * footers.  Blocks are never coalesced or reused. Realloc is
 * implemented directly using mm_malloc and mm_free.
 *
 * NOTE TO STUDENTS: Replace this header comment with your own header
 * comment that gives a high level description of your solution.
 */
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>
#include <unistd.h>
#include <string.h>

#include "mm.h"
#include "memlib.h"

/*********************************************************
 * NOTE TO STUDENTS: Before you do anything else, please
 * provide your team information in the following struct.
 ********************************************************/
team_t team = {
    /* Team name */
    "ateam",
    /* First member's full name */
    "Harry Bovik",
    /* First member's email address */
    "bovik@cs.cmu.edu",
    /* Second member's full name (leave blank if none) */
    "",
    /* Second member's email address (leave blank if none) */
    ""
};

/* single word (4) or double word (8) alignment */
#define ALIGNMENT 8

/* rounds up to the nearest multiple of ALIGNMENT */
#define ALIGN(size) (((size) + (ALIGNMENT-1)) & ~0x7)
#define SIZE_T_SIZE (ALIGN(sizeof(size_t)))

/* some define about using free list*/
#define WSIZE 4
#define DSIZE 8
#define CHUNKSIZE (1 << 12)

#define MAX(x, y) ((x) > (y) ? (x) : (y))
//处理头部或尾部
#define PACK(size, alloc) ((size) | (alloc))
//获取p处的指；将p处的指改为val
#define GET(p) (*(unsigned int *)(p))
#define PUT(p, val) (*(unsigned int *)(p) = (val))
//获取头部的size和alloc字段
#define GET_SIZE(p) (GET(p) & ~0x7)
#define GET_ALLOC(p) (GET(p) & 0x1)
//根据bp获取它的头部指针或尾部指针
#define HDRP(bp) ((char *)(bp) - WSIZE)
#define FTRP(bp) ((char *)(bp) + GET_SIZE(HDRP(bp)) - DSIZE)
//根据bp获取它前一个结点的bp或后一个结点的bp
#define NEXT_BLKP(bp) ((char *)(bp) + GET_SIZE(((char *)(bp) - WSIZE)))
#define PREV_BLKP(bp) ((char *)(bp) - GET_SIZE(((char *)(bp) - DSIZE)))

/* 堆的头：指向序言段的后面 */
static char *heap_listp;

/* 定义函数 */
static void* extend_heap(size_t words);
static void* coalesce(void *bp);
static void* find_fit(size_t asize);
static void place(void* bp, size_t asize);
int mm_init(void);
void *mm_malloc(size_t size);
void mm_free(void *bp);
void *mm_realloc(void *ptr, size_t size);

/* 
 * extend_heap - 开辟一块新空间，返回 
 */
static void *extend_heap(size_t words) {
    char *bp;
    size_t size;

    size = (words % 2) ? (words + 1) * WSIZE : words * WSIZE;//双字对齐
    if ((long)(bp = mem_sbrk(size)) == (-1))
        return NULL;

    PUT(HDRP(bp), PACK(size, 0));//旧的结尾块此时变成了新申请的空闲块的头部
    PUT(FTRP(bp), PACK(size, 0));
    PUT(HDRP(NEXT_BLKP(bp)), PACK(0, 1));//新结尾块

    return coalesce(bp);//合并前面的空闲块（如果有）
}

/*
 * coalesce - 合并bp附近的空闲块
 */
static void* coalesce(void *bp) {
    int prev_alloc = GET_ALLOC(HDRP(PREV_BLKP(bp)));
    int next_alloc = GET_ALLOC(HDRP(NEXT_BLKP(bp)));
    size_t size = GET_SIZE(HDRP(bp));

    //case1 - 前面和后面块都分配了
    if (prev_alloc && next_alloc)
        return bp; 

    //case2 - 前面块没分配,后面块分配了
    else if (!prev_alloc && next_alloc) {
        size += GET_SIZE(HDRP(PREV_BLKP(bp)));
        PUT(FTRP(bp), PACK(size, 0));
        PUT(HDRP(PREV_BLKP(bp)), PACK(size, 0));
        bp = PREV_BLKP(bp);
    }

    //case3 - 后面块没分配，前面块分配了
    else if (prev_alloc && !next_alloc) {
        size += GET_SIZE(HDRP(NEXT_BLKP(bp)));
        PUT(HDRP(bp), PACK(size, 0));
        PUT(FTRP(bp), PACK(size, 0));//因为头部被设置过了，所以此时调用FTRP获取到的是后一个块的尾部，他们以及合并了
    }

    //case4 - 都没分配
    else {
        size += (GET_SIZE(HDRP(PREV_BLKP(bp))) + GET_SIZE(HDRP(NEXT_BLKP(bp))));
        PUT(HDRP(PREV_BLKP(bp)), PACK(size, 0));
        PUT(FTRP(NEXT_BLKP(bp)), PACK(size, 0));
        bp = PREV_BLKP(bp);
    }
    
    return bp;
}

/*
 * find_fit - 寻找合适的空闲块，这里使用first fit匹配策略
 */
static void* find_fit(size_t asize) {
    for (char *bp = heap_listp; GET_SIZE(HDRP(bp)) > 0; bp = NEXT_BLKP(bp)) {
        if (GET_ALLOC(HDRP(bp)) == 0 && GET_SIZE(HDRP(bp)) >= asize) {
            return bp;
        }
    }
    //查不到
    return NULL;
}


/*
 * place - 设置bp为已分配
 */
static void place(void* bp, size_t asize) {
    size_t csize = GET_SIZE(HDRP(bp));

    if ((csize - asize) >= 2 * DSIZE) {
        PUT(HDRP(bp), PACK(asize, 1));
        PUT(FTRP(bp), PACK(asize, 1));
        bp = NEXT_BLKP(bp);
        PUT(HDRP(bp), PACK(csize - asize, 0));
        PUT(FTRP(bp), PACK(csize - asize, 0));
    } else {
        PUT(HDRP(bp), PACK(csize, 1));
        PUT(FTRP(bp), PACK(csize, 1));
    }
}



/* 
 * mm_init - initialize the malloc package.
 */
int mm_init(void)
{
    if ((heap_listp = mem_sbrk(4 * WSIZE)) == (void *)-1)
        return -1;
    //设置序言块和结尾块
    PUT(heap_listp, 0);
    PUT(heap_listp + WSIZE, PACK(DSIZE, 1));
    PUT(heap_listp + DSIZE, PACK(DSIZE, 1));
    PUT(heap_listp + (3 * WSIZE), PACK(0, 1));
    heap_listp += (2 * WSIZE);

    if (extend_heap(CHUNKSIZE / WSIZE) == NULL)
        return -1;

    return 0;
}

/* 
 * mm_malloc - Allocate a block by incrementing the brk pointer.
 *     Always allocate a block whose size is a multiple of the alignment.
 */
void *mm_malloc(size_t size)
{
    char *bp;
    size_t asize;//对齐调整后的size
    size_t extendsize;

    if (size == 0)
        return NULL;

    if (size <= DSIZE)
        asize = 2 * DSIZE;
    else
        asize = DSIZE * (((size) + (DSIZE) + (DSIZE - 1)) / DSIZE);//按八字节向上取整，例如size = 13，则asize = （13 + 8 + 7）/ 8 * 8 = 24；   
    
    if ((bp = find_fit(asize)) != NULL) {
        place(bp, asize);
        return bp;
    }

    extendsize = MAX(asize, CHUNKSIZE);
    if ((bp = extend_heap(extendsize / WSIZE)) == NULL) {
        return NULL;
    }
    place(bp, asize);
    return bp;
}

/*
 * mm_free - Freeing a block does nothing.
 */
void mm_free(void *ptr)
{
    size_t size = GET_SIZE(HDRP(ptr));

    PUT(HDRP(ptr), PACK(size, 0));
    PUT(FTRP(ptr), PACK(size, 0));

    coalesce(ptr);
}

/*
 * mm_realloc - Implemented simply in terms of mm_malloc and mm_free
 */
void *mm_realloc(void *ptr, size_t size)
{
    if (ptr == NULL) 
        return mm_malloc(size);
    
    if (size == 0) {
        mm_free(ptr);
        return NULL;
    }

    size_t psize = GET_SIZE(HDRP(ptr));
    //空间不变
    if (psize == size) 
        return ptr;

    char * new_ptr = mm_malloc(size); 
    if (new_ptr == NULL)
        return NULL;       
    //请求扩大空间
    if (psize < size) {
        memcpy(new_ptr, ptr, psize - WSIZE);//此处我认为应该是-DSIZE，因为尾部应该可以不需要保证相同，但检测程序好像认为尾部也要复制过去
        mm_free(ptr);
        return new_ptr;
    }
    //请求缩小空间
    else {
        memcpy(new_ptr, ptr, size - WSIZE);//同上
        mm_free(ptr);
        return new_ptr;
    }
}
```

![image-20230613212657298](C:\Users\xuliz\AppData\Roaming\Typora\typora-user-images\image-20230613212657298.png)

匹配算法换成best fit

```c
static void* find_fit2(size_t asize) {
    char *bset_fit = NULL;
    size_t best_size = 0xffffffff;

    for (char *bp = heap_listp; GET_SIZE(HDRP(bp)) > 0; bp = NEXT_BLKP(bp)) {
        if (!GET_ALLOC(HDRP(bp)) && GET_SIZE(HDRP(bp)) > asize) {
            if (GET_SIZE(HDRP(bp)) < best_size) {
                best_size = GET_SIZE(HDRP(bp));
                bset_fit = bp;
            }
        }
    }
    return bset_fit;
}
```

分数还下降了，不过总算是把书上的实现自己调通了，接下来再写一个更好的版本吧。

![image-20230613213818300](C:\Users\xuliz\AppData\Roaming\Typora\typora-user-images\image-20230613213818300.png)

### 3、分离适配

![image-20230613221300249](C:\Users\xuliz\AppData\Roaming\Typora\typora-user-images\image-20230613221300249.png)

```c
/*
 * mm-naive.c - The fastest, least memory-efficient malloc package.
 * 
 * In this naive approach, a block is allocated by simply incrementing
 * the brk pointer.  A block is pure payload. There are no headers or
 * footers.  Blocks are never coalesced or reused. Realloc is
 * implemented directly using mm_malloc and mm_free.
 *
 * NOTE TO STUDENTS: Replace this header comment with your own header
 * comment that gives a high level description of your solution.
 */
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>
#include <unistd.h>
#include <string.h>

#include "mm.h"
#include "memlib.h"

/*********************************************************
 * NOTE TO STUDENTS: Before you do anything else, please
 * provide your team information in the following struct.
 ********************************************************/
team_t team = {
    /* Team name */
    "ateam",
    /* First member's full name */
    "Harry Bovik",
    /* First member's email address */
    "bovik@cs.cmu.edu",
    /* Second member's full name (leave blank if none) */
    "",
    /* Second member's email address (leave blank if none) */
    ""
};

/* single word (4) or double word (8) alignment */
#define ALIGNMENT 8

/* rounds up to the nearest multiple of ALIGNMENT */
#define ALIGN(size) (((size) + (ALIGNMENT-1)) & ~0x7)
#define SIZE_T_SIZE (ALIGN(sizeof(size_t)))

/* some define about using free list*/
#define WSIZE 4
#define DSIZE 8
#define CHUNKSIZE (1 << 12)
#define CLASSSIZE 20

#define MAX(x, y) ((x) > (y) ? (x) : (y))
//处理头部或尾部
#define PACK(size, alloc) ((size) | (alloc))
//获取p处的指；将p处的指改为val
#define GET(p) (*(unsigned int *)(p))
#define PUT(p, val) (*(unsigned int *)(p) = (val))
//获取头部的size和alloc字段
#define GET_SIZE(p) (GET(p) & ~0x7)
#define GET_ALLOC(p) (GET(p) & 0x1)
//根据bp获取它的头部指针或尾部指针
#define HDRP(bp) ((char *)(bp) - WSIZE)
#define FTRP(bp) ((char *)(bp) + GET_SIZE(HDRP(bp)) - DSIZE)
//根据bp获取它前一个结点的bp或后一个结点的bp
#define NEXT_BLKP(bp) ((char *)(bp) + GET_SIZE(((char *)(bp) - WSIZE)))
#define PREV_BLKP(bp) ((char *)(bp) - GET_SIZE(((char *)(bp) - DSIZE)))
//获取空闲块的前驱后继
#define GET_PREV(bp) ((unsigned int *)(long)GET(bp))
#define GET_NEXT(bp) ((unsigned int *)(long)GET((char *)bp + WSIZE))  

/* 堆的头：指向序言段的后面 */
static char *heap_listp;

//获取序号为i的链表头结点
#define GET_HEAD(num) ((unsigned int *)(long)(GET(heap_listp + WSIZE * num)))

/* 定义函数 */
static void* extend_heap(size_t words);
static void* coalesce(void *bp);
static void* find_fit(size_t asize);
static void place(void* bp, size_t asize);
static int search(size_t size);
static void insert(void *bp);
static void delete(void *bp);
int mm_init(void);
void *mm_malloc(size_t size);
void mm_free(void *bp);
void *mm_realloc(void *ptr, size_t size);

/* 
 * extend_heap - 开辟一块新空间，返回 
 */
static void *extend_heap(size_t words) {
    char *bp;
    size_t size;

    size = (words % 2) ? (words + 1) * WSIZE : words * WSIZE;//双字对齐
    if ((long)(bp = mem_sbrk(size)) == (-1))
        return NULL;

    PUT(HDRP(bp), PACK(size, 0));//旧的结尾块此时变成了新申请的空闲块的头部
    PUT(FTRP(bp), PACK(size, 0));
    PUT(HDRP(NEXT_BLKP(bp)), PACK(0, 1));//新结尾块

    return coalesce(bp);//合并前面的空闲块（如果有）
}

/*
 * coalesce - 合并bp附近的空闲块
 */
static void* coalesce(void *bp) {
    int prev_alloc = GET_ALLOC(HDRP(PREV_BLKP(bp)));
    int next_alloc = GET_ALLOC(HDRP(NEXT_BLKP(bp)));
    size_t size = GET_SIZE(HDRP(bp));

    //case1 - 前面和后面块都分配了
    if (prev_alloc && next_alloc) {
        insert(bp);
        return bp; 
    }
        

    //case2 - 前面块没分配,后面块分配了
    else if (!prev_alloc && next_alloc) {
        delete(PREV_BLKP(bp));
        size += GET_SIZE(HDRP(PREV_BLKP(bp)));
        PUT(FTRP(bp), PACK(size, 0));
        PUT(HDRP(PREV_BLKP(bp)), PACK(size, 0));
        bp = PREV_BLKP(bp);
        insert(bp);
    }

    //case3 - 后面块没分配，前面块分配了
    else if (prev_alloc && !next_alloc) {
        delete(NEXT_BLKP(bp));
        size += GET_SIZE(HDRP(NEXT_BLKP(bp)));
        PUT(HDRP(bp), PACK(size, 0));
        PUT(FTRP(bp), PACK(size, 0));//因为头部被设置过了，所以此时调用FTRP获取到的是后一个块的尾部，他们以及合并了
        insert(bp);
    }

    //case4 - 都没分配
    else {
        delete(PREV_BLKP(bp));
        delete(NEXT_BLKP(bp));
        size += (GET_SIZE(HDRP(PREV_BLKP(bp))) + GET_SIZE(HDRP(NEXT_BLKP(bp))));
        PUT(HDRP(PREV_BLKP(bp)), PACK(size, 0));
        PUT(FTRP(NEXT_BLKP(bp)), PACK(size, 0));
        bp = PREV_BLKP(bp);
        insert(bp);
    }
    
    return bp;
}

/*
 * find_fit - 寻找合适的空闲块，这里使用first fit匹配策略
 */
static void* find_fit(size_t asize) {
    int num = search(asize);
    char *bp;

    while (num < CLASSSIZE) {
        bp = GET_HEAD(num);
        while (bp) {
            if (GET_SIZE(HDRP(bp)) > asize)
                return (void *)bp;
            bp = GET_NEXT(bp);
        }
        num ++;
    }
    return NULL;
}


/*
 * place - 设置bp为已分配
 */
static void place(void* bp, size_t asize) {
    size_t csize = GET_SIZE(HDRP(bp));

    if ((csize - asize) >= 2 * DSIZE) {
        delete(bp);
        PUT(HDRP(bp), PACK(asize, 1));
        PUT(FTRP(bp), PACK(asize, 1));
        bp = NEXT_BLKP(bp);
        PUT(HDRP(bp), PACK(csize - asize, 0));
        PUT(FTRP(bp), PACK(csize - asize, 0));
        insert(bp);
    } else {
        delete(bp);
        PUT(HDRP(bp), PACK(csize, 1));
        PUT(FTRP(bp), PACK(csize, 1));
    }
}

/* 
 * search - 返回size大小属于哪个组 
 * 组别设置为:{16},{17~32},...,{2^22+1~INF}
 */
static int search(size_t size) {
    for (int i = 4; i <= 22; i++) {
        int tmp = 1 << i;
        if (size <= tmp) {
            return i - 4;
        }
    }
    return 19; 
}

/* 
 * insert - 将bp插入空闲链表
 */
static void insert(void *bp) {
    size_t size = GET_SIZE(HDRP(bp));
    int num = search(size);

    if (GET_HEAD(num) == NULL) {
        PUT(heap_listp + WSIZE * num, bp);
        PUT(bp, NULL);
        PUT((unsigned int *)bp + 1, NULL);
    } else {
        PUT((unsigned int *)bp + 1, GET_HEAD(num));
        PUT(bp, NULL);
        PUT(GET_HEAD(num), bp);
        PUT(heap_listp + WSIZE * num, bp);
    }
}

/* 
 * delete - 将bp从空闲链表中删除
 */
static void delete(void *bp) {
    size_t size = GET_SIZE(HDRP(bp));
    int num = search(size);

    if (GET_PREV(bp) == NULL && GET_NEXT(bp) == NULL) {
        PUT(heap_listp + WSIZE * num, NULL);
    }
    else if (GET_PREV(bp) != NULL && GET_NEXT(bp) == NULL) {
        PUT(GET_PREV(bp) + 1, NULL);
    }
    else if (GET_NEXT(bp) != NULL && GET_PREV(bp) == NULL) {
        PUT(heap_listp + WSIZE * num, GET_NEXT(bp));
        PUT(GET_NEXT(bp), NULL);
    }
    else if (GET_PREV(bp) != NULL && GET_NEXT(bp) != NULL){
        PUT(GET_PREV(bp) + 1, GET_NEXT(bp));
        PUT(GET_NEXT(bp), GET_PREV(bp));
    }

}


/* 
 * mm_init - initialize the malloc package.
 */
int mm_init(void)
{
    if ((heap_listp = mem_sbrk((4 + CLASSSIZE)* WSIZE)) == (void *)-1)
        return -1;
    
    //设置起始空块
    PUT(heap_listp, 0);
    heap_listp += WSIZE;
    //设置空闲块头指针
    for (int i = 0; i < CLASSSIZE; i++) {
        PUT(heap_listp + i * WSIZE, NULL);
    }
    //设置序言块
    PUT(heap_listp + CLASSSIZE * WSIZE, PACK(DSIZE, 1));
    PUT(heap_listp + (1 + CLASSSIZE) * WSIZE, PACK(DSIZE, 1));
    PUT(heap_listp + (2 + CLASSSIZE) * WSIZE, PACK(0, 1));
    
    if (extend_heap(CHUNKSIZE / WSIZE) == NULL)
        return -1;

    return 0;
}

/* 
 * mm_malloc - Allocate a block by incrementing the brk pointer.
 *     Always allocate a block whose size is a multiple of the alignment.
 */
void *mm_malloc(size_t size)
{
    char *bp;
    size_t asize;//对齐调整后的size
    size_t extendsize;

    if (size == 0)
        return NULL;

    if (size <= DSIZE)
        asize = 2 * DSIZE;
    else
        asize = DSIZE * (((size) + (DSIZE) + (DSIZE - 1)) / DSIZE);//按八字节向上取整，例如size = 13，则asize = （13 + 8 + 7）/ 8 * 8 = 24；   
    
    if ((bp = find_fit(asize)) != NULL) {
        place(bp, asize);
        return bp;
    }

    extendsize = MAX(asize, CHUNKSIZE);
    if ((bp = extend_heap(extendsize / WSIZE)) == NULL) {
        return NULL;
    }
    place(bp, asize);
    return bp;
}

/*
 * mm_free - Freeing a block does nothing.
 */
void mm_free(void *ptr)
{
    size_t size = GET_SIZE(HDRP(ptr));

    PUT(HDRP(ptr), PACK(size, 0));
    PUT(FTRP(ptr), PACK(size, 0));

    coalesce(ptr);
}

/*
 * mm_realloc - Implemented simply in terms of mm_malloc and mm_free
 */
void *mm_realloc(void *ptr, size_t size)
{
    if (ptr == NULL) 
        return mm_malloc(size);
    
    if (size == 0) {
        mm_free(ptr);
        return NULL;
    }

    size_t psize = GET_SIZE(HDRP(ptr));
    //空间不变
    if (psize == size) 
        return ptr;

    char * new_ptr = mm_malloc(size); 
    if (new_ptr == NULL)
        return NULL;       
    //请求扩大空间
    if (psize < size) {
        memcpy(new_ptr, ptr, psize - WSIZE);//此处我认为应该是-DSIZE，因为尾部应该可以不需要保证相同，但检测程序好像认为尾部也要复制过去
        mm_free(ptr);
        return new_ptr;
    }
    //请求缩小空间
    else {
        memcpy(new_ptr, ptr, size - WSIZE);//同上
        mm_free(ptr);
        return new_ptr;
    }
}
```

![image-20230614114913707](C:\Users\xuliz\AppData\Roaming\Typora\typora-user-images\image-20230614114913707.png)



优化一下realloc函数，得到87分，就不继续优化了。

```c
/*
 * mm_realloc - Implemented simply in terms of mm_malloc and mm_free
 */
void *mm_realloc(void *ptr, size_t size)
{
    if (ptr == NULL) 
        return mm_malloc(size);
    if (size == 0) {
        mm_free(ptr);
        return NULL;
    }
    size_t psize = GET_SIZE(HDRP(ptr));
    //空间不变
    if (psize == size) 
        return ptr;

    int next_alloc = GET_ALLOC(HDRP(NEXT_BLKP(ptr)));
    size_t next_size = GET_SIZE(HDRP(NEXT_BLKP(ptr)));
    
    if (size > psize && !next_alloc && next_size + psize >= size) {
        delete(NEXT_BLKP(ptr));
        PUT(HDRP(ptr), PACK(next_size + psize, 1));
        PUT(FTRP(ptr), PACK(next_size + psize, 1));
        return ptr;
    }
    else if (!next_size && size >= psize) {
        size_t extend_size = (size - psize);
        if((long)(mem_sbrk(extend_size)) == -1)            
            return NULL;                 
        PUT(HDRP(ptr), PACK(psize + extend_size, 1));        
        PUT(FTRP(ptr), PACK(psize + extend_size, 1));        
        PUT(HDRP(NEXT_BLKP(ptr)), PACK(0, 1));
        return ptr;
    }
    else {
        char * new_ptr = mm_malloc(size); 
        if (new_ptr == NULL)
            return NULL;
        memcpy(new_ptr, ptr, MIN(size, psize));
        mm_free(ptr);
        return new_ptr;
    }
}
```

![image-20230614134137761](C:\Users\xuliz\AppData\Roaming\Typora\typora-user-images\image-20230614134137761.png)
