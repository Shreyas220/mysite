---
title: "Async"
meta_title: ""
description: "Understanding Fault Tolerance in Streaming Systems "
date: 2025-03-10T05:00:00Z
image: "/images/tokio.png"
categories: ["rust", "async"]
author: "Shreyas Mishra"
tags: ["rust", "async"]
draft: true
---

# Diving Deeper into Tokio: Task Creation, Scheduling, and the Mechanics of Await

In our previous exploration of Tokio, we covered the foundational concepts. Now, let's dive deeper into how Tokio manages concurrent tasks, how its scheduler works, and what really happens when you use the `.await` keyword.

## Task Creation and Management in Tokio

When you create tasks in Tokio, you're essentially telling the runtime to manage independent units of work that can progress concurrently. Let's build a more complex example to understand this better:

```rust
use tokio::time::{sleep, Duration};
use std::sync::{Arc, Mutex};

#[tokio::main]
async fn main() {
    // Shared state to track task completion
    let counter = Arc::new(Mutex::new(0));
    
    // Store join handles to await them later
    let mut handles = Vec::new();
    
    // Create 10 tasks with different behaviors
    for i in 0..10 {
        let counter_clone = Arc::clone(&counter);
        
        // Spawn a new task
        let handle = tokio::spawn(async move {
            // Simulate different processing times
            sleep(Duration::from_millis(i * 100)).await;
            
            // Critical section - update shared state
            {
                let mut count = counter_clone.lock().unwrap();
                *count += 1;
                println!("Task {} completed. Counter: {}", i, *count);
            }
            
            // Return a result from this task
            i * 10
        });
        
        handles.push(handle);
    }
    
    // Process task results as they complete
    for handle in handles {
        match handle.await {
            Ok(result) => println!("Got result: {}", result),
            Err(e) => eprintln!("Task failed: {}", e),
        }
    }
    
    // Final counter value
    println!("All tasks done. Final count: {}", *counter.lock().unwrap());
}
```

### What Happens Under the Hood When a Task is Created

When `tokio::spawn()` is called, several important steps occur:

1. **Future Transformation**: Your async block is transformed into a state machine that implements `Future`.

2. **Task Allocation**: Tokio creates a `Task<T>` struct that wraps your future. This includes:
   - The future itself
   - Scheduling metadata
   - Cancellation state
   - Memory for the output value

3. **Header Creation**: Each task has a header containing:
   ```rust
   struct Header {
       /// Current state of the task
       state: AtomicUsize,
       
       /// Pointer back to the owning scheduler
       owner: *const Scheduler,
       
       /// Queue links for the scheduler
       queue_next: UnsafeCell<*const Header>,
       
       /// Reference count
       ref_count: AtomicUsize,
   }
   ```

4. **Task Registration**: The task is registered with the scheduler by adding it to the appropriate queue.

5. **Waker Setup**: A waker is configured for the task so the runtime knows how to resume it.

The actual implementation in Tokio is more complex, using specialized memory management and optimizations, but this is the conceptual structure.

## Tokio's Scheduler: The Brain of the Operation

Tokio's multi-threaded scheduler is a sophisticated work-stealing system inspired by Golang's scheduler. Let's explore how tasks are actually scheduled.

### Scheduler Architecture

```rust
// Conceptual representation of Tokio's scheduler
struct Scheduler {
    // Worker threads
    workers: Vec<Worker>,
    
    // Global task queue
    global_queue: UnboundedSender<Task>,
    
    // Shutdown signal
    shutdown: AtomicBool,
}

struct Worker {
    // Thread handle
    thread: Option<JoinHandle<()>>,
    
    // Local task queue
    local_queue: Deque<Task>,
    
    // Reference to all workers for stealing
    stealers: Vec<Stealer<Task>>,
    
    // Worker ID
    id: usize,
}
```

### Task Scheduling Flow

1. **Initial Placement**: When a task is spawned, it's placed in the local queue of the worker thread that spawned it.

2. **Work Stealing**: If a worker runs out of tasks in its local queue:
   - It checks the global queue first
   - If the global queue is empty, it attempts to steal tasks from other workers
   - It steals half of another worker's queue to minimize contention

3. **Task Execution**: When a worker picks a task to run:
   - It calls `poll()` on the task's future
   - If the future completes, the result is stored and any waiters are notified
   - If the future is pending, the task is suspended until its waker is called

Let's visualize how our 10 tasks from the earlier example might be distributed:

