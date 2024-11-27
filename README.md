# Concurrency Study Guide

These notes serves as my guide to studying concurrency, focusing on three key areas:  

1. **Grand Central Dispatch (GCD)**  
2. **Modern Concurrency**  
3. **Multithreading Pitfalls**  

The goal is to consolidate all relevant information gathered from various books and articles into one comprehensive resource. It is designed to facilitate in-depth understanding and act as a quick reference during learning and development.

## What is concurrency?

Concurrency refers to the ability to perform multiple tasks simultaneously, enabling our app to remain responsive and efficient. It allows you to execute different parts of your code independently, making the most of the device's hardware resources, such as its multiple cores. Concurrency is essential for tasks that could block the main thread, such as network requests, file operations, or heavy computations.

### Parallelism vs. Concurrency

- Concurrency: Structuring a program to handle multiple tasks, which may or may not run at the same time.
- Parallelism: Executing multiple tasks literally at the same time, which depends on the system's hardware capabilities.

## Grand Central Dispatch

GCD is Apple’s implementation of C’s libdispatch library. Its purpose is to queue up tasks (either a method or a closure) that can be run in parallel, depending on availability of resources; it then executes the tasks on an available processor core.

All of the tasks that GCD manages for you are placed into GCD-managed first-in, **first-out** (FIFO) queues. Each task that you submit to a queue is then executed against a pool of threads fully managed by the system.

### Synchronous and Asynchronous Tasks

**sync**

- The task is executed immediately on the current thread.
- The current thread is blocked until the task completes.
- Typically used when the operation is small or needs to be performed sequentially.

**async**

- The task is scheduled to run on a different thread or later on the current thread.
- The current thread is **not blocked** and can continue executing other tasks.
- Typically used for long-running or non-blocking operations like network requests or file I/O.

##### Key Difference:
- **Sync** blocks the current thread.
- **Async** does not block the current thread.

#### Example: Sync vs. Async with DispatchQueue

```swift
import Foundation

// Simulate synchronous and asynchronous execution
func syncExample() {
    let queue = DispatchQueue.global()
    print("Before sync task")
    
    queue.sync { // Task runs synchronously on the global queue
        for i in 1...3 {
            print("Sync task \(i)")
        }
    }
    
    print("After sync task")
}

func asyncExample() {
    let queue = DispatchQueue.global()
    print("Before async task")
    
    queue.async { // Task runs asynchronously on the global queue
        for i in 1...3 {
            print("Async task \(i)")
        }
    }
    
    print("After async task")
}

// Call both examples
print("--- Synchronous Example ---")
syncExample()

print("\n--- Asynchronous Example ---")
asyncExample()

// Give some time for async task to complete (only needed in playgrounds)
Thread.sleep(forTimeInterval: 1)
```

##### Synchronous Example:
```
--- Synchronous Example ---
Before sync task
Sync task 1
Sync task 2
Sync task 3
After sync task
```

- The `sync` block ensures that the task completes before moving to the next line (`After sync task`).

##### Asynchronous Example:
```
--- Asynchronous Example ---
Before async task
After async task
Async task 1
Async task 2
Async task 3
```

- The `async` block schedules the task to run on a background thread, allowing the main thread to continue execution immediately (`After async task` is printed before the async task completes).

#### Practical Use Case
- sync: Performing a sequence of dependent operations (e.g., reading a file before parsing it).
- async: Fetching data from a network without blocking the UI thread.

### Serial and Concurrent Queues

In **Grand Central Dispatch (GCD)**, queues are used to execute tasks. There are two main types of queues:

- Serial queues only have a single thread associated with them and thus only allow a single task to be executed at any given time.
- Concurrent queue, on the other hand, is able to utilize as many threads as the system has resources for. Threads will be created and released as necessary on a concurrent queue.

#### **Concurrent Queue**
- Executes multiple tasks concurrently.
- Tasks start in the order they are added, but they may complete in any order since they can run simultaneously on different threads.
- Useful for performing multiple tasks in parallel.

#### **Serial Queue**
- Executes tasks one at a time in the order they are added.
- Ensures that each task finishes before the next one starts.
- Useful for ensuring sequential execution or avoiding race conditions when accessing shared resources.

