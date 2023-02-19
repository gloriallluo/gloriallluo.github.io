---
layout: post
title: Notes - Rust Future
subtitle: 学习 async Rust 的笔记
tags: [Rust]
comments: true
---

# 1. Basics

## 1.1 Why futures?

对于并发任务，OS 一般采用 Preemptive Multitasking 的方式，即所有线程都由系统被动暂停/终止，控制权掌握在系统手里；而 Cooperative Multitasking，即任务主动让出控制权，则一般适用于细粒度的并行任务，由编程语言提供。Futures 是 Rust 为轻量级的并发任务提供了一种解决方案，其主要的使用场景是，当一个资源要求未满足时，放弃对线程的控制权，等到资源满足时再调度回去。相比于多线程的并发方式，在性能（没有 context switching）上和内存占用（线程上的所有任务可以共享一个栈）上都有更好的表现。

Futures 的竞争者们：
1. OS threads：栈较大，系统调用较多，线程调度由 OS 负责不太自由。
2. Greenthreads：相比于 OS threads 切换开销少了很多，但是每个线程一个栈的方式仍然导致扩展性不好。
3. Callbacks：为每个任务注册一个回调函数（本质上是一个函数指针），但是给编程带来很大的困难 -- 回调地狱！
4. Promises：JS 的 Promise 和 Rust 的 Future 差不多，除了一个是 eagerly evaluated 另一个是 lazily evaluated，此外它的编程过程中 `and_then` chaining 给跨 step 的所有权转移带来一定的困难（但是 async/await 则可以用同步的风格写代码）。

所以说，对于数量较多的轻量级任务，Future 机制是很适合的。

## 1.2 How does a future work?

Future 之间的关系可以被理解成一棵树，执行顺序则类似树的遍历。Futures 主要分为两种，一种是 leaf futures，完全依赖外部资源，例如一个 socket、或者计时器事件，一种是 non-leaf futures，它们 await 许多子 future 的完成。在很多调度器实现中，Future 树是调度的基本单位。

### Roles

- `Executor`: more like a scheduler which manages `Future`s in queues and polls them.
  In some libs, top-level Futures are called `Task`s, which are placed in READY/WAITING queue.
- `Reactor`: keeps an eye on the leaf futures and wakes the `Waker` which notifies the `Executor` to do some scheduling.
  `Waker`: The `Waker` struct is held by resources and is used to signal that a task is *runnable* and be pushed into the scheduler's run queue.
- `Future`: state machine advanced by `poll`s.
  The **state machine** must keep track of the current state internally, i.e. the required variables. An `enum` variant is generated by the compiler, each field of which represents a state with variables attached. In the `poll` function, the state of `self` is matched. And in each match arm, its child future is polled, and if a `Ready` is returned, pick the variables for the next state and then assign `self` with a new `enum` field.


### 一个 Future 执行的 timeline

```rust
let non_leaf_fut = async {
    // A
    let leaf_fut = Reactor::new_io();
    leaf_fut.await
    // B
};
runtime.block_on(non_leaf_fut)
```

以上是一个 non-leaf Future 调用一个 leaf Future 的例子，时间顺序上发生的事情大概是这样：

| User / External incidents | Runtime |
| --- | --- |
| Create a Future which is compiled to a state machine. |  |
| Call block_on to this Future. | The Future is passed to the Executor and a Waker is created whose reference is passes to the Reactor later. |
|  | The Executor polls this Waker, and the code block A will be ran. |
| await the leaf future. | The Executor polls the leaf future’s Waker and gets a Poll::Pending, the Executor then puts the non_leaf_future in the WAITING queue and picks another task (i.e. top-level future) from the READY queue. |
| The I/O event is ready. | The Reactor is notified that an event is ready and then calls Waker::wake() which puts the future in the READY queue and notifies the Executor. |
|  | The Executor picks the Future from the READY queue and then polls it, this time a Poll::Ready(output) is returned. The state machine advances one more step, code block B in ran. |

{: .box-note}
**💡 Why not let the `Executor` to do the waking job?**\\
By having a wake up mechanism that is *not* tied to the thing that executes the future, runtime-implementors can come up with interesting new wake-up mechanisms.\\
An example of this can be spawning a thread to do some work that eventually notifies the future, completely independent of the current runtime.\\
Without a waker, the executor would be the *only* way to notify a running task, whereas with the waker, we get a loose coupling where it's easy to extend the ecosystem with new leaf-level tasks.

