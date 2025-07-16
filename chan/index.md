# channel 解读

# GO channel 详解

 本文是尽可能罗列channela知识点用来复习


## 0.设计哲学

```
"Don't communicate by sharing memory; share memory by communicating."
不要通过共享内存来通信，而要通过通信来共享内存。
```

Channel是Go实现CSP（Communicating Sequential Processes）并发模型的核心机制。

## 1. 定义与使用

#### 1）基本语法 

```go
// 声明channel
var ch chan int              // 声明一个传输int的channel
ch = make(chan int)          // 创建unbuffered channel
ch = make(chan int, 10)      // 创建buffered channel，容量为10

// 简化写法
ch := make(chan int)         // unbuffered
ch := make(chan int, 10)     // buffered

// 只读和只写的单向 channel
var readOnly <-chan int      // 只能接收
var writeOnly chan<- int     // 只能发送

```


#### 2) Channel的三种状态

````go
func channelStates() {
    var ch chan int
    
    // 1. nil channel
    fmt.Printf("nil channel: %v\n", ch == nil) // true
    
    // 2. open channel
    ch = make(chan int)
    fmt.Printf("open channel: %v\n", ch != nil) // true
    
    // 3. closed channel
    close(ch)
    // ch仍然不是nil，但已关闭
}
````

#### 3)发送操作 (ch <- value)

````go
func sendExample() {
    ch := make(chan int, 2)
    
    // 非阻塞发送（有缓冲空间）
    ch <- 1
    ch <- 2
    
    // 这个发送会阻塞（缓冲区满）
    go func() {
        ch <- 3 // 会阻塞直到有接收者
    }()
    
    // 接收一个值，释放缓冲空间
    <-ch
    
    time.Sleep(time.Millisecond) // 让goroutine有机会接受
}
````

#### 4) 接收操作 (<-ch)

````go
func receiveExample() {
    ch := make(chan int, 2)
    ch <- 1
    ch <- 2
    close(ch)
    
    // 方式1：基本接收
    value := <-ch
    fmt.Println("Received:", value) // 1
    
    // 方式2：检查channel是否关闭
    value, ok := <-ch
    fmt.Println("Received:", value, "OK:", ok) // 2, true
    
    // 从已关闭的channel接收
    value, ok = <-ch
    fmt.Println("Received:", value, "OK:", ok) // 0, false
}
````

#### 5) 关闭操作 (close(ch))
关闭后接受是0值，发送 panic

````go
func closeExample() {
    ch := make(chan int, 2)
    ch <- 1
    ch <- 2
    
    close(ch)
    
    // 仍然可以接收已发送的数据
    fmt.Println(<-ch) // 1
    fmt.Println(<-ch) // 2
    
    // 从关闭的channel接收零值
    fmt.Println(<-ch) // 0
    
    // 不能向已关闭的channel发送（会panic）
    // ch <- 3 // panic: send on closed channel
}
````

## 2.Select语句

### 1. 基本用法

````go
func selectExample() {
    ch1 := make(chan string)
    ch2 := make(chan string)
    
    go func() {
        time.Sleep(time.Second)
        ch1 <- "from ch1"
    }()
    
    go func() {
        time.Sleep(2 * time.Second)
        ch2 <- "from ch2"
    }()
    
    // select会选择第一个就绪的case
    select {
    case msg1 := <-ch1:
        fmt.Println("Received:", msg1)
    case msg2 := <-ch2:
        fmt.Println("Received:", msg2)
    case <-time.After(3 * time.Second):
        fmt.Println("Timeout")
    }
}
````

### 2. 非阻塞操作
对于channel，默认情况下，读/写 channel 都是阻塞模式，只有在 select 语句组成的多路复用分支中，与 channel 的交互会变成非阻塞模式

````go
func nonBlockingOperations() {
    ch := make(chan int, 1)
    
    // 非阻塞发送
    select {
    case ch <- 1:
        fmt.Println("Sent 1")
    default:
        fmt.Println("Channel full")
    }
    
    // 非阻塞接收
    select {
    case value := <-ch:
        fmt.Println("Received:", value)
    default:
        fmt.Println("Channel empty")
    }
}
````

### 3. Select的随机性

源码内存在伪随机函数确保随机选择

````go
func selectRandomness() {
    ch1 := make(chan int)
    ch2 := make(chan int)
    
    // 两个channel同时就绪时，select随机选择
    go func() {
        ch1 <- 1
    }()
    
    go func() {
        ch2 <- 2
    }()
    
    time.Sleep(time.Millisecond) // 确保两个goroutine都就绪
    
    select {
    case v := <-ch1:
        fmt.Println("From ch1:", v)
    case v := <-ch2:
        fmt.Println("From ch2:", v)
    }
}
````


