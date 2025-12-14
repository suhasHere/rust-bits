The `Default` trait in Rust provides a way to create a "default" value for a 
type by implementing `fn default() -> Self`. In this code:

Example (illustrates the trait and usage):
```rust
// rust
#[derive(Debug)]
struct Foo { x: u32 }

impl Default for Foo {
    fn default() -> Self { Self { x: 42 } }
}

fn main() {
    let f = Foo::default(); // calls impl Default::default
    println!("{:?}", f);
}
```

Below is a real world example for building configs using builder pattern

```rust
impl Default for TransportConfig {
    fn default() -> Self {
        Self {
            tls_cert_filename: None,
            tls_key_filename: None,
            time_queue_init_queue_size: 1000,
            time_queue_max_duration: 2000,
            time_queue_bucket_interval: 1,
            time_queue_rx_size: 1000,
            debug: false,
            quic_cwin_minimum: 131072,
            quic_wifi_shadow_rtt_us: 20000,
            idle_timeout: Duration::from_secs(30),
            use_reset_wait_strategy: false,
            use_bbr: true,
            quic_qlog_path: None,
        }
    }
}

```

- `impl Default for TransportConfig { ... }` defines the canonical default transport settings
- (timeouts, queue sizes, flags, etc.).

```rust
impl TransportConfig {
    /// Create a new transport configuration with default values
    pub fn new() -> Self {
        Self::default()
    }

    /// Create a builder for transport configuration
    pub fn builder() -> TransportConfigBuilder {
        TransportConfigBuilder::default()
    }
}
```
  
 `TransportConfig::new()` calls `Self::default()` so it returns the same default instance.

```rust
#[derive(Debug, Default)]
pub struct TransportConfigBuilder {
    config: TransportConfig,
}

impl TransportConfig {
    /// Create a new transport configuration with default values
    pub fn new() -> Self {
        Self::default()
    }

    /// Create a builder for transport configuration
    pub fn builder() -> TransportConfigBuilder {
        TransportConfigBuilder::default()
    }
}
```
  
- `#[derive(Default)]` on `TransportConfigBuilder` uses `TransportConfig::default()` to
  initialize its `config` field, allowing `TransportConfigBuilder::default()`
  (and thus `TransportConfig::builder()`) to produce a builder pre-filled with default transport values.

Because `Default` is in Rust's prelude, no `use` is necessary. 

Typical usage: `let cfg = TransportConfig::default()` or via the builder: `let b = TransportConfig::builder()`.


```