```
Worker 1: [Task 0] [Task 4] [Task 8]
Worker 2: [Task 1] [Task 5] [Task 9]
Worker 3: [Task 2] [Task 6]
Worker 4: [Task 3] [Task 7]
```

If Worker 4 processes its tasks quickly, it might steal tasks from Worker 1:

```
Worker 1: [Task 0] [Task 4]
Worker 2: [Task 1] [Task 5] [Task 9]
Worker 3: [Task 2] [Task 6]
Worker 4: [Task 8] [Task 3] [Task 7]
```

### Real Implementation Details

In reality, Tokio uses a more sophisticated approach:

1. **LIFO Local Queues**: Each worker has a LIFO (stack) queue for better cache locality
2. **FIFO Stealing**: When stealing, tasks are taken in FIFO order
3. **Thread Parking**: Workers will park their threads when no work is available
4. **Priorities**: Some tasks can be given higher priority

## The Mechanics of Await: Where the Magic Happens

The `.await` keyword is where Rust's async system and Tokio's runtime truly connect. Let's demystify what happens when you call `.await`.

### Conceptual Example

```rust
async fn process_data(data: Vec<u32>) -> u32 {
    // This will be transformed into a state machine
    let intermediate = transform_data(data).await;
    let result = validate_result(intermediate).await;
    result
}
```

### The State Machine Transformation

The compiler transforms this into a state machine with approximately these states:

```rust
enum ProcessDataStateMachine {
    // Initial state
    Start(Vec<u32>),
    
    // Waiting for transform_data to complete
    WaitingOnTransform(TransformFuture),
    
    // Waiting for validate_result to complete
    WaitingOnValidate(ValidateFuture, IntermediateData),
    
    // Completed
    Done,
}
```

### When `.await` is Called

Let's walk through what happens at runtime when `.await` is encountered:

1. **Future Creation**: The expression before `.await` creates a future

2. **Initial Poll**: The future is polled once immediately

3. **Ready Check**: If the future is immediately ready:
   - The result is extracted
   - Execution continues to the next statement
   
4. **Suspension**: If the future is not ready:
   - The current task's state is saved (locals, position in code)
   - The future registers interest in some event with a waker
   - Control returns to the scheduler, which can run other tasks
   
5. **Wakeup**: When the event occurs:
   - The waker is called, marking the task as ready
   - The scheduler places the task back in a queue
   
6. **Resumption**: When the scheduler runs the task again:
   - The future is polled again
   - If ready, execution continues after the `.await`
   - If still pending, steps 4-6 repeat

Let's see this in a more concrete example:

```rust
#[tokio::main]
async fn main() {
    // Create multiple tasks that interact
    let (tx, rx) = tokio::sync::oneshot::channel();
    
    // Task 1: sender
    let sender_task = tokio::spawn(async move {
        println!("Sender: preparing data...");
        // Simulate work
        tokio::time::sleep(Duration::from_millis(500)).await;
        
        println!("Sender: sending data");
        tx.send("Hello from task 1").unwrap();
    });
    
    // Task 2: receiver
    let receiver_task = tokio::spawn(async move {
        println!("Receiver: waiting for data...");
        
        // This await will suspend this task until data arrives
        match rx.await {
            Ok(message) => {
                println!("Receiver: got message: {}", message);
                // More processing
                tokio::time::sleep(Duration::from_millis(200)).await;
                println!("Receiver: processed message");
                message
            }
            Err(_) => {
                println!("Receiver: sender dropped");
                "default message"
            }
        }
    });
    
    // Wait for both tasks
    let _ = sender_task.await.unwrap();
    let result = receiver_task.await.unwrap();
    println!("Main: final result: {}", result);
}
```

### Detailed Execution Flow With State Transitions

Let's trace the exact execution flow of the above code:

1. **Runtime Start**: 
   - Tokio runtime initializes (thread pool, I/O drivers, etc.)
   - Main task enters the scheduler

2. **Task Creation**:
   - Two tasks are created and their futures are wrapped in `Task` structures
   - Tasks are pushed to the scheduler's queue

3. **Main Task Suspension**:
   - Main task awaits `sender_task` completion
   - Since `sender_task` isn't done, main task suspends
   - Waker for `sender_task` is configured to wake main task

4. **Scheduler Decisions**:
   - Scheduler picks the next task (likely sender or receiver)
   - Let's say receiver runs first

5. **Receiver Task Execution**:
   - Prints "Receiver: waiting for data..."
   - Calls `.await` on `rx`
   - Since no data is available, registers interest in the channel
   - Suspends and returns control to scheduler

