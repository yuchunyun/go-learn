package:  sync

sync.Cond 对标 同步原语“条件变量”，它可以阻塞一个，或同时阻塞多个线程，直到另一个线程 
1. 修改了共享变量
2. 通知该条件变量

首先，我们把概念搞清楚，条件变量的作用是控制多个线程对一个共享变量的读写。我们有三类主体：

- 共享变量：条件变量控制多个线程对该变量的读写
- 等待线程：被条件变量阻塞的线程，有一个或多个
- 更新线程：更新共享变量，并唤起一个或多个等待线程

> 比喻：KTV里只有一个话筒，N个人都想唱歌，但一个话筒只能被一个人使用。这个话筒就是共享变量，这个N-1个人就是等待线程，一个人告诉大家唱完了就是更新线程。而不需要在一个人唱歌的时候N-1个人都一直盯着话筒，他们只需要等待唱完了的人告诉他们就行了
# 应用场景

多个goroutine都需要操作共享资源时，为了维持数据统一会在操作前加锁，操作完成后解锁，这也会导致某一时刻只有一个goroutine正在操作资源，其他goroutine出于等待锁的状态。如果有的goroutine只是在共享资源满足某些条件时才需要操作资源呢？是不是这些goroutine也需要循环的去抢占锁，然后判断条件，然后解锁呢？

# 应用模拟
```
package main

import (
        "fmt"
        "sync"
        "time"
)

type Test struct {
        lock sync.RWMutex
        ID   int
        cond *sync.Cond
}

func main() {
        test := &Test{ID: 1}
        test.cond = sync.NewCond(&test.lock)
        go test.worker1()
        go test.worker2()
        test.write()
}

func (t *Test) write() {
        for t.ID != 100 {
                t.lock.Lock()
                t.ID += 1
                t.lock.Unlock()
                if t.ID%3 == 0 {
                        t.cond.Broadcast()
                }
                time.Sleep(time.Duration(2) * time.Second)
        }
        return
}

func (t *Test) worker1() {
        for {
                fmt.Println("in worker1")
                t.lock.Lock()
                t.cond.Wait()
                fmt.Printf("id equal %d, multiple of 3, so add 10!\n", t.ID)
                t.ID += 10
                t.lock.Unlock()
        }
        return
}

func (t *Test) worker2() {
        for {
                fmt.Println("in worker2")
                t.lock.Lock()
                t.cond.Wait()
                fmt.Printf("id equal %d, multiple of 3, so print!\n", t.ID)
                t.lock.Unlock()
        }
        return
}
```

- write: 负责持续变更共享变量t，当共享变量是3的倍数时通知也需要操作共享变量的goroutine
- worker1: 收到通知前挂起goroutine，收到通知后进行加10(写操作)
- worker2: 收到同之前挂起goroutine，收到通知后进行打印（读操作）

# 结构体
```
type Cond struct {
    noCopy noCopy               // noCopy可以嵌入到结构中，在第一次使用后不可复制,使用go vet作为检测使用
    L Locker                    // 根据需求初始化不同的锁，如*Mutex 和 *RWMutex
    notify  notifyList          // 通知列表,使用链表实现,调用Wait()方法的goroutine会被放入list中,每次唤醒,从这里取出
    checker copyChecker         // 复制检查,检查cond实例是否被复制
}

type notifyList struct {
    wait   uint32       //等待数量。无限自增，有新的goroutine等待时+1
    notify uint32       //通知数量。无限自增，有新的唤醒信号的时候+1
    lock   uintptr	//锁对象
    head   *sudog	//链表头
    tail   *sudog	//链表尾
}

type copyChecker uintptr

func (c *copyChecker) check() {
    if uintptr(*c) != uintptr(unsafe.Pointer(c)) &&
        !atomic.CompareAndSwapUintptr((*uintptr)(c), 0, uintptr(unsafe.Pointer(c))) &&
        uintptr(*c) != uintptr(unsafe.Pointer(c)) {
        panic("sync.Cond is copied")
    }
}


```
# 方法

## NewCond

创建一个带锁的条件变量，Locker 通常是一个 *Mutex 或 *RWMutex

```
func NewCond(l Locker) *Cond {
    return &Cond{L: l}
}
```


## Broadcast

唤醒所有因等待条件变量 c 阻塞的 goroutine
```
func (c *Cond) Broadcast() {
    c.checker.check()                       // 检查c是否是被复制的，如果是就panic
    runtime_notifyListNotifyAll(&c.notify)  // 唤醒等待队列中所有的goroutine
}
```
按照加入链表的顺序唤醒所有
```
func notifyListNotifyAll(l *notifyList) {
    // Fast-path: if there are no new waiters since the last notification
    // we don't need to acquire the lock.
    if atomic.Load(&l.wait) == atomic.Load(&l.notify) {
        return
    }

    // Pull the list out into a local variable, waiters will be readied
    // outside the lock.
    lock(&l.lock)
    s := l.head
    l.head = nil
    l.tail = nil

    // Update the next ticket to be notified. We can set it to the current
    // value of wait because any previous waiters are already in the list
    // or will notice that they have already been notified when trying to
    // add themselves to the list.
    atomic.Store(&l.notify, atomic.Load(&l.wait))
    unlock(&l.lock)

    // Go through the local list and ready all waiters.
    for s != nil {
        next := s.next
        s.next = nil
        readyWithTime(s, 4)
        s = next
    }
}
```
## Signal
  
  唤醒一个因等待条件变量 c 阻塞的 goroutine
