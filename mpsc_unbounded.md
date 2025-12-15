Overview
- `tokio::sync::mpsc::unbounded_channel` creates an unbounded, multi-producer single-consumer channel.
- `UnboundedSender<T>` is cloneable; `UnboundedReceiver<T>` `.recv().await` yields `Some(T)` or `None`
  when all senders are dropped.
- Unbounded channels have no backpressure — send never awaits. Use them for signals/notifications or
  where you can tolerate unbounded buffering.
- Values sent across threads must meet `Send` (if you move senders to other threads).

Simple example — single producer, single consumer
- Demonstrates creating an unbounded channel, spawning a producer task that sends values and a
  consumer that awaits them.
- Shows dropping the sender to close the channel and end the consumer loop.

Brief explanation: producer sends integers via an unbounded channel; consumer receives until channel closes.
```rust
// rust
use tokio::sync::mpsc::unbounded_channel;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let (tx, mut rx) = unbounded_channel::<u32>();

    // Producer: send numbers 0..5 with a small delay
    let p = tokio::spawn(async move {
        for i in 0..6 {
            tx.send(i).expect("receiver alive");
            println!("sent {}", i);
            sleep(Duration::from_millis(50)).await;
        }
        // tx dropped here -> receiver will see None when queue drained
    });

    // Consumer: receive until channel closes
    let c = tokio::spawn(async move {
        while let Some(v) = rx.recv().await {
            println!("recv {}", v);
        }
        println!("channel closed, consumer done");
    });

    let _ = p.await;
    let _ = c.await;
}
```

Advanced example — dispatcher (single consumer) with many producers and worker tasks
- Use one receiver as a dispatcher that routes messages to worker-specific bounded channels.
- Shows how `unbounded_channel` is useful to gather events from many producers without blocking them,
  while a dispatcher does backpressured routing.

Brief explanation: many producers send jobs into a global unbounded queue; a dispatcher pulls jobs 
and distributes them to bounded worker queues with backpressure.
```rust
// rust
use tokio::sync::{mpsc::{unbounded_channel, channel}, oneshot};
use tokio::time::{sleep, Duration};
use std::sync::Arc;

#[derive(Debug)]
struct Job(u32);

#[tokio::main]
async fn main() {
    // global unbounded intake (many producers -> single dispatcher)
    let (intake_tx, mut intake_rx) = unbounded_channel::<Job>();

    // create worker bounded queues to exert backpressure per worker
    let mut workers = Vec::new();
    for id in 0..3 {
        let (w_tx, mut w_rx) = channel::<Job>(4); // bounded
        workers.push(w_tx);
        tokio::spawn(async move {
            while let Some(job) = w_rx.recv().await {
                println!("worker {} processing {:?}", id, job);
                sleep(Duration::from_millis(120)).await; // simulate work
            }
            println!("worker {} shutting down", id);
        });
    }

    // dispatcher: round-robin distribute jobs to workers
    let dispatcher = tokio::spawn(async move {
        let mut idx = 0usize;
        while let Some(job) = intake_rx.recv().await {
            let w = &workers[idx % workers.len()];
            // If worker queue full, try_send fails; we can choose to drop, block via a bounded channel,
            // or route elsewhere. Here we block with a oneshot workaround if necessary.
            match w.try_send(job) {
                Ok(()) => {}
                Err(e) => {
                    // Backpressure strategy: block until there's capacity
                    let job = e.into_inner();
                    // convert to blocking send via an async await
                    // (Note: try_send failing rarely; this illustrates a policy)
                    let _ = w.send(job).await;
                }
            }
            idx += 1;
        }
        // intake closed -> close workers by dropping their senders
        println!("dispatcher done, intake closed");
    });

    // many producers: spawn tasks that send quickly to intake (no await on send)
    let producers: Vec<_> = (0..5).map(|p| {
        let tx = intake_tx.clone();
        tokio::spawn(async move {
            for j in 0..6 {
                let job = Job(p * 100 + j);
                tx.send(job).expect("dispatcher alive");
                sleep(Duration::from_millis(20)).await;
            }
        })
    }).collect();

    drop(intake_tx); // optional: close when no further producers from this scope

    for p in producers { let _ = p.await; }
    let _ = dispatcher.await;
    // worker channels will close once dispatcher drops workers' senders
    sleep(Duration::from_millis(500)).await; // wait for workers to finish printing
}
```

Advanced considerations / best practices
- Unbounded = no backpressure. Avoid using it for high-throughput streams unless you accept
- unbounded memory growth.
- If you need backpressure, prefer `channel` with capacity and `.send().await`.
- Use `.recv().await` in the single consumer; `.try_recv()` is available for polling.
- Dropping all senders closes the channel; receiver loop should handle `None`.
- `UnboundedSender::send` is synchronous and returns `Result<(), SendError<T>>`. Failure only
- happens if receiver is closed.

FFI example — safe patterns to receive C callbacks into a Rust task via `unbounded_channel`
- Typical scenario: a C library accepts `user_data: *mut c_void` and a callback invoked from C threads.
  Rust wants to forward those events into async context.
