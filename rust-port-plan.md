# Rust Port of PVXS — Design Document

## What Is PVXS?

PVXS is a C++ library implementing the **PVAccess (PVA)** network protocol for EPICS (Experimental Physics and Industrial Control System) — a framework used in particle accelerators and scientific instruments worldwide. It provides:

- A **client** that can Get, Put, Monitor, and RPC against named *process variables* (PVs) over TCP/UDP
- A **server** that hosts PVs and responds to those operations
- A **binary wire codec** for the PVA protocol
- A polymorphic **Value** container carrying arbitrary structured data (scalars, structs, unions, arrays)

The C++ implementation uses `libevent2` for async I/O. The Rust port replaces that with `tokio` and idiomatic Rust patterns throughout.

---

## Cargo Workspace Layout

```
pvxs-rs/
├── Cargo.toml               # workspace manifest
│
├── pvxs/                    # thin facade crate — re-exports everything for users
│   └── src/lib.rs
│
├── pvxs-core/               # data model: no I/O, no async
│   └── src/
│       ├── lib.rs
│       ├── value.rs         # Value, TypeCode, FieldDesc, BitSet
│       ├── shared_array.rs  # SharedArray<T> (type-erased, ref-counted)
│       ├── nt.rs            # Normative Types: NTScalar, Alarm, TimeStamp
│       └── error.rs         # PvxsError enum + Result alias
│
├── pvxs-proto/              # pure codec: encode/decode PVA frames, no sockets
│   └── src/
│       ├── lib.rs
│       ├── header.rs        # PvaHeader (8 bytes: magic, version, flags, cmd, len)
│       ├── messages.rs      # structs for every PVA message type
│       ├── encode.rs        # PvaEncode trait + impls
│       ├── decode.rs        # PvaDecode trait + impls
│       └── type_cache.rs    # per-connection FieldDesc cache (0xfd/0xfe/0xff markers)
│
├── pvxs-client/             # async tokio client
│   └── src/
│       ├── lib.rs
│       ├── context.rs       # ClientContext: connection pool, UDP search dispatch
│       ├── channel.rs       # Channel state machine
│       ├── ops.rs           # GetBuilder, PutBuilder, MonitorBuilder, RpcBuilder
│       ├── conn.rs          # TCP connection task (Framed<TcpStream, PvaCodec>)
│       └── udp.rs           # UDP search sender + response listener
│
├── pvxs-server/             # async tokio server
│   └── src/
│       ├── lib.rs
│       ├── server.rs        # ServerBuilder + Server lifecycle
│       ├── conn.rs          # per-client TCP connection task
│       ├── source.rs        # Source trait + StaticSource registry
│       ├── shared_pv.rs     # SharedPV: in-process PV with post/open/close
│       └── udp.rs           # Beacon broadcaster + SearchRequest responder
│
└── pvxs-tools/              # CLI binaries
    └── src/bin/
        ├── pvxget.rs
        ├── pvxput.rs
        ├── pvxmon.rs
        └── pvxlist.rs
```

---

## Key Dependencies

| Crate | Why |
|-------|-----|
| `tokio` (features = ["full"]) | Async runtime, TCP/UDP sockets, timers, channels |
| `bytes` | `Bytes` / `BytesMut` — zero-copy buffers replacing `EvInBuf`/`EvOutBuf` |
| `tokio-util` | `codec::Framed` for framing TCP streams |
| `futures` | `Stream` / `Sink` traits for Monitor subscriptions |
| `thiserror` | Derive-based error types |
| `tracing` | Structured logging replacing `DEFINE_LOGGER` / `log_debug` |
| `tracing-subscriber` | Log output in tools/tests |
| `arc-swap` | Lock-free `Arc` swap for SharedPV live updates |
| `dashmap` | Concurrent `HashMap` for channel / connection registries |
| `async-trait` | `async fn` in trait definitions (`Source`) |
| `clap` | CLI argument parsing for tools |

---

## Core Data Model (`pvxs-core`)

### TypeCode

Maps directly to the PVA wire type byte. Values are the same as the C++ enum.

