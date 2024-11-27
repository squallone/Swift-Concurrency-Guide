# Concurrency
Notes about concurrency

## What is concurrency?

Concurrency refers to the ability to perform multiple tasks simultaneously, enabling our app to remain responsive and efficient. It allows you to execute different parts of your code independently, making the most of the device's hardware resources, such as its multiple cores. Concurrency is essential for tasks that could block the main thread, such as network requests, file operations, or heavy computations.

### Parallelism vs. Concurrency

- Concurrency: Structuring a program to handle multiple tasks, which may or may not run at the same time.
- Parallelism: Executing multiple tasks literally at the same time, which depends on the system's hardware capabilities.

## Grand Central Dispatch

GCD is Apple’s implementation of C’s libdispatch library. Its purpose is to queue up tasks (either a method or a closure) that can be run in parallel, depending on availability of resources; it then executes the tasks on an available processor core.

All of the tasks that GCD manages for you are placed into GCD-managed first-in, **first-out** (FIFO) queues. Each task that you submit to a queue is then executed against a pool of threads fully managed by the system.


## Modern Concurrency

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