| Feature         | Serial Queue                         | Concurrent Queue                   |
|-----------------|-------------------------------------|------------------------------------|
| Task Execution  | One task at a time                 | Multiple tasks simultaneously      |
| Order           | Tasks execute in the order added   | Tasks start in the order added, but may finish in any order |
| Use Case        | Sequential tasks, thread safety    | Parallel tasks, performance optimization |

#### Example: Serial Queue

```swift
import Foundation

func serialQueueExample() {
    let serialQueue = DispatchQueue(label: "com.example.serialQueue")
    
    print("Start Serial Queue")
    
    serialQueue.async {
        for i in 1...3 {
            print("Task 1 - \(i)")
            sleep(1) // Simulate a long task
        }
    }
    
    serialQueue.async {
        for i in 1...3 {
            print("Task 2 - \(i)")
            sleep(1) // Simulate a long task
        }
    }
    
    serialQueue.async {
        for i in 1...3 {
            print("Task 3 - \(i)")
            sleep(1) // Simulate a long task
        }
    }
    
    print("End Serial Queue")
}

serialQueueExample()
```

##### Output (sequential execution):
```
Start Serial Queue
End Serial Queue
Task 1 - 1
Task 1 - 2
Task 1 - 3
Task 2 - 1
Task 2 - 2
Task 2 - 3
Task 3 - 1
Task 3 - 2
Task 3 - 3
```

- Tasks are executed one at a time in the order they were added.

#### Example: Concurrent Queue

```swift
import Foundation

func concurrentQueueExample() {
    let concurrentQueue = DispatchQueue(label: "com.example.concurrentQueue", attributes: .concurrent)
    
    print("Start Concurrent Queue")
    
    concurrentQueue.async {
        for i in 1...3 {
            print("Task 1 - \(i)")
            sleep(1) // Simulate a long task
        }
    }
    
    concurrentQueue.async {
        for i in 1...3 {
            print("Task 2 - \(i)")
            sleep(1) // Simulate a long task
        }
    }
    
    concurrentQueue.async {
        for i in 1...3 {
            print("Task 3 - \(i)")
            sleep(1) // Simulate a long task
        }
    }
    
    print("End Concurrent Queue")
}

concurrentQueueExample()
```

#### Output (concurrent execution, order may vary):
```
Start Concurrent Queue
End Concurrent Queue
Task 1 - 1
Task 2 - 1
Task 3 - 1
Task 1 - 2
Task 2 - 2
Task 3 - 2
Task 1 - 3
Task 2 - 3
Task 3 - 3
```

- Tasks begin executing in the order they are added but may overlap because they are running concurrently.

#### When to Use

##### Serial Queue:
- When tasks need to execute one at a time, such as updating shared resources.
- For predictable task order and avoiding race conditions.

##### Concurrent Queue:
- When tasks are independent and can run in parallel to improve performance, such as downloading multiple files or performing batch processing.

### Asynchronous Doesn’t Mean Concurrent

While the difference seems subtle at first, **just because your tasks are asynchronous doesn’t mean they will run concurrently**. You’re actually able to submit asynchronous tasks to either a serial queue or a concurrent queue. Being synchronous or asynchronous simply identifies whether or not the queue on which you’re running the task must wait for the task to complete before it can spawn the next task.

### **Example: Async Task on a Serial Queue**

```swift
import Foundation

func asyncTaskOnSerialQueue() {
    let serialQueue = DispatchQueue(label: "com.example.serialQueue")
    
    print("Start asyncTaskOnSerialQueue")
    
    serialQueue.async {
        print("Async Task 1 - Start")
        sleep(2) // Simulate a long-running task
        print("Async Task 1 - End")
    }
    
    serialQueue.async {
        print("Async Task 2 - Start")
        sleep(2) // Simulate a long-running task
        print("Async Task 2 - End")
    }
    
    serialQueue.async {
        print("Async Task 3 - Start")
        sleep(2) // Simulate a long-running task
        print("Async Task 3 - End")
    }
    
    print("End asyncTaskOnSerialQueue")
}

asyncTaskOnSerialQueue()
```

#### Output:
```
Start asyncTaskOnSerialQueue
End asyncTaskOnSerialQueue
Async Task 1 - Start
Async Task 1 - End
Async Task 2 - Start
Async Task 2 - End
Async Task 3 - Start
Async Task 3 - End
```