```rust
#[repr(u8)]
pub enum TypeCode {
    Bool    = 0x00,  BoolA   = 0x08,
    Int8    = 0x20,  Int16   = 0x21,  Int32   = 0x22,  Int64 = 0x23,
    UInt8   = 0x24,  UInt16  = 0x25,  UInt32  = 0x26,  UInt64 = 0x27,
    Int8A   = 0x28,  /* ... */        Int64A  = 0x2b,
    UInt8A  = 0x2c,  /* ... */        UInt64A = 0x2f,
    Float32 = 0x42,  Float64 = 0x43,
    Float32A= 0x4a,  Float64A= 0x4b,
    String  = 0x60,  StringA = 0x68,
    Struct  = 0x80,  Union   = 0x81,  Any  = 0x82,
    StructA = 0x88,  UnionA  = 0x89,  AnyA = 0x8a,
    Null    = 0xff,
}
```

### FieldDesc (type descriptor tree)

```rust
pub struct FieldDesc {
    pub id:       String,                            // e.g. "epics:nt/NTScalar:1.0"
    pub code:     TypeCode,
    pub children: Vec<(String, Arc<FieldDesc>)>,     // (field_name, field_type)
}
```

### Value (universal data container)

Replaces the entire `PVField` class hierarchy from pvDataCPP. Null-safe — all methods are safe to call on an empty Value.

```rust
pub struct Value {
    desc:    Arc<FieldDesc>,
    storage: Arc<ValueData>,
    valid:   BitSet,           // which fields carry live data
}

enum ValueData {
    Null,
    Bool(bool),
    Int64(i64),
    UInt64(u64),
    Float64(f64),
    String(Arc<str>),
    Struct(Vec<Value>),
    Union { selector: u32, inner: Box<Value> },
    Array(SharedArray),
}
```

### SharedArray

Type-erased ref-counted array, mirrors C++ `shared_array<void>`.
Freeze/thaw semantics: mutable while building, immutable once shared.

```rust
pub struct SharedArray {
    elem_type: TypeCode,
    data: Arc<dyn Any + Send + Sync>,
}
```

---

## Wire Protocol Codec (`pvxs-proto`)

### Message Header (8 bytes)

```
Byte 0:    0xCA  (magic)
Byte 1:    0x02  (PVA version 2)
Byte 2:    flags  [7]=big-endian [6]=server→client [5:4]=segmentation [0]=ctrl
Byte 3:    command
Bytes 4-7: body length (uint32, host byte order after swap)
```

```rust
pub struct PvaHeader {
    pub magic:   u8,
    pub version: u8,
    pub flags:   u8,
    pub command: u8,
    pub length:  u32,
}
```

### TCP Framing

Custom `tokio_util::codec::Decoder`:
1. Buffer until 8 bytes available → parse header
2. Buffer until `header.length` more bytes available → yield frame
3. Return `PvaFrame { header, body: Bytes }`

### Codec Traits

```rust
pub trait PvaEncode {
    fn encode(&self, buf: &mut BytesMut, big_endian: bool);
}
pub trait PvaDecode: Sized {
    fn decode(header: &PvaHeader, body: &mut Bytes) -> Result<Self, ProtoError>;
}
```

### Variable-Length Size Encoding

Identical to C++ `to_wire(Buffer&, Size)`:

```
< 254       → 1 byte
= 254 (0xfe) → 1 byte marker + 4 byte uint32
= 255 (0xff) → null (empty Union)
```

### Type Descriptor Cache

Per-connection cache of `Arc<FieldDesc>`, keyed by a slot index.

| Wire byte | Meaning |
|-----------|---------|
| `0xfd`    | Full descriptor follows; assign next cache slot |
| `0xfe`    | Reference to cached slot (2-byte index follows) |
| `0xff`    | Null type |

### Message Types to Implement

| Command | Name |
|---------|------|
| `0x00`  | Beacon |
| `0x01`  | ConnectionValidation |
| `0x02`  | Echo |
| `0x03`  | Search |
| `0x04`  | SearchResponse |
| `0x07`  | CreateChannel |
| `0x08`  | DestroyChannel |
| `0x0A`  | Get |
| `0x0B`  | Put |
| `0x0C`  | PutGet |
| `0x0D`  | Monitor |
| `0x11`  | GetField |
| `0x14`  | RPC |

---

## UDP Discovery

### Default Ports
- Server listens: **5076** (TCP + UDP)
- Multicast group: `224.0.0.128`

### Server-Side UDP Task