6. **Sender Task Execution**:
   - Prints "Sender: preparing data..."
   - Calls `.await` on `sleep`
   - Registers interest in the timer with the time driver
   - Suspends and returns control to scheduler

7. **Idle Period**:
   - All tasks are waiting on events
   - Tokio's time driver monitors the timer

8. **Timer Completion**:
   - After 500ms, time driver triggers the waker for the sleep future
   - Sender task is marked ready and queued

9. **Sender Task Resumption**:
   - Sender continues after the sleep
   - Prints "Sender: sending data"
   - Calls `tx.send(...)` which completes the channel
   - This triggers the waker for the channel
   - Receiver task is marked ready and queued
   - Sender task completes, notifying `sender_task.await` in main

10. **Receiver Task Resumption**:
    - Receiver continues after `rx.await`
    - Prints message
    - Awaits another sleep
    - Suspends again

11. **Main Task Partial Resumption**:
    - Main continues after `sender_task.await`
    - Awaits `receiver_task` completion
    - Suspends again

12. **Final Steps**:
    - After 200ms, receiver's timer completes
    - Receiver task resumes, completes
    - Main task wakes up, prints final result

## Implementing Mini-Tokio: Understanding Through Building

To truly understand Tokio's inner workings, let's build a simplified version that demonstrates the core concepts of tasks and the runtime. This isn't production-ready, but it illustrates the concepts:

```rust
use futures::future::BoxFuture;
use futures::{Future, FutureExt};
use std::cell::RefCell;
use std::collections::VecDeque;
use std::pin::Pin;
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll, Wake, Waker};

/// A very simplified task scheduler
struct MiniTokio {
    // Tasks that are ready to run
    ready_queue: Arc<Mutex<VecDeque<Task>>>,
}

// A spawned future with its waker
type Task = Arc<TaskInner>;

struct TaskInner {
    // The future to execute
    future: RefCell<BoxFuture<'static, ()>>,
    
    // Queue to push this task to when ready
    queue: Arc<Mutex<VecDeque<Task>>>,
}

// Implementation of the waker that notifies our scheduler
impl Wake for TaskInner {
    fn wake(self: Arc<Self>) {
        // Push this task to the ready queue
        self.queue.lock().unwrap().push_back(self.clone());
    }
}

impl MiniTokio {
    fn new() -> Self {
        MiniTokio {
            ready_queue: Arc::new(Mutex::new(VecDeque::new())),
        }
    }
    
    // Spawn a new task
    fn spawn<F>(&self, future: F) 
    where
        F: Future<Output = ()> + Send + 'static,
    {
        let task = Arc::new(TaskInner {
            future: RefCell::new(future.boxed()),
            queue: self.ready_queue.clone(),
        });
        
        // Push the task to the ready queue
        self.ready_queue.lock().unwrap().push_back(task);
    }
    
    // Run the mini runtime
    fn run(&self) {
        loop {
            // Get the next task
            let task = match self.ready_queue.lock().unwrap().pop_front() {
                Some(task) => task,
                None => break, // No more tasks to execute
            };
            
            // Create a waker for this task
            let waker = waker_from_task(task.clone());
            let mut context = Context::from_waker(&waker);
            
            // Poll the future
            let mut future = task.future.borrow_mut();
            
            // Check if the future has completed
            if let Poll::Ready(_) = Pin::new(&mut *future).poll(&mut context) {
                // Task completed, don't push it back to the queue
                continue;
            }
            
            // If Poll::Pending, the future will wake us up via the waker when ready
        }
    }
}

// Helper function to create a waker from a task
fn waker_from_task(task: Task) -> Waker {
    // This would use unsafe in real Tokio to create a waker directly
    // Here we'll use the simplification from futures crate
    futures::task::waker(task)
}

// Example usage
#[tokio::main] // We still need tokio for the async/await functionality
async fn main() {
    // Create our mini runtime
    let mini_tokio = MiniTokio::new();
    
    // Spawn a task
    mini_tokio.spawn(async {
        println!("Task 1: Starting");
        // Simulate an async operation
        // In a real implementation, this would be non-blocking
        std::thread::sleep(std::time::Duration::from_millis(100));
        println!("Task 1: Completed");
    });
    
    // Spawn another task
    mini_tokio.spawn(async {
        println!("Task 2: Starting");
        // Simulate a longer async operation
        std::thread::sleep(std::time::Duration::from_millis(200));
        println!("Task 2: Completed");
    });
    
    // Run our mini runtime
    mini_tokio.run();
}
```

This simplified implementation shows the core concepts:

1. **Tasks**: Each task contains a future and knows how to wake itself up
2. **Wakers**: The connection between an event and task resumption
3. **Scheduler**: The queue and execution loop that drives tasks forward
4. **Polling Mechanism**: How futures are driven to completion

## Advanced Tokio Scheduler Behaviors

With our foundation in place, let's look at some more advanced aspects of Tokio's scheduler:

### Blocking Detection and Mitigation

Tokio can detect when a task is blocking the thread for too long and will take action:

```rust
#[tokio::main]
async fn main() {
    // This spawns on the current thread pool
    tokio::spawn(async {
        // This would block the thread
        let result = tokio::task::block_in_place(|| {
            // Expensive CPU operation
            compute_fibonacci(1000000)
        });
        
        println!("Result: {}", result);
    });
    
    // This spawns on a dedicated blocking thread pool
    tokio::task::spawn_blocking(|| {
        // CPU-intensive work
        for i in 0..1000 {
            std::thread::sleep(std::time::Duration::from_millis(1));
        }
    }).await.unwrap();
}
```

Under the hood, Tokio:
1. Detects thread blocking with a combination of timing and heuristics
2. Spawns a replacement worker thread when needed
3. Maintains a separate thread pool for known blocking operations

### Task Budget and Yielding

Tokio implements a "task budget" system to ensure fair execution:

```rust
async fn cpu_intensive_task() {
    let mut counter = 0;
    
    loop {
        // Do some work
        for _ in 0..1_000_000 {
            counter += 1;
        }
        
        // Yield to let other tasks run
        tokio::task::yield_now().await;
        
        println!("Counter: {}", counter);
    }
}
```

When a task exceeds its budget (measured in "ticks" of CPU time):
1. The runtime automatically yields it, even without an explicit `.await`
2. Other tasks get a chance to run
3. The yielded task goes back to the end of the queue

### Cooperative Cancellation

Tasks can be cancelled through dropping their join handles:

```rust
#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        loop {
            println!("I'm still running!");
            tokio::time::sleep(Duration::from_secs(1)).await;
        }
    });
    
    // Wait for a bit
    tokio::time::sleep(Duration::from_secs(3)).await;
    
    // Cancel the task by dropping its handle
    drop(handle);
    
    // Give some time to see it's gone
    tokio::time::sleep(Duration::from_secs(2)).await;
    println!("Task should be cancelled now");
}
```

The cancellation process:
1. When the join handle is dropped, the task is marked for cancellation
2. When the task reaches its next `.await` point, cancellation is checked
3. If cancellation is detected, the future's memory is dropped and resources freed
4. There's no forceful termination - tasks must cooperate by having `.await` points

## Conclusion: The Power and Elegance of Tokio

Tokio's design represents a masterful balance between performance, safety, and ergonomics. By understanding how tasks are created, scheduled, and how `.await` works under the hood, we gain appreciation for the machinery that makes asynchronous Rust so powerful.

The real beauty of Tokio is that it handles all this complexity while presenting a clean API that lets developers focus on business logic rather than concurrency mechanics. The same principles that power Tokio - efficient scheduling, cooperative multitasking, and event-driven I/O - power many of the world's most performance-critical systems.

Whether you're building a high-performance web server, a data processing pipeline, or just learning asynchronous programming, the knowledge of Tokio's internals will help you write more efficient and correct asynchronous Rust code.


# Yielding in Infinite Workers: A Crucial Concept in Async Rust

Yes, you've touched on a critical insight about async Rust! For an infinitely running worker (like a long-running loop or continuous processing task), you absolutely need to implement explicit yielding to allow other tasks to execute.

## Why Yielding is Necessary

In Rust's cooperative multitasking model, tasks only give up control at `.await` points. Without an `.await` somewhere in your loop, an infinite worker would monopolize the thread and effectively starve all other tasks:

```rust
// Problematic infinite worker - will prevent other tasks from running
async fn bad_infinite_worker() {
    loop {
        // Process messages, check conditions, do work...
        // This loop never yields, so it will block the thread forever
    }
}
```

This task would grab the thread and never let go, which defeats the whole purpose of the async model.

## How to Implement Yielding in Infinite Workers

Here are the main approaches to properly implement yielding in an infinite worker:

### 1. Using `tokio::task::yield_now()`

The most direct approach is to periodically call `yield_now()`:

```rust
async fn good_infinite_worker() {
    loop {
        // Do some bounded amount of work
        for _ in 0..1000 {
            process_message();
        }
        
        // Explicitly yield to the scheduler to let other tasks run
        tokio::task::yield_now().await;
    }
}
```

This is saying "I'm voluntarily giving up my turn so other tasks can run." It's the simplest and most explicit way to yield.

### 2. Using `tokio::time::sleep()`

Another common approach is to introduce a tiny sleep:

```rust
async fn good_infinite_worker_with_sleep() {
    loop {
        // Do some work
        process_batch();
        
        // Sleep for a very short time - this both yields and adds a tiny delay
        tokio::time::sleep(Duration::from_millis(1)).await;
    }
}
```

This approach not only yields but also prevents the worker from consuming 100% CPU time when idle, which can be beneficial in some scenarios.

### 3. Building on Real Async Operations

The most natural approach is to base your worker on truly asynchronous operations that will yield naturally:

```rust
async fn natural_yielding_worker() {
    loop {
        // This naturally yields when the channel is empty
        match channel.recv().await {
            Some(message) => process_message(message),
            None => break, // Channel closed
        }
    }
}
```

When your worker waits for real I/O or other async events (like channel operations), it naturally yields when those operations aren't immediately ready.

## What Happens If You Don't Yield?

If you have an infinite worker without yielding:

1. The worker task will monopolize the thread it's running on
2. Other tasks scheduled on that thread will be starved (never get a chance to run)
3. If using Tokio's multi-threaded runtime, only tasks on that one thread are affected
4. If using single-threaded runtime, all tasks are blocked

This can lead to:
- Timeouts in other tasks
- Reduced throughput
- Unresponsive applications
- Deadlocks if the monopolizing task is waiting for something from another starved task

## A Complete Example

Here's a more complete example showing proper yielding in an infinite worker:

```rust
use tokio::sync::mpsc;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    // Channel for communication
    let (tx, rx) = mpsc::channel(100);
    
    // Spawn the worker task
    let worker = tokio::spawn(infinite_worker(rx));
    
    // Spawn a task that sends messages to the worker
    let producer = tokio::spawn(async move {
        for i in 0..1000 {
            tx.send(i).await.unwrap();
            sleep(Duration::from_millis(10)).await;
        }
    });
    
    // Spawn a task that just prints periodically to show it's still running
    let heartbeat = tokio::spawn(async {
        for i in 0..100 {
            println!("Heartbeat {}", i);
            sleep(Duration::from_millis(100)).await;
        }
    });
    
    // Wait for the producer to finish
    producer.await.unwrap();
    // Wait for the heartbeat to finish
    heartbeat.await.unwrap();
    // Cancel the worker (it would run forever otherwise)
    worker.abort();
}

async fn infinite_worker(mut rx: mpsc::Receiver<i32>) {
    let mut counter = 0;
    
    loop {
        // Approach 1: Process a batch of messages if available
        for _ in 0..10 {
            match rx.try_recv() {
                Ok(msg) => {
                    // Process message
                    counter += msg;
                }
                Err(_) => break,
            }
        }
        
        // Simulate some CPU-bound work
        let mut sum = 0;
        for i in 0..10000 {
            sum += i;
        }
        
        // Approach 2: Explicitly yield to let other tasks run
        tokio::task::yield_now().await;
        
        // Approach 3: Or use a minimal sleep
        // sleep(Duration::from_micros(1)).await;
        
        // Approach 4: Or wait for the next message (natural yielding)
        // if let Some(msg) = rx.recv().await {
        //     counter += msg;
        // } else {
        //     break; // Channel closed
        // }
        
        if counter % 1000 == 0 {
            println!("Worker processed counter = {}", counter);
        }
    }
}
```

## Best Practices for Infinite Workers

1. **Have a clear yield strategy**: Decide how and when your worker will yield.

2. **Choose an appropriate yield frequency**: Too frequent yields add overhead, too infrequent yields cause starvation.

3. **Consider work batch sizes**: Process a reasonable batch of work between yields.

4. **Use natural async points when possible**: Base your worker on channel operations, I/O, or other naturally async operations.

5. **Add monitoring**: Consider adding metrics to track how long your worker runs between yields.

6. **Consider backpressure**: If your worker can't keep up, implement mechanisms to handle backpressure.

The most elegant workers often combine both natural async operations with occasional explicit yields as a safety mechanism. This provides the best balance between performance and cooperative behavior.

Remember: In the cooperative world of async Rust, being a good citizen means yielding regularly to let other tasks have their turn!