执行器是用来执行 task 的，每个状态机则事实上是由子 future 驱动的，归根结底整个状态机是由叶子 future 驱动的。

当一个叶子 future 完成时，我们如何通知执行器，进而让这个叶子 future 的顶层 task 被 wake up 呢？用户态程序得知一个外部事件被完成这件事情一般由外部驱动的实现提供。通过将 Executor 和 Reactor 解耦合，在“从用户态程序得到外部事件，到执行器去 poll 对应的 task”这一过程中，运行时库被给予了更大的自由。而 Reactor 通知 Executor 这一过程，则是通过 Waker 来实现的。

## 1.3 Implementation details

### Pinning

Future 需要被标记为 `Pin`，这本质上因为是它是因为一个自引用结构体，不可以暴露 `&mut self` 以至于被 move。Future 本质上是一个枚举结构，每种枚举类型是一个状态，其中存储了当前有的变量。有的时候这些变量是另一些变量的引用，即 Future 中一些 field 指向另一些 field，这就导致 Future 变成了一个自引用结构体 (self-referential struct)。


### Waker and Context

对于什么时候被 poll 然后状态机推进这件事，non-leaf futures 是由它的子 future 驱动的，而 leaf future 则需要借助 `Waker` 的帮助来通知 `Executor` 进行调度。

在 `Future` trait 的 `poll` 方法中，有一个参数是一个 `Context`，它可以理解为一个 `Waker` 的 wrapper，为了以后的迭代而设计，可以放一些 task-local storage 或者 debugging hooks 等等。

调用 `Waker::wake()` 涉及运行时多态，一个 `&dyn Trait` 本质上就是一个胖指针，前 8 个字节指向 trait object，后 8 个字节指向一个 vtable，这个 vtable 属于一个 Trait implementation，存了一些 `Drop` 函数的指针、trait function 的指针、数据大小等等。


## Reference