```rust
async fn udp_server_task(
    socket: UdpSocket,
    guid: [u8; 12],              // unique 12-byte server ID
    registry: Arc<ChannelRegistry>,
    beacon_interval: Duration,
) {
    // 1. Send Beacon periodically (timer)
    // 2. On SearchRequest: check registry, reply SearchResponse for known PVs
}
```

### Client-Side UDP Task

```rust
async fn udp_client_task(
    socket: UdpSocket,
    search_rx: mpsc::Receiver<Vec<String>>,  // PV names to search for
    found_tx: mpsc::Sender<(String, SocketAddr)>,  // PV name + server addr
) {
    // 1. On search_rx: send SearchRequest to broadcast + configured addresses
    // 2. On SearchResponse: forward (pv_name, server_addr) to found_tx
    // 3. On Beacon: trigger re-search for disconnected channels
}
```

---

## Client (`pvxs-client`)

### ClientContext

```rust
pub struct ClientContext {
    inner: Arc<ClientInner>,
}

struct ClientInner {
    config:      ClientConfig,
    connections: DashMap<SocketAddr, Arc<ClientConn>>,
    channels:    DashMap<String, Arc<ChannelState>>,
    search_tx:   mpsc::Sender<Vec<String>>,
}

impl ClientContext {
    pub fn new() -> Self { ... }                  // reads PVXS_* env vars
    pub fn from_config(cfg: ClientConfig) -> Self { ... }
    pub fn get(&self, pv: &str) -> GetBuilder { ... }
    pub fn put(&self, pv: &str) -> PutBuilder { ... }
    pub fn monitor(&self, pv: &str) -> MonitorBuilder { ... }
    pub fn rpc(&self, pv: &str, req: Value) -> RpcBuilder { ... }
}
```

### Builder Pattern (mirrors C++ fluent API)

```rust
// One-shot Get — awaitable
let value: Value = ctx.get("SR:CURRENT")
    .request("field(value,alarm,timeStamp)")
    .await?;

// One-shot Put
ctx.put("SR:SETPOINT")
    .set(|mut v| { v["value"] = 1.5f64.into(); v })
    .await?;

// Streaming Monitor — returns impl Stream<Item = Result<Value>>
let mut sub = ctx.monitor("SR:CURRENT")
    .request("field(value)")
    .subscribe();
while let Some(update) = sub.next().await {
    println!("{:?}", update?);
}
```

### Channel State Machine

```
Searching ──UDP hit──▶ Connecting ──TCP open──▶ Creating ──server ack──▶ Connected
    ▲                                                                         │
    └──────────────────────── disconnect / error ─────────────────────────────┘
```

Each `ClientConn` is a `tokio::task` owning a `Framed<TcpStream, PvaCodec>`.
Channels are multiplexed over connections by `serverChannelID` / `clientChannelID` pairs.

---

## Server (`pvxs-server`)

### Server Builder

```rust
pub struct Server { inner: Arc<ServerInner> }

impl Server {
    pub fn builder() -> ServerBuilder { ... }
    pub fn run_until_signal(self) -> impl Future<Output = ()> { ... }
}

impl ServerBuilder {
    pub fn bind(self, addr: impl ToSocketAddrs) -> Self { ... }
    pub fn add_source(self, priority: i32, src: impl Source + 'static) -> Self { ... }
    pub fn build(self) -> Result<Server, PvxsError> { ... }
}
```

### Source Trait

```rust
#[async_trait]
pub trait Source: Send + Sync {
    fn list(&self) -> Vec<String>;
    async fn connect(&self, op: ConnectOp);
}

pub struct ConnectOp {
    pub pv_name:     String,
    pub credentials: ClientCredentials,
    // reply methods: op.set_type(desc), op.get_handler(fn), op.monitor_handler(fn), ...
}
```

### SharedPV

Simple in-process PV storage; the default building block for servers.

```rust
pub struct SharedPV { inner: Arc<SharedPvInner> }

impl SharedPV {
    pub fn open(&self, initial: Value) { ... }   // publish type + first value
    pub fn post(&self, update: Value) { ... }    // push to all active monitors
    pub fn close(&self) { ... }                  // disconnect all subscribers
}
```

Internally uses `arc_swap::ArcSwap<Option<Value>>` for the current value and a `Vec<mpsc::Sender<Value>>` for subscriber channels.

