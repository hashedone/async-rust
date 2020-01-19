# From blocking to async

----

## Disclaimer 1
This presentations is created with people fairly new to Rust, so some things may be obvious.

## Disclaimer 2
I am basing on current Rust standard, some APIs may differ for what you know from futures 0.1.

----

## Agenda
* What async means
* Building blocks
* Old way: continuators
* New way: async/await
* Crates

----

## Computation models

* Sequential calculations
* Parallel calculations
* Async calculations

---

### Asynchronius programming is defining calculations as a graph and delegate actual computation to runtime.

----

## Futures

### Calculations which may eventually give some result in future

---

## Futures

```rust
trait Future {
    fn poll(self: Pin<&mut Self>, cx: &mut Context)
        ->  Poll<Self::Output>;
}

enum Poll {
    Ready(T),
    Pending,
}
```

Note:
Emphase `Pin`

---

## Futures - continuators

```rust
trait FutureExt {
    fn map<U, F>(self, f: F)
        -> impl Future<Item = U>;
    fn then<Fut, F>(self, f: F)
        -> impl Future<Item = Fut::Output>;
    fn inspect<F>(self, f: F)
        -> impl Future<Item = Self::Output>;
}
```

----

## Streams

### Sources of data which may become available in future

---

## Streams

```rust
trait Stream {
    fn poll_next(self: Pin<&mut Self>, cx: &mut Context)
        -> Poll<Option<Self::Item>>;
}

enum Poll {
    Ready(T),
    Pending,
}
```

Note:
Emphase `Pin`

---

## Streams - combinators

```rust
trait StreamExt {
    fn map<T, F>(self, f: F)
        -> impl Stream<Item = T>;
    fn filter<Fut, F>(self, f: F)
        -> impl Stream<Item = Self::Item>;
    fn filter_map<Fut, T, F>(self, f: F)
        -> impl Stream<Item = T>;
    fn then<Fut, F>(self, f: F)
        -> impl Stream<Item = Fut::Output>;

    fn collect<C>(self)
        -> impl Future<Output = C>;
    // ...
}
```

Note:
Compare to `Iterator`
```

----

## Pin

Pin is a way to ensure, that if the type cares about not being ever moved, it will never move

## Unpin

Unping is a way to say, that type doesn't care about being moved

---

## Pin

```rust
fn main() {
    let s = create_some_stream();
    // Compile error - s is not pinned
    let polled = s.poll_next(cx);
}
```
---

## Pin

```rust
fn main() {
    let s = create_some_stream();
    // Pinning s to stack - s may not be ever moved
    pin_mut!(s);
    let polled = s.poll_next(cx);
}
```

```rust
fn main() {
    let s = create_some_stream();
    // Pinning s to heap - s may not be ever moved
    // (but whole box may)
    let s = Box::pin(s);
    let polled = s.poll_next(cx);
}
```

---

The `Pin` API is highly unsafe - it is not a good idea to deal with it directly!

----

## Async

Simplifyinig a little, `async` is just syntax sugar for `impl Future<...>`, but allowing usage of `await`.

```rust
async fn foo() -> Result<u8, String> {
    Ok(3)
}
```

```rust
fn foo() -> impl Future<Item=Result<u8, String>> {
    future::ready(Ok(3))
}
```

---

## Async
But it can be also used on blocks, to make them futures.

```rust
let _ = async {
    3
};
```

```rust
let _ = future::ready(3);
```

----

## Await

Again simiplifying, `await` is replacement for `and_then`/`map`, but cleaner.

```rust
let _ = async {
    let a = foo().await;
    a.do_thing(4).await
};
```
```rust
let _ = foo()
    .and_then(|a| a.do_thing(4));
