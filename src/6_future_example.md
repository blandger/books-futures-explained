# Implementing Futures - main example

We'll create our own Futures together with a fake reactor and a simple
executor which allows you to edit, run an play around with the code right here
in your browser.

I'll walk you through the example, but if you want to check it out closer, you
can always [clone the repository][example_repo] and play around with the code
yourself or just copy it from the next chapter.

There are several branches explained in the readme, but two are
relevant for this chapter. The `main` branch is the example we go through here,
and the `basic_example_commented` branch is this example with extensive
comments.

> If you want to follow along as we go through, initialize a new cargo project
> by creating a new folder and run `cargo init` inside it. Everything we write
> here will be in `main.rs`

## Implementing our own Futures

Let's start off by getting all our imports right away so you can follow along

```rust, noplaypen, ignore
use std::{
    future::Future, pin::Pin, sync::{ mpsc::{channel, Sender}, Arc, Mutex,},
    task::{Context, Poll, RawWaker, RawWakerVTable, Waker}, mem,
    thread::{self, JoinHandle}, time::{Duration, Instant}, collections::HashMap
};
```

## The Executor

The executors responsibility is to take one or more futures and run them to completion.

The first thing an `executor` does when it gets a `Future` is polling it.

**When polled one of three things can happen:**

- The future returns `Ready` and we schedule whatever chained operations to run
- The future hasn't been polled before so we pass it a `Waker` and suspend it
- The futures has been polled before but is not ready and returns `Pending`

Rust provides a way for the Reactor and Executor to communicate through the `Waker`. The reactor stores this `Waker` and calls `Waker::wake()` on it once
a `Future` has resolved and should be polled again.