```
func (c *Cond) Signal() {
    c.checker.check()                       // 检查c是否是被复制的，如果是就panic
    runtime_notifyListNotifyOne(&c.notify)  // 通知等待列表中的一个 
}
```
唤醒链表头
```
func notifyListNotifyOne(l *notifyList) {
    if atomic.Load(&l.wait) == atomic.Load(&l.notify) {
        return
    }

    lock(&l.lock)

    t := l.notify
    if t == atomic.Load(&l.wait) {
        unlock(&l.lock)
        return
    }

    atomic.Store(&l.notify, t+1)
    
    for p, s := (*sudog)(nil), l.head; s != nil; p, s = s, s.next {
        if s.ticket == t {
            n := s.next
            if p != nil {
                p.next = n
            } else {
                l.head = n
            }
            if n == nil {
                l.tail = p
            }
            unlock(&l.lock)
            s.next = nil
            readyWithTime(s, 4)
            return
        }
    }
    unlock(&l.lock)
}
```

## Wait
  
  自动解锁 c.L 并挂起 goroutine。只有当被 Broadcast 和 Signal 唤醒，Wait 才能返回，返回前会锁定 c.L
```
func (c *Cond) Wait() {  
    c.checker.check()                       // 检查c是否是被复制的，如果是就panic
    t := runtime_notifyListAdd(&c.notify)   // 将当前goroutine加入等待队列
    c.L.Unlock()                            // 解锁
    runtime_notifyListWait(&c.notify, t)    // 等待队列中的所有的goroutine执行等待唤醒操作
    c.L.Lock()                              //再次上锁
}
```
获取当前goroutine添加到链表尾
```
func notifyListWait(l *notifyList, t uint32) {
    lock(&l.lock)

    // Return right away if this ticket has already been notified.
    if less(t, l.notify) {
        unlock(&l.lock)
        return
    }

    // Enqueue itself.
    s := acquireSudog()
    s.g = getg()
    s.ticket = t
    s.releasetime = 0
    t0 := int64(0)
    if blockprofilerate > 0 {
        t0 = cputicks()
        s.releasetime = -1
    }
    if l.tail == nil {
        l.head = s
    } else {
        l.tail.next = s
    }
    l.tail = s
    goparkunlock(&l.lock, "semacquire", traceEvGoBlockCond, 3)
    if t0 != 0 {
        blockevent(s.releasetime-t0, 2)
    }
    releaseSudog(s)
}
```
> 若没有Wait()，通知后也不会报错

> 在调用 Signal，Broadcast 之前，应确保目标 Go 程序进入 Wait 阻塞状态

> 条件变量并不是被用来共享资源的，它是用于协调想要访问共享资源的那些线程的。当共享资源的状态发生变化时，它可以被用来通知被互斥锁阻塞的线程

> 条件变量在这里的最大优势就是在效率方面的提升。当共享资源的状态不满足条件的时候，想操作它的线程再也不用循环往复地做检查了，只要等待通知就好了


```
for !condition{
    c.Wait()
}
```
挂起当前的goroutine，直到有signal或者broadcast给它
```
for !condition{
    continue
    //time.sleep(time.second*3)
{
```
这样实际上cpu还是被当前的goroutine占据执行

# 特点
- 不能被复制
  
  因为 Cond 内部维护着一个所有等待在这个 Cond 的 Go 程队列，如果 Cond 允许值传递，则这个队列在值传递的过程中会进行复制，导致在唤醒 goroutine 的时候出现错误。
- 唤醒顺序

  FIFO
  
  也有资料说这种效率不高，可以换成随机唤醒

# 示例
```
package main

import (
    "fmt"
    "sync"
    "time"
)

var locker = new(sync.Mutex)
var cond = sync.NewCond(locker)

func main() {
    for i := 0; i < 10; i++ {
        go func(x int) {
            cond.L.Lock()         //获取锁
            defer cond.L.Unlock() //释放锁
	    fmt.Println(“add to list”,x)
            cond.Wait()           //等待通知，阻塞当前 goroutine
            
            // do something. 这里仅打印
            fmt.Println(x)
        }(i)
    }   
    time.Sleep(time.Second * 1)	// 睡眠 1 秒，等待所有 goroutine 进入 Wait 阻塞状态
    fmt.Println("Signal...")
    cond.Signal()               // 1 秒后下发一个通知给已经获取锁的 goroutine
    time.Sleep(time.Second * 1)
    fmt.Println("Signal...")
    cond.Signal()               // 1 秒后下发下一个通知给已经获取锁的 goroutine
    time.Sleep(time.Second * 1)
    cond.Broadcast()            // 1 秒后下发广播给所有等待的goroutine
    fmt.Println("Broadcast...")
    time.Sleep(time.Second * 1)	// 睡眠 1 秒，等待所有 goroutine 执行完毕
}
```

>	在for循环里，感觉只有第一个goroutine能够获取锁，其他的goroutine会在第一个goroutine执行完才能获取锁，其实不然，是wait()方法在搞事情



# 注意
- 调用 Wait() 函数前，需要先获得条件变量的成员锁，原因是需要互斥地变更条件变量的等待队列
