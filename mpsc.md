A short explanation and examples.

- What it is: `Mutex<mpsc::UnboundedReceiver<Status>>` (here `tokio::sync::Mutex`) stores the single receiver inside an async-aware mutex so multiple async methods/tasks can share access. `UnboundedReceiver<T>` requires exclusive mutable access to call `recv()` (it is not `Sync`), so wrapping it in a `Mutex` lets you `lock().await` and then call `recv().await`.

- Key caveat: calling `recv().await` while holding the mutex blocks other tasks from locking the same mutex until a message arrives. If many tasks need to receive concurrently, prefer a `broadcast` channel or restructure to have one dedicated receiver task that fans out messages.

Examples:

```rust
// rust
use tokio::sync::{mpsc, Mutex};
use std::sync::Arc;
use tokio::time::{sleep, Duration, timeout};

#[derive(Debug, Clone, Copy)]
enum Status { Ok, Error }

#[tokio::main]
async fn main() {
    // create unbounded channel
    let (tx, rx) = mpsc::unbounded_channel::<Status>();

    // wrap receiver in an async Mutex so multiple async functions can share it
    let shared_rx = Arc::new(Mutex::new(rx));

    // sender task
    let s = tx.clone();
    tokio::spawn(async move {
        s.send(Status::Ok).unwrap();
        sleep(Duration::from_millis(50)).await;
        s.send(Status::Error).unwrap();
    });

    // consumer example 1: lock and await recv()
    {
        let rx = Arc::clone(&shared_rx);
        tokio::spawn(async move {
            let mut guard = rx.lock().await;             // acquire exclusive access
            if let Some(st) = guard.recv().await {       // await a message while holding the lock
                println!("task1 got: {:?}", st);
            }
            // guard is dropped here
        });
    }

    // consumer example 2: try_recv loop to avoid long await while holding lock
    {
        let rx = Arc::clone(&shared_rx);
        tokio::spawn(async move {
            loop {
                {
                    let mut guard = rx.lock().await;
                    match guard.try_recv() {             // non-async, immediate
                        Ok(st) => {
                            println!("task2 got (try): {:?}", st);
                            continue;
                        }
                        Err(_) => { /* no message now */ }
                    }
                    // drop guard before sleeping
                }
                sleep(Duration::from_millis(10)).await;
            }
        });
    }

    // consumer example 3: use timeout with the locked receiver
    {
        let rx = Arc::clone(&shared_rx);
        tokio::spawn(async move {
            let mut guard = rx.lock().await;
            match timeout(Duration::from_millis(100), guard.recv()).await {
                Ok(Some(st)) => println!("task3 timeout got: {:?}", st),
                Ok(None) => println!("channel closed"),
                Err(_) => println!("timeout waiting for status"),
            }
        });
    }

    // wait a bit for tasks to run
    sleep(Duration::from_millis(200)).await;
}
```

- When to use:
  - Use this pattern if you must share a single receiver between async methods on the same object.
  - If multiple independent consumers are needed, use `tokio::sync::broadcast` or spawn a dedicated receiver task that redistributes messages (avoids holding a mutex across awaits).