- Pattern: create an `UnboundedSender<Event>` in Rust, store a stable pointer to it (e.g., in a `Box`)
  and pass that raw pointer to C as `user_data`. In the C callback, cast the pointer back
  to `&UnboundedSender<Event>` and call `.send(...)` (synchronous and safe). When unregistering,
  convert raw pointer back with `Box::from_raw` to drop it.
- Important safety rules:
  - Ensure the pointer remains valid while C might call the callback.
  - Do not call `Box::from_raw` in the callback (that would take ownership) unless you
    immediately restore it with `Box::into_raw`.
  - Prefer using a stable heap allocation (`Box` or `Arc`) that you explicitly free when unregistering.

Brief explanation: the fake C library stores a `user_data` pointer and calls the callback from a thread. 
Rust passes a boxed `UnboundedSender<Event>` pointer. Callback casts to a reference and calls `.send()`. 
On unregister we drop the boxed sender to close the channel.
```rust
// rust
use std::ffi::c_void;
use std::thread;
use std::time::Duration;
use tokio::sync::mpsc::unbounded_channel;

#[derive(Debug)]
struct Event { value: i32 }

// Fake C library simulator: stores a callback and user_data and calls it on a background thread
type CCallback = extern "C" fn(user_data: *mut c_void, value: i32);

struct FakeCLib {
    cb: Option<CCallback>,
    user_data: *mut c_void,
}

impl FakeCLib {
    fn new() -> Self { Self { cb: None, user_data: std::ptr::null_mut() } }

    fn register(&mut self, cb: CCallback, user_data: *mut c_void) {
        self.cb = Some(cb);
        self.user_data = user_data;
    }

    fn unregister(&mut self) -> *mut c_void {
        let ud = self.user_data;
        self.cb = None;
        self.user_data = std::ptr::null_mut();
        ud
    }

    fn invoke(&self, v: i32) {
        if let Some(cb) = self.cb {
            cb(self.user_data, v);
        }
    }
}

// C-style callback that will be called from C threads.
// We cast user_data to a reference to UnboundedSender<Event> and call send.
extern "C" fn c_callback_forward(user_data: *mut c_void, value: i32) {
    if user_data.is_null() { return; }
    // SAFETY: `user_data` was created by `Box::into_raw(Box::new(tx))`
    let tx_ptr = user_data as *const tokio::sync::mpsc::UnboundedSender<Event>;
    let tx_ref = unsafe { &*tx_ptr };
    // `send` is synchronous and returns Err if receiver closed
    let _ = tx_ref.send(Event { value });
}

#[tokio::main]
async fn main() {
    // Create unbounded channel to receive events into async code
    let (tx, mut rx) = unbounded_channel::<Event>();

    // Box the sender so we can hand a stable pointer to C
    let boxed_tx = Box::new(tx);
    let user_data = Box::into_raw(boxed_tx) as *mut c_void;

    // Register with fake C lib
    let mut clib = FakeCLib::new();
    clib.register(c_callback_forward, user_data);

    // Simulate C invoking callbacks from a separate OS thread
    let clib_backref = std::sync::Arc::new(std::sync::Mutex::new(clib));
    let th = {
        let cl = std::sync::Arc::clone(&clib_backref);
        thread::spawn(move || {
            // two calls from "C"
            thread::sleep(Duration::from_millis(50));
            cl.lock().unwrap().invoke(10);
            thread::sleep(Duration::from_millis(50));
            cl.lock().unwrap().invoke(20);
        })
    };

    // In async context: collect two events then unregister
    if let Some(ev) = rx.recv().await {
        println!("async received {:?}", ev);
    }
    if let Some(ev) = rx.recv().await {
        println!("async received {:?}", ev);
    }

    // Unregister from the fake C lib and recover the boxed sender
    let raw = clib_backref.lock().unwrap().unregister();
    if !raw.is_null() {
        // SAFETY: pointer came from Box::into_raw
        let boxed: Box<tokio::sync::mpsc::UnboundedSender<Event>> =
            unsafe { Box::from_raw(raw as *mut _) };
        // dropping boxed will close the channel (if no other clones exist)
        drop(boxed);
    }

    let _ = th.join();
    println!("done");
}
```

FFI notes and safety checklist
- Ensure the pointer passed to C remains valid for the whole time C may call the callback.
- When C gives the pointer back (unregister), reconstruct with `Box::from_raw` exactly once to free it.
- Avoid `Box::from_raw` inside callbacks unless you immediately `Box::into_raw` to restore
  ownership (to avoid double-free).
- If C can call the callback concurrently from many threads, ensure the pointee is thread-safe;
  `UnboundedSender<T>` is `Sync`/`Send` when `T: Send`.
- If you need multiple independent Rust consumers of C events, consider an `UnboundedSender`
  cloned per consumer or use a `broadcast`/fan-out approach in Rust.

This covers basic usage, an advanced dispatcher pattern, and an FFI integration pattern that 
safely forwards C callbacks into an async Rust task via `unbounded_channel`.