## Channel的实现原理

### hchan结构体

````go
// runtime/chan.go
type hchan struct {
    qcount   uint           // 当前channel队列中的元素数量
    dataqsiz uint           // 环形队列的大小(当前 channel 能存放的元素容量)
    buf      unsafe.Pointer // 指向环形队列的指针
    elemsize uint16         // channel 元素类型的大小
    closed   uint32         // 是否关闭
    elemtype *_type         // 元素类型
    sendx    uint           // 发送索引(发送元素进入环形缓冲区的 index）
    recvx    uint           // 接收索引(接收元素所处的环形缓冲区的 index)
    recvq    waitq          // 接收等待队列(因接收而陷入阻塞的协程队列)
    sendq    waitq          // 发送等待队列(因发送而陷入阻塞的协程队列)
    lock     mutex          // 互斥锁
}

// 等待队列
type waitq struct {
    first *sudog         //队列头部
    last  *sudog         //队列尾部
}

// 等待的goroutine
type sudog struct {
    g      *g             // 等待的goroutine
    
    next   *sudog         // 链表下一个
    prev   *sudog         // 链表上一个
    elem   unsafe.Pointer // 数据元素?  读取/写入 channel 的数据的容器; TODO

    isSelect bool        //标识当前协程是否处在 select 多路复用的流程中
    
    c      *hchan         // channel
}
````



### 1. 发送流程

1. case1：写时存在阻塞读协程
• 加锁；
• 从阻塞度协程队列中取出一个 goroutine 的封装对象 sudog；
• 在 send 方法中，会基于 memmove 方法，直接将元素拷贝交给 sudog 对应的 goroutine；
• 在 send 方法中会完成解锁动作.

2. case2：写时无阻塞读协程但环形缓冲区仍有空间
• 加锁；
• 将当前元素添加到环形缓冲区 sendx 对应的位置；
• sendx++;
• qcount++;
• 解锁，返回.

3. case3：写时无阻塞读协程且环形缓冲区无空间
• 加锁；
• 构造封装当前 goroutine 的 sudog 对象；
• 完成指针指向，建立 sudog、goroutine、channel 之间的指向关系；
• 把 sudog 添加到当前 channel 的阻塞写协程队列中；
• park 当前协程；
• 倘若协程从 park 中被唤醒，则回收 sudog（sudog能被唤醒，其对应的元素必然已经被读协程取走）；
• 解锁，返回



````go
// 简化的发送流程
func chansend(c *hchan, ep unsafe.Pointer, block bool) bool {
    // 1. 检查channel状态
    if c == nil {
        if !block {
            return false
        }
        // 向nil channel发送会永久阻塞
        gopark(...)
    }
    
    lock(&c.lock)
    
    // 2. 检查是否已关闭
    if c.closed != 0 {
        unlock(&c.lock)
        panic("send on closed channel")
    }
    
    // 3. 如果有等待的接收者，直接传递
    if sg := c.recvq.dequeue(); sg != nil {
        send(c, sg, ep)
        unlock(&c.lock)
        return true
    }
    
    // 4. 如果缓冲区有空间，存入缓冲区
    if c.qcount < c.dataqsiz {
        // 计算存储位置
        qp := chanbuf(c, c.sendx)
        // 复制数据
        typedmemmove(c.elemtype, qp, ep)
        c.sendx++
        if c.sendx == c.dataqsiz {
            c.sendx = 0
        }
        c.qcount++
        unlock(&c.lock)
        return true
    }
    
    // 5. 缓冲区满，需要阻塞
    if !block {
        unlock(&c.lock)
        return false
    }
    
    // 6. 将当前goroutine加入发送等待队列
    gp := getg()
    mysg := acquireSudog()
    mysg.g = gp
    mysg.elem = ep
    mysg.c = c
    
    c.sendq.enqueue(mysg)
    unlock(&c.lock)
    
    // 7. 阻塞等待
    gopark(...)
    
    return true
}
````

### 2. 接收流程

1. 读nil会park挂起，直接死锁
2. channel关闭，读取直接解锁返回0值
3. 如果有写的协程在等待，
  - 加锁；
  - 从阻塞写协程队列中获取到一个写协程；
  - 倘若 channel 无缓冲区，则直接读取写协程元素，并唤醒写协程；
  - 倘若 channel 有缓冲区，则读取缓冲区头部元素，并将写协程元素写入缓冲区尾部后唤醒写写成；
  - 解锁，返回.
  
4. 读时无阻塞写协程且缓冲区有元素
  - 加锁；
  - 获取到 recvx 对应位置的元素；
  - recvx++
  - qcount--
  - 解锁，返回

5. 读时 无阻塞写协程且缓冲区无元素  即无数据可接收，需要阻塞
  - 加锁；
  - 构造 封装 当前goroutine的 sudog 对象；
  - 完成指针指向，建立 sudog、goroutine、channel 之间的指向关系；
  - 把 sudog 添加到当前 channel 的阻塞读协程队列中；
  - park 当前协程；
  - 倘若协程从 park 中被唤醒，则回收 sudog（sudog能被唤醒，其对应的元素必然已经被写入）；
  - 解锁，返回

````go
// 简化的接收流程
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    // 1. 检查nil channel
    if c == nil {
        if !block {
            return false, false
        }
        // 从nil channel接收会永久阻塞
        gopark(...)
    }
    
    lock(&c.lock)
    
    // 2. 检查已关闭且无数据的情况
    if c.closed != 0 && c.qcount == 0 {
        unlock(&c.lock)
        if ep != nil {
            // 设置零值
            typedmemclr(c.elemtype, ep)
        }
        return true, false
    }
    
    // 3. 如果有等待的发送者，直接接收
    if sg := c.sendq.dequeue(); sg != nil {
        recv(c, sg, ep)
        unlock(&c.lock)
        return true, true
    }
    
    // 4. 如果缓冲区有数据，从缓冲区接收
    if c.qcount > 0 {
        qp := chanbuf(c, c.recvx)
        if ep != nil {
            typedmemmove(c.elemtype, ep, qp)
        }
        typedmemclr(c.elemtype, qp)
        c.recvx++
        if c.recvx == c.dataqsiz {
            c.recvx = 0
        }
        c.qcount--
        unlock(&c.lock)
        return true, true
    }
    
    // 5. 无数据可接收，需要阻塞
    if !block {
        unlock(&c.lock)
        return false, false
    }
    
    // 6. 将当前goroutine加入接收等待队列
    gp := getg()
    mysg := acquireSudog()
    mysg.g = gp
    mysg.elem = ep
    mysg.c = c
    
    c.recvq.enqueue(mysg)
    unlock(&c.lock)
    
    // 7. 阻塞等待
    gopark(...)
    
    return true, true
}
````


## 常见陷阱和最佳实践

### 1. Goroutine泄露

````go
// ❌ 错误：可能导致goroutine泄露
func badPattern() {
    ch := make(chan int)
    
    go func() {
        // 如果没有接收者，这个goroutine会永久阻塞
        ch <- 1
    }()
    
    // 如果这里提前返回，goroutine就泄露了
    return
}

// ✅ 正确：使用context或close channel来避免泄露
func goodPattern(ctx context.Context) {
    ch := make(chan int)
    
    go func() {
        select {
        case ch <- 1:
            // 发送成功
        case <-ctx.Done():
            // 被取消，退出goroutine
            return
        }
    }()
    
    select {
    case result := <-ch:
        fmt.Println("Received:", result)
    case <-ctx.Done():
        return
    }
}
````

### 2. Channel方向

````go
// ✅ 使用单向channel提高类型安全
func producer() <-chan int {
    ch := make(chan int)
    go func() {
        defer close(ch)
        for i := 0; i < 10; i++ {
            ch <- i
        }
    }()
    return ch
}

func consumer(ch <-chan int) {
    for value := range ch {
        fmt.Println("Consumed:", value)
    }
}
````

### 3. 正确关闭Channel

````go
// ✅ 发送者负责关闭channel
func correctClose() {
    ch := make(chan int, 10)
    
    // 发送者
    go func() {
        defer close(ch) // 发送完毕后关闭
        for i := 0; i < 5; i++ {
            ch <- i
        }
    }()
    
    // 接收者
    for value := range ch { // range会在channel关闭时退出
        fmt.Println(value)
    }
}
````

### 4.  写异常
• 对于未初始化的 chan，写入操作会引发死锁；
• 对于已关闭的 chan，写入操作会引发 panic.

## 总结

Channel是Go并发编程的核心：

1. **设计哲学**：通过通信来共享内存
2. **内部结构**：基于环形缓冲区和等待队列
3. **操作语义**：发送、接收、关闭的详细行为
4. **使用模式**：生产者-消费者、Fan-out/in、工作池、管道
5. **性能考虑**：缓冲大小、内存使用、延迟vs吞吐量
6. **最佳实践**：避免泄露、正确关闭、类型安全

掌握Channel是写出高质量Go并发程序的关键。

