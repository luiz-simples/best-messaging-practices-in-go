# 🚀 Advanced Golang  Guide

> goroutines, channels, and concurrency patterns with visual explanations
>

---

## 📚 Table of Contents

1. [Concept Visualizations](#-concept-visualizations)
2. [Goroutines & Concurrency Fundamentals](#1-goroutines--concurrency-fundamentals)
3. [Channels - Deep Dive](#2-channels---deep-dive)
4. [Worker Pool Pattern](#3-worker-pool-pattern)
5. [Singleflight Pattern](#4-singleflight-pattern)
6. [Synchronization Primitives](#5-synchronization-primitives)
7. [Memory Management](#6-memory-management)
8. [Context Package](#7-context-package)
9. [Advanced Patterns & Anti-patterns](#8-advanced-patterns--anti-patterns)
10. [Testing Concurrent Code](#9-testing-concurrent-code)
11. [Performance Considerations](#10-performance-considerations)
12. [Common Pitfalls](#-common-pitfalls)

---

## 🎨 Concept Visualizations

### GMP Scheduler Model

```
┌─────────────────────────────────────────────────────┐
│           Go Runtime Scheduler (M:N Model)          │
└─────────────────────────────────────────────────────┘

    Goroutines (G) - Lightweight user-space threads
    ┌────┐  ┌────┐  ┌────┐  ┌────┐  ┌────┐  ┌────┐
    │ G1 │  │ G2 │  │ G3 │  │ G4 │  │ G5 │  │ G6 │
    └─┬──┘  └─┬──┘  └─┬──┘  └─┬──┘  └─┬──┘  └─┬──┘
      │       │       │       │       │       │
      └───┬───┴───┬───┘       └───┬───┴───┬───┘
          │       │               │       │
          ▼       ▼               ▼       ▼
    ┌─────────┐ ┌─────────┐   ┌─────────┐
    │   P1    │ │   P2    │   │   P3    │  Processors (P)
    │ (queue) │ │ (queue) │   │ (queue) │  - Scheduling context
    └────┬────┘ └────┬────┘   └────┬────┘  - GOMAXPROCS
         │           │             │
         ▼           ▼             ▼
    ┌─────────┐ ┌─────────┐   ┌─────────┐
    │   M1    │ │   M2    │   │   M3    │  OS Threads (M)
    │ Thread  │ │ Thread  │   │ Thread  │  - Execute goroutines
    └─────────┘ └─────────┘   └─────────┘  - Syscalls, blocking I/O

    Work-Stealing: P can steal G from other P's queue
```

### Channel Communication Patterns

```
┌──────────────────────────────────────────────────────┐
│         Unbuffered vs Buffered Channels              │
└──────────────────────────────────────────────────────┘

UNBUFFERED (Synchronous):
    Sender                Channel              Receiver
    ┌────┐               ┌──────┐              ┌────┐
    │ G1 │─────Send─────>│  ch  │────Recv─────>│ G2 │
    └────┘               └──────┘              └────┘
     BLOCKS              Size: 0              BLOCKS
    ↓ waits for receiver                  ↓ waits for sender

    Both must be ready simultaneously!

BUFFERED (Asynchronous):
    Sender                Channel              Receiver
    ┌────┐            ┌──────────────┐        ┌────┐
    │ G1 │───Send────>│ [•][•][ ]    │───Recv>│ G2 │
    └────┘            │  Buffer: 3   │        └────┘
    Only blocks       └──────────────┘      Only blocks
    when full                               when empty

    Sender can proceed if buffer has space
```

### Goroutine Lifecycle

```
┌────────────────────────────────────────────────────────┐
│              Goroutine State Machine                   │
└────────────────────────────────────────────────────────┘

    ┌─────────┐
    │ Created │  go func() {...}
    └────┬────┘
         │
         ▼
    ┌─────────┐
    │ Runnable│◄─────┐
    └────┬────┘      │ Unblocked
         │           │
         ▼           │
    ┌─────────┐      │
    │ Running │──────┤
    └────┬────┘      │
         │           │
         ├───────────┘
         │
         ├─────────┐
         │         │
         ▼         ▼
    ┌─────────┐ ┌──────────┐
    │ Blocked │ │  Dead    │
    │(I/O,ch) │ │(returned)│
    └─────────┘ └──────────┘
```

### Memory Stack vs Heap

```
┌──────────────────────────────────────────────────────┐
│           Stack vs Heap Allocation                   │
└──────────────────────────────────────────────────────┘

STACK (Fast, automatic cleanup):
    ┌─────────────────┐
    │  main()         │  ← Stack grows
    ├─────────────────┤     downward
    │  func1()        │
    │  ├─ local vars  │  ✓ Fast allocation
    │  └─ parameters  │  ✓ Auto cleanup
    ├─────────────────┤  ✓ Cache-friendly
    │  func2()        │
    └─────────────────┘

HEAP (Slower, GC managed):
    ┌─────────────────────────────────┐
    │  ┌────┐  ┌────┐  ┌────┐        │
    │  │obj1│  │obj2│  │obj3│  ...   │
    │  └────┘  └────┘  └────┘        │
    │                                 │
    │  ✗ Slower allocation            │
    │  ✗ GC overhead                  │
    │  ✓ Flexible lifetime            │
    │  ✓ Shared across goroutines     │
    └─────────────────────────────────┘

Variables escape to heap when:
  • Returned by pointer
  • Assigned to interface
  • Too large for stack
  • Size unknown at compile time
```

### Mutex vs RWMutex

```
┌──────────────────────────────────────────────────────┐
│         Mutex vs RWMutex Access Patterns             │
└──────────────────────────────────────────────────────┘

MUTEX (Exclusive Lock):
    Time ─────────────────────────────►

    G1  [████ Lock ████]
    G2              [████ Lock ████]
    G3                          [████ Lock ████]

    Only ONE goroutine at a time

RWMUTEX (Read-Write Lock):
    Time ─────────────────────────────►

    G1  [──── RLock ────]
    G2  [──── RLock ────]  ← Multiple readers OK
    G3    [── RLock ──]
    G4                  [████ Lock ████]  ← Writer waits
    G5                              [─ RLock ─]

    Multiple readers OR single writer
```

### Channel Select Pattern

```
┌──────────────────────────────────────────────────────┐
│            Select Statement Flow                     │
└──────────────────────────────────────────────────────┘

    select {
    case msg1 := <-ch1:  ─┐
    case msg2 := <-ch2:  ─┼──► Random selection
    case ch3 <- value:   ─┤    if multiple ready
    case <-time.After(): ─┘
    default:             ──────► Non-blocking
    }

    ┌─────────┐
    │ Select  │
    └────┬────┘
         │
    ┌────┴────────────────────┐
    │ All cases blocked?      │
    └────┬────────────────┬───┘
         │ Yes            │ No
         ▼                ▼
    ┌─────────┐      ┌──────────┐
    │ Wait    │      │ Random   │
    │ (block) │      │ pick one │
    └─────────┘      └──────────┘
         │                │
         │ One ready      │
         ▼                ▼
    ┌──────────────────────┐
    │  Execute case        │
    └──────────────────────┘
```

### Worker Pool Architecture

```
┌──────────────────────────────────────────────────────┐
│              Worker Pool Pattern                     │
└──────────────────────────────────────────────────────┘

Input Stream                Workers              Output Stream
    │                                                │
    ▼                                                ▼
┌────────┐          ┌──────────┐              ┌────────┐
│ Job 1  │─┐        │ Worker 1 │─────────────>│Result 1│
│ Job 2  │─┤        └──────────┘              │Result 2│
│ Job 3  │─┼──────> ┌──────────┐              │Result 3│
│ Job 4  │─┤  jobs  │ Worker 2 │─────────────>│  ...   │
│  ...   │─┤  chan  └──────────┘  results chan└────────┘
└────────┘─┘        ┌──────────┐
                    │ Worker 3 │─────────────>
                    └──────────┘
                    ┌──────────┐
                    │ Worker N │─────────────>
                    └──────────┘

    Controlled concurrency with N workers
```

### Context Cancellation Tree

```
┌──────────────────────────────────────────────────────┐
│           Context Cancellation Hierarchy             │
└──────────────────────────────────────────────────────┘

    Background/TODO (root)
            │
            ▼
    ┌───────────────┐
    │  ctx1         │  WithCancel/Timeout
    │  (canceled)   │
    └───────┬───────┘
            │
            ├──────────────┬──────────────┐
            ▼              ▼              ▼
    ┌──────────┐   ┌──────────┐   ┌──────────┐
    │  ctx2    │   │  ctx3    │   │  ctx4    │
    │ (canceled)│   │(canceled)│   │(canceled)│
    └──────────┘   └────┬─────┘   └──────────┘
                        │
                        ▼
                 ┌──────────┐
                 │  ctx5    │
                 │(canceled)│
                 └──────────┘

    Canceling parent cancels ALL children!
```

---

## 1. Goroutines & Concurrency Fundamentals

### 🎯 Key Concepts

**Goroutines** are lightweight threads managed by the Go runtime, starting with only **2KB** of stack space (can grow/shrink dynamically).

### Characteristics:

- ✅ **Cheap creation**: Millions can run concurrently
- ✅ **M:N Threading Model**: M goroutines multiplexed onto N OS threads
- ✅ **Cooperative scheduling**: With preemption points
- ✅ **Dynamic stack**: Grows from 2KB to GBs if needed

### 📝 Common Points

**Q: Difference between goroutines and OS threads?**

| Feature | Goroutines | OS Threads |
| --- | --- | --- |
| Stack Size | 2KB (growable) | 1-2MB (fixed) |
| Creation Cost | ~2μs | ~1ms |
| Context Switch | ~200ns | ~1-2μs |
| Managed By | Go Runtime | OS Kernel |
| Scalability | Millions | Thousands |

**Q: How does the Go scheduler work (GMP model)?**

The GMP model consists of:
- **G (Goroutine)**: User-level thread
- **M (Machine)**: OS thread that executes goroutines
- **P (Processor)**: Scheduling context with local run queue

**Work-stealing algorithm**: Idle processors steal goroutines from other processors’ queues for load balancing.

### 💻 Code Examples

### Basic Goroutine

```go
// Simple goroutine launch
go func() {
	fmt.Println("Hello from goroutine")
}()
```

### Avoiding Closure Issues

### Common Goroutine Leak Scenario

```go
// ❌ WRONG - Captures loop variable
for i := 0; i < 5; i++ {
	go func() {
		fmt.Println(i) // All print 5!
	}()
}
// ✅ CORRECT - Pass as parameter
for i := 0; i < 5; i++ {
	go func(i int) {
		fmt.Println(i) // Prints 0,1,2,3,4
	}(i)
}
// ✅ CORRECT (Go 1.22+) - Loop variable per iteration
for i := 0; i < 5; i++ {
	go func() {
		fmt.Println(i)
		// Now safe!
	}()
}
```

```go
// ⚠️ GOROUTINE LEAK!
func leak() {
	ch := make(chan int)
	go func() {
		ch <- 42 // Blocks forever if no receiver
	}()    // Function returns, channel never read
}

// ✅ FIX - Use buffered channel or ensure receiver
func noLeak() {
	ch := make(chan int, 1) // Buffered
	go func() {
		ch <- 42 // Won't block
	}()
}
```

### 🔍 Detecting Goroutine Leaks

```go
// Use runtime to check goroutine count
before := runtime.NumGoroutine()// ... run code ...
after := runtime.NumGoroutine()
if after > before {
	fmt.Printf("Leaked %d goroutines\n", after-before)
}
// Use pprof for detailed analysis
import _ "net/http/pprof"// Visit http://localhost:6060/debug/pprof/goroutine
```

---

## 2. Channels - Deep Dive

### 📊 Channel Types & Characteristics

### Unbuffered Channels (Synchronous)

```go
ch := make(chan int) // Unbuffered
// Sender blocks until receiver ready
// Receiver blocks until sender ready
// Perfect synchronization point
```

**Properties:**
- Zero capacity
- Synchronous communication
- Guarantees delivery

### Buffered Channels (Asynchronous)

```go
ch := make(chan int, 10) // Buffer size: 10
// Sender blocks only when buffer is FULL
// Receiver blocks only when buffer is EMPTY
// Can proceed independently up to capacity
```

**Properties:**
- Fixed capacity
- Asynchronous up to buffer size
- Decouples sender/receiver timing

### Directional Channels

```go
// Send-only channel
func send(ch chan<- int) {
	ch <- 42    // <-ch // Compile error!
}

// Receive-only channel
func receive(ch <-chan int) int {
	return <-ch
  // ch <- 42 // Compile error!
  }

// Bi-directional converts to directional
ch := make(chan int)
send(ch) // Converts to send-only
receive(ch) // Converts to receive-only
```

### 🎭 Channel Operations & Patterns

### Select Statement

```go
select {
	case msg := <-ch1:    // Handle message from ch1
		fmt.Println("Received:", msg)
	case ch2 <- value:    // Send value to ch2
		fmt.Println("Sent:", value)
	case <-time.After(1 * time.Second):    // Timeout after 1 second
		fmt.Println("Timeout!")
	default:   //Non-blocking fallback Executes immediately if no other case ready
		fmt.Println("No channel ready")
}
```

**Important**: If multiple cases are ready, Go **randomly** selects one to ensure fairness.

### Channel Closing Patterns

```go
// Proper way to check if channel is closed
if val, ok := <-ch; ok {
	// Channel is open, val is valid
	fmt.Println("Received:", val)
} else {
	    // Channel is closed, val is zero value
	   fmt.Println("Channel closed")
	  }

for val := range ch {    // Range over channel (stops when closed)
 // Process val
 // Loop exits when ch is closed
}
// Only sender should close
close(ch)
// Sending to closed channel = PANIC!
// Closing closed channel = PANIC!
// Receiving from closed channel = zero value + false
```


```jsx
A send to a nil channel blocks forever
A receive from a nil channel blocks forever
A send to a closed channel panics
A receive from a closed channel returns the zero value immediately

```

### Nil Channel Behavior

```go
var ch chan int // nil channel
<-ch    // Blocks forever
ch <- 1 // Blocks forever
close(ch) // PANIC!

// Use case: Disable a case in select
select {
	case <-ch1:    // Will never be selected if ch1 is nil
	case <-ch2:    // ...
}
```

### 🌊 Fan-in/Fan-out Patterns

```jsx
So why isn’t there a version of close() that lets you check if a channel is closed ?

if !isClosed(c) {
        // c isn't closed, send the value
        c <- v
}
But this function would have an inherent race. Someone may close the channel after we checked isClosed(c) but before the code gets to c <- v.

Solutions for dealing with this fan in problem are discussed in the 2nd article linked at the bottom of this post.

```

https://go.dev/blog/pipelines

### Fan-Out (Distribute Work)

```go
// Distribute work across multiple workers
func fanOut(input <-chan Work, workers int) []<-chan Result {
	outputs := make([]<-chan Result, workers)
	for i := 0; i < workers; i++ {
	output := make(chan Result)
	outputs[i] = output
        go worker(input, output)
  }
	return outputs
}

func worker(input <-chan Work, output chan<- Result) {
	for work := range input {
		result := process(work)
		output <- result
	}
	close(output)}
```

### Fan-In (Merge Results)

```go
// Merge multiple channels into one
func fanIn(inputs ...<-chan Result) <-chan Result {
	output := make(chan Result)
	var wg sync.WaitGroup
    // Start a goroutine for each input channel
    for _, input := range inputs {
	    wg.Add(1)
	    go func(ch <-chan Result) {
	    defer wg.Done()
	    for result := range ch {
		    output <- result
      }
    }(input)
  }
  // Close output when all inputs are done
  go func() {
	  wg.Wait()
	  close(output)
	}()
	return output
}
```

### Pipeline Pattern

```go
func pipeline() <-chan int {
	numbers := generate(1, 10)
	squares := square(numbers)
	filtered := filter(
		squares,
		func(n int) bool {
		return n > 50
	})
	return filtered
}

func generate(start, end int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for i := start; i <= end; i++ {
			out <- i
    }
  }()

  return out
}

func square(in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for n := range in {
			out <- n * n
		}
	}()

	return out
}

func filter(in <-chan int, predicate func(int) bool) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for n := range in {
			if predicate(n) {
				out <- n
	     }
	   }
	 }()

 return out
}
```

---

## 3. Worker Pool Pattern

### 🏊 Basic Worker Pool Implementation

```go
type Job struct {
	ID   int
	Data interface{}
}

type Result struct {
	JobID int
	Data  interface{}
	Error error
}

func workerPool(jobs <-chan Job, results chan<- Result, numWorkers int) {
	var wg sync.WaitGroup

	// Start worker goroutines
	for i := 0; i < numWorkers; i++ {
		wg.Add(1)
		go func(workerID int) {
			defer wg.Done()
			for job := range jobs {
				// Process job
				result := processJob(job)
				results <- result
			}
		}(i)
	}

	// Close results channel when all workers finish
	go func() {
		wg.Wait()
		close(results)
	}()
}

func processJob(job Job) Result {
	// Simulate work
	time.Sleep(time.Millisecond * 100)
	return Result{
		JobID: job.ID,
		Data:  fmt.Sprintf("Processed: %v", job.Data),
	}
}

// Usage
func main() {
	jobs := make(chan Job, 100)
	results := make(chan Result, 100)

	// Start worker pool with 5 workers
	workerPool(jobs, results, 5)

	// Send jobs
	go func() {
		for i := 0; i < 20; i++ {
			jobs <- Job{ID: i, Data: fmt.Sprintf("Job %d", i)}
		}
		close(jobs)
	}()

	// Collect results
	for result := range results {
		fmt.Printf("Result %d: %v\n", result.JobID, result.Data)
	}
}

```

### 🎯 Advanced Worker Pool with Context

```go
func workerPoolWithContext(
	ctx context.Context,
	jobs <-chan Job,
	results chan<- Result,
	numWorkers int,
) {
	var wg sync.WaitGroup

	for i := 0; i < numWorkers; i++ {
		wg.Add(1)
		go func(workerID int) {
			defer wg.Done()
			for {
				select {
				case job, ok := <-jobs:
					if !ok {
						// Jobs channel closed
						return
					}
					// Check context before processing
					if ctx.Err() != nil {
						return
					}
					result := processJob(job)
					// Try to send result, respect context
					select {
					case results <- result:
					case <-ctx.Done():
						return
					}
				case <-ctx.Done():
					// Context canceled, exit
					return
				}
			}
		}(i)
	}

	// Wait and close results
	go func() {
		wg.Wait()
		close(results)
	}()
}

// Usage with timeout
func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	jobs := make(chan Job, 100)
	results := make(chan Result, 100)

	workerPoolWithContext(ctx, jobs, results, 5)

	// Process...
}

```

### 💡 Worker Pool Benefits

- ✅ **Controlled concurrency**: Limit number of concurrent operations
- ✅ **Resource management**: Prevent resource exhaustion
- ✅ **Better performance**: Reuse goroutines instead of creating new ones
- ✅ **Graceful shutdown**: Easy to implement with context

---

## 4. Singleflight Pattern

### 🎯 Problem Singleflight Solves

**Scenario**: Multiple goroutines request the same expensive resource simultaneously (cache miss, API call, database query).

**Without Singleflight**:

```
Request 1 ──► API Call 1
Request 2 ──► API Call 2  } All make the same call!
Request 3 ──► API Call 3
Request 4 ──► API Call 4
```

**With Singleflight**:

```
Request 1 ──┐
Request 2 ──┤──► Single API Call ──► Shared Result
Request 3 ──┤
Request 4 ──┘
```

### 💻 Implementation Example

```go
type Cache struct {
	mu    sync.RWMutex
	items map[string]interface{}
	group singleflight.Group
}

func (c *Cache) Get(
	key string,
	fetch func() (interface{}, error),
) (interface{}, error) {
	// Check cache first
	c.mu.RLock()
	if val, ok := c.items[key]; ok {
		c.mu.RUnlock()
		return val, nil
	}
	c.mu.RUnlock()

	// Use singleflight to prevent duplicate fetches
	val, err, shared := c.group.Do(key, func() (interface{}, error) {
		// Only ONE goroutine executes this
		data, err := fetch()
		if err == nil {
			c.mu.Lock()
			c.items[key] = data
			c.mu.Unlock()
		}
		return data, err
	})

	if shared {
		fmt.Println("Result was shared from another goroutine")
	}

	return val, err
}

// Usage
func main() {
	cache := &Cache{items: make(map[string]interface{})}
	var wg sync.WaitGroup

	// Simulate 100 concurrent requests for same key
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			val, err := cache.Get("expensive-key", func() (interface{}, error) {
				// This will only be called ONCE!
				fmt.Println("Fetching data from API...")
				time.Sleep(time.Second)
				return "expensive-data", nil
			})
			if err != nil {
				fmt.Printf("Goroutine %d: Error: %v\n", id, err)
			} else {
				fmt.Printf("Goroutine %d: %v\n", id, val)
			}
		}(i)
	}

	wg.Wait()
}

```

### 📊 Singleflight Methods

```go
// Do: Execute and cache result for all callers
val, err, shared := group.Do(key, func() (interface{}, error) {
    return fetch()
})

// DoChan: Returns channel for async result
ch := group.DoChan(key, func() (interface{}, error) {
    return fetch()
})
result := <-ch

// Forget: Remove key so next call will execute
group.Forget(key)

```

---

## 5. Synchronization Primitives

### 🔒 Mutex vs RWMutex

### Mutex - Exclusive Access

```go
var mu sync.Mutex
var counter int// Only ONE goroutine can hold the lockmu.Lock()counter++mu.Unlock()// ⚠️ Common mistake: Forgetting to unlockmu.Lock()defer mu.Unlock() // Better: Use defercounter++

// Only ONE goroutine can hold the lock
mu.Lock()
counter++
mu.Unlock()

// ⚠️ Common mistake: Forgetting to unlock
mu.Lock()
// Do some work
mu.Unlock()

// Better: Use defer
mu.Lock()
defer mu.Unlock()
counter++
```

### RWMutex - Multiple Readers, Single Writer

```go
var rwmu sync.RWMutex
var data map[string]int// Multiple readers can hold read lock simultaneouslyrwmu.RLock()val := data["key"]rwmu.RUnlock()// Write lock is exclusive (no readers or writers)rwmu.Lock()data["key"] = 42rwmu.Unlock()

// Multiple readers can hold read lock simultaneously
rwmu.RLock()
val := data["key"]
rwmu.RUnlock()

// Write lock is exclusive (no readers or writers)
rwmu.Lock()
data["key"] = 42
rwmu.Unlock()
```

**When to use RWMutex**:
- ✅ Read-heavy workloads (reads >> writes)
- ✅ Expensive data structures
- ❌ Write-heavy workloads (use Mutex)

### ⏳ WaitGroup

```go
var wg sync.WaitGroup
// Launch multiple goroutines
for i := 0; i < 10; i++ {
	wg.Add(1) // Increment counter
	go func(i int) {
		defer wg.Done() // Decrement counter
		// Do work
		fmt.Println("Worker", i)
		time.Sleep(time.Second)
	}(i)
}
// Wait for all goroutines to finish
wg.Wait()
fmt.Println("All workers done!")
```

**⚠️ Common Mistakes**:

```go
// ❌ WRONG: Add inside goroutine (race condition)
go func() {
	wg.Add(1) // TOO LATE!
	defer wg.Done()
}()

// ✅ CORRECT: Add before launching
wg.Add(1)
go func() {
	defer wg.Done()
}()

```

### 1️⃣ Once - Execute Exactly Once

```go
var (
	once     sync.Once
	instance *Singleton
)

type Singleton struct {
	// Add fields as needed
}

func (s *Singleton) Initialize() {
	// Initialize the singleton
}

func GetInstance() *Singleton {
	once.Do(func() {
		// This is executed exactly once
		// Even if called by multiple goroutines
		instance = &Singleton{}
		instance.Initialize()
	})
	return instance
}

// Thread-safe singleton pattern
func main() {
	var wg sync.WaitGroup

	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			s := GetInstance() // All get same instance
			fmt.Println(s)
		}()
	}

	wg.Wait()
}

```

### 🚦 Cond - Conditional Variables

```go
import (
	"fmt"
	"sync"
	"time"
)

var (
	mu    sync.Mutex
	cond  = sync.NewCond(&mu)
	ready bool
	data  []int
)

// Producer goroutine
go func() {
	time.Sleep(2 * time.Second)
	mu.Lock()
	data = []int{1, 2, 3, 4, 5}
	ready = true
	mu.Unlock()
	cond.Broadcast() // Wake all waiting goroutines
	// cond.Signal() // Wake one waiting goroutine
}()

// Consumer goroutines
for i := 0; i < 3; i++ {
	go func(id int) {
		mu.Lock()
		// Wait for condition
		for !ready {
			cond.Wait() // Releases lock and waits
		}
		// Process data
		fmt.Printf("Consumer %d: %v\n", id, data)
		mu.Unlock()
	}(i)
}

```

**Cond methods**:
- `Wait()`: Release lock, sleep until signaled, reacquire lock
- `Signal()`: Wake one waiting goroutine
- `Broadcast()`: Wake all waiting goroutines

---

## 6. Memory Management

### 📦 Stack vs Heap Allocation

### Stack Allocation (Fast)

```go
func processUser() {    // Allocated on stack
	u := User{Name: "John", Age: 30}
	fmt.Println(u.Name)    // Automatically cleaned up when function returns
}
```

**Characteristics**:
- ⚡ **Fast**: Simple pointer bump
- 🧹 **Auto cleanup**: No GC needed
- 📏 **Limited size**: Default 1MB on Linux
- 🔒 **Thread-local**: No synchronization needed

### Heap Allocation (Slower)

```go
func createUser() *User {
	// ESCAPES to heap (returned by pointer)
	u := User{Name: "John", Age: 30}
return &u
}
```

**Characteristics**:
- 🐌 **Slower**: GC managed
- 🗑️ **GC overhead**: Must be garbage collected
- 📈 **Unlimited size**: Only limited by system memory
- 🔄 **Shared**: Can be accessed by multiple goroutines

### 🔍 Escape Analysis

Variables “escape” to heap when:

```go
// 1. Returned by reference
func escape1() *int {
	x := 42
	return &x // x escapes
}

// 2. Assigned to interface
func escape2() {
	var i interface{}
	x := 42
	i = x // x escapes
}

// 3. Sent to channel
func escape3() {
	ch := make(chan *int)
	x := 42
	ch <- &x // x escapes
}

// 4. Too large for stack
func escape4() {
	// Large array escapes
	x := [1000000]int{}
	use(x)
}

// 5. Size unknown at compile time
func escape5(n int) {
	x := make([]int, n) // Escapes (n not constant)
}

func use(x [1000000]int) {
	// Use the array
}

// Check escape analysis
// go build -gcflags="-m" main.go

```

### 🎯 Memory Optimization Tips

### 1. Prefer Value Types

```go
// ❌ Slower: Pointer slice
func process(users []*User) {
	// More allocations, more GC pressure
}

// ✅ Faster: Value slice
func process(users []User) {
	// Better cache locality, less GC
}

type User struct {
	ID   int
	Name string
}

```

### 2. Reuse Slices

```go
var buf []byte

func process() {
	// ❌ New allocation every time
	buf := make([]byte, 1024)

	// ✅ Reuse buffer
	buf = buf[:0] // Reset length, keep capacity

	// Use buf...
}
```

### 3. sync.Pool for Frequent Allocations

```go

var bufferPool = sync.Pool{
	New: func() interface{} {
		return make([]byte, 0, 1024)
	},
}

func processData(data []byte) {
	// Get buffer from pool
	buf := bufferPool.Get().([]byte)
	defer bufferPool.Put(buf[:0]) // Return to pool

	// Use buffer
	buf = append(buf, data...)

	// Process...
}
```

### 4. Avoid String Concatenation

```go
// ❌ Slow: Creates many temporary strings
func concat(strs []string) string {
	result := ""
	for _, s := range strs {
		result += s // New allocation each time!
	}
	return result
}

// ✅ Fast: Single allocation
func concat(strs []string) string {
	var builder strings.Builder
	for _, s := range strs {
		builder.WriteString(s)
	}
	return builder.String()
}

```

### 🗑️ Garbage Collector

### Tricolor Mark & Sweep Algorithm

```
WHITE: Not yet visited
GRAY:  Visited, children not scanned
BLACK: Fully processed

1. All objects start WHITE
2. Roots (globals, stack) marked GRAY
3. Process GRAY objects:
   - Mark referenced objects GRAY
   - Mark current object BLACK
4. Repeat until no GRAY objects
5. Sweep WHITE objects (garbage)
```

### GC Tuning

```go
// GOGC controls GC frequency
// Default: 100 (run GC when heap grows 100%)
// Higher = less frequent GC, more memory
// Lower = more frequent GC, less memory
// GOGC=off disables GC
// Set via environment variable
// export GOGC=200// Or programmatically
debug.SetGCPercent(200)// Force GC (rarely needed)runtime.GC()
// Get GC statsvar stats runtime.MemStats
runtime.ReadMemStats(&stats)fmt.Printf("Alloc: %d MB\n", stats.Alloc/1024/1024)fmt.Printf("NumGC: %d\n", stats.NumGC)
```

---

## 7. Context Package

### 🎯 Context Usage Patterns

```go
// 1. Context with timeout
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel() // Always call cancel to release resources

// 2. Context with cancellation
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

// 3. Context with deadline
deadline := time.Now().Add(10 * time.Second)
ctx, cancel := context.WithDeadline(context.Background(), deadline)
defer cancel()

// 4. Context with value (use sparingly!)
ctx = context.WithValue(ctx, "userID", 123)
ctx = context.WithValue(ctx, "requestID", "abc-123")

// Retrieve value
if userID, ok := ctx.Value("userID").(int); ok {
    fmt.Println("User ID:", userID)
}
```

### 📋 Best Practices

```go
// ✅ Accept context as first parameter
func ProcessRequest(ctx context.Context, data Data) error {
	// ...
	return nil
}

// ✅ Pass context through call chain
func Handler(ctx context.Context) {
	result := DatabaseQuery(ctx, "SELECT ...")
	CacheStore(ctx, result)
}

// ✅ Check context cancellation
func LongRunning(ctx context.Context) error {
	for {
		select {
		case <-ctx.Done():
			return ctx.Err() // Canceled or timed out
		default:
			// Do work
		}
	}
}

// ❌ Don't store context in struct
type Server struct {
	ctx context.Context // WRONG!
}

// ✅ Pass as parameter instead
func (s *Server) Handle(ctx context.Context) {
	// Use ctx parameter
}

// ❌ Don't pass nil context
func bad() {
	ProcessRequest(nil, data{}) // WRONG!
}

// ✅ Use context.Background() or context.TODO()
func good() {
	ctx := context.Background()
	ProcessRequest(ctx, data{})
}
```

### 🔄 Context Cancellation Propagation

```go
func example() {
	// Parent context
	parent, parentCancel := context.WithCancel(context.Background())
	defer parentCancel()

	// Child contexts
	child1, cancel1 := context.WithTimeout(parent, 5*time.Second)
	defer cancel1()

	child2, cancel2 := context.WithCancel(parent)
	defer cancel2()

	// Canceling parent cancels ALL children
	go func() {
		time.Sleep(2 * time.Second)
		parentCancel() // Cancels child1 and child2
	}()

	// Children detect cancellation
	<-child1.Done() // Canceled by parent
	<-child2.Done() // Canceled by parent
}
```

### 💡 Context Use Cases

```go
// 1. HTTP Request handling
func handleRequest(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context() // Get request context

	// Context canceled when client disconnects
	data, err := fetchData(ctx)
	if ctx.Err() != nil {
		return // Client gone, stop work
	}

	json.NewEncoder(w).Encode(data)
}

// 2. Database queries
func queryUser(ctx context.Context, id int) (*User, error) {
	query := "SELECT * FROM users WHERE id = ?"
	// Query respects context timeout/cancellation
	row := db.QueryRowContext(ctx, query, id)
	var user User
	err := row.Scan(&user.ID, &user.Name)
	return &user, err
}

// 3. Graceful shutdown
func server() {
	ctx, cancel := context.WithCancel(context.Background())

	// Cancel on SIGINT/SIGTERM
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, os.Interrupt, syscall.SIGTERM)
	go func() {
		<-sigCh
		fmt.Println("Shutting down...")
		cancel() // Propagates to all workers
	}()

	// Workers check context
	for i := 0; i < 10; i++ {
		go worker(ctx, i)
	}

	<-ctx.Done()
	fmt.Println("All workers stopped")
}

type Data struct{}
type User struct {
	ID   int
	Name string
}

var db *sql.DB

func DatabaseQuery(ctx context.Context, query string) interface{} {
	return nil
}

func CacheStore(ctx context.Context, result interface{}) {}

func fetchData(ctx context.Context) (interface{}, error) {
	return nil, nil
}

func worker(ctx context.Context, i int) {
	for {
		select {
		case <-ctx.Done():
			return
		default:
			// Do work
		}
	}
}

```

---

## 8. Advanced Patterns & Anti-patterns

### 🔄 Pipeline Pattern

```go
// Generator stage
func generate(nums ...int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for _, n := range nums {
			out <- n
		}
	}()
	return out
}

// Transformation stage
func square(in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for n := range in {
			out <- n * n
		}
	}()
	return out
}

// Filter stage
func filter(in <-chan int, predicate func(int) bool) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for n := range in {
			if predicate(n) {
				out <- n
			}
		}
	}()
	return out
}

// Pipeline Usage
func pipelineExample() {
	// Pipeline: generate → square → filter
	numbers := generate(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
	squared := square(numbers)
	filtered := filter(squared, func(n int) bool {
		return n > 50
	})

	// Consume results
	for result := range filtered {
		fmt.Println(result) // 64, 81, 100
	}
}
```

### ⏱️ Rate Limiting

### Token Bucket Pattern

```go
// Rate limiter: 10 operations per second
func rateLimiter(rps int) <-chan struct{} {
	rate := time.Second / time.Duration(rps)
	ticker := time.NewTicker(rate)
	tokens := make(chan struct{}, rps) // Burst capacity

	go func() {
		defer ticker.Stop()
		// Fill bucket initially
		for i := 0; i < rps; i++ {
			tokens <- struct{}{}
		}
		// Refill at rate
		for range ticker.C {
			select {
			case tokens <- struct{}{}:
			default: // Bucket full, skip
			}
		}
	}()

	return tokens
}

// Token bucket usage
func tokenBucketExample() {
	limiter := rateLimiter(10) // 10 ops/sec
	for i := 0; i < 100; i++ {
		<-limiter // Wait for token
		go makeRequest(i)
	}
}
```

### golang.org/x/time/rate

```go
// Rate limiter using golang.org/x/time/rate
func advancedRateLimiterExample() {
	// Create limiter: 10 requests/sec, burst of 20
	limiter := rate.NewLimiter(10, 20)

	// Wait for permission
	err := limiter.Wait(context.Background())
	if err != nil {
		// Context canceled
		return
	}

	// Or check without blocking
	if limiter.Allow() {
		// Permission granted
		makeRequest()
	} else {
		// Rate limited
		fmt.Println("Rate limited")
	}
}

func makeRequest(i int) {
	fmt.Printf("Making request %d\n", i)
}

func makeRequest() {
	fmt.Println("Making request")
}

```

### ❌ Common Anti-patterns

### 1. Goroutine Leaks

```go
// ❌ LEAK: Goroutine waits forever
func leak() {
	ch := make(chan int)
	go func() {
		ch <- 42 // Blocks forever
	}()
}

// Function returns, goroutine stuck
// ✅ FIX: Use buffered channel or context
func noLeak() {
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()
	ch := make(chan int)
	go func() {
		select {
			case ch <- 42:
			case <-ctx.Done():
			return // Goroutine can exit
		}
	}()
}
```

### 2. Race Conditions

```go
// ❌ RACE: Concurrent map access
var m = make(map[string]int)

func race() {
	go func() { m["key"] = 1 }()
	go func() { m["key"] = 2 }()
	go func() { fmt.Println(m["key"]) }()
}

// ✅ FIX: Use mutex or sync.Map
var (
	mu sync.RWMutex
	m2  = make(map[string]int)
)

func safe() {
	go func() {
		mu.Lock()
		m2["key"] = 1
		mu.Unlock()
	}()
	go func() {
		mu.RLock()
		fmt.Println(m2["key"])
		mu.RUnlock()
	}()
}
```

### 3. Channel Misuse

```go
// ❌ WRONG: Using channel for simple synchronization
var done = make(chan bool)

func wait() {
	<-done
}

func signal() {
	done <- true
}

// ✅ BETTER: Use sync.WaitGroup or context
var wg sync.WaitGroup

func work() {
	defer wg.Done()
	// ...
}

func main() {
	wg.Add(1)
	go work()
	wg.Wait()
}
```

### 4. Context Misuse

```go
// ❌ WRONG: Storing context in struct
type Handler struct {
	ctx context.Context // Don't do this!
}

// ✅ CORRECT: Pass as parameter
type Handler2 struct {
	// Other fields
}

func (h *Handler2) Handle(ctx context.Context) {
	// Use ctx parameter
}
```

---

## 9. Testing Concurrent Code

### 🏁 Race Detection

```bash
# Run tests with race detectorgo test -race ./...
# Run program with race detectorgo run -race main.go
# Build with race detectorgo build -race
```

**Race detector finds**:
- Concurrent map access
- Unprotected shared variables
- Channel races
- Data races

### 🧪 Testing with Goroutines

```go
func TestConcurrent(t *testing.T) {
	var wg sync.WaitGroup
	errors := make(chan error, 10)

	// Launch 10 concurrent operations
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			if err := someOperation(id); err != nil {
				errors <- fmt.Errorf("goroutine %d: %w", id, err)
			}
		}(i)
	}

	// Wait and collect errors
	wg.Wait()
	close(errors)
	for err := range errors {
		t.Error(err)
	}
}
func someOperation(id int) error {
	return nil
}

```

### ⏱️ Testing with Timeouts

```go

func TestWithTimeout(t *testing.T) {
	done := make(chan bool)
	go func() {
		// Test code
		result := longRunningOperation()
		if result != expected {
			t.Errorf("got %v, want %v", result, expected)
		}
		done <- true
	}()

	select {
	case <-done:
		// Test passed
	case <-time.After(5 * time.Second):
		t.Fatal("Test timed out")
	}
}

func longRunningOperation() interface{} {
	return nil
}

```

### 🔍 Table-Driven Concurrent Tests

```go
func TestConcurrentOperations(t *testing.T) {
	tests := []struct {
		name     string
		input    int
		expected int
	}{
		{"case1", 1, 2},
		{"case2", 5, 10},
		{"case3", 10, 20},
	}

	for _, tt := range tests {
		tt := tt // Capture range variable
		t.Run(tt.name, func(t *testing.T) {
			t.Parallel() // Run tests concurrently
			result := operation(tt.input)
			if result != tt.expected {
				t.Errorf("got %d, want %d", result, tt.expected)
			}
		})
	}
}

var expected interface{} = nil

func operation(input int) int {
	return input * 2
}

```

---

## 10. Performance Considerations

### 📊 Profiling

```go
import _ "net/http/pprof"
func main() {
	// Start pprof server
	go func() {
		log.Println(http.ListenAndServe("localhost:6060", nil))
	}()
	// Your application code
}
```

**Access profiles**:

```bash
# CPU profilego tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
# Memory profilego tool pprof http://localhost:6060/debug/pprof/heap
# Goroutine profilego tool pprof http://localhost:6060/debug/pprof/goroutine
# Block profilego tool pprof http://localhost:6060/debug/pprof/block
# Mutex profilego tool pprof http://localhost:6060/debug/pprof/mutex
```

### 🏆 Benchmarking

```go
func BenchmarkChannelVsMutex(b *testing.B) {
	b.Run("Channel", func(b *testing.B) {
	ch := make(chan int, 1)
	ch <- 0
	b.ResetTimer()
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
		val := <-ch
    val++
    ch <- val
  }
 })
})

 b.Run("Mutex", func(b *testing.B) {
 var mu sync.Mutex
 var counter int
 b.ResetTimer()
 b.RunParallel(func(pb *testing.PB) {
	 for pb.Next() {
		 mu.Lock()
		 counter++
		 mu.Unlock()
	 }
 })
})
}

 // Run benchmarks
 // go test -bench=. -benchmem
```

### 💡 Performance Tips

### 1. Choose Right Synchronization

```go
// Channel: ~150ns per operation
// Mutex:   ~20ns per operation
// Atomic:  ~1ns per operation
// Use atomic for simple countersvar counter int64atomic.AddInt64(&counter, 1)// Use mutex for complex statevar mu sync.Mutex
mu.Lock()
complexState.update()
mu.Unlock()// Use channels for coordination
results := make(chan Result)
go worker(results)
result := <-results
```

### 2. Reduce Allocations

```go
// Use benchstat to compare
// go test -bench=. -count=10 | benchstat
```

### 3. Optimal GOMAXPROCS

```go
// Default: number of CPU cores
runtime.GOMAXPROCS(runtime.NumCPU())// Can adjust based on workload
// CPU-bound: NumCPU()// I/O-bound: NumCPU() * 2
```

---

## 🎓 Common Pitfalls

### Q1: When to use channels vs mutexes?

**A**: Use **channels** for communication and coordination between goroutines. Use **mutexes** for protecting shared state.

**Rule of thumb**: “Don’t communicate by sharing memory; share memory by communicating.”

```go
// Channel: Passing datach := make(chan Data)
// go func() {
// 	ch <- fetchData()
// }()
// data := <-ch
// Mutex: Protecting statevar mu sync.Mutex
mu.Lock()balance += amount
mu.Unlock()
```

---

### Q2: What’s the difference between buffered and unbuffered channels?

**A**:
- **Unbuffered** (`make(chan T)`): Synchronous. Sender blocks until receiver is ready.
- **Buffered** (`make(chan T, n)`): Asynchronous up to buffer capacity. Sender blocks only when full.

**Use unbuffered** for synchronization points.
**Use buffered** to decouple sender/receiver timing.

---

### Q3: How do you prevent goroutine leaks?

**A**: Ensure all goroutines have a way to exit:

1. **Close channels** when done
2. **Use context** for cancellation
3. **Set timeouts** for operations
4. **Use buffered channels** if receiver might not always read
5. **Profile regularly** with `go tool pprof`

```go
// Good patternfunc worker(ctx context.Context) {
	for {
		select {
			case work := <-workCh:
			process(work)
			case <-ctx.Done():
			return // Can exit!
		}
	}
}
```

---

### Q4: Explain the GMP model

**A**: The GMP model is Go’s scheduler:

- **G (Goroutine)**: User-space thread (~2KB stack)
- **M (Machine)**: OS thread that executes goroutines
- **P (Processor)**: Scheduling context with local run queue

**How it works**:
1. Each P has a local queue of G’s
2. M executes G’s from its P
3. When P’s queue is empty, M steals from other P’s (work-stealing)
4. When G blocks on syscall, M detaches from P
5. Another M can attach to P and continue executing G’s

**Benefits**: Efficient scheduling, minimal context switching, excellent scalability.

---

### Q5: What’s the zero value of a channel?

**A**: `nil`

**Behavior**:
- Receiving from `nil` channel: **blocks forever**
- Sending to `nil` channel: **blocks forever**
- Closing `nil` channel: **panic**

**Use case**: Disable a case in select

```go
var ch chan int // nil
select {
	case <-ch: // Never selected
	case <-other:    // ...
}
```

---

### Q6: How does select work with multiple ready channels?

**A**: Go **randomly** selects one of the ready cases to ensure fairness and prevent starvation.

```go
ch1 := make(chan int, 1)
ch2 := make(chan int, 1)
ch1 <- 1
ch2 <- 2
select {
	case <-ch1: // Might be selected
	case <-ch2: // Or this might be selected
}// Random selection between ready cases
```

---

### Q7: What causes variables to escape to the heap?

**A**: Variables escape when:
1. Returned by pointer reference
2. Assigned to interface value
3. Sent to channel
4. Too large for stack
5. Size unknown at compile time
6. Stored in heap-allocated structure

Check with: `go build -gcflags="-m"`

---

### Q8: Explain sync.Pool and when to use it

**A**: `sync.Pool` is a cache for reusable objects, reducing GC pressure.

**When to use**:
- Objects allocated and deallocated frequently
- Temporary objects (like buffers)
- Costly allocation

**When NOT to use**:
- Long-lived objects
- Objects with important state (pool can clear anytime)

---

### Q9: What’s the difference between concurrency and parallelism?

**A**:
- **Concurrency**: Structure of program (multiple tasks in progress)
- **Parallelism**: Execution (multiple tasks running simultaneously)

*“Concurrency is about dealing with lots of things at once. Parallelism is about doing lots of things at once.”* - Rob Pike

---

### Q10: How do you handle errors from goroutines?

**A**: Use channels or error groups:

```go
// Option 1: Error channel
func errorChannelExample() {
	numWorkers := 5
	errors := make(chan error, numWorkers)

	for i := 0; i < numWorkers; i++ {
		go func() {
			if err := work(); err != nil {
				errors <- err
			}
		}()
	}
}

// Option 2: errgroup
func errgroupExample() error {
	numWorkers := 5
	g, ctx := errgroup.WithContext(context.Background())

	for i := 0; i < numWorkers; i++ {
		i := i
		g.Go(func() error {
			return work(ctx, i)
		})
	}

	if err := g.Wait(); err != nil {
		// Handle error
		return err
	}

	return nil
}

func work() error {
	return nil
}

func work(ctx context.Context, i int) error {
	return nil
}

```

---

## 🎯 Final Tips

### Demonstrate Understanding

1. **Explain tradeoffs**: Why choose one approach over another
2. **Consider edge cases**: What happens if channel is closed? Context canceled?
3. **Think about scale**: Will this work with 10K goroutines?
4. **Memory efficiency**: Stack vs heap, allocations
5. **Testing**: How would you test this?

### Common Coding Challenges

- Implement worker pool
- Rate limiter
- Cache with expiration
- Concurrent map
- Pipeline processing
- Graceful shutdown

### Key Principles to Mention

- ✅ “Share memory by communicating”
- ✅ Start with simple synchronization, add complexity if needed
- ✅ Prefer channels for ownership transfer
- ✅ Use context for cancellation
- ✅ Always provide way for goroutines to exit
- ✅ Profile before optimizing

---

## 📚 Additional Resources

- [Effective Go](https://golang.org/doc/effective_go)
- [Go Concurrency Patterns](https://www.youtube.com/watch?v=f6kdp27TYZs) - Rob Pike
- [Advanced Go Concurrency Patterns](https://www.youtube.com/watch?v=QDDwwePbDtw) - Sameer Ajmani
- [Go Memory Model](https://golang.org/ref/mem)
- [golang.org/x/sync](https://pkg.go.dev/golang.org/x/sync) - Extended sync primitives

---
