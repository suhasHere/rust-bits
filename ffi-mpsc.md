```rust
struct ClientCallbackData {
    status_tx: mpsc::UnboundedSender<Status>,
    server_setup_tx: mpsc::UnboundedSender<ServerSetup>,
}
```

Explanation of `ClientCallbackData`

- Purpose: holds the channel senders used by the C callbacks to forward events
  (status changes and server setup) into the async Rust side.
- Fields:
  - `status_tx: mpsc::UnboundedSender<Status>` — non-blocking sender used to deliver `Status`
    updates from the C callback to the `status_rx` receiver inside `Client`.
  - `server_setup_tx: mpsc::UnboundedSender<ServerSetup>` — sender used to deliver `ServerSetup`
    info to the `server_setup_rx` receiver.

How it’s used
- An instance is allocated as a `Box<ClientCallbackData>` and stored on the `Client` so
  its memory remains stable.
- The raw pointer (`user_data`) passed to the C callbacks points into that boxed data.
  The callbacks unsafely cast the pointer back to `&ClientCallbackData` and call `.send(...)`
  on the appropriate sender.
- The `UnboundedSender::send` call is synchronous and non\-blocking, suitable for use inside
  an FFI callback where async/await cannot be used. The receiving side (inside `Client`) awaits
  on the corresponding `UnboundedReceiver`.

Safety notes
- The `Box` must outlive any callbacks invoked by the C library; keeping it on the `Client`
  ensures that while the `Client` exists the pointer remains valid.
- Callbacks use `unsafe` to cast the pointer; ensure the pointer always points at a valid
  `ClientCallbackData` to avoid UB.
- The senders are cheap to clone if needed; they are thread\-safe for sending from the C
  callback thread into Tokio tasks.