**Explanation**:
- Tasks are executed **one at a time** in the order they are added (serial queue behavior).
- The **main thread** is not blocked; it moves on immediately after scheduling the tasks (`End asyncTaskOnSerialQueue` is printed before any task completes).

### **Example: Sync Task on a Serial Queue**

```swift
import Foundation

func syncTaskOnSerialQueue() {
    let serialQueue = DispatchQueue(label: "com.example.serialQueue")
    
    print("Start syncTaskOnSerialQueue")
    
    serialQueue.sync {
        print("Sync Task 1 - Start")
        sleep(2) // Simulate a long-running task
        print("Sync Task 1 - End")
    }
    
    serialQueue.sync {
        print("Sync Task 2 - Start")
        sleep(2) // Simulate a long-running task
        print("Sync Task 2 - End")
    }
    
    serialQueue.sync {
        print("Sync Task 3 - Start")
        sleep(2) // Simulate a long-running task
        print("Sync Task 3 - End")
    }
    
    print("End syncTaskOnSerialQueue")
}

syncTaskOnSerialQueue()
```
#### Output:
```
Start syncTaskOnSerialQueue
Sync Task 1 - Start
Sync Task 1 - End
Sync Task 2 - Start
Sync Task 2 - End
Sync Task 3 - Start
Sync Task 3 - End
End syncTaskOnSerialQueue
```

**Explanation**:
- Each task is executed **one at a time** in the order they are added (serial queue behavior).
- The **main thread is blocked** until each task completes sequentially. `End syncTaskOnSerialQueue` is only printed after all tasks are finished.

#### Combining Both
You can mix **async** and **sync** tasks on the same serial queue to achieve flexible behavior:

```swift
let serialQueue = DispatchQueue(label: "com.example.mixedQueue")

serialQueue.async {
    print("Async Task - Start")
    sleep(1)
    print("Async Task - End")
}

serialQueue.sync {
    print("Sync Task - Start")
    sleep(2)
    print("Sync Task - End")
}

serialQueue.async {
    print("Another Async Task - Start")
    sleep(1)
    print("Another Async Task - End")
}
```

#### Expected Output:
```
Async Task - Start
Async Task - End
Sync Task - Start
Sync Task - End
Another Async Task - Start
Another Async Task - End
``` 

This demonstrates how combining async and sync tasks can help balance responsiveness and sequential execution when working with serial queues in Swift.

### Operations
Operations are a higher-level abstraction over dispatch queues. They are part of the Operation and OperationQueue classes, which provide more control and flexibility for managing tasks compared to GCD’s raw dispatch queues.

- Tasks are encapsulated as Operation objects, making them reusable and easier to manage.
- Operations can have dependencies, meaning one operation can wait for another to complete before starting.
- You can set priorities for operations to control their execution order.
- Operations can be canceled gracefully using the cancel() method.
- Operations can be executed concurrently or serially within an OperationQueue.

Operations can exist in any of the following states:

- `isReady`
- `isExecuting`
- `isCancelled`
- `isFinished`

### Creating and Using Operations

#### BlockOperation

The `BlockOperation` class allows you to execute blocks of code as operations. `BlockOperation` subclasses Operation for you and manages the concurrent execution of one or more closures on the default global queue. However, being an actual `Operation` subclass lets you take advantage of all the other features of an operation.

```swift
import Foundation

func blockOperationExample() {
    let operationQueue = OperationQueue() // Create an operation queue
    
    // Create a BlockOperation
    let operation = BlockOperation {
        print("Task 1 - Start")
        sleep(2) // Simulate a long-running task
        print("Task 1 - End")
    }
    
    // Add another block to the same operation
    operation.addExecutionBlock {
        print("Task 2 - Start")
        sleep(1) // Simulate another task
        print("Task 2 - End")
    }
    
    // Add operation to the queue
    operationQueue.addOperation(operation)
    
    print("All tasks added to the queue")
}

blockOperationExample()
```

#### Output:
```
All tasks added to the queue
Task 1 - Start
Task 2 - Start
Task 2 - End
Task 1 - End
```

- Tasks in the same `BlockOperation` run concurrently on different threads.

##### 2. **Using Custom `Operation`**
You can subclass `Operation` to create a custom task with more control.

