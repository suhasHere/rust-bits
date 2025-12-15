`Arc` is an atomically reference counted pointer for sharing ownership of 
immutable (or internally mutable) data across threads/tasks. 
Use `Arc::clone(&arc)` to cheaply increment the ref count. For mutable shared 
state combine `Arc` with `Mutex` / `RwLock`. Use `Weak` to avoid reference cycles.

Rust code examples (simple and advanced):

```rust
// rust
// -------------------------
// Simple examples (std::thread)
// -------------------------
use std::sync::{Arc, Mutex};
use std::thread;

// 1) Sharing read-only data between threads
fn simple_read_only_example() {
    let data = Arc::new("hello from Arc".to_string());
    let mut handles = Vec::new();

    for i in 0..4 {
        let data = Arc::clone(&data); // cheap clone of Arc
        handles.push(thread::spawn(move || {
            println!("thread {}: {}", i, data);
        }));
    }

    for h in handles {
        h.join().unwrap();
    }
}

// 2) Shared mutable counter using Arc + Mutex
fn simple_mutable_example() {
    let counter = Arc::new(Mutex::new(0usize));
    let mut handles = Vec::new();

    for _ in 0..8 {
        let c = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            for _ in 0..100 {
                let mut guard = c.lock().unwrap();
                *guard += 1;
                // guard dropped here
            }
        }));
    }

    for h in handles {
        h.join().unwrap();
    }

    println!("counter = {}", *counter.lock().unwrap());
}

// -------------------------
// Advanced examples
// -------------------------
use std::sync::{atomic::{AtomicBool, Ordering}, Weak};
use tokio::sync::RwLock;
use tokio::time::{sleep, Duration};

// Item with interior mutability (AtomicBool)
#[derive(Debug)]
struct Item {
    id: u32,
    active: AtomicBool,
}
impl Item {
    fn new(id: u32) -> Self {
        Self { id, active: AtomicBool::new(true) }
    }
    fn is_active(&self) -> bool { self.active.load(Ordering::SeqCst) }
    fn set_active(&self, v: bool) { self.active.store(v, Ordering::SeqCst) }
}

// 3) Pattern: Arc\<RwLock\<Vec\<Arc<Item\>\>\>\> with async tasks
#[tokio::main]
async fn advanced_rwlock_example() {
    // Shared store of Arc<Item>
    let store: Arc<RwLock<Vec<Arc<Item>>>> = Arc::new(RwLock::new(Vec::new()));

    // Populate store (writer)
    {
        let mut w = store.write().await;
        for id in 0..5 {
            w.push(Arc::new(Item::new(id)));
        }
        // write guard dropped here
    }

    // Inspector task: take read lock, clone handles, drop lock, then work
    let inspector_store = Arc::clone(&store);
    let inspector = tokio::spawn(async move {
        for _ in 0..3 {
            // Acquire read lock, filter and clone Arc handles
            let clones: Vec<Arc<Item>> = {
                let r = inspector_store.read().await;
                r.iter()
                    .filter(|it| it.is_active())
                    .cloned()
                    .collect()
                // read guard dropped here
            };

            // Work on clones outside the lock (no holding across .await)
            for it in clones {
                println!("inspector sees item id={}", it.id);
                sleep(Duration::from_millis(30)).await;
            }
            sleep(Duration::from_millis(100)).await;
        }
    });

    // Cleaner task: mutate items and remove inactive ones
    let cleaner_store = Arc::clone(&store);
    let cleaner = tokio::spawn(async move {
        sleep(Duration::from_millis(120)).await;
        let mut w = cleaner_store.write().await;
        // mark id 1 and 3 inactive
        for it in w.iter() {
            if it.id == 1 || it.id == 3 {
                it.set_active(false);
            }
        }
        // remove inactive
        w.retain(|it| it.is_active());
        // write guard dropped here
    });

    inspector.await.unwrap();
    cleaner.await.unwrap();
}

// 4) Avoiding cycles: Arc + Weak example
fn weak_example() {
    use std::sync::Mutex;
    use std::rc::Rc; // for single-threaded example; replace with Arc in multi-threaded

    // Node structure with a Weak back-reference to avoid cycle
    struct Node {
        name: String,
        parent: Mutex<Option<Weak<Node>>>, // use Arc/Weak for thread-safe version
    }

    // Note: This snippet is conceptual. For multi-threaded code, use Arc/Mutex instead of Rc.
    println!("Use Weak to break Arc reference cycles.");
}

// -------------------------
// Run simple examples
// -------------------------
fn main() {
    simple_read_only_example();
    simple_mutable_example();

    // advanced examples: the tokio example runs its own async main above;
    // to try it separately, run `advanced_rwlock_example()` as a tokio program.
}
```

Short best practices:
- Use `Arc::clone(&arc)` to share ownership (do not call `clone()` free function).
- Prefer interior mutability (`Atomic*`, `Mutex`) inside items instead of holding a
  global write lock for long work.
- Never hold locks across `.await` points; clone `Arc` handles while holding the lock,
  then drop the guard and `await`.
- Use `Weak` to break cycles when objects refer to each other.
