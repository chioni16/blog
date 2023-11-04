---
title: "Can you please take over now?"
date: 2023-11-02T12:50:55+05:30
---

I use [Go](https://go.dev/) at my current job and recently I had to dive into the innards of go scheduler to understand a problem we were facing. 
I learnt a lot during this time and decided to implement my own user space threads library that is inspired by golang, but with a generous number of simplifications.

## Theory
There are loads of well written articles[^1] that explain the internal workings of Go runtime. So, I will limit my focus on the components that I have incorporated in my own implementation. 

### Threads
[Threads](https://en.wikipedia.org/wiki/Thread_(computing)) represent a unit of work to be performed. They differ from [processes](https://en.wikipedia.org/wiki/Process_(computing)) in that threads that belong to the same process share memory, files and other resources. Modern operating systems implement threads and control the way they are run. Threads managed by operating system kernel are called kernel threads.

But, golang implements its own threads in the userspace, called [Goroutines](https://go.dev/tour/concurrency/1). Userspace threads come with a bunch of advantages, like:
- Faster creation, destruction and [context switching](https://en.wikipedia.org/wiki/Context_switch).
- Smaller memory footprint.
- Greater control over their implementation, which allows for specialised scheduling algorithm for the expected workload.

This means that large number of userspace threads is less of an issue compared to kernel threads, which helps implement a given idea in a more straight-forward fashion, without having to resort to tricks that make code less readable.
This also allows for specialised scheduling algorithms that can provide the best performance for the expected workload.

But there's no such thing as a free lunch. In order to avail the benefits mentioned above, we need to implement threads ourselves in the userspace, which is hardly simple. The userspace threads will have to work in conjunction with kernel threads, as at the end of the day it's the kernel threads that are run on processors and not the userspace threads directly. Go follows [M:P:N model](https://flylib.com/books/en/3.19.1.51/1/). Userspace threads have to maintain their own stacks, which is separate from the stack used by the kernel thread that they are paired with. Scheduler should also be able to save and restore thread contexts between thread switches.

My implementation schedules user space threads on a single kernel thread. This simplifies the implementation a lot as I don't have to worry about atomics and memory ordering as there is only one thing actually running at any point in time. Thread scheduling takes place [cooperatively](https://en.wikipedia.org/wiki/Cooperative_multitasking) where threads yield control to other fellow threads voluntarily. This is a huge simplification compared to the highly sophisticated go scheduler which offers features like [preemption](https://en.wikipedia.org/wiki/Preemption_(computing)), direct switch between threads, fairness etc. 

Although the above decisions help simplify our implementation, it also limits the kind of operations that can be run concurrently. A compute intensive task, like a busy loop, can hog all the resources, without ever letting a different task to execute.

### Channels
Golang uses channels for communication and synchronisation between goroutines. Channels, along with goroutines, are what make concurrency in Go fun to work with. Golang follows the principle 
```
    Do not communicate by sharing memory; instead, share memory by communicating
```
which is [made possible by channels](https://go.dev/blog/codelab-share). Channels have their origin in [Communicating Sequential Processes](https://dl.acm.org/doi/10.1145/359576.359585) by C.A.R Hoare.

Channels help with synchronisation[^2] at `send` or `receive` points when the channel is either full or empty respectively.

## Implementation

### Runtime

The runtime is a collection of threads that have started but haven't completed yet. 
```rust
pub struct Runtime {
    /// All active threads, i.e, which haven't completed.
    /// Can store threads that are not currently running,
    /// but are waiting to be chosen by the runtime or for some other event to occur.
    threads: Vec<Thread>,
    /// Id of thread that is currently running.
    current: Id,
    /// Shows the total number of threads created up until a certain point.
    /// Used to generate unique thread IDs for threads spawned by a runtime.
    count: usize,
}
```

The runtime is stored as a global variable to avoid having to pass it into every function that works with it. And because we are only working with one `Runtime` at any point in time, this shouldn't be an issue.
```rust
static mut RUNTIME: *mut Runtime = std::ptr::null_mut();
```

### Threads

Userspace threads are represented by the `Thread` struct. This stores all the information needed for the thread to do to its job successfully, even the information needed to restore its state between switches. 
```rust
pub struct Thread {
    /// Uniquely identifies a thread.
    pub id: Id,
    /// Stack used by the thread to run the function passed.
    pub stack: Box<[u8]>,
    /// Stores the thread context between successive runs.
    pub ctx: Context,
    /// Represents the current state of the thread.
    pub state: State,
    /// Stores the value sent by the channel, if any.
    pub chan_val: Option<usize>,
}
```

As mentioned above, a userspace thread has to manage its own stack. We represent our stack as a heap allocated slice of bytes. The size of the stack is fixed and cannot be changed, unlike [goroutine stacks](https://blog.cloudflare.com/how-stacks-are-handled-in-go/). 

We prepare the stack by writing the addresses of the function we want to run. This is done following the [x86_64 System V](https://en.wikipedia.org/wiki/X86_calling_conventions) calling convention. Along with the user function to be run, we also add a few functions that take care of cleanup once the user task is complete. 

```rust 
unsafe {
    let s_ptr = thread.stack.as_mut_ptr().add(thread.stack.len());
    let s_ptr = (s_ptr as usize & !15) as *mut u8;
    // add cleanup functions that are run when the user function returns
    std::ptr::write(s_ptr.offset(-16) as *mut usize, done as usize);
    // aligns stack to a 16 byte boundary
    std::ptr::write(s_ptr.offset(-24) as *mut usize, do_nothing as usize);
    // user function
    std::ptr::write(s_ptr.offset(-32) as *mut usize, f as usize);
    // bookkeeping
    thread.ctx.rsp = s_ptr.offset(-32) as u64;
}
```

Information needed for the thread to run correctly has to be restored between thread switches. We represent this information in the `Context` struct. At the moment, `Context` contains the [callee-saved registers](https://web.stanford.edu/class/archive/cs/cs107/cs107.1174/guide_x86-64.html). 

```rust
#[derive(Debug, Default)]
#[repr(C)]
pub struct Context {
    pub rsp: u64,
    pub r15: u64,
    pub r14: u64,
    pub r13: u64,
    pub r12: u64,
    pub rbx: u64,
    pub rbp: u64,
}
```

This information will have to be saved before yielding control to another thread and then restored once the control is gained back. 
I have used Rust's [naked functions](https://github.com/rust-lang/rust/issues/32408), as compiler doesn't implicitly add prologue and epilogue to the function. [Inline assembly](https://doc.rust-lang.org/reference/inline-assembly.html) is used to 
store current context in the `ctx` field of `Thread` struct representing the current task and then to update the registers with the context of the next task to be run. We follow x86_64 System V calling convention again here .
```rust
#[naked]
#[no_mangle]
unsafe extern "C" fn switch() {
    asm!(
        "mov [rdi + 0x00], rsp",
        "mov [rdi + 0x08], r15",
        "mov [rdi + 0x10], r14",
        "mov [rdi + 0x18], r13",
        "mov [rdi + 0x20], r12",
        "mov [rdi + 0x28], rbx",
        "mov [rdi + 0x30], rbp",
        "mov rsp, [rsi + 0x00]",
        "mov r15, [rsi + 0x08]",
        "mov r14, [rsi + 0x10]",
        "mov r13, [rsi + 0x18]",
        "mov r12, [rsi + 0x20]",
        "mov rbx, [rsi + 0x28]",
        "mov rbp, [rsi + 0x30]",
        "ret",
        options(noreturn)
    );
}
```

Every thread has a state associated with it. State determines the possible operations that can be performed on a given thread. For example, only the `Ready` threads are considered when choosing the next thread, to which the control is yielded and not those that are blocked on something.
```rust
pub enum State {
    /// Thread is making progress.
    Running,
    /// Thread is ready to be run and is not waiting on any external event.
    Ready,
    /// Thread is unable to send a value to a channel and is hence blocked until the channel frees up.
    ChannelBlockSend,
    /// Thread is waiting to receive a value from the channel.
    ChannelBlockRecv,
}
```

Threads call `yield_thread` method to give control to other threads. `yield_control` is either called explicitly by the user code or implicitly when a thread is blocked on sending or receiving from a channel. The method finds a suitable thread to run next and then changes the contents of the registers, while also saving the context of the current thread so that it can be restored later.

```rust
    fn yield_thread(&mut self) -> bool {
        // get the next thread to run.
        let cur_pos = self.cur_pos();
        let Some(next_pos) = self.round_robin(cur_pos) else {
            // return false when no other runnable thread is found.
            return false;
        };

        ...

        // store and restore the thread contexts and jump to the target thread.
        unsafe {
            let old: *mut Context = &mut self.threads[cur_pos].ctx;
            let new: *const Context = &self.threads[next_pos].ctx;

            #[cfg(target_os = "linux")]
            asm!("call switch", in("rdi") old, in("rsi") new, clobber_abi("C"));
            // symbols in macos need an underscore at the beginning.
            #[cfg(target_os = "macos")]
            asm!("call _switch", in("rdi") old, in("rsi") new, clobber_abi("C"));
        }
        
        ...
    }
```

### Channels

`Channel`s contain a `buffer` to store the values sent to the channel but that are not yet consumed. They also contain two queues to hold the threads that are blocked on sending and receiving values to the channel. Setting the size of the `buffer` to 0 will give an unbuffered `Channel`. 
```rust
pub struct Channel<T> {
    pub buffer: CircularBuffer<T>,
    pub sendq: CircularBuffer<(Id, T)>,
    pub recvq: CircularBuffer<Id>,
}
```

In Golang, goroutines can directly write values in another goroutine's stack. This helps with optimisations where the values are directly passed between goroutines instead of going through the channel buffer. This is possible as the go compiler has full knowledge of channels in the program and the goroutines that interact with the channels and hence can produce assembly instructions to make enough space in stacks for the incoming values from other goroutines.

But we don't have this luxury. Instead, we have the `chan_val` field in `Thread` struct to hold the value that was received from a `Channel`. A `Thread` can only be waiting on one channel at any point in time, so it's fine to have just one spot for it. 

When sending or receiving a value to/from a `Channel`, threads always check if there is another thread waiting to receive/send a value respectively. If so, the value is directly exchanged between the threads instead of going through the `Channel` buffer. Otherwise, the value is added to the `Channel` buffer. If the buffer is full/empty, then the thread is added to the corresponding `Channel` queues. Threads in the `Channel` queues are assigned suitable `State`s (`ChannelBlockSend` or `ChannelBlockRecv`) and are reverted back to `Ready` once they manage to send / receive values to the channel.

```rust
pub fn chan_recv<T: Debug>(chan: *mut Channel<T>) -> T {
    ...

    // if there's a sender blocked on sending, get its value
    if let Ok((sender, val)) = chan.sendq.read() {
        ...
    } else {
        // fetch value from channel buffer
        match chan.buffer.read() {
            Ok(val) => {
                return val;
            }
            // if no value present in the buffer, block
            Err(()) => {
                let curr_id = get_current_thread();
                // add the current thread to waiting list
                chan.recvq.write(curr_id).expect("No more space in recvq");
                
                ...
            }
        }
    }
}

pub fn chan_send<T: Debug>(chan: *mut Channel<T>, val: T) {
    ... 

    // if there's a thread waiting to receive a value, 
    // directly give the value to the waiting thread.
    // And change the state of the receiving thread to Ready
    if let Ok(receiver) = chan.recvq.read() {
        ...
    }
    // try adding the value to the channel buffer
    else if let Err(val) = chan.buffer.write(val) {
        // In case the buffer is full, add the sender to the waiting list
        let curr_id = get_current_thread();
        chan.sendq
            .write((curr_id, val))
            .expect("No more space in sendq");
        
        ...
    }
}
```

My implementation can be found [here](https://github.com/chioni16/uthreads).

## What's next?

The implementation, as it stands at the time of writing this post, is good enough as a proof of concept. However, there are a few features that I'd like to add, which can help the project leave the realm of PoC.

- Provide a [monadic](https://en.wikipedia.org/wiki/Monad_(functional_programming)) interface.
    - As we all know, working with Javascript Promises is a much more pleasant experience compared to using callbacks everywhere. I refuse to respect anyone who thinks otherwise /s.
- Implement priority-based scheduling.
    - Currently, a simple [round-robin algorithm](https://en.wikipedia.org/wiki/Round-robin_scheduling) is used to choose the next user thread that runs. I'd like to have a priority-based scheduling algorithm, where each task is assigned a priority when it is spawned.
- Support non-blocking I/O and timers.
    - Adding a simple TCP support would be a good start.
- Channels shouldn't be global variables.

[^1]: - https://go.dev/src/runtime/HACKING
      - https://ardanlabs.com/blog/2018/08/scheduling-in-go-part1.html
      - https://www.youtube.com/watch?v=-K11rY57K7k
      - https://www.youtube.com/watch?v=YHRO5WQGh0k
[^2]: - https://dmitryvorobev.blogspot.com/2016/08/golang-channels-implementation.html
      - https://medium.com/@ravikumar19997/exploring-the-depths-of-golang-channels-a-comprehensive-guide-53e1a97cafe6
      - https://www.youtube.com/watch?v=KBZlN0izeiY
