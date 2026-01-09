## Floresta-CLI

`floresta-cli` is a thin CLI that talks to `florestad`'s JSON-RPC server. It uses a simple JSON-RPC client that can call any method defined by the `FlorestaRPC` trait from `floresta-rpc`.

### Floresta-rpc Traits

The `floresta-rpc` library defines two traits: `FlorestaRPC` and `JsonRPCClient`.

- `FlorestaRPC` lists all methods exposed by the `florestad` JSON-RPC server.
- `JsonRPCClient` is implemented by clients, requiring only a single `call` method.

```rust
# // Path: floresta-rpc/src/rpc.rs
#
/// A trait specifying all possible methods for floresta's json-rpc
pub trait FlorestaRPC {
    /// Get the BIP158 filter for a given block height
    ///
    /// BIP158 filters are a compact representation of the set of transactions in a block,
    /// designed for efficient light client synchronization. This method returns the filter
    /// for a given block height, encoded as a hexadecimal string.
    /// You need to have enabled block filters by setting the `blockfilters=1` option
    fn get_block_filter(&self, height: u32) -> Result<String>;
    // ...
```

Any type that implements `JsonRPCClient` will automatically implement `FlorestaRPC`, which makes this RPC interface available to any client implementation.

Many of the `FlorestaRPC` methods mirror Bitcoin Core's RPC interface, but with changes that reflect our pruned and utreexo-based design. There are also utreexo-specific methods like `get_roots`. You can list all available RPC methods by running:

```bash
cargo run --bin floresta-cli -- --help
```

### JSON-RPC Client

`floresta-cli` uses a simple client that is also defined in `floresta-rpc`, available via the default `with-jsonrpc` feature. It is implemented as a simple wrapper around the `jsonrpc` library, which implements the mentioned `JsonRPCClient` trait, and by extension, `FlorestaRPC`.

```rust
# // Path: floresta-rpc/src/jsonrpc_client.rs
#
// Define a Client struct that wraps a jsonrpc::Client
#[derive(Debug)]
pub struct Client(jsonrpc::Client);
```

```rust
# // Path: floresta-rpc/src/jsonrpc_client.rs
#
// Implement the JsonRPCClient trait for Client
impl JsonRPCClient for Client {
    fn call<T: for<'a> serde::de::Deserialize<'a> + Debug>(
        &self,
        method: &str,
        params: &[serde_json::Value],
    ) -> Result<T, crate::rpc_types::Error> {
        self.rpc_call(method, params)
    }
}
```

### Usage

To use `floresta-cli`, you first need to start `florestad` with the `json-rpc` feature enabled (already enabled by default). Once the daemon is running, you can issue commands like:

```bash
# Rescan from height 100 to 200
floresta-cli rescanblockchain 100 200

# Get the current blockchain info
floresta-cli getblockchaininfo
```

For more details about the `floresta-cli` usage and RPC commands, you can check the [Floresta doc folder](https://github.com/getfloresta/Floresta/blob/master/doc).
