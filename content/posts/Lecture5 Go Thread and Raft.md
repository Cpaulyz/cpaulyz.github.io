+++

title = "MIT6.824 Lecture5 Go Thread and Raft"

date = "2022-01-29"

tags = [
    "MIT6.824",
    "distributed system",
]

+++

# Lecture5 Go Thread and Raft

> 讲义：http://nil.csail.mit.edu/6.824/2020/notes/l-go-concurrency.txt
>
> 视频：https://www.bilibili.com/video/BV1qk4y197bB?p=5
>
> 前半部分主要介绍Go并发相关知识，后半部分介绍raft lab的实现，只针对前半部分的一些内容做了摘要。

## Go相关

### closure&wait group

good

```go
func main(){
  var wg sync.WaitGroup
  for i:=0; i< 5; i+ {
		wg.Add(1)
		go func(x int) {
			sendRPC(x)
			wg.Done()
		}
  }
	wg.Wait()
}

func sendRPC(i int){
	println(i)
}
```

bad

```go
func main() {
	var wg sync.WaitGroup
	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func() {
			sendRPC(i)
			wg.Done()
		}()
	}
	wg.Wait()
}

func sendRPC(i int) {
	println(i)
}
```



### Mutex

 加锁，确保在关键代码区互斥

bad

```go
import "time"

func main() {
	counter := 0
	for i := 0; i < 1000; i++ {
		go func() {
			counter = counter + 1
		}()
	}

	time.Sleep(1 * time.Second)
	println(counter)
}
```

good

```go
import "time"

func main() {
	counter := 0
	var mu sync.Mutex
	for i := 0; i < 1000; i++ {
		go func() {
			mu.Lock()
			defer mu.Unlock()
			counter = counter + 1
		}()
	}

	time.Sleep(1 * time.Second)
	mu.Lock()
	println(counter)
	mu.Unlock()
}
```

### 条件变量

* `func NewCond(l Locker) *Cond`

  使⽤锁 I 创建一个 *Cond。 Cond条件变量，总是要和锁结合使用。

* `func (c *Cond) Broadcast()`

  Broadcast唤醒所有等待c的线程。调⽤者在调⽤本⽅法时，建议（但并⾮必须）保持c.L的锁定。

* `func (c *Cond) Signal()`

  Signal唤醒等待c的⼀个线程（如果存在）。调⽤者在调⽤本⽅法时，建议（但并⾮必须）保持c.L的锁定。 发送通知给⼀个⼈。

* `func (c *Cond) Wait()`

  a) Wait⾃⾏解锁c.L并阻塞当前线程，在之后线程恢复执⾏时， Wait⽅法会在返回前锁定c.L。和其他系统不同， Wait除⾮被Broadcast或者Signal唤醒，不会主动返回。 ⼴播给所有⼈。

  b) 因为线程中Wait⽅法是第⼀个恢复执⾏的，⽽此时c.L未加锁。调⽤者不应假设Wait恢复时条件已满⾜，相反，调⽤者应在循环中等待。

```go
import "sync"
import "time"
import "math/rand"

func main() {
	rand.Seed(time.Now().UnixNano())

	count := 0
	finished := 0
	var mu sync.Mutex
	cond := sync.NewCond(&mu)

	for i := 0; i < 10; i++ {
		go func() {
			vote := requestVote()
			mu.Lock()
			defer mu.Unlock()
			if vote {
				count++
			}
			finished++
			cond.Broadcast()
		}()
	}

	mu.Lock()
	for count < 5 && finished != 10 {
		cond.Wait()
	}
	if count >= 5 {
		println("received 5+ votes!")
	} else {
		println("lost")
	}
	mu.Unlock()
}
```

pattern

```go
mu.Lock()
// do something that might affect the condition
cond.Broadcast() // or.signal() wake up one thread
mu.Unlock()

----

mu.Lock()
while condition == false {
	cond.Wait()
}
// now condition is true, and we have the lock
mu.Unlock()
```

Broadcast和signal的效果其实差不多，broadcast唤醒所有等待者，signal只唤醒一个。signal也许可以提高一些性能。

### channel 

可以用作wait group

```go
package main

func main() {
	done := make(chan bool)
	for i := 0; i < 5; i++ {
		go func(x int) {
			sendRPC(x)
			done <- true
		}(i)
	}
	for i := 0; i < 5; i++ {
		<-done
	}
}

func sendRPC(i int) {
	println(i)
}
```

代码中应该避免使用带buffer的chan，除非有非常清晰的目的。