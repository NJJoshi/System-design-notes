## 📌 Java Virtual Threads: Key Concepts & Takeaways

### 1. Carrier Threads

* **Definition:** Carrier threads are standard platform threads (OS-managed threads) that execute virtual threads under the hood.
* **Pool Management:** Managed by a dedicated internal `ForkJoinPool` whose size defaults to the number of available CPU cores/processors.
* **Role:** Acts as the execution carrier for virtual threads.

---

### 2. Virtual Threads

* **Scheduling:** Cannot be scheduled or executed directly by the OS scheduler; they are scheduled at the JDK level.
* **Mounting & Unmounting:**
* **Execution:** A virtual thread is **mounted** onto a carrier thread to run code.
* **In-Memory / CPU Tasks:** Keeps running continuously on the carrier thread until the computation finishes.
* **Blocking I/O:** When a virtual thread hits a blocking operation (e.g., database call, network request, sleep), it **unmounts** from the carrier thread, freeing up the carrier thread to serve other virtual threads.


* **Memory Management:** The execution context and stack frames of a virtual thread are stored as objects on the **JVM Heap** rather than using fixed native stack memory.

---

### 3. Key Operational Rules & Pitfalls

* **Best Suited For:** High-throughput, **I/O-bound** workloads (e.g., HTTP requests, database transactions).
* **Not Recommended For:** Heavy **CPU-bound** tasks (does not accelerate CPU-bound workloads as they require continuous computation on platform threads).
* **Thread Pinning:**
* Occurs when a virtual thread executes a synchronized block/method or native call during a blocking operation.
    For e.g.
    ```
    //synchronized method
    public synchronized void ioTask() {
        .....
    }

    //synchronized block
    public void ioTask() {
        ...
        (synchronized) {
            ...
            ...
        }
        ...
    }

    //JNI
    private native void doSomethingNative();

    ```
* **Impact:** Prevents the virtual thread from unmounting, blocking the underlying carrier thread.
* **Fix:** Replace traditional `synchronized` blocks with `ReentrantLock` where blocking I/O occurs.
Based on the slide currently displayed in the video, here is a concise summary of the key concept:

---

## 📌 Thread Pinning: Java 24+

### **What is Thread Pinning?**

* **Definition:** Pinning occurs when a virtual thread **must stay attached** to its underlying carrier thread and **cannot be unmounted** while executing specific code.
* **Impact:** Because the virtual thread cannot unmount, the JVM is prevented from switching the carrier thread to another virtual thread, which reduces overall application scalability.

---

### **When Does Pinning Occur?**

1. **Native Code Execution (JNI):** Occurs when executing native methods (e.g., `private native void someNativeMethod();`).
* *Note:* Executing JNI/native code is **not a common scenario** for most Java developers.


2. **Synchronized Blocks/Methods (Prior to Java 24):** In modern Java releases (Java 24+), execution inside standard `synchronized` blocks or methods no longer causes thread pinning during I/O operations (as indicated by the green checkmark on the code snippet).

---

### **Summary Takeaway**

In Java 24 and later, thread pinning primarily applies to **native code (JNI) execution**, removing one of the major caveats previously associated with using `synchronized` blocks alongside Virtual Threads.

### Virtual Thread
- Great for I/O Tasks to achieve "non blocking benefits behind scene".
- No need to use for CPU intensive Tasks.
- Thread Per Task
- Never Pool it

### ExecutorService
- Simple framework for high level concurrency
- We submit task and get result via Future object
- For Virtual Thread - we have Thread Per Task executor

### ExecutorService with Platform Thread
- Single/fixed/cached/scheduled/fork-join-pool
- These implementation pools thread
- With this implementation usage, don't use Virtual Thread Factory

### ExecutorService with Virtual Thread
- We can achieve single/fixed using Semaphore + Queue
- Cached pool -> more or less same as thread-per-task
- scheduled -> user platform thread to schedule and virtual thread to execute