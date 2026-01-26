## Node Contexts

A Bitcoin client goes through different phases during its lifetime, each with distinct responsibilities and requirements. To manage these phases, Floresta uses node contexts, which encapsulate the specific logic and behavior needed for each phase.

Instead of creating a monolithic `UtreexoNode` struct with all possible logic and conditional branches, the design separates shared functionality and phase-specific logic. The base `UtreexoNode` struct handles common features, while contexts, passed as generic parameters implementing the `NodeContext` trait, handle the specific behavior for each phase. This modular design simplifies the codebase and makes it more maintainable.

### Default Floresta Contexts

The three node contexts in Floresta are:

1. **ChainSelector**: This context identifies the best proof-of-work (PoW) chain by downloading and evaluating multiple candidates. It quickly determines the chain with the highest PoW, as further client operations depend on this selection.

2. **SyncNode**: Responsible for downloading and verifying all blocks in the selected chain, this context ensures the chain's validity. Although it is computationally expensive and time-consuming, it guarantees that the chain is fully validated.

3. **RunningNode**: The primary context during normal operation, it starts operating after `ChainSelector` finishes. This context processes new blocks (even if `SyncNode` is still running) and handles user requests.

`RunningNode` is the top-level context. When a `UtreexoNode` is first created, it will internally switch to the other contexts as needed, and then return to `RunningNode`. In practice, this makes `RunningNode` the default context used by `florestad` to run the node.

### The NodeContext Trait

The following is the `NodeContext` trait definition, which holds many useful node constants and provides one method. Click _see more_ in the snippet if you would like to see all the comments for each `const`.

Filename: p2p_wire/node_context.rs

```rust
# // Path: floresta-wire/src/p2p_wire/node_context.rs
#
use bitcoin::p2p::ServiceFlags;

pub trait NodeContext {
    # /// How long we wait for a peer to respond to our request
    const REQUEST_TIMEOUT: u64;
    # /// Max number of simultaneous connections we initiates we are willing to hold
    const MAX_OUTGOING_PEERS: usize = 10;
    # /// We ask for peers every ASK_FOR_PEERS_INTERVAL seconds
    const ASK_FOR_PEERS_INTERVAL: u64 = 60 * 60; // One hour
    # /// Save our database of peers every PEER_DB_DUMP_INTERVAL seconds
    const PEER_DB_DUMP_INTERVAL: u64 = 30; // 30 seconds
    # /// Attempt to open a new connection (if needed) every TRY_NEW_CONNECTION seconds
    const TRY_NEW_CONNECTION: u64 = 10; // 10 seconds
    # /// If ASSUME_STALE seconds passed since our last tip update, treat it as stale
    const ASSUME_STALE: u64 = 15 * 60; // 15 minutes
    # /// While on IBD, if we've been without blocks for this long, ask for headers again
    const IBD_REQUEST_BLOCKS_AGAIN: u64 = 30; // 30 seconds
    # /// How often we broadcast transactions
    const BROADCAST_DELAY: u64 = 30; // 30 seconds
    # /// Max number of simultaneous inflight requests we allow
    const MAX_INFLIGHT_REQUESTS: usize = 1_000;
    # /// Interval at which we open new feeler connections
    const FEELER_INTERVAL: u64 = 30; // 30 seconds
    # /// Interval at which we rearrange our addresses
    const ADDRESS_REARRANGE_INTERVAL: u64 = 60 * 60; // 1 hour
    # /// How long we ban a peer for
    const BAN_TIME: u64 = 60 * 60 * 24;
    # /// How often we check if we haven't missed a block
    const BLOCK_CHECK_INTERVAL: u64 = 60 * 5; // 5 minutes
    # /// How often we send our addresses to our peers
    const SEND_ADDRESSES_INTERVAL: u64 = 60 * 60; // 1 hour
    # /// How long should we wait for a peer to respond our connection request
    const CONNECTION_TIMEOUT: u64 = 30; // 30 seconds
    # /// How many blocks we can ask in the same request
    const BLOCKS_PER_GETDATA: usize = 5;
    # /// How many concurrent GETDATA packages we can send at the same time
    const MAX_CONCURRENT_GETDATA: usize = 10;
    # /// How often we perform the main loop maintenance tasks (checking for timeouts, peers, etc.)
    const MAINTENANCE_TICK: Duration = Duration::from_secs(1);

    fn get_required_services(&self) -> ServiceFlags {
        ServiceFlags::NETWORK
    }
}

pub(crate) type PeerId = u32;
```

The `get_required_services` implementation defaults to returning `ServiceFlags::NETWORK`, meaning the node is capable of serving the complete blockchain to peers on the network. This is a placeholder implementation, and as we’ll see, each context will provide its own `ServiceFlags` based on its role. Similarly, the constants can be customized when implementing `NodeContext` for a specific type.

> `bitcoin::p2p::ServiceFlags` is a fundamental type that represents the various services that nodes in the network can offer. Floresta extends this functionality by defining two additional service flags in `floresta-common`, which can be converted `into` the `ServiceFlags` type.
> 
> ```rust
> # // Path: floresta-common/src/lib.rs
> #
> /// Non-standard service flags that aren't in rust-bitcoin yet
> pub mod service_flags {
>     /// This peer supports UTREEXO messages
>     pub const UTREEXO: u64 = 1 << 24;
> 
>     /// This peer supports UTREEXO filter messages
>     pub const UTREEXO_FILTER: u64 = 1 << 25;
> }
> ```

Finally, we find the `PeerId` type alias that will be used to give each peer a specific identifier. This is all the code in _node_context.rs_.

### Implementation Example

When we implement `NodeContext` for any meaningful type, we also implement the context-specific methods for `UtreexoNode`. Let's suppose we have a `MyContext` type:

```rust
// Example implementation for MyContext

impl NodeContext for MyContext {
    fn get_required_services(&self) -> bitcoin::p2p::ServiceFlags {
        // The node under MyContext can serve the whole blockchain as well
        // as segregated witness (SegWit) data to other peers
        ServiceFlags::WITNESS | ServiceFlags::NETWORK
    }
    const REQUEST_TIMEOUT: u64 = 10 * 60; // 10 minutes
}

// The following `impl` block is only for `UtreexoNode`s that use MyContext
impl<Chain> UtreexoNode<MyContext, Chain>
where
    WireError: From<<Chain as BlockchainInterface>::Error>,
    Chain: BlockchainInterface + UpdatableChainstate + 'static,
{
    // Methods for UtreexoNode in MyContext
}
```

Since `UtreexoNode<RunningNode, _>`, `UtreexoNode<SyncNode, _>`, and `UtreexoNode<ChainSelector, _>` are all entirely different types (because Rust considers each generic parameter combination a separate type), we cannot mistakenly use methods that are not intended for a specific context.

The shared functionality is what is implemented generically for `T: NodeContext`, which all three contexts satisfy (as we have seen [in the previous section](ch06-00-utreexonode-in-depth.md)).

In the next sections within this chapter we will see some of these shared functions and methods implementations.