```swift
import Foundation

class CustomOperation: Operation {
    override func main() {
        if isCancelled { return } // Check for cancellation
        print("Custom Operation - Start")
        sleep(2) // Simulate a task
        if isCancelled { return }
        print("Custom Operation - End")
    }
}

func customOperationExample() {
    let operationQueue = OperationQueue()
    
    // Create a custom operation
    let customOperation = CustomOperation()
    
    // Add operation to the queue
    operationQueue.addOperation(customOperation)
    
    print("Custom operation added to the queue")
}

customOperationExample()
```

##### Output:
```
Custom operation added to the queue
Custom Operation - Start
Custom Operation - End
```

##### 3. **Adding Dependencies**
Operations can depend on other operations. This ensures that one operation waits for another to finish before starting.

```swift
import Foundation

func dependencyExample() {
    let operationQueue = OperationQueue()
    
    let operation1 = BlockOperation {
        print("Operation 1 - Start")
        sleep(1)
        print("Operation 1 - End")
    }
    
    let operation2 = BlockOperation {
        print("Operation 2 - Start")
        sleep(1)
        print("Operation 2 - End")
    }
    
    let operation3 = BlockOperation {
        print("Operation 3 - Start")
        sleep(1)
        print("Operation 3 - End")
    }
    
    // Set dependencies
    operation2.addDependency(operation1) // Operation 2 waits for Operation 1
    operation3.addDependency(operation2) // Operation 3 waits for Operation 2
    
    // Add operations to the queue
    operationQueue.addOperations([operation1, operation2, operation3], waitUntilFinished: false)
    
    print("All operations added to the queue")
}

dependencyExample()
```

##### Output:
```
All operations added to the queue
Operation 1 - Start
Operation 1 - End
Operation 2 - Start
Operation 2 - End
Operation 3 - Start
Operation 3 - End
```

- `Operation 2` waits for `Operation 1` to complete, and `Operation 3` waits for `Operation 2`.

### **When to Use Operations**

- Use operations when tasks require dependencies or monitoring.
- Encapsulate reusable tasks in custom `Operation` subclasses.
- Operations are ideal when you need to cancel tasks or track their progress and state.
- For finer control over execution threads and priorities.


# Modern Concurrency

### async/await


# Multithreading Pitfalls

Concurrency and multithreading have long been considered challenging problems in software development. Since their inception in computing, developers have worked to create paradigms and tools that simplify the complexities of managing concurrency. The difficulty arises from the fact that most programmers think procedurally, following a linear flow of execution. Designing code that runs asynchronously or at unpredictable times requires a different mindset, often leading to errors and unexpected behavior if not handled carefully.


## Deadlocks

A deadlock occurs when two different processes are waiting on the other to finish, effectively making it impossible for any of them to finish to begin with. This can happen when both processes are sharing a resource.

Thread A may try to access Resource C and Resource D, one after the other, and how Thread B may try to do the same but in different order. In this example, the deadlock will happen rather soon, because Thread A will get hold of Resource C while Thread B gets hold of Resource D. Whoever needs each other’s resource first will cause the deadlock .

<img src="https://github.com/user-attachments/assets/9fb68d66-d73c-435d-b357-887eb42c514a" width=50% style="display: block; margin: 0 auto"/>

### Solving the Deadlocking Problem

The deadlock problem has many established solutions. Mutex and Semaphores being the most used ones. There is also Inter-process communication through pipes.

### Mutex

Mutex is short for Mutually exclusive lock (or flag). A mutex will signal other processes that a process is currently using some resource by adding a lock to it and preventing other processes from grabbing until that lock is freed. Ideally, a process will acquire a lock to all the resources it will need at once, even before using them. This way, if Thread A needs Resource C and Resource D, it can lock them before Thread B tries to access them. Thread B will wait until all the locks are freed before attempting to access the resources itself.

<img src="https://github.com/user-attachments/assets/01b36ea0-384e-4b5e-9ad2-ee173fdcf1d3" width=50%/>

#### How a Mutex Works
A mutex ensures mutual exclusion by:
1. Blocking threads that attempt to acquire the lock if another thread already holds it.
2. Allowing only one thread to access the critical section at a time.
3. Releasing the lock after the critical section is executed, letting another thread acquire the lock.

