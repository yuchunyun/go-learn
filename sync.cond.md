package:  sync

Cond实现了一个条件变量，一个等待或宣布事件发生的goroutines的集合点

# 结构体
```
type Cond struct {
    noCopy noCopy               // noCopy可以嵌入到结构中，在第一次使用后不可复制,使用go vet作为检测使用
    L Locker                    // 根据需求初始化不同的锁，如*Mutex 和 *RWMutex
    notify  notifyList          // 通知列表,调用Wait()方法的goroutine会被放入list中,每次唤醒,从这里取出
    checker copyChecker         // 复制检查,检查cond实例是否被复制
}

type notifyList struct {
    wait   uint32       //无限自增，有新的goroutine等待时+1
    notify uint32       //无限自增，有新的唤醒信号的时候+1
    lock   uintptr
    head   unsafe.Pointer
    tail   unsafe.Pointer
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
## Signal
  
  唤醒一个因等待条件变量 c 阻塞的 goroutine
```
func (c *Cond) Signal() {
    c.checker.check()                       // 检查c是否是被复制的，如果是就panic
    runtime_notifyListNotifyOne(&c.notify)  // 通知等待列表中的一个 
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
  
  也有资料说这种效率不高，可以换成随机唤醒，但代码里是唤醒最后一个
  
```
func notifyListNotifyOne(l *notifyList) {

   if atomic.Load(&l.wait) == atomic.Load(&l.notify) { //step a
        return
   }

   lock(&l.lock) //step b

   t := l.notify
   if t == atomic.Load(&l.wait) {
        unlock(&l.lock)
        return
   }

   atomic.Store(&l.notify, t+1)  

   for p, s := (*sudog)(nil), l.head; s != nil; p, s = s, s.next { //step c
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
- 条件变量和锁结合使用，在并发时如果逻辑不严谨容易发生死锁，所以尽量不要使用条件变量，推荐用 sync.WaitGroup 来实现并发时 Go 程序间的同步。