```

---

## Await

But it also makes branching easy.

```rust
let _ = async {
    while let Some(item) = stream.next() {
        let item = foo(item).await;
        if check_something_happened() {
            do_exceptional_stuff().await;
        }
        bar(item).await;
    }

    finalize().await
};
```

Excercise: do it with continuators (this is actually possible).

---

## Await

And it also helps with borrowing.

```rust
let msg = mesage_to_be_send();
let _ = async move {
    log_msg(&msg).await;
    send_msg(&msg).await;
    wait_resp(&msg).await
}
```

Excercise: do it with continuators.

---

## Await

```rust
enum FutStage<'a> {
    BeforeLog(&'a Message),
    BeforeSend(&'a Message),
    BeforeWait(&'a Message),
    WaitingResp(WaitResp<'a>),
}

struct Fut {
    msg: Message,
    stage: FutStage<'???>,
}
```

This may be possible in future, with Polonius, but it is not for now, but `async`/`await` can figure out lifetime for `FutStage` safely - just because it may ensure `msg` will never move.

---

## Async/await

`Async`/`await` is commonly traeted just like syntax sugar, and making code cleaner is probably the most important benefit of it. However it is good to have in mind,
that it also prevents for unnecessary overhead, like obsolete clonning (which is commonly avoided by `Arc`, but `Arc` is an overhead on its own).

----

# Case study

1. Send message
2. Wait response
    1. If future resolves, forward the result
    2. If timeout occured before response is received:
        1. If there were less than 3 attempts, goto 1)
        2. Otherwise return error
3. Return response

---

### Design

* `register_for_response` method setups some synchronization primitive for waiting for response
* `send` method sends message
* function should not block - if it will, it will be executed on dedicated thread

----

### Parallel solution

```rust
fn send_request(&self, message: &Message)
-> Result<Message, Err> {
    let (cvar, mutex) = self.register_for_response(message.id);
    for _ in 0..3 {
        self.send(message.clone());
        let resp = cvar.wait_timeout(mutex.lock().unwrap(),
                                     Duration::seconds(3));
        if !resp.timed_out() {
            return self.get_response(mutex.lock().unwrap());
        }
    }
    Result::Err(Timeout)
}
```

Note:
`register_for_response` locks `RWGuard` on some `HashMap`, where it keeps additional `Mutex` and `ConditionalVariable`
for sending message through it (and returns those primitives)
`get_response` returns actuall response using given sync primitives and removes entry in `HashMap`.

---

## Pros
* Fairly simple both to read and write

## Cons
* Involves new thread for every request
* Synchronizations is a bit nasty

Note:
The problem with synchronious parallel execution is that computers doesn't handle many actual threads well - both context switching and locking mutex is slow

----

### Async solution with continuators

```rust
fn send_request(&self, message: &Message)
-> impl Future<Output=Result<Message, Err>> {
    let response = self.register_for_response(message.id);
    let shared = self.clone();

    let retransmit = stream::unfold((), move |_| {
        shared.send(message.clone())
            .and_then(tokio::Delay::new(Instant::now + TIMEOUT))
	    .map(|_| Some((), ()))
    }).take(3)
    .try_for_each(|_| future::ready(()))
    .then(|_| feature::ready(Result::Err(Timeout)));

    select(response, retransmit).factor_first()
}
```

Note:
`register_for_response` locks `RWGuard` on some `HashMap` where it keeps `oneshot::Receiver` channel where
response should appear.
Removing receiver from map is missing here - it should be done as a continuation for actuall result.

---

## Pros
* Threads are controlled by runtime

## Cons
* WTF/min count
* Additional shared pointer is introduced - it's obsolete

----

## Async solution with async/await

```rust
async fn send_request(&self, message: &Message)
-> Result<Message, Err> {
    let response = self.register_for_response(message.id);
    let retransmit = async {
        let i = tokio::interval(TIMEOUT);
        for _ in 0..3 {
            self.send(message.clone()).await?;
            i.tick().await;
        }
        Resutl::Err(Timeout)
    };
    let res = select(response, retransmit).await.0;
    self.cancel_response(message.id); res
}
```

Note:
`register_for_response` locks `RWGuard` on some `HashMap` where it keeps `oneshot::Receiver` channel where
response should appear.

---

## Pros
* Looks very straightforward
* Threads are controlled by runtime
* No unnecessary overhead

## Cons
* New syntax to get used to

----

## Problems

There is no syntax for async closures... yet.

Proposed syntax (`async_closure` in nighlty):
```rust
async |_| { /* ... */ }
```

Workaround:

```rust
move |_| async move { /* ... */ }
```

----

## Usefull crates
* Futures-rs
* Tokio
* Async-std
* Async-stream
* Async-std

----

## Futures-RS

* Futures continuators
* Streams combinators
* Tools for easy creation of own Futures/Streams
* Basic synchronization primitives

----

## Tokio

* Runtime
* IO Streams (FS, Net, Signals)
* Time handling
* Less basic synchronization primitives

---

## Tokio

```rust
#[tokio::main]
async fn main() {
    // ...
}
```

```rust
#[tokio::test]
async fn test() {
    assert_eq!(3, foo().await.unwrap())
}
```

----

## Async-std

Kind of mariage of Futures-RS and Tokio, but pretends to mimic std.

---

## Async-std

```rust
#[async_std::main]
async fn main() {
    // ...
}
```

```rust
#[async_std::test]
async fn test() {
    assert_eq!(3, foo().await.unwrap())
}
```

----

## Async-stream

Allows to create custom streams very easly.

```rust
let s = stream! {
    for i in 0..3 {
        yield i;
    }
};
```

----

## Pin project

Allows reasonable cooperation with `Pin`.

```rust
#[pin_project]
struct Struct<T, U> {
    #[pin]
    pinned: T,
    unpinned: U,
}

impl<T, U> Struct<T, U> {
    fn foo(self: Pin<&mut Self>) {
        let this = self.project();
        let _: Pin<&mut T> = this.pinned;
        let _: &mut U = this.unpinned;
    }
}
```

----

# Questions?

----

# Thank you

Find me on github:

https://github.com/hashedone/

Find me on Rust Wrocław:

http://www.rust-wroclaw.pl/
