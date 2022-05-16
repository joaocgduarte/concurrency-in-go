# Table of contents
- [Bibliography](#bibliography)
- [Why is concurrency hard?](#why-is-concurrency-hard)
  * [Concurrency problems:](#concurrency-problems)
- [Communicating Sequencial Processes (CSP)](#communicating-sequencial-processes-csp)
  * [What is the difference between concurrency and parallelism?](#what-is-the-difference-between-concurrency-and-parallelism)
  * [What is CSP?](#what-is-csp)
  * [How does CSP (and Golang) help?](#how-does-csp-and-golang-help)
  * [Go's Philosophy on Concurrency](#gos-philosophy-on-concurrency)
- [Go's Concurrency Building Blocks](#gos-concurrency-building-blocks)
  * [Goroutines](#goroutines)
    + [The M:N Scheduler](#the-mn-scheduler)
  * [The `sync` package](#the-sync-package)
    + [WaitGroup](#waitgroup)
    + [Mutex and RWMutex](#mutex-and-rwmutex)
    + [Cond](#cond)
    + [Once](#once)
    + [Pool](#pool)
  * [Channels](#channels)
    + [Buffered vs unbuffered](#buffered-vs-unbuffered)
    + [Channel owners and channel consumers](#channel-owners-and-channel-consumers)
  * [The `select` statement](#the-select-statement)
- [Concurrency Patterns in Go](#concurrency-patterns-in-go)
  * [Confinement](#confinement)
  * [The for-select loop](#the-for-select-loop)
  * [How to prevent goroutine leaks?](#how-to-prevent-goroutine-leaks)
  * [The or-channel](#the-or-channel)
  * [Error handling](#error-handling)
  * [Pipelines](#pipelines)
  * [Useful generators](#useful-generators)
    + [Repeat](#repeat)
    + [Take](#take)
    + [The repeatFn](#the-repeatfn)
    + [Fan-out, fan-in](#fan-out-fan-in)
    + [The or-done channel](#the-or-done-channel)
    + [The tee-channel](#the-tee-channel)
    + [The bridge-channel](#the-bridge-channel)
  * [The context package](#the-context-package)
- [Concurrency at scale](#concurrency-at-scale)
  * [Error Propagation](#error-propagation)

# Introduction
I decided to properly start learning concurrency in Golang. I had previously worked with it but didn't really understood it. So I bought Concurrency in Go by Katherine Cox-Buday and started to take notes of what I was learning. This is basically a **very** slimmed down version of the contents of the book. I think it's something useful and decided to share it. 

# Bibliography
1. Concurrency in Go - Katherine Cox-Buday - [buy here](https://www.amazon.com/Concurrency-Go-Tools-Techniques-Developers/dp/1491941197)

Everything in this article is based on this book by Katherine Cox-Buday. All the knowledge and nearly all examples comes from her book. This article is purely my annotations of the book and what I learned from it. All credits go to Katherine.

# Why is concurrency hard?
## Concurrency problems:
### Race Conditions
1. When 2 or more operations must execute in the correct order, but the program has not been written so that this order is guaranteed.
2. It shows up as a data race, when two threads are competing for the same variable or resource.

### Atomicity
1. Something is atomic when, within a context, something is indivisible and uninterruptible by any other thing.
2. If a variable/resource is inside a goroutine (context) and its not shared with other routines, then it's atomic. Changes to that resource are uninterruptible and indivisible

### Memory Access Syncronization
1. When there is a data race, we can wrap that **critical section** (when we need exclusive access to a shared resource) with a mutex, locking and unlocking the mutex. When we do this, we are synchronising memory access.
2. This fixes the data race, but it doesn't fix the race condition (threads still compete for first write/read).
3. It can create maintenance and performance problems.
We should try to keep them only to critical sections. If it gets inefficient, make them broader.

### Deadlocks
1. When all concurrent processes (threads or goroutines) are blocked, waiting for another routine to continue. The program will never resume without external intervention.
2. Happens when a shared mutex is locked twice. by two different routines.
3. *Coffman Conditions* help identity when we have a deadlock:
	1. *Mutual exclusions* - When a concurrent process has exclusive rights to a shared resource
	2. *Wait for condition* - When a process holds a resource and simultaneously is waiting for another resource
	3. *No preemption* - when a resource held by the process can only be released by that process
	4. Circular wait - a routine waiting for a routine that is waiting for that routine.

### Livelocks
1. When concurrent processes do nothing to advance the state of the program
2. For instance, when trying to manually avoiding a deadlock, we might try to make the process avoid that other process. And so both of the processes get caught in a loop avoiding each other but not advancing
3. Like when two people are staring at each other, trying to let the other one pass, going left and right, but not succeeding.

### Starvation
1. When a resource is too greedy and prevents other resources to do their job, for instance with a shared mutex in which it keeps it too much time locked.
2. It might seem that some process is more efficient than the other, but the reality is that process is just consuming more resources (exclusive memory access to a resource) and leaving nothing to the other.

## When writing code that uses concurrency, try to answer these questions to the reader (of the func):
1. Who is responsible for initialising the concurrency/goroutines? Me or the function?
2. Who is responsible for the synchronisation of memory?
3. How is the problem space mapped onto concurrency primitives?

# Communicating Sequencial Processes (CSP)
## What is the difference between concurrency and parallelism?
1. We write concurrent code that we except to run in parallel.
2. We should be ignorant to the fact that a concurrent code is running in parallel (ideally).
3. Concurrency is running/managing multiple computations at the same time, parallelism is running multiple computations simultaneously.

## What is CSP?
1. Paper written by Charles Hoare (in 1978), from which Go takes its principles in regards to concurrency, basically where channels come from.
2. Said that inputs and outputs needed to be primitives to facilitate communication between parallel processes (so as do channels)
3. Made this communication easier to make without the need of exclusive memory access

## How does CSP (and Golang) help?
1. Golang provides channels (communication inputs and outputs between processes), goroutines (processes) and the select statement.
2. Channels:
	1. We can easily communicate between different goroutines.
	2. There is less need to do memory access synchronisations.
	3. Allows us to compose the outputs and inputs to other subsystems, and combine it with timeouts or cancellations.
3. Goroutines:
	1. Are very lightweight, we don't have to worry about creating one (unlike Java threads for instance)
	2. We don't need to think about parallelism in our code, and allows us to model problems closer to their natural level of concurrency.
	3. Are multiplexed onto OS Threads automatically and scheduled for us (Go does that). We don't have to worry about these optimisations.
	4. Go scales all our gouroutines dynamically, we don't have to worry about specific OS/hardware limit, because it does it for us.
4. Select:
	1. Complement to channels
	2. enables all the difficult bits of composing channels. Waits for events, selects messages from competing channels, continue if there are no messages waiting...

## Go's Philosophy on Concurrency
1. "Share memory by communicating, don't communicate by sharing memory"
2. Aim for simplicity. Use channels when possible, treat goroutines like a free resource.
3. If necessary only, use memory access synchronisation (mutexes).

# Go's Concurrency Building Blocks
## Goroutines
1. Every program in Go has at least one goroutine: the main routine.
2. A cheap and basic unit of organisation in a Go program.
3. It's a function that runs concurrently alongside other code.
4. They run in the same memory address they were created in (share variables)
5. The functions might not run if main ends before the goroutines start/print to stdout.
6. They are very lightweight (few kilobytes).
7. It's possible to create thousands of goroutines in the same address space.

### Examples of a goroutine instantiation:

```go
func main() {
	go sayHello()
	go func() {
		fmt.Println("cenas")
	}()
	// continue do other things.
}

func sayHello() {
	fmt.Println("hello world")
}
```

### The M:N Scheduler
1. Go's mechanism for hosting goroutines
2. Maps `M` green threads into `N` OS threads
	1. **Green threads** - threads managed by the language's runtime.
	2. **OS Threads** - expensive
3. Goroutines are scheduled into the green threads.
4. The scheduler handles the distribution accross the available threads and ensures that when these goroutines become blocks, other goroutines can run.

## The `sync` package
1. Contain concurrency primitives that are useful for low level memory synchronisation.

### WaitGroup
1. Allows us to wait for a set of concurrent operations to complete when we don't care about the result of the concurrent operation.

#### Example
```go
var wg sync.WaitGroup

wg.Add(1) // Add an incremental to the group (plus one routine running)
go func() {
	defer wg.Done() // Say that the routine is finished to the group.
	fmt.Println("first goroutine sleeping")
	time.Sleep(1)
} ()

wg.Add(1)
go func() {
	defer wg.Done()
	fmt.Println("second goroutine sleeping")
	time.Sleep(1)
} ()

wg.Wait() // Waits for routines in waitgroup are finished.
fmt.Println("all goroutines are complete.")
```

### Mutex and RWMutex
1. Allows us to wrap critical sections and exclusively write to a memory address (memory synchronisation).
2. Try to only use in an encapsulated manner (in a struct that allows for concurrent writes for instance).
3. **Locking the same mutex twice without Unlocking will cause a deadlock!**
4. RWMutex is the same as regular Mutex, but constrains wether we want to synchronise in reading or writing.

#### Example
```go
type ConcurrentCounter struct {
	mu sync.Mutex
	value int
}

func (c *ConcurrentCounter) Increment() {
	c.mu.Lock() // restricts the memory pointers here. Guarantees only one write to value.
	value++
	c.mu.Unlock() // Being called with defer is also nice.
}
```

### Cond
1. A rendezvous for goroutines waiting for or announcing the occurence of an event/signal.

#### Example
This example makes a queue with limit of 5, and keeps adding to the queue. We want to remove from the queue every time the queue gets to length of 2, concurrently.

```go
c := sync.NewCond(&sync.Mutex{})
queue := make([]interface{}, 0, 5)

removeFromQueue := func(delay time.Duration) {
	time.Sleep(delay)
	c.L.Lock() // enter the critical section.
	queue = queue[1:]
	fmt.Println("removed from queue")
	c.L.Unlock() // exit the critical section.
	c.Signal() // signal that our operation is done.
}

for i := 0; i < 5; i++ {
	c.L.Lock() // Enter a critical section. Prevent concurrent writes.
	for len(queue) == 2 {
		c.Wait() // We suspend the main routine here. Wait for a signal to continue.
		// c.Wait() calls c.L.Unlock() after it's called, and c.L.Lock() when it exits.
	}
	fmt.Println("adding to queue")
	queue = append(queue, struct{}{})
	go removeFromQueue(time.Second*1)
	c.L.Unlock() // Exit critical section.
}

Output:
adding to queue
adding to queue
removed from queue
adding to queue
removed from queue
adding to queue
removed from queue
adding to queue
removed from queue
adding to queue
```

We can also use `Broadcast()` instead of `Signal()`. `Broadcast()` tells all goroutines that something happened. `Signal()` tells the goroutine that is waiting the longest that something happened.

### Once
1. Ensures that only one call to Do ever calls the function that is passed, even on different goroutines.
#### Example
```go
var count int
increment := func() {
	count++
}

var once sync.Once

var wg sync.WaitGroup
wg.Add(100)
for i := 0; i < 100; i++ {
	go func() {
		defer wg.Done()
		once.Do(increment)
	}()
}

wg.Wait()
fmt.Printf("count is %d\n", count)

Output:
count is 1
```

### Pool
1. Concurrent safe implementation of the design pattern "Pool".
2. Creates and makes available a fixed number (or pool) of things for use. Used for expensive things such as database connection. Only N instances are created, but M operations can request access from these instances.
3. Can basically improve response time, but try to avoid using it.

#### Example
```go
pool := &sync.Pool {
	New: func() interface{} {
		fmt.Println("creating new instance")
		return struct{}{}
	}
}

pool.Get() // Gets an available instance or calls pool.New()
instance := pool.Get() 
pool.Put(instance) // Put previously retrieved instance back in pool. Use it to seed the pool or to reuse instances. Saves memory.
pool.Get()

Output:
creating new instance
creating new instance
```

## Channels
1. Streams of data (input and output) that allow communication between goroutines. Derived from Hoare's CSP.
2. Can be unidirectional or not (only read or write, or both). Useful so we can define who owns and who consumes the channel.

```go
var receiveStream <-chan string
var sendStream chan<- string
dataStream := make(chan string)

// valid statements
receiveStream = dataStream
sendStream = dataStream

go func() {
	dataStream <- "hello world"
}()
fmt.Println(<-dataStream) // prints hello world. Waits until there is something in the channel.
```

3. We can close the channel, with `close(dataStream)` and receive a the close message with `data, ok := <-dataStream`, where ok is `false` and data is `""`
4. We can loop through a channel, using `for data := range dataStream {}`, and it will end the loop as soon as `dataStream` is closed. It will block the execution to wait for the data as well.

### Buffered vs unbuffered
1. Buffered channels have a limit `make(chan int, 3)`, whereas  unbuffered channels don't `make(chan int)`
2. Buffered channels are a FIFO (first in first out) queue.
3. Unbuffered channels are used for synchronous communication
4. Unbuffered channels send data to other routines as soon as it receives it.
5. Buffered channels have to be filled, and then the routines that consume the channel receive. 
6. Unbuffered Channel will block the goroutine whenever it is empty and waiting to be filled.
7. Buffered Channel will also block the goroutine either when it is empty and waiting to be filled or it's on its full-capacity and there's a statement that want to fill the channel.

### Channel owners and channel consumers
#### Owners should:
1. Instantiate the channel.
2. Perform writes, or pass ownership to another goroutine.
3. Close the channel.
4. Encapsulate 1,2 and 3 and expose them via a reader channel.

This reduces the risk of deadlocks (by writing to a nil channel), panics (by closing a nil channel or writing to a closed channel, or closing a channel more than once)

#### Consumers should:
1. Know when a channel is closed.
2. Responsibly handle blocking for any reason (using the `select` statement).

#### Example:
This way, the lifecycle of resultStream is encapsulated, easy to read and maintain.
```go
chanOwner := func() <-chan int { //read only chan return
	resultStream := make(chan int, 5)
	go func() {
		defer close(resultStream)
		for i := 0; i <= 5; i++ {
			resultStream <- i
		}
	} ()
	return resultStream
}

resultStream := chanOwner()
for result := range resultStream {
	fmt.Printf("received: %d\n", result)
}
fmt.Println("done receiving")

output:
received 0
received 1
received 2
received 3
received 4
received 5
done receiving
```

## The `select` statement
1. Binds channels together.
2. Waits for a channel to receive something and does logic accordingly

### Examples:
```go
done := make(chan interface{})
go func() {
	time.Sleep(5*time.Second)
	close(done)
} ()

workCounter := 0
for {
	select {
	case <-done: 
		break
	case <-time.After(10 * time.Second): // example of a timeout with a select, if we never get anything on the done channel for 10 seconds.
		break
	default:
	}
	//simulate work
	workCounter++
	time.Sleep(1*time.Second)
}
fmt.Printf("achieved %d work cycles", workCounter)

output:
achieved 5 work cycles.
```

# Concurrency Patterns in Go
## Confinement
Means encapsulating the data, having owners and consumers, so that the owners can close the channel and guarantee that the operations are atomic.

**Example:**
```go
chanOwner := func() <-chan int {
	results := make(chan int, 5)
	go func() {
		defer close(results)
		for i := 0; i < 5; i++ {
			results <- i
		}
	}()
	return results
}

consumer := func(results <-chan int) {
	for result := range results {
		fmt.Printf("received: %d\n", result)
	}
	fmt.Println("done receiving")
}

results := chanOwner()
consumer(results)
```

## The for-select loop
We can loop indefinitely and wait to be stopped, or loop through iteration variables, and do a select statement to return from the loop.

**Example:**
```go
for {
	select {
	case <-done:
		return
	default:
	}
	// do non-preemptable work
}
```

## How to prevent goroutine leaks?
Although goroutines are very lightweight, they are not eliminated by the garbage collector, which means that we need to prevent them to run indefinitely. 
We do that by using a read only channel on the owners of the goroutine, and exiting the routine once that channel is closed. The consumers, or the parent routine that has access to the routine, that knows when that routine must end, are able to do it just by closing the channel.

**Example:**
```go
newRandStream := func(done <-chan interface{}) <-chan int {
	randStream := make(chan int)
	go func() {
		defer fmt.Println("newRandStream has exited.")
		defer close(randStream)
		for {
			select {
			case randStream <- rand.Int():
			case <-done:
				return
			}
		}
	} ()
	return randStream
}

done := make(chan interface{})
randStream := newRandStream(done)
fmt.Println("3 random ints:")
for i := 1; i <= 3; i++ {
	fmt.Printf("%d: %d", i, <-randStream)
}
close(done)

//simulate on going work
time.Sleep(1 * time.Second)

Output:
3 random ints:
1: 121324212
2: 423344243
3: 123123231
newRandStream has exited.
```

If we didn't send the close channel, we would never have gotten the "newRandStream" message.

## The or-channel
You can combine multiple done channels into a single done channel that closes if any of its component channels close. This is done by the use of a recursive function, and it's a good pattern to have. The next example is to check if any of X channels (or goroutines) are closed, and it's X/2 complexity.

**Example:**
```go
or := func (channels ...<-chan interface{}) <-chan interface{} {
	switch len(channels) {
	case 0:
		return nil
	case 1:
		return channels[0]
	}
	
	orDone := make(chan interface{})
	go func() {
		defer close(orDone)
		switch len(channels) {
		case 2:
			select {
			case <-channels[0]:
			case <-channels[1]:
			}
		default:
			select {
				case <-channels[0]:
				case <-channels[1]:
				case <-channels[2]:
				case <-or(append(channels[3:], orDone)...)
			}	
		}
		return orDone
	}()
}

sig := func(after time.Duration) <-chan interface{} {
	c := make(chan interface{})
	go func() {
		defer close(c)
		time.Sleep(after)
	}()
	return c
}

start := time.Now()
<-or(
	sig(time.Second * 5)
	sig(time.Second * 1)
	sig(time.Hour * 2)
	sig(time.Minute * 1)
)
fmt.Printf("time since start: %v", time.Since(start))

output: 
time since start: 1.000004s
```

## Error handling
Parent goroutines should handle the errors of the children goroutines, or any other routine that has more access to the context, or state, of the whole application. We can do that by encapsulating the return result of the return channel.

**Example:**
```go
type Result struct {
	Error err
	Response *http.Response
}

checkStatus := func(done <-chan interface{}, urls ...string) <-chan Result {
	results := make(chan Result)
	go func() {
		defer close(results)
		for _, url := range urls {
			resp, err := http.Get(url)
			result := Result{Error: err, Result: resp}
			select {
			case <-done:
				return
			case results <- result:
			}
		}
	} ()
	return results
}

done := make(chan interface{})
defer close(done)

urls := []string{"a", "https://www.google.com", "b", "c", "d"}
errCount := 0
for result := range checkStatus(done, urls) {
	if result.Error != nil {
		errCount++
		fmt.Printf("error: %v", err)
		if errCount == 3 {
			fmt.Println("too many errors, exiting process")
			break
		}
		continue
	}
	fmt.Printf("Response: %d\n", result.Response.Status)
}

output:
error: a doesnt exist
Response: 200
error: b doesnt exist
error: c doesnt exist
```

## Pipelines
Pipelines are something that comes from functional programming. Are functions that call another function result. They can be a **Batch Pipeline** (an array as input and output), or a **Stream Pipeline** (single input as an input and output). Here is an example of a simple pipeline:

```go
add := func(nums []int, addition int) []int {
	result := make([]int, len(nums))
	for i, num := range nums {
		result[i] = num + addition
	}
	return result
}

multiply := func(nums []int, multiplier int) []int {
	result := make([]int, len(nums))
	for i, num := range nums {
		result[i] = num * multiplier
	}
	return result
}

initialArr := []int{1, 2, 3, 4}
result := add(multiply(initialArr, 2), 1)
fmt.Println(result)

Output:
[3, 5, 7, 9]
```

### How can we use pipelines in concurrency?
1. Use channels as the pipeline's inputs.

***Example:***

```go
generator := func (<- done interface{}, integers... int) <-chan int {
	intStream := make(chan int)
	go func () {
		defer close (intStream)
		for _, i := range integers {
			select {
			case <-done:
				return
			case intStream <- i:
			}
		}
	} ()
	return intStream
}

multiply := func(
	done  <-chan interface,
	intStream <-chan int,
	multiplier int,
) <-chan int {
	multipliedStream := make(chan int)
	go func () {
		defer close(multipliedStream)
		for i := range intStream {
			select {
			case <-done:
				return
			case multipliedStream <- i*multiplier:
			}
		}
	} ()
	return multipliedStream
}

add := func(
	done  <-chan interface,
	intStream <-chan int,
	additive int,
) <-chan int {
	additionStream := make(chan int)
	go func () {
		defer close(additionStream)
		for i := range intStream {
			select {
			case <-done:
				return
			case additionStream <- i*additive:
			}
		}
	} ()
	return additionStream
}

done := make(chan interface)
defer close(done)

pipeline := multiply(done, add(done, multiply(done, generator(done, 1, 2, 3, 4), 2), 1), 2)

for v := range pipeline {
	fmt.Println(v)
}

Output:
6
10
14
18
```

## Useful generators
This is a collection of useful functions/snippets that you might commonly use or see in concurrency projects.

### Repeat
This function will repeat the values you pass to it infinitely until you tell it to stop.

```go
repeat := func(
	done <-chan interface{},
	values ...interface{},
) <-chan interface{} {
	valueStream := make(chan interface{})
	go func() {
		defer close(valueStream)
		for {
			for _, v := range values {
				select {
				case <-done:
					return
				case valueStream <- v:
				}
			}
		}
	}()
	return valueStream
}
```

### Take
This "takes" the first number of items off an incoming stream, and then exit.

```go
take := func(
	done <-chan interface{},
	valueStream <-chan interface{},
	num int,
) <-chan interface {
	takeStream := make(chan interface{})
	go func() {
		defer close(takeStream)
		for i := 0; i < num; i++ {
			select {
			case <-done:
				return
			case takeStream <- <-valueStream:
			}
		}
	}()
	return takeStream
}

done := make(chan interface{})
defer close(done)
for num := range take(done, repeat(done, 1), 5) {
	fmt.Printf("%v ", num)
}

Output:
1 1 1 1 1
```

### The repeatFn
If we expand the repeat and add a callback, we can use that to infinitely call a function and return a channel of the desired type that you want, here's an example:

```go
repeat := func(
	done <-chan interface{},
	fn func() interface{},
) <-chan interface{} {
	valueStream := make(chan interface{})
	go func() {
		defer close(valueStream)
		for {
			for _, v := range values {
				select {
				case <-done:
					return
				case valueStream <- fn():	
				}
			}
		}
	}()
	return valueStream
}

done := make(chan interface{})
defer close(done)
rand := func() interface{} {
	return rand.Int()
}

for num := range take(done, repeatFn(done, rand), 5) {
	fmt.Println(num)
}

Output:
1234
54332
3467567
234
34456
```

### Fan-out, fan-in
Fan-out is the process of starting multiple goroutines to handle input from the pipeline, and fan-in is what we call the process of combining multiple results into one channel. When should we use this pattern?
1. when it doesn't rely on values that the stage had calculated before;
2. when it takes a long time to run;
3. when the order of the output doesn't matter.

Fan-out is easy, just launch multiple versions of a particular stage:
```go
numFinders := 4
finders := make([]<-chan int, numFinders)
for i := 0; i < numFinders; i++ {
	finders[i] := primeFinder(done, randIntStream)
}
```

The fan-in joins together (the term is *multiplexing*) the multiple streams of data into a single-stream. Here is an example:

```go
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
			case multiplexedStream <- c:
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
	}
	
	return multiplexedStream
}
```

We can now utilize this multiplexedStream, that combines all of the channels (that are running on multiple goroutines) into one.

### The or-done channel
It's a way to improve readability. When utilising this pattern, we don't need to check if the channel is closed when reading from it, because the function does it for us:

```go
orDone := func(done, c <-chan interface{}) <-chan interface{} {
	valStream := make(chan interface{})
	go func() {
		defer close(valStream)
		for {
			select {
			case <-done:
				return
			case v, ok := <-c:
				if ok == false {
					return
				}
				select {
				case valStream <-v:
				case <-done:
				}
			}
		}
	}()
	return valStream
}

for val := range orDone(done, myChan) {
	...
}
```

### The tee-channel
This reads from one input stream, and exposes it to two other input streams. This way you can send data to two parts of your system.
```go
tee := func(
	done <-chan interface{},
	in <-chan interface{},
) (<-chan interface{}, <-chan interface{}) {
	out1 := make(chan interface{})
	out2 := make(chan interface{})
	go func() {
		defer close(out1)
		defer close(out2)
		for val := range orDone(done, in) {
			var out1, out2 := out1, out2
			for i := 0; i < 2; i++ {
				select{
				case out1 <- val:
					out1 = nil
				case out2 <- val:
					out2 = nil
				}
			}
		}
	}()
	return out1, out2
}
```

This will always wait until both out1 and out2 have written data on it. Both of these channels should always have the same data for each iteration of the input channel.

### The bridge-channel
This is a useful way of combining multiple streams of data into a single one. It lets our consumers handle only one problem at the time, when we might have a channel of channels as an input.

```go
bridge := func(
	done <-chan interface{},
	chanStream <-chan <-chan interface{},
) <-chan interface{} {
	valStream := make(chan interface{})
	go func() {
		defer close(valStream)
		for {
			var stream <-chan interface{}
			select {
			case maybeStream, ok := <-chanStream:
				if ok == false {
					return
				}
				stream = maybeStream
			}
		case <-done:
			return
		}
		for val := range orDone(done, stream) {
			select {
			case valStream <- val:
			case <-done:
			}
		}
	}()
	return valStream
}
```

## The context package
We can use the context package to add validations whether or not we want to close a channel and end a goroutine. This gives us some control with timeouts and cancelations.

```go
work := func(ctx context.Context) {
    defer wg.Done()
    for i := 0; i < 200; i++ {
        select {
        case <-time.After(5 * time.Second):
            fmt.Println("starting...", i)

        case <-ctx.Done():
            fmt.Println("context was canceled", i)
        }
    }
}

ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

wg.Add(1)
go work(ctx)
wg.Wait()

fmt.Println("finished")

Output:
context was canceled
finished
```

# Concurrency At Scale
These techniques will allow us to make our systems more scalable.

## Error Propagation
1. Errors should always be first class citizens in our Go code.
2. Errors shouldn't just be dumped in front of the user.
3. We should try to make error handling an asset in our systems
4. Errors should include the following critical information:
	1. What happened;
	2. When and where it happened;
	3. A friendly user-facing message;
	4. How the user can get more information.
5. This allows us to know that if our error isn't wrapped, then it's an unhandled bug or error case that we didn't account for.
6. It gives us a lot more context on our errors.

**Example of a good error handling**
```go
// Error handler
type MyError struct {
	Inner error // wraps the original error
	Message string
	Stacktrace string
	Misc map[string]interface{} // where we can put additional details such as a hash of the stack trace, an ID, any king of contextual information
}

func wrapError(err error, messagef string, msgArgs ...interface{}) {
	return MyError{
		Inner: err,
		Message: fmt.Sprintf(messagef, msgArgs...)
		Stacktrace: string(debug.Stack()),
		Misc: make(map[string]interface{})
	}
}

func (err MyError) Error() string {
	return err.Message
}

// Let's call this next module "LowLevel" Module
type LowLevelErr struct {
	error
}

func isGloballyExec(path string) (bool, error) {
	info, err := os.Stat(path)
	if err != nil {
		return false, LowLevelErr{(wrapError(err, err.Error()))}
	}
	return info.Mode().Perm()&0100 == 0100, nil
}

// Let's call this next module "IntermediateLevel" Module

type IntermediateErr struct {
	error
}

func runJob(id string) error {
	const jobBinPath := "/path/to/binary"
	isExecutable, err := isGloballyExec(jobBinPath)
	if err != nil {
		return IntermediateErr{wrapError(
			err,
			"cannot run job %q: binaries are not available",
			id,
		)}
	} else if isExecutable == false {
		return wrapError(
			nil,
			"cannot run job %q: binaries are not executable",
			id,
		)
	}
	return exec.Command(jobBinPath, "--id="+id).Run()
}

// main
func handleError(key int, err error, message string) {
	log.SetPrefix(fmt.Sprintf("[logID: %v]:", key))
	log.Printf("%#v", err)
	fmt.Println("[%v] %v", key, message)
}

func main() {
	log.SetOutput(os.Stdout)
	log.SetFlags(log.Ltime|log.LUTC)
	
	err := runJob("1")
	if err != nil {
		msg := "Unexpected error, please contact someone"
		if _, ok := err.(IntermediateErr); ok {
			msg = err.Error()
		}
		handleError(1, err, msg)
	}
}

Output:
[logID: 1]: 22:00:00 main.IntermediateErr{error: main.MyError{Inner: main.LowLevelErr{error: main.MyError(Inner: (*os.PathError)(0xc4200123f0), Message: "stat: \"/path/to/binary\" no such file or directory", Stacktrace: "stacktrace")}}}
```
