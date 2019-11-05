### 线程创建

1. 继承Thread
2. 实现Runnable

### 线程安全

synchronize

### Thread api

1. currentThread() 返回当前代码段 被哪个线程调度

2. isAlive() 当前线程是否 存活

3. sleep() 当前线程休眠 x 毫秒

4. getId() 返回当前线程的id

5. interrupt() 中断线程标记 修改为 中断，

6. interrupted() 当前发起该interrupted()函数程序 所处 线程 ，中断标记返回 类方法

   调用会重置 中断标记

7. isInterrupted() 实例方法，返回实例线程 的中断标记， 调用不重置中断标记

8. ~~stop()~~ 过期弃用方法，

   缺点： 暴力停止线程，**中途暴力直接stop程序，容易造成数据不一致**。

9. ~~suspend()~~,~~resume()~~ 暂停，恢复线程。

    缺点：~~suspend()~~会对公共对象进行独占，在suspend期间，其他线程无法进行访问。**容易导致程序卡住的情况**。程序中途suspend导致数据不一致。

10. yield()  类方法， 主动让出cpu

11. setPriority() 线程优先级1-10

12. setDeamon() 设置是否为守护线程，所有用户线程结束后，守护线程结束。