- [Futures Explained in 200 Lines of Rust](https://cfsamson.github.io/books-futures-explained/)
- [Async/Await](https://os.phil-opp.com/async-await/)

# 2. More Facts About Async

## 2.1 The designs of `async_trait`

复读这篇文章：
[Baby Steps - why async fn in traits are hard](http://smallcultfollowing.com/babysteps/blog/2019/10/26/async-fn-in-traits-are-hard/)


### About `async_trait`

[GitHub - dtolnay/async-trait: Type erasure for async trait methods](https://github.com/dtolnay/async-trait)

一个过程宏，由于奇怪的原因，trait 中的方法不可以是 async 的，因此 async trait 实际将 trait 中的 async fn 编译为普通的函数，其返回值是 `Pin<Box<'_ + dyn Future<...> + Send>>` （不一定是 `Send`，可以把它关掉）。


### Why not `impl Future`?

这个答案也可以解答“为什么 trait 中不可以有 async fn”，因为 async fn 本质上就是一个函数体为 async block、返回值为 `impl Future<...>` 的函数。

{: .box-note}
**💡 impl Trait v.s. dyn Trait**\\
Rust 需要某种机制来确定某个 Trait 具体是哪个实现。impl 关键字就是要求在编译的时候确定实现，称为 static dispatching，其实质是在编译的时候消除泛型，变成确定类型的代码；而 dyn 则是在运行时确定实现，类似于 C++ 通过 vtable 来解决，也会带来一些运行时的性能开销。

在 trait fn 的返回值不可以是 `impl Trait`，因为 Rust 要求函数的返回值是已知大小。对于普通的函数，如果返回值是 `impl Future`，会要求编译期间知道这个 Future 实现具体是什么，普通的函数自然可以完成这一点。对于 trait 中的函数，这一点只能在实例化这个 trait 时才能得知，一个返回 `impl Xxx` 的 trait fn 会被 Rust 编译器新建一个匿名 generic associated type（我的理解：带生命周期参数的关联类型），并且这个函数的返回值也会被修改为这个关联类型。

{: .box-note}
**💡 为什么要建这个匿名关联类型？**\\
https://github.com/rust-lang/chalk

在我们使用自己编写的 async trait 的时候，我们需要将语法改成诸如 `MyAsyncTrait<AnonymousGAT = Xxx>` 的形式，而对于一个 future，我们除了知道它实现了 `Future` trait，往往没有具体的类型名称，这给编程带来极大的不方便。


## 2.2 The designs of `futures`

From Aron Turon’s blogs:
- [Zero-cost futures in Rust](https://aturon.github.io/blog/2016/08/11/futures/)
- [Designing futures for Rust](https://aturon.github.io/blog/2016/09/07/futures-design/)


### The goals of futures crate

Whats futures solve: tracking a lot of asynchronous tasks and dispatching those to the right callbacks. (When an async event arrives, only one dynamic dispatch is required.)

**Aims:**
- Robust: error handling, thread-safety, etc.
- Ergonimic: making writing async codes as easy as synchronous code → leverage Rust’s traits and closures.
- Zero-cost: equivalent or better than a hand-driven state machine. **demand-driven** rather than callback-oriented, that is, composing the futures together without creating intermediate callbacks.

**Asynchrony**: you get a *future* right away, without blocking, even though the *value* the future represents will become ready only at some unknown time in the… future.

{: .box-note}
**💡 Rust’s traits — without heap allocation and dynamic dispatch**\\
Rust trait 是一种编译时多态。当我们调用一个 trait 函数时，实际上确定实现 trait 的具体类型是在编译期间完成的，例如为 `i32` 和 `i64` 实现了 `hash()` 函数，编译器会实现 `_hash_i32()` 和 `_hash_i64()` 两个函数，编译期间将对应的 `hash()` 调用替换为 `_hash_i32()` 或者 `_hash_i64()`。\\
所以确定到底是哪种 `impl Future` 并且去调用对应的 `poll` 实际上是在编译期间做的。


### How to achieve zero-cost?

此处考察了几种实现思路的权衡：
- State machines or callbacks?
- Static dispatch or dynamic dispatch?

Rust 均选择了前者，下面的内容将解释前者为什么是相比于后者性能更高的选择。

#### State machines V.S. callbacks:

分别地对应了两种思路：demand-driven V.S. completion-based，前者被 poll 推动状态机，后者则一般是注册完成时的回调。

**Q1. What does a completion-based future look like?**

一个类似于回调的 Future trait 大概提供了一个注册回调的接口，即将一个 FnOnce 作为参数。缺点在于需要反复地分配内存以及动态分发。

在 join 的场景下，用户将两个 future 交给 join 并且自己实现了回调，join 也返回一个 future，现在 join 的实现者需要实现这个回调，这个回调需要可以被两个 future 的任意一个回调触发（因为不知道两个 future 谁先完成）。在不同的回调之间涉及大量的数据共享，一般用 Arc，这就涉及了大量的堆内存申请。            

此外，当一个被等待的资源（heterogeneous collections）到来时，需要将它分配到一个回调函数上，此处大概需要动态分发。（我的理解是，callback 的实现一般是有一个“全局”事件 → callback 的表。）
            
**Q2. What does a demand-driven future look like?**

- 不需要反复分配内存：不用在不同的 FnOnce 之间共享数据，状态机代码可以自然地分享变量，且状态机的大小在编译期间就确定了。
- 较少的动态分发：自然地由子 future 去驱动，动态分发只发生在 `Waker::wake()` 的过程中。
  子 future 被 poll，返回 `Poll::Ready`，回到父 future 的状态机中，继续进行下一个状态。
- 一个副作用是需要一个外部力量去 poll 这些 future (Executor)，进而又需要一个外部力量去通知 Executor (Reactor)。

#### Static dispatch V.S. dynamic dispatch:

- Static dispatch: 在编译期间确定具体的 Future 类型，没有运行时开销。
- Dynamic dispatch: 一般的用途是 heterogeneous collections。
        
> You need to group together a number of objects which may have different underlying types but all implement the same trait. Trait objects must always be behind a pointer, which in practice usually requires heap allocation.


### Implementation details

- Futures that execute the futures:
  - “wait”：阻塞当前线程，future 被分配在栈上。
  - “spawn”：从线程池抓一个线程去执行，future 被分配在堆上。
- In a way, the task model is an instance of “green” threading: we schedule a potentially large number of asynchronous tasks onto a much smaller number of real OS threads, and most of those tasks are blocked on some event most of the time.

### A little summary

State machines → No heap allocation, no dynamic dispatching → Zero-cost!