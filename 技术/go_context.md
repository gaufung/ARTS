---
title: go in concurrency
date: 2018-06-23
status: public
---
# 1 Backgroud
Last week, I took a presentation on `An awesome go & brief introduction`. The topics  were basic as considering few folks in company with Golang backgroud.  The leader urged me to share more about go concurrency when he early overviewed the slide. Notes were as follow:
- `sync` package
- `goroutine secheduler` 
- `go concurrency patterns`
# 2 `sync` package
This package provides part of features those in other language synchronize area. You will be familiar with them if you get used to writing synchronize codes. 
## 2.1 Interface and Types
- Locker: 
An interface includes `Lock` and `Unlock` methods. 
- Mutex 
A mutual exclusion lock that implementing `Locker` interface.
- RWMutex
A reader/writer exclusion lock which can be held by a arbitrary number of readers or a single writer. Of course, it implements `Locker` interface. 
- WaitGroup
Wating a group goroutines to finish. `Add(delta int)` method increases the number of goroutine to wait. And `Done()` decreases it by one. `Wait()` will block until all goroutines have finished.
- Pool 
A pool contains a set of temporary object to reuse.  It is safe for multiple goroutines to be simultaneousely. It consist of   only one exposed  fields:
```go
type fool struct {
    New func() interface{}
}
```
- Map
It is used as  goroutine-safe `map[interface{}]interface{}` type and two common use cases:
1. writen once, and read many times. 
2. many goroutines read, write and overwrite.

