tokio::sync::RwLock tutorial
=================================================

Brief: `tokio::sync::RwLock` is an async-aware reader/writer lock. Many readers can hold the 
lock concurrently with `.read().await`, writers need exclusive access with `.write().await`. 
Use `Arc` to share the lock across tasks and `Arc` inside the collection for cheap clones of items.

Simple example — concurrent readers + single writer
---------------------------------------------------
Explanation: a shared `Arc<RwLock<Vec<u32>>>` is cloned into multiple tasks. Readers take a 
read lock, clone the small snapshot they need, drop the lock, then do work. The writer takes a 
write lock to mutate the `Vec`.

```rust
// rust
use std::sync::Arc;
use tokio::sync::RwLock;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let shared = Arc::new(RwLock::new(Vec::<u32>::new()));

    // Writer task: push numbers every 100ms
    let w = {
        let shared = Arc::clone(&shared);
        tokio::spawn(async move {
            for i in 0..10 {
                {
                    let mut write = shared.write().await;
                    write.push(i);
                    // drop(write) here when scope ends
                }
                sleep(Duration::from_millis(100)).await;
            }
        })
    };

    // Spawn multiple readers that periodically snapshot the vec
    let mut readers = Vec::new();
    for id in 0..3 {
        let shared = Arc::clone(&shared);
        readers.push(tokio::spawn(async move {
            for _ in 0..6 {
                // Acquire read lock
                let read = shared.read().await;
                // Clone what we need while holding the read lock
                let snapshot: Vec<u32> = read.iter().cloned().collect();
                // drop(read) here (end of scope) to allow other writers
                drop(read);

                // Do heavier work outside the lock
                println!("reader {} snapshot: {:?}", id, snapshot);
                sleep(Duration::from_millis(150)).await;
            }
        }));
    }

    w.await.unwrap();
    for r in readers { r.await.unwrap(); }
}
```

Advanced example — `RwLock<Vec<Arc<Item>>>`, safe iteration and removal
--------------------------------------------------------------------
Explanation: store `Arc<Item>` in the `Vec` so callers can hold item handles without 
owning the vector. When iterating, clone `Arc`s under a read lock, drop the lock, 
then operate on the clones. For removals, take a write lock and `retain` unwanted items. 
Use atomic fields or interior `Mutex` inside `Item` for per-item mutation.

```rust
// rust
use std::sync::{
    atomic::{AtomicBool, Ordering},
    Arc,
};
use tokio::sync::RwLock;
use tokio::time::{sleep, Duration};

#[derive(Debug)]
struct Item {
    id: u32,
    registered: AtomicBool,
}

impl Item {
    fn new(id: u32) -> Self {
        Self { id, registered: AtomicBool::new(false) }
    }

    fn is_registered(&self) -> bool {
        self.registered.load(Ordering::SeqCst)
    }

    fn set_registered(&self, v: bool) {
        self.registered.store(v, Ordering::SeqCst)
    }
}

#[tokio::main]
async fn main() {
    let store: Arc<RwLock<Vec<Arc<Item>>>> = Arc::new(RwLock::new(Vec::new()));

    // add items (writer)
    {
        let mut w = store.write().await;
        for id in 0..5 {
            let item = Arc::new(Item::new(id));
            item.set_registered(true); // pretend registering
            w.push(item);
        }
        // write guard dropped here
    }

    // background task: periodically inspect registered items
    let reader_store = Arc::clone(&store);
    let inspector = tokio::spawn(async move {
        for _ in 0..3 {
            // 1) take read lock, clone Arc handles of items we care about
            let clones: Vec<Arc<Item>> = {
                let read = reader_store.read().await;
                read.iter()
                    .filter(|it| it.is_registered())
                    .cloned()
                    .collect()
                // read guard dropped here
            };

            // 2) operate on clones outside the lock
            for it in clones {
                println!("inspector sees registered item id={}", it.id);
                // Simulate expensive work; safe because we don't hold the RwLock
                sleep(Duration::from_millis(50)).await;
            }
            sleep(Duration::from_millis(200)).await;
        }
    });

    // background task: unregister some items (writer)
    let writer_store = Arc::clone(&store);
    let cleaner = tokio::spawn(async move {
        // Wait a bit, then unregister id 1 and 3
        sleep(Duration::from_millis(300)).await;
        let mut w = writer_store.write().await;
        for it in w.iter() {
            if it.id == 1 || it.id == 3 {
                it.set_registered(false);
            }
        }
        // Remove unregistered items
        w.retain(|it| it.is_registered());
        // write guard dropped here
    });

    inspector.await.unwrap();
    cleaner.await.unwrap();
}
```

Concise best practices
- Use `Arc::clone(&arc)` to cheaply clone shared handles.
- Never hold a `read()` or `write()` guard across `.await` on an unrelated future;
  clone what you need and drop the guard first.
- For per-item mutation, prefer interior mutability (`Atomic*`, `Mutex`) inside the item
  rather than keeping a big write lock.
- Use `try_read()`/`try_write()` for opportunistic access when you want non-blocking attempts.
- For maps with heavy concurrent access consider concurrent containers (e.g., `dashmap`)
  instead of a global `RwLock`.

This provides safe patterns for `tokio::sync::RwLock` both for simple and more complex concurrent designs.
