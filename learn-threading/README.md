

### 线程和进程

### Python线程与GIL

### 线程的创建

Python提供两个模块进行多线程的操作，分别是`thread`和`threading`，前者是比较低级的模块，用于更底层的操作，一般应有级别的开发不常用。后者则封装了更多高级的接口，类似java的多线程风格，提供`run`方法和`start`调用。

```python
import time
import threading

class MyThread(threading.Thread):
    def run(self):
        for i in range(5):
            print 'thread {}, @number: {}'.format(self.name, i)
            time.sleep(1)

def main():
    print "Start main threading"
    # 创建三个线程
    threads = [MyThread() for i in range(3)]
    # 启动三个线程
    for t in threads:
        t.start()

    print "End Main threading"


if __name__ == '__main__':
    main()
```

大概输入如下：

```
Start main threading
thread Thread-1, @number: 0
thread Thread-2, @number: 0
thread Thread-3, @number: 0
End Main threading
thread Thread-1, @number: 1
thread Thread-3, @number: 1
thread Thread-2, @number: 1
thread Thread-3, @number: 2
thread Thread-1, @number: 2
 thread Thread-2, @number: 2
thread Thread-2, @number: 3
thread Thread-1, @number: 3
thread Thread-3, @number: 3

```

每个线程都依次打印 0 - 3 三个数字，可是从输出的结果，我们看到线程并不是顺序的执行，而是三个线程之间相互交替执行。此外，我们的主线程执行结束，将会打印 `End Main threading`。从输出结果可以知道，主线程结束后，新建的线程还在运行。


### 线程`join`方法

上述的例子中，主线程结束了，子线程还在运行。如果需要主线程等待子线程执行完毕再退出，可是使用线程的`join`方法。join方法官网文档大概是

> `join(timeout)`方法将会等待直到进程结束。这将阻塞正在调用的线程，直到被调用join()方法的线程结束。

也就是说，主线程或者某个函数如果创建了子线程，只要调用了子线程的join方法，那么主线程就会被子线程所阻塞，直到子线程执行完毕再轮到主线程执行。这样就会在所有子线程执行完毕，才打印 `End Main threading`。只需要修改上面的`main`函数

```python

def main():
    print "Start main threading"

    threads = [MyThread() for i in range(3)]

    for t in threads:
        t.start()
    
    # 一次让新创建的线程执行 join
    for t in threads:
        t.join()

    print "End Main threading"

```

输入如下：

```
Start main threading
thread Thread-1, @number: 0
thread Thread-2, @number: 0
thread Thread-3, @number: 0
thread Thread-2, @number: 1
....
thread Thread-3, @number: 4
End Main threading

Process finished with exit code 0

```

所有子线程结束了才会执行也行`print "End Main threading"`。有人会这么想，如果在 `t.start()`之后join会怎么样？结果也能阻塞主线程，但是每个线程都是依次执行，变得有顺序了。其实join很好理解，就字面上的意思就是子线程 “加入”（join）主线程嘛。在CPU执行时间片段上“等于”主线程的一部分。在start之后join，也就是每个子线程由被后来新建的子线程给阻塞了，因此线程之间变得有顺序了。

借用[_moxie_](http://zipperary.com/2013/07/28/python-thread-join/)的总结：

> 1 join方法的作用是阻塞主进程（挡住，无法执行join以后的语句），专注执行多线程。
>
> 2 多线程多join的情况下，依次执行各线程的join方法，前头一个结束了才能执行后面一个。
>
> 3 无参数，则等待到该线程结束，才开始执行下一个线程的join。
>
> 4 设置参数后，则等待该线程这么长时间就不管它了（而该线程并没有结束）。不管的意思就是可以执行后面的主进程了


### 线程同步与互斥锁

线程之所以比进程轻量，其中一个原因就是他们共享内存。也就是各个线程可以平等的访问内存的数据，如果在短时间“同时并行”读取修改内存的数据，很可能造成数据不同步。例如下面的例子：

```python
count = 0
class MyThread(threading.Thread):
    def run(self):
        global count
        time.sleep(1)
        for i in range(100):
            count += 1
        print 'thread {} add 1, count is {}'.format(self.name, count)


def main():
    print "Start main threading"
    for i in range(10):
        MyThread().start()

    print "End Main threading"


main()
```
输出结果如下，十个线程，每个线程增加100，运算结果应该是1000：

Start main threading
End Main threading
thread Thread-6 add 1, count is 161thread Thread-1 add 1, count is 433
thread Thread-7 add 1, count is 482
thread Thread-2 add 1, count is 100
 thread Thread-9 add 1, count is 125

thread Thread-8 add 1, count is 335
 thread Thread-5 add 1, count is 533thread Thread-3 add 1, count is 533
 thread Thread-10 add 1, count is 261

thread Thread-4 add 1, count is 308

为了避免线程不同步造成是数据不同步，可以对资源进行加锁。也就是访问资源的线程需要获得锁，才能访问。threading模块正好提供了一个Lock功能，修改代码如下：

```python
# 创建锁
mutex = threading.Lock()

class MyThread(threading.Thread):
    def run(self):
        global count
        time.sleep(1)
        # 获取锁，修改资源
        if mutex.acquire():
            for i in range(100):
                count += 1
            print 'thread {} add 1, count is {}'.format(self.name, count)
            # 释放锁
            mutex.release()
```

### 死锁

有锁就可以方便处理线程同步问题，可是多线程的复杂度和难以调试的根源也来自于线程的锁。利用不当，甚至会带来更多问题。比如死锁就是需要避免的问题。