> Notice that this chapter has a bonus section called [A  Proper Way to Park our Thread](./6_future_example.md#bonus-section---a-proper-way-to-park-our-thread) which shows how to avoid `thread::park`.

**Our Executor will look like this:**

```rust, noplaypen, ignore
// Our executor takes any object which implements the `Future` trait
fn block_on<F: Future>(mut future: F) -> F::Output {

    // the first thing we do is to construct a `Waker` which we'll pass on to
    // the `reactor` so it can wake us up when an event is ready.
    let mywaker = Arc::new(MyWaker{ thread: thread::current() });
    let waker = waker_into_waker(Arc::into_raw(mywaker));

    // The context struct is just a wrapper for a `Waker` object. Maybe in the
    // future this will do more, but right now it's just a wrapper.
    let mut cx = Context::from_waker(&waker);

    // So, since we run this on one thread and run one future to completion
    // we can pin the `Future` to the stack. This is unsafe, but saves an
    // allocation. We could `Box::pin` it too if we wanted. This is however
    // safe since we shadow `future` so it can't be accessed again and will
    // not move until it's dropped.
    let mut future = unsafe { Pin::new_unchecked(&mut future) };

    // We poll in a loop, but it's not a busy loop. It will only run when
    // an event occurs, or a thread has a "spurious wakeup" (an unexpected wakeup
    // that can happen for no good reason).
    let val = loop {
        match Future::poll(future.as_mut(), &mut cx) {

            // when the Future is ready we're finished
            Poll::Ready(val) => break val,

            // If we get a `pending` future we just go to sleep...
            Poll::Pending => thread::park(),
        };
    };
    val
}
```

In all the examples you'll see in this chapter I've chosen to comment the code
extensively. I find it easier to follow along that way so I'll not repeat myself
here and focus only on some important aspects that might need further explanation.

It's worth noting that simply calling `thread::sleep` as we do here can lead to
both deadlocks and errors. We'll explain a bit more later and fix this if you
read all the way to the [Bonus Section](./6_future_example.md##bonus-section---a-proper-way-to-park-our-thread) at
the end of this chapter.

For now, we keep it as simple and easy to understand as we can by just going
to sleep.

Now that you've read so much about `Generator`s and `Pin` already this should
be rather easy to understand. `Future` is a state machine, every `await` point
is a `yield` point. We could borrow data across `await` points and we meet the
exact same challenges as we do when borrowing across `yield` points.

> `Context` is just a wrapper around the `Waker`. At the time of writing this
book it's nothing more. In the future it might be possible that the `Context`
object will do more than just wrapping a `Future` so having this extra
abstraction gives some flexibility.

As explained in the [chapter about generators](./3_generators_pin.md), we use
`Pin` and the guarantees that give us to allow `Future`s to have self
references.

## The `Future` implementation

Futures has a well defined interface, which means they can be used across the
entire ecosystem.

We can chain these `Future`s so that once a **leaf-future** is
ready we'll perform a set of operations until either the task is finished or we
reach yet another **leaf-future** which we'll wait for and yield control to the
scheduler.

**Our Future implementation looks like this:**

```rust, noplaypen, ignore
// This is the definition of our `Waker`. We use a regular thread-handle here.
// It works but it's not a good solution. It's easy to fix though, I'll explain
// after this code snippet.
#[derive(Clone)]
struct MyWaker {
    thread: thread::Thread,
}

// This is the definition of our `Future`. It keeps all the information we
// need. This one holds a reference to our `reactor`, that's just to make
// this example as easy as possible. It doesn't need to hold a reference to
// the whole reactor, but it needs to be able to register itself with the
// reactor.
#[derive(Clone)]
pub struct Task {
    id: usize,
    reactor: Arc<Mutex<Box<Reactor>>>,
    data: u64,
}

// These are function definitions we'll use for our waker. Remember the
// "Trait Objects" chapter earlier.
fn mywaker_wake(s: &MyWaker) {
    let waker_ptr: *const MyWaker = s;
    let waker_arc = unsafe {Arc::from_raw(waker_ptr)};
    waker_arc.thread.unpark();
}

// Since we use an `Arc` cloning is just increasing the refcount on the smart
// pointer.
fn mywaker_clone(s: &MyWaker) -> RawWaker {
    let arc = unsafe { Arc::from_raw(s) };
    std::mem::forget(arc.clone()); // increase ref count
    RawWaker::new(Arc::into_raw(arc) as *const (), &VTABLE)
}

// This is actually a "helper funtcion" to create a `Waker` vtable. In contrast
// to when we created a `Trait Object` from scratch we don't need to concern
// ourselves with the actual layout of the `vtable` and only provide a fixed
// set of functions
const VTABLE: RawWakerVTable = unsafe {
    RawWakerVTable::new(
        |s| mywaker_clone(&*(s as *const MyWaker)),   // clone
        |s| mywaker_wake(&*(s as *const MyWaker)),    // wake
        |s| (*(s as *const MyWaker)).parker.unpark(), // wake by ref (don't decrease refcount)
        |s| drop(Arc::from_raw(s as *const MyWaker)), // decrease refcount
    )
};

// Instead of implementing this on the `MyWaker` object in `impl Mywaker...` we
// just use this pattern instead since it saves us some lines of code.
fn waker_into_waker(s: *const MyWaker) -> Waker {
    let raw_waker = RawWaker::new(s as *const (), &VTABLE);
    unsafe { Waker::from_raw(raw_waker) }
}

impl Task {
    fn new(reactor: Arc<Mutex<Box<Reactor>>>, data: u64, id: usize) -> Self {
        Task { id, reactor, data }
    }
}

// This is our `Future` implementation
impl Future for Task {
    type Output = usize;

    // Poll is the what drives the state machine forward and it's the only
    // method we'll need to call to drive futures to completion.
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {

        // We need to get access the reactor in our `poll` method so we acquire
        // a lock on that.
        let mut r = self.reactor.lock().unwrap();

        // First we check if the task is marked as ready
        if r.is_ready(self.id) {

            // If it's ready we set its state to `Finished`
            *r.tasks.get_mut(&self.id).unwrap() = TaskState::Finished;
            Poll::Ready(self.id)

        // If it isn't finished we check the map we have stored in our Reactor
        // over id's we have registered and see if it's there
        } else if r.tasks.contains_key(&self.id) {

            // This is important. The docs says that on multiple calls to poll,
            // only the Waker from the Context passed to the most recent call
            // should be scheduled to receive a wakeup. That's why we insert
            // this waker into the map (which will return the old one which will
            // get dropped) before we return `Pending`.
            r.tasks.insert(self.id, TaskState::NotReady(cx.waker().clone()));
            Poll::Pending
        } else {

            // If it's not ready, and not in the map it's a new task so we
            // register that with the Reactor and return `Pending`
            r.register(self.data, cx.waker().clone(), self.id);
            Poll::Pending
        }

        // Note that we're holding a lock on the `Mutex` which protects the
        // Reactor all the way until the end of this scope. This means that
        // even if our task were to complete immidiately, it will not be
        // able to call `wake` while we're in our `Poll` method.

        // Since we can make this guarantee, it's now the Executors job to
        // handle this possible race condition where `Wake` is called after
        // `poll` but before our thread goes to sleep.
    }
}
```

This is mostly pretty straight forward. The confusing part is the strange way
we need to construct the `Waker`, but since we've already created our own
_trait objects_ from raw parts, this looks pretty familiar. Actually, it's
even a bit easier.

We use an `Arc` here to pass out a ref-counted borrow of our `MyWaker`. This
is pretty normal, and makes this easy and safe to work with. Cloning a `Waker`
is just increasing the refcount in this case.

Dropping a `Waker` is as easy as decreasing the refcount. Now, in special
cases we could choose to not use an `Arc`. So this low-level method is there
to allow such cases.

Indeed, if we only used `Arc` there is no reason for us to go through all the
trouble of creating our own `vtable` and a `RawWaker`. We could just implement
a normal trait.

Fortunately, in the future this will probably be possible in the standard
library as well. For now, [this trait lives in the nursery][arc_wake], but my
guess is that this will be a part of the standard library after some maturing.

We choose to pass in a reference to the whole `Reactor` here. This isn't normal.
The reactor will often be a global resource which let's us register interests
without passing around a reference.

> ### Why using thread park/unpark is a bad idea for a library
>
> It could deadlock easily since anyone could get a handle to the `executor thread`
> and call park/unpark on our thread. I've made [an example with comments on the
> playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=b2343661fe3d271c91c6977ab8e681d0) that showcases how such an error could occur. You can also read a bit more about this in [issue 2010](https://github.com/rust-lang/futures-rs/pull/2010)
> in the futures crate.

## The Reactor

This is the home stretch, and not strictly `Future` related, but we need one
to have an example to run.

Since concurrency mostly makes sense when interacting with the outside world (or
at least some peripheral), we need something to actually abstract over this
interaction in an asynchronous way.

This is the Reactors job. Most often you'll see reactors in Rust use a library
called [Mio][mio], which provides non blocking APIs and event notification for
several platforms.

The reactor will typically give you something like a `TcpStream` (or any other
resource) which you'll use to create an I/O request. What you get in return is a
`Future`.

>If our reactor did some real I/O work our `Task` in would instead be represent
>a non-blocking `TcpStream` which registers interest with the global `Reactor`.
>Passing around a reference to the Reactor itself is pretty uncommon but I find
>it makes reasoning about what's happening easier.

Our example task is a timer that only spawns a thread and puts it to sleep for
the number of seconds we specify. The reactor we create here will create a
**leaf-future** representing each timer. In return the Reactor receives a waker
which it will call once the task is finished.

To be able to run the code here in the browser there is not much real I/O we
can do so just pretend that this is actually represents some useful I/O operation
for the sake of this example.


**Our Reactor will look like this:**

```rust, noplaypen, ignore
// The different states a task can have in this Reactor
enum TaskState {
    Ready,
    NotReady(Waker),
    Finished,
}

// This is a "fake" reactor. It does no real I/O, but that also makes our
// code possible to run in the book and in the playground
struct Reactor {

    // we need some way of registering a Task with the reactor. Normally this
    // would be an "interest" in an I/O event
    dispatcher: Sender<Event>,
    handle: Option<JoinHandle<()>>,

    // This is a list of tasks
    tasks: HashMap<usize, TaskState>,
}

// This represents the Events we can send to our reactor thread. In this
// example it's only a Timeout or a Close event.
#[derive(Debug)]
enum Event {
    Close,
    Timeout(u64, usize),
}

impl Reactor {

    // We choose to return an atomic reference counted, mutex protected, heap
    // allocated `Reactor`. Just to make it easy to explain... No, the reason
    // we do this is:
    //
    // 1. We know that only thread-safe reactors will be created.
    // 2. By heap allocating it we can obtain a reference to a stable address
    // that's not dependent on the stack frame of the function that called `new`
    fn new() -> Arc<Mutex<Box<Self>>> {
        let (tx, rx) = channel::<Event>();
        let reactor = Arc::new(Mutex::new(Box::new(Reactor {
            dispatcher: tx,
            handle: None,
            tasks: HashMap::new(),
        })));

        // Notice that we'll need to use `weak` reference here. If we don't,
        // our `Reactor` will not get `dropped` when our main thread is finished
        // since we're holding internal references to it.

        // Since we're collecting all `JoinHandles` from the threads we spawn
        // and make sure to join them we know that `Reactor` will be alive
        // longer than any reference held by the threads we spawn here.
        let reactor_clone = Arc::downgrade(&reactor);

        // This will be our Reactor-thread. The Reactor-thread will in our case
        // just spawn new threads which will serve as timers for us.
        let handle = thread::spawn(move || {
            let mut handles = vec![];

            // This simulates some I/O resource
            for event in rx {
                println!("REACTOR: {:?}", event);
                let reactor = reactor_clone.clone();
                match event {
                    Event::Close => break,
                    Event::Timeout(duration, id) => {

                        // We spawn a new thread that will serve as a timer
                        // and will call `wake` on the correct `Waker` once
                        // it's done.
                        let event_handle = thread::spawn(move || {
                            thread::sleep(Duration::from_secs(duration));
                            let reactor = reactor.upgrade().unwrap();
                            reactor.lock().map(|mut r| r.wake(id)).unwrap();
                        });
                        handles.push(event_handle);
                    }
                }
            }

            // This is important for us since we need to know that these
            // threads don't live longer than our Reactor-thread. Our
            // Reactor-thread will be joined when `Reactor` gets dropped.
            handles.into_iter().for_each(|handle| handle.join().unwrap());
        });
        reactor.lock().map(|mut r| r.handle = Some(handle)).unwrap();
        reactor
    }

    // The wake function will call wake on the waker for the task with the
    // corresponding id.
    fn wake(&mut self, id: usize) {
        self.tasks.get_mut(&id).map(|state| {

            // No matter what state the task was in we can safely set it
            // to ready at this point. This lets us get ownership over the
            // the data that was there before we replaced it.
            match mem::replace(state, TaskState::Ready) {
                TaskState::NotReady(waker) => waker.wake(),
                TaskState::Finished => panic!("Called 'wake' twice on task: {}", id),
                _ => unreachable!()
            }
        }).unwrap();
    }

    // Register a new task with the reactor. In this particular example
    // we panic if a task with the same id get's registered twice
    fn register(&mut self, duration: u64, waker: Waker, id: usize) {
        if self.tasks.insert(id, TaskState::NotReady(waker)).is_some() {
            panic!("Tried to insert a task with id: '{}', twice!", id);
        }
        self.dispatcher.send(Event::Timeout(duration, id)).unwrap();
    }

    // We simply checks if a task with this id is in the state `TaskState::Ready`
    fn is_ready(&self, id: usize) -> bool {
        self.tasks.get(&id).map(|state| match state {
            TaskState::Ready => true,
            _ => false,
        }).unwrap_or(false)
    }
}

impl Drop for Reactor {
    fn drop(&mut self) {
        // We send a close event to the reactor so it closes down our reactor-thread.
        // If we don't do that we'll end up waiting forever for new events.
        self.dispatcher.send(Event::Close).unwrap();
        self.handle.take().map(|h| h.join().unwrap()).unwrap();
    }
}
```

It's a lot of code though, but essentially we just spawn off a new thread
and make it sleep for some time which we specify when we create a `Task`.

Now, let's test our code and see if it works. Since we're sleeping for a couple
of seconds here, just give it some time to run.

In the last chapter we have the [whole 200 lines in an editable window](./8_finished_example.md)
which you can edit and change the way you like.

```rust, edition2018
# use std::{
#     future::Future, pin::Pin, sync::{ mpsc::{channel, Sender}, Arc, Mutex,},
#     task::{Context, Poll, RawWaker, RawWakerVTable, Waker}, mem,
#     thread::{self, JoinHandle}, time::{Duration, Instant}, collections::HashMap
# };
#
fn main() {
    // This is just to make it easier for us to see when our Future was resolved
    let start = Instant::now();

    // Many runtimes create a global `reactor` we pass it as an argument
    let reactor = Reactor::new();

    // We create two tasks:
    // - first parameter is the `reactor`
    // - the second is a timeout in seconds
    // - the third is an `id` to identify the task
    let future1 = Task::new(reactor.clone(), 1, 1);
    let future2 = Task::new(reactor.clone(), 2, 2);

    // an `async` block works the same way as an `async fn` in that it compiles
    // our code into a state machine, `yielding` at every `await` point.
    let fut1 = async {
        let val = future1.await;
        println!("Got {} at time: {:.2}.", val, start.elapsed().as_secs_f32());
    };

    let fut2 = async {
        let val = future2.await;
        println!("Got {} at time: {:.2}.", val, start.elapsed().as_secs_f32());
    };

    // Our executor can only run one and one future, this is pretty normal
    // though. You have a set of operations containing many futures that
    // ends up as a single future that drives them all to completion.
    let mainfut = async {
        fut1.await;
        fut2.await;
    };

    // This executor will block the main thread until the futures are resolved
    block_on(mainfut);
}
# // ============================= EXECUTOR ====================================
# fn block_on<F: Future>(mut future: F) -> F::Output {
#     let mywaker = Arc::new(MyWaker {
#         thread: thread::current(),
#     });
#     let waker = waker_into_waker(Arc::into_raw(mywaker));
#     let mut cx = Context::from_waker(&waker);
#
#     // SAFETY: we shadow `future` so it can't be accessed again.
#     let mut future = unsafe { Pin::new_unchecked(&mut future) };
#     let val = loop {
#         match Future::poll(future.as_mut(), &mut cx) {
#             Poll::Ready(val) => break val,
#             Poll::Pending => thread::park(),
#         };
#     };
#     val
# }
#
# // ====================== FUTURE IMPLEMENTATION ==============================
# #[derive(Clone)]
# struct MyWaker {
#     thread: thread::Thread,
# }
#
# #[derive(Clone)]
# pub struct Task {
#     id: usize,
#     reactor: Arc<Mutex<Box<Reactor>>>,
#     data: u64,
# }
#
# fn mywaker_wake(s: &MyWaker) {
#     let waker_ptr: *const MyWaker = s;
#     let waker_arc = unsafe { Arc::from_raw(waker_ptr) };
#     waker_arc.thread.unpark();
# }
#
# fn mywaker_clone(s: &MyWaker) -> RawWaker {
#     let arc = unsafe { Arc::from_raw(s) };
#     std::mem::forget(arc.clone()); // increase ref count
#     RawWaker::new(Arc::into_raw(arc) as *const (), &VTABLE)
# }
#
# const VTABLE: RawWakerVTable = unsafe {
#     RawWakerVTable::new(
#         |s| mywaker_clone(&*(s as *const MyWaker)),   // clone
#         |s| mywaker_wake(&*(s as *const MyWaker)),    // wake
#         |s| (*(s as *const MyWaker)).thread.unpark(), // wake by ref (don't decrease refcount)
#         |s| drop(Arc::from_raw(s as *const MyWaker)), // decrease refcount
#     )
# };
#
# fn waker_into_waker(s: *const MyWaker) -> Waker {
#     let raw_waker = RawWaker::new(s as *const (), &VTABLE);
#     unsafe { Waker::from_raw(raw_waker) }
# }
#
# impl Task {
#     fn new(reactor: Arc<Mutex<Box<Reactor>>>, data: u64, id: usize) -> Self {
#         Task { id, reactor, data }
#     }
# }
#
# impl Future for Task {
#     type Output = usize;
#     fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
#         let mut r = self.reactor.lock().unwrap();
#         if r.is_ready(self.id) {
#             println!("POLL: TASK {} IS READY", self.id);
#             *r.tasks.get_mut(&self.id).unwrap() = TaskState::Finished;
#             Poll::Ready(self.id)
#         } else if r.tasks.contains_key(&self.id) {
#             println!("POLL: REPLACED WAKER FOR TASK: {}", self.id);
#             r.tasks.insert(self.id, TaskState::NotReady(cx.waker().clone()));
#             Poll::Pending
#         } else {
#             println!("POLL: REGISTERED TASK: {}, WAKER: {:?}", self.id, cx.waker());
#             r.register(self.data, cx.waker().clone(), self.id);
#             Poll::Pending
#         }
#     }
# }
#
# // =============================== REACTOR ===================================
# enum TaskState {
#     Ready,
#     NotReady(Waker),
#     Finished,
# }
# struct Reactor {
#     dispatcher: Sender<Event>,
#     handle: Option<JoinHandle<()>>,
#     tasks: HashMap<usize, TaskState>,
# }
#
# #[derive(Debug)]
# enum Event {
#     Close,
#     Timeout(u64, usize),
# }
#
# impl Reactor {
#     fn new() -> Arc<Mutex<Box<Self>>> {
#         let (tx, rx) = channel::<Event>();
#         let reactor = Arc::new(Mutex::new(Box::new(Reactor {
#             dispatcher: tx,
#             handle: None,
#             tasks: HashMap::new(),
#         })));
#
#         let reactor_clone = Arc::downgrade(&reactor);
#         let handle = thread::spawn(move || {
#             let mut handles = vec![];
#             // This simulates some I/O resource
#             for event in rx {
#                 println!("REACTOR: {:?}", event);
#                 let reactor = reactor_clone.clone();
#                 match event {
#                     Event::Close => break,
#                     Event::Timeout(duration, id) => {
#                         let event_handle = thread::spawn(move || {
#                             thread::sleep(Duration::from_secs(duration));
#                             let reactor = reactor.upgrade().unwrap();
#                             reactor.lock().map(|mut r| r.wake(id)).unwrap();
#                         });
#                         handles.push(event_handle);
#                     }
#                 }
#             }
#             handles.into_iter().for_each(|handle| handle.join().unwrap());
#         });
#         reactor.lock().map(|mut r| r.handle = Some(handle)).unwrap();
#         reactor
#     }
#
#     fn wake(&mut self, id: usize) {
#         self.tasks.get_mut(&id).map(|state| {
#             match mem::replace(state, TaskState::Ready) {
#                 TaskState::NotReady(waker) => waker.wake(),
#                 TaskState::Finished => panic!("Called 'wake' twice on task: {}", id),
#                 _ => unreachable!()
#             }
#         }).unwrap();
#     }
#
#     fn register(&mut self, duration: u64, waker: Waker, id: usize) {
#         if self.tasks.insert(id, TaskState::NotReady(waker)).is_some() {
#             panic!("Tried to insert a task with id: '{}', twice!", id);
#         }
#         self.dispatcher.send(Event::Timeout(duration, id)).unwrap();
#     }
#
#     fn is_ready(&self, id: usize) -> bool {
#         self.tasks.get(&id).map(|state| match state {
#             TaskState::Ready => true,
#             _ => false,
#         }).unwrap_or(false)
#     }
# }
#
# impl Drop for Reactor {
#     fn drop(&mut self) {
#         self.dispatcher.send(Event::Close).unwrap();
#         self.handle.take().map(|h| h.join().unwrap()).unwrap();
#     }
# }
```

I added a some debug printouts so we can observe a couple of things:

1. How the `Waker` object looks just like the _trait object_ we talked about in an earlier chapter
2. The program flow from start to finish

The last point is relevant when we move on the the last paragraph.

> There is one subtle thing to note about our example. What happens if we pass
> in the same `id` for both events?
>
> ```rust, ignore
> let future1 = Task::new(reactor.clone(), 1, 1);
> let future2 = Task::new(reactor.clone(), 2, 1);
> ```
>
> We'll discuss this a bit more under exercises in the last chapter where we
> also look at ways to fix it. For now, just make a note of it so you're aware
> of the problem.

## Async/Await and concurrency

The `async` keyword can be used on functions as in `async fn(...)` or on a
block as in `async { ... }`. Both will turn your function, or block, into a
`Future`.

These Futures are rather simple. Imagine our generator from a few chapters
back. Every `await` point is like a `yield` point.

Instead of `yielding` a value we pass in, we yield the result of calling `poll` on
the next `Future` we're awaiting.

Our `mainfut` contains two non-leaf futures which it will call `poll` on. **Non-leaf-futures**
has a `poll` method that simply polls their inner futures and these state machines
are polled until some "leaf future" in the end either returns `Ready` or `Pending`.

The way our example is right now, it's not much better than regular synchronous
code. For us to actually await multiple futures at the same time we somehow need
to `spawn` them so the executor starts running them concurrently.

Our example as it stands now returns this:

```ignore
Future got 1 at time: 1.00.
Future got 2 at time: 3.00.
```

If these Futures were executed asynchronously we would expect to see:

```ignore
Future got 1 at time: 1.00.
Future got 2 at time: 2.00.
```

> Note that this doesn't mean they need to run in parallel. They _can_ run in
parallel but there is no requirement. Remember that we're waiting for some
external resource so we can fire off many such calls on a single thread and
handle each event as it resolves.

Now, this is the point where I'll refer you to some better resources for
implementing a better executor. You should have a pretty good understanding of
the concept of Futures by now helping you along the way.

The next step should be getting to know how more advanced runtimes work and
how they implement different ways of running Futures to completion.

[If I were you I would read this next, and try to implement it for our example.](./conclusion.md#building-a-better-exectuor).

That's actually it for now. There as probably much more to learn, this is enough
for today.

I hope exploring Futures and async in general gets easier after this read and I
do really hope that you do continue to explore further.

Don't forget the exercises in the last chapter 😊.

## Bonus Section - a Proper Way to Park our Thread

As we explained earlier in our chapter, simply calling `thread::sleep` is not really
sufficient to implement a proper reactor. You can also reach a tool like the `Parker`
in crossbeam: [crossbeam::sync::Parker][crossbeam_parker]

Since it doesn't require many lines of code to create a working solution ourselves we'll show how
we can solve that by using a `Condvar` and a `Mutex` instead.

Start by implementing our own `Parker` like this:

```rust, ignore
#[derive(Default)]
struct Parker(Mutex<bool>, Condvar);

impl Parker {
    fn park(&self) {

        // We aquire a lock to the Mutex which protects our flag indicating if we
        // should resume execution or not.
        let mut resumable = self.0.lock().unwrap();

            // We put this in a loop since there is a chance we'll get woken, but
            // our flag hasn't changed. If that happens, we simply go back to sleep.
            while !*resumable {

                // We sleep until someone notifies us
                resumable = self.1.wait(resumable).unwrap();
            }

        // We immidiately set the condition to false, so that next time we call `park` we'll
        // go right to sleep.
        *resumable = false;
    }

    fn unpark(&self) {
        // We simply acquire a lock to our flag and sets the condition to `runnable` when we
        // get it.
        *self.0.lock().unwrap() = true;

        // We notify our `Condvar` so it wakes up and resumes.
        self.1.notify_one();
    }
}
```

The `Condvar` in Rust is designed to work together with a Mutex. Usually, you'd think that we don't
release the mutex-lock we acquire in `self.0.lock().unwrap();` before we go to sleep. Which means
that our `unpark` function never will acquire a lock to our flag and we deadlock.

Using `Condvar` we avoid this since the `Condvar` will consume our lock so it's released at the
moment we go to sleep.

When we resume again, our `Condvar` returns our lock so we can continue to operate on it.

This means we need to make some very slight changes to our executor like this:

```rust, ignore
fn block_on<F: Future>(mut future: F) -> F::Output {
    let parker = Arc::new(Parker::default()); // <--- NB!
    let mywaker = Arc::new(MyWaker { parker: parker.clone() }); <--- NB!
    let waker = mywaker_into_waker(Arc::into_raw(mywaker));
    let mut cx = Context::from_waker(&waker);

    // SAFETY: we shadow `future` so it can't be accessed again.
    let mut future = unsafe { Pin::new_unchecked(&mut future) };
    loop {
        match Future::poll(future.as_mut(), &mut cx) {
            Poll::Ready(val) => break val,
            Poll::Pending => parker.park(), // <--- NB!
        };
    }
}
```

And we need to change our `Waker` like this:

```rust, ignore
#[derive(Clone)]
struct MyWaker {
    parker: Arc<Parker>,
}

fn mywaker_wake(s: &MyWaker) {
    let waker_arc = unsafe { Arc::from_raw(s) };
    waker_arc.parker.unpark();
}
```

And that's really all there is to it.

> If you checked out the playground link that showcased how park/unpark could [cause subtle
> problems](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=b2343661fe3d271c91c6977ab8e681d0)
> you can [check out this example](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=bebef0f8a8ce6a9d0d32442cc8381595) which shows how our final version avoids this problem.

The next chapter shows our finished code with this
improvement which you can explore further if you wish.

[mio]: https://github.com/tokio-rs/mio
[arc_wake]: https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.13/futures/task/trait.ArcWake.html
[example_repo]: https://github.com/cfsamson/examples-futures
[playground_example]:https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=ca43dba55c6e3838c5494de45875677f
[spurious_wakeup]: https://cfsamson.github.io/book-exploring-async-basics/9_3_http_module.html#bonus-section
[condvar]: https://doc.rust-lang.org/stable/std/sync/struct.Condvar.html
[crossbeam_parker]: https://docs.rs/crossbeam/0.7.3/crossbeam/sync/struct.Parker.html