Keep in mind that in this specific situation, it means that Thread A and Thread B, while multithreaded, will not run strictly at the same time, because Thread B needs the same resources as Thread A and it will wait for them to be free. For this reason, it is important to identify tasks that can be run in parallel before designing a multithreaded system.

#### Example Using `NSLock`

This example demonstrates how to use a lock to protect access to a shared resource, like a counter, in a multithreaded environment.

```swift
import Foundation

class Counter {
    private var value: Int = 0
    private let lock = NSLock() // A mutex lock
    
    func increment() {
        lock.lock() // Acquire the lock
        value += 1
        print("Counter incremented to \(value)")
        lock.unlock() // Release the lock
    }
    
    func getValue() -> Int {
        lock.lock() // Acquire the lock
        let currentValue = value
        lock.unlock() // Release the lock
        return currentValue
    }
}

// Example Usage
let counter = Counter()
let queue = DispatchQueue.global()

// Simulate multiple threads incrementing the counter
DispatchQueue.concurrentPerform(iterations: 10) { _ in
    counter.increment()
}

print("Final counter value: \(counter.getValue())")
```

##### Explanation:
1. **`NSLock`**:
   - The `lock.lock()` method acquires the lock, preventing other threads from entering the critical section.
   - The `lock.unlock()` method releases the lock, allowing other threads to access the critical section.

2. **Thread Safety**:
   - The lock ensures that only one thread at a time can modify the `value` property.
   - Without the lock, multiple threads could increment the counter simultaneously, leading to race conditions and incorrect results.

### Visualizing the Problem Without a Lock

To understand the importance of a lock, here's the same example **without using a mutex**:

```swift
class UnsafeCounter {
    private var value: Int = 0

    func increment() {
        value += 1
        print("Counter incremented to \(value)")
    }
    
    func getValue() -> Int {
        return value
    }
}

// Example Usage
let unsafeCounter = UnsafeCounter()
DispatchQueue.concurrentPerform(iterations: 10) { _ in
    unsafeCounter.increment()
}

print("Final counter value: \(unsafeCounter.getValue())")
```

##### Potential Issue:
- The `value` property is modified concurrently by multiple threads, leading to race conditions.
- The final counter value might not match the number of increments due to threads overwriting each other's changes.

### Semaphores

A Semaphore is a type of lock, very similar to a mutex . With this solution, a task will acquire a lock to a resource. Any other tasks that arrive and need that resource will see that the resource is busy. When the original thread frees the resources, it will signal interested parties that the resource is free, and they will follow the same process of locking and signaling as they interact with the resource.
Unlike a mutex, which allows only one thread at a time, a semaphore can allow multiple threads depending on the initial value of the semaphore. In Swift, semaphores are managed using the DispatchSemaphore class.

#### How Semaphores Work
- A semaphore with a value of **1** behaves like a mutex, allowing only one thread to access the resource.
- A semaphore with a value greater than **1** can allow multiple threads to access the resource simultaneously.


#### Example Using `DispatchSemaphore`

This example demonstrates how to use a semaphore to safely increment a shared counter.

```swift
import Foundation

class Counter {
    private var value: Int = 0
    private let semaphore = DispatchSemaphore(value: 1) // Semaphore with an initial value of 1
    
    func increment() {
        semaphore.wait() // Wait for access (decrease semaphore count)
        value += 1
        print("Counter incremented to \(value)")
        semaphore.signal() // Signal that access is released (increase semaphore count)
    }
    
    func getValue() -> Int {
        semaphore.wait() // Wait for access
        let currentValue = value
        semaphore.signal() // Release access
        return currentValue
    }
}

// Example Usage
let counter = Counter()
let queue = DispatchQueue.global()

// Simulate multiple threads incrementing the counter
DispatchQueue.concurrentPerform(iterations: 10) { _ in
    counter.increment()
}

print("Final counter value: \(counter.getValue())")
```

#### Explanation
1. **`DispatchSemaphore`**:
   - The semaphore is initialized with a value of `1`, meaning only one thread can access the critical section at a time.
   - The `wait()` method decreases the semaphore's count, blocking the thread if the count reaches zero (no resources available).
   - The `signal()` method increases the semaphore's count, allowing blocked threads to proceed.

2. **Thread Safety**:
   - The semaphore ensures that only one thread accesses the critical section (the `increment` or `getValue` methods) at any given time.
   - This prevents race conditions and ensures the counter is incremented safely.