---

## Error Handling

```rust
#[derive(Debug, thiserror::Error)]
pub enum PvxsError {
    #[error("I/O: {0}")]          Io(#[from] std::io::Error),
    #[error("protocol: {0}")]     Protocol(String),
    #[error("type mismatch: expected {expected}, got {actual}")]
                                  TypeMismatch { expected: String, actual: String },
    #[error("PV not found: {0}")] NotFound(String),
    #[error("cancelled")]         Cancelled,
    #[error("remote: {0}")]       Remote(String),
    #[error("timeout")]           Timeout,
}

pub type Result<T> = std::result::Result<T, PvxsError>;
```

---

## Normative Types (`pvxs-core/src/nt.rs`)

Standard EPICS structured types built on top of `Value`. Mirrors `src/pvxs/nt.h`.

```rust
pub struct NTScalar { pub value: Value, pub alarm: Alarm, pub timestamp: TimeStamp, pub display: Option<Display> }
pub struct Alarm    { pub severity: AlarmSeverity, pub status: AlarmStatus, pub message: String }
pub struct TimeStamp { pub seconds_past_epoch: i64, pub nanoseconds: i32, pub user_tag: i32 }
// EPICS epoch = Jan 1, 1990 (not Unix epoch)
```

---

## Logging

Replace C++ `DEFINE_LOGGER` / `log_debug_printf` with `tracing`:

```rust
tracing::debug!(pv = %name, "search sent");
tracing::warn!(conn = %addr, "TCP connection lost, reconnecting");
tracing::error!(err = %e, "fatal codec error");
```

---

## Implementation Phases

| Phase | Crate(s) | Deliverable |
|-------|----------|-------------|
| 1 | `pvxs-core` | Value, TypeCode, FieldDesc, SharedArray, NT types, errors |
| 2 | `pvxs-proto` | Header, all message structs, encode/decode traits, type cache |
| 3 | `pvxs-server/udp`, `pvxs-client/udp` | UDP Beacon + Search (no TCP yet) |
| 4 | `pvxs-server` | TCP listener, Source trait, SharedPV, server-side ops |
| 5 | `pvxs-client` | ClientContext, connection pool, Get/Put/Monitor/RPC |
| 6 | `pvxs-tools` | `pvxget`, `pvxput`, `pvxmon`, `pvxlist` CLIs |
| 7 | Integration | In-process client+server tests; optional interop with C++ pvxs |

---

## Testing Strategy

### Unit Tests (per crate, `#[cfg(test)]`)
- `pvxs-proto`: Round-trip encode→decode for every message type (port `testxcode.cpp`)
- `pvxs-proto`: Variable-length size encoding edge cases (port `testendian.cpp`)
- `pvxs-core`: Value field get/set, type coercion, SharedArray freeze/thaw

### Integration Tests (`#[tokio::test]`)
```rust
// Helper spins up a real in-process server+client pair
async fn test_pair() -> (Server, ClientContext) { ... }

#[tokio::test]
async fn roundtrip_get() {
    let (srv, ctx) = test_pair().await;
    let pv = SharedPV::default();
    pv.open(Value::from(42.0f64));
    srv.add_pv("TEST:VAL", pv.clone()).await;
    let v = ctx.get("TEST:VAL").await.unwrap();
    assert_eq!(v.as_f64(), Some(42.0));
}
```

### Interoperability (optional, manual)
- Run Rust `pvxget` against a C++ `softIocPVX` server
- Run C++ `pvxget` against the Rust server

---

## C++ → Rust Mapping Summary

| C++ | Rust |
|-----|------|
| `libevent2` event loop | `tokio` runtime |
| `std::function` callbacks | `async fn` / `impl Fn` / `mpsc` channels |
| virtual `Source` / `Handler` | `dyn Source` trait object |
| `shared_array<void>` | custom `SharedArray` (type-erased `Arc`) |
| `EvInBuf` / `EvOutBuf` | `bytes::Bytes` / `bytes::BytesMut` |
| `epicsAtomicGet`, mutex | `tokio::sync::Mutex`, `arc_swap::ArcSwap` |
| `ConnBase` class hierarchy | composition + traits |
| EPICS `make` build system | `cargo` workspace |
