Short explanation:

- `Arc<PublishTrack>`: provides shared, thread\-safe ownership of a `PublishTrack`. Storing `Arc<T>` in the `Vec` lets the client and any callers hold independent owned references to the same track without copying the track or requiring a single owner.
- `RwLock<Vec<Arc<PublishTrack>>>` (from `tokio::sync::RwLock`): protects the `Vec` so multiple async tasks can access it concurrently. Readers can acquire a shared read lock concurrently; writers acquire an exclusive write lock to mutate the vector.
- Combined: the vector holds lightweight shared pointers (`Arc`) so you can hand out clones of `Arc` while the vector remains protected by the `RwLock`. This minimizes copying and lets you iterate or return track handles safely across threads/tasks.

Common usage patterns (Rust, `tokio`):

```rust
// Read access: iterate without blocking other readers. Clone Arc when you need an owned handle.
let read_guard = self.publish_tracks.read().await;
for track_arc in read_guard.iter() {
    let track = Arc::clone(track_arc); // cheap, increments ref count
    // use `track` outside the lock if needed
}
// read_guard is dropped here, releasing the read lock

// Write access: push a new track (exclusive)
let mut write_guard = self.publish_tracks.write().await;
let new_arc = Arc::new(publish_track);
write_guard.push(Arc::clone(&new_arc)); // store shared pointer
// write_guard dropped here, releasing the write lock

// Removing or unregistering: take write lock and mutate (retain, remove, etc.)
let mut write_guard = self.publish_tracks.write().await;
write_guard.retain(|arc| !arc.is_registered());
// then dropped
```

Notes:
- Use `Arc::clone(&arc)` (or `Arc::clone`) — do not call a free `clone` function.
- Always drop the lock guard (usually by leaving scope) before performing blocking or long operations that might reacquire locks.
- `tokio::sync::RwLock` is async\-aware; use `.read().await` / `.write().await` inside async contexts.
