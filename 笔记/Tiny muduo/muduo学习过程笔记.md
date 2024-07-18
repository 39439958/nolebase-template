# EventLoop

> 带有__thread属性得t_loopInThisThread怎么判断当前线程是否只有一个EventLoop对象？

__thread：用于声明一个变量为线程局部变量每个线程都有自己的独立副本，这些副本在不同线程之间是独立的，互不影响

```cpp
if (t_loopInThisThread)
{
    LOG_FATAL("Another EventLoop %p exists in this thread %d \n", t_loopInThisThread, threadId_);
}
else 
{
    t_loopInThisThread = this;
}
```