## 2.2 dead lock
As a golang user, a Go's motto *Share memory by comunicating, don't communicate by sharing memory* is a buzzword. But Should we choose channel rather than primitives methods like `sync` package at any  circumstances？Here is your deecision tree.
![](./_image/decision_tree.png)
### example
All concurrent processes waiting on one another.
```go
type value struct {
    mu sync.Mutex
    value int
}
var wg sync.WaitGroup
printSum := func(v1, v2 *value){
    defer wg.Done()
    v1.mu.Lock()
    defer v1.mu.Unlock
    time.Sleep(1 * time.Second)
    
    v2.mu.Lock()
    defer v2.mu.Unlock
}
var a, b value
wg.Add(2)
go printSum(&a, &b) //goroutine 1
go printSum(&b, &a) //goroutine 2
wg.Wait()
```
In above case, go routine 1 locks `a` and tries to lock `b`, vice versa,  go routine 2. So all gouroutines are dead. Go runtime can detect it. 
[Dining phiolopher problem](https://en.wikipedia.org/wiki/Dining_philosophers_problem) is a well-known example to illustrate  synchronization issue. 
**Solutions**
- partial order to resources
All resources will be requested by a certain convention. So we change a little bit of `printSum` function.
```go
printSum := func(v1, v2 *value){
    values := make([]*value, 0, 2)
    if fmt.Sprintf("%p", v1) > fmt.Sprintf("%p", v2) {
        values[0]=v1
        values[1]=v2
    }else{
        values[0]=v2
        values[1]=v1
    }
    defer wg.Done()
    values[0].Lock()
    defer values[1].Unlock()
    time.Sleep(1 * time.Second)
    values[1].Lock()
    defer values[1].Unlock()
}
```
- Lock bigger resource
The deadlock situation rises from request different resource at different time. So we will permit one goroutine to lock bigger resources. Meanwhile, other goroutine only waiting them but do nothing.
- comunitcating with gorotines
It can be solved by channel, Seeing  later.

# 3 Goroutine Secheduler
There are 3 usual models for threading. 
- N:1  where several userspace threading are running on single OS thread.  
    - advantage: context switching is quite easy and quick. 
    - disadvantage: multi-core systems may be useless. And once a thread invokes system call, all other threads will be blocked.
*many coroutines libraries use this model, like gevent in python*
-  1:1 One executing thread maps to one OS Thread.
    - advantage: take full advantage of multi-core system.
    - disadvantage: context switching iw slow, since it has to trap though the OS
*many language thread libraries use this model, , like java.lang.Thread*
- M:N an arbitrary number of goroutines run on an arbitrary number of OS threads.

## 3.1 M, P and G entities

![](./_image/mgp.jpg)
**M**: The triangle in above illustration represents OS thread.
**G**: The circle in that represents a goroutine.
**P**: The rectangle in above represents a context for scheduling. That plays an important role in tranform N:1 scheuduler into M:N scheduler
## 3.2 How it works 

![](./_image/mgprunning.jpg)
In order to run goroutine, a **M** must attach to  context **P** . Several goroutines are in context **P**  queue, waiting to be executed by **M**.  In above illustration, there are two **M** machines which means two threads can run at the same time. The number of **P** is set on program startup by the **GOMAXPROCS**  environment variable or throguh the `runtime` function `GOMAXPROCES()`. Usually, the numeber equals to the cores in your computer. 

## 3.3 What if system call or blocked ?

![](./_image/syscall.jpg)

In above illustration left part, `G0` routines invoke `sys-call` method. Since a thread is blocked, it gives up its context **P** to make another thread **M1** run it. The syscalling thread holds on the `G0` goroutines until it finishs. After that, the **G0** continues joining other context's queue. 

# 4 Go Concurrency Patterns
## 4.1 Goroutine leaks
Though goroutine is cheap and easy to creat, it still do cost resources. Worse more, goroutine is not garbage collected by the runtime. Here are situations when terminate the goroutines:
- When it has finished its task.
- When it fails to carry on its work due to an unexpected error.
- When it's asked to stop work. 
The first two situations are controlled by your algorithm, what about cancellation?  Let's begin with a simple example of a goroutine leak.
```go
doWork = func (strings <- chan string)  <- chan interface{}{
    completed := make(chan interface{})
    go func(){
        defer close(completed)
        for s := range strings {
            // do soemthing
            fmt.Println(s)
        }
    }()
    return completed
}
doWork(nil)
```
Here we pass a `nil` channel into the `doWork`. Therefore, the `strings` channel will never receive a channel from it. So it will evaluately cause system resource leak.  How we improve it ?
```go
doWork = func (done <- chan interface{}, strings <- chan string)<-chan interface{} {
    completed := make(chan interface{})
    go func(){
            defer close(completed)
            for {
                select {
                    case s:=<- strings:
                        //do something
                    case <-done:
                        return
                }
            }
    }()
    return completed
}
done := make(chan interface{})
completed := doWork(done, nil)
go func(){
    time.Sleep(1 * time.Second)
    close(done)
}()
<- completed
```
In This case, it still can exits goroutine when you pass `nil` for `strings` channel.

## 4.2 Pipeline
A pipleline is tool that takes data from upstream to porecess and then pass it to downstream. The data like water flows between different stages. It helps you imporve your program's abstractions. The only thing you concerned is how to divide data process into separate stages and connect them by channels. Most of all, all stages can work simultaneously. 
```go
generator := func(done <-chan interface{}, integers ... int) <- chan int {
    intStream := make(chan int)
    go func(){
            defer close(intStream)
            for _, i := range integers {
                select {
                    case <- done:
                        return 
                    case intStream <- i:
                }
            }
    }()
    return intStream
}
multiply := func(done <-chan interface{}, intStream<-chan int, multiplier int) <-chan int {
    multipiedStream := make(chan int)
    go func(){
       defer close(multipiedStream)
       for i := range intStream {
           select {
               case <- done:
                    return 
                case multipiedStream <- i * multiplier
            }
       } 
    }()
    return multipiedStream
}
add := func(done <- chan interface{}, intStream <- chan int, adder int) <-chan int {
    addedStream := make(chan int)
    go func(){
        defer close(addedStream)
        for i := range intStream {
            selct {
                case <- done:
                        return 
                    case  addedStream <- i + adder
                }
        }
    }()
    return addedStream
}
done := make(chan interface{})
for i := range adder(done, multiply(done, generator(done, 1,2 ,3 ,4),4, 3){
    fmt.Println("%d", i)
}
```
## 4.3 Fan-out, Fan-in
`Pipeline` pattern seems to be very awesome. What if one of the stage is very expensive that means it will take a gread deal of  time to complete task for each received data, the performance of the  whole  task is depended on that stage.  If you don't care about the order of data, `fan out and fan-in` pattern is reasonable choice.
```go
primerFinder := func(
		done <-chan interface{},
		valueStream <-chan interface{},
) <-chan interface{} {
		primerStream := make(chan interface{})
		go func() {
			defer close(primerStream)
			for {
				select {
				case <-done:
					return
				case value := <-valueStream:
					switch value := value.(type) {
					case int:
						var factor bool
						for i := 2; i <= value/2; i++ {
							if value%i == 0 {
								factor = true
								break
							}
						}
						if factor == false {
							primerStream <- value
						}
					}
				}
			}
		}()
		return primerStream
	}
	fanIn := func(
		done <-chan interface{},
		channels ...<-chan interface{},
	) <-chan interface{} {
		var wg sync.WaitGroup
		multiplexedStream := make(chan interface{})
		multiplex := func(c <-chan interface{}) {
			defer wg.Done()
			for i := range c {
				select {
				case <-done:
					return
				case multiplexedStream <- i:
				}
			}
		}
		wg.Add(len(channels))
		for _, c := range channels {
			go multiplex(c)
		}
		go func() {
			wg.Wait()
			close(multiplexedStream)
		}()
		return multiplexedStream
	}
generator := func(done <- chan interfce{}, f func() interface{} , num int ) <- chan interface{} {
    generateStream := make(chan interface{})
    go func () {
        defer close(generateStream)
        for i := 0; i<num ; i ++ {
            select {
                case <- done:
                        return 
                case  generateStream <- f()
            }
        }
    }()
    return generateStream
}
rand := func() interface{} { return rand.Intn(50000000) }
num := 20
done := make(chan interface{})
randIntStream := generator(done, rand, num)
finders := make([]<-chan interface{}, num)
for i:= 0; i<num ; i++ {
    finder[i] = primerFinder(done, randIntStream)
}
for prime := range fanIn(done, finders...) {
		fmt.Printf("\t%d\n", prime)
}
close(done)
```

## 4.4 The context Package
Above idioms  help us to terminate goroutines by `done` channel. But we still have other demands in concurrency.
- Why cancellation was occuring?
- What if a function has a deadling?
- How to communicate extra information between goroutines?
Since `Go 1.7`, the statndard library provides `context` package to meet those demands. 
### 4.4.1 Peek pakcage
The package exposed memebers are very simple:
```go
var canceled = errors.New("Context canceled")
var DeadlineExecuted error = deadlineExecutedError{}
type CancelFunc
type Context
func Backgroud() Context
func TODO() Contenxt
func WithCancel(parent Context)(Context,  CancelFunc)
func WithDeadline(parent Context, deadline time.Time)(Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration)(Context, CancelFunc)
func WithValue(parent Context, key, val interface{})) Context
```
The `Context` interface plays an important role in this package. Let's dive into it:
```go
type Context interface{
    Deadline()(deadline time.Time, ok bool)
    Done<- chan strcut{}
    Err() error
    Value(key interface{}) interface{}
}
```
- `Done()`: returns a channel that's closed when goroutine is to be preempted
- `Deadline()` indicates if a goroutine will be canceled after a certain time.
- `Err()` indicates the error information when the goroutine is canceled.
- `Value()` is used to share value between goroutines.
### 4.4.2 Situations
Let's take a flashblack, the cancellation in the a goroutine has three aspects:
1. A goroutine's paraent wants to cancel it.
2. A goroutine wants to canncel its children.
3. A goroutine caontins a blocking operation needs to be preemptable. 
This package can handle mentioned three aspects prefectly. 
- `WitchCancel` returns a new Context that closes its done chanel when the returned `cancel` was called.
- `WithDeadline` returns a new Context that closes its done channel whten the machine's clock advances past the given deadline
- `WithTimout` return a new Context that closes its done channel after the given timeout duration. 
At the top a asynchronous call-graph, you can strat chain by calling `Backgroud`. It just merely return an empty Context. 

### 4.4.3 Example
Taking `http server` for example, client make a request and the server connects to database to fetch data. The scheduler illustration as follows:
![](./_image/perfect.png)
What if the client cancels its request  while the DB is exeucting query? The control flow may seem like that:
![](./_image/without-cancel.jpg)
We can cancel Db's execution as long as the client request cancelled by using `context` package.

![](./_image/timg-withcancel.jpg)
```go
func main() {
  // Create an HTTP server that listens on port 8000
  http.ListenAndServe(":8000", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    // This prints to STDOUT to show that processing has started
    fmt.Fprint(os.Stdout, "processing request\n")
    // We use `select` to execute a peice of code depending on which
    // channel receives a message first
    select {
    case <-time.After(2 * time.Second):
      // If we receive a message after 2 seconds
      // that means the request has been processed
      // We then write this as the response
      w.Write([]byte("request processed"))
    case <-ctx.Done():
      // If the request gets cancelled, log it
      // to STDERR
      fmt.Fprint(os.Stderr, "request cancelled\n")
    }
  }))
}
```
As long as you close your browser in 2 second. you will see `request cancelled` in standard error output.

