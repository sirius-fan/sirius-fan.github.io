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
关闭后接受是0值，o发送 panic

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
