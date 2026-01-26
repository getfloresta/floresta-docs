# UtreexoNode In-Depth

We have already explored most of the `floresta-chain` crate! Now, it’s time to dive into the higher-level `UtreexoNode`, first introduced in the [utreexonode](ch01-01-utreexonode.md) section of Chapter 1.

This chapter focuses entirely on the `floresta-wire` crate, so stay tuned!

## Revisiting the UtreexoNode Type

The two fields of `UtreexoNode` are the `NodeCommon<Chain>` struct and the generic `Context`.

`NodeCommon` represents the core state and data required for node operations, independent of the context. It keeps the `Chain` backend, optional compact block filters, the mempool, the `UtreexoNodeConfig` and `Network`, peer connection data, networking configurations, and time-based events to enable effective node behavior and synchronization.

Filename: floresta-wire/src/p2p_wire/node/mod.rs

```rust
# // Path: floresta-wire/src/p2p_wire/node/mod.rs
#
pub struct NodeCommon<Chain: ChainBackend> {
    // 1. Core Blockchain and Transient Data
    pub(crate) chain: Chain,
    pub(crate) blocks: HashMap<BlockHash, InflightBlock>,
    pub(crate) mempool: Arc<tokio::sync::Mutex<Mempool>>,
    pub(crate) block_filters: Option<Arc<NetworkFilters<FlatFiltersStore>>>,
    pub(crate) last_filter: BlockHash,

    // 2. Peer Management
    pub(crate) peer_id_count: u32,
    pub(crate) peer_ids: Vec<u32>,
    pub(crate) peers: HashMap<u32, LocalPeerView>,
    pub(crate) peer_by_service: HashMap<ServiceFlags, Vec<u32>>,
    pub(crate) max_banscore: u32,
    pub(crate) address_man: AddressMan,
    pub(crate) added_peers: Vec<AddedPeerInfo>,

    // 3. Internal Communication
    pub(crate) node_rx: UnboundedReceiver<NodeNotification>,
    pub(crate) node_tx: UnboundedSender<NodeNotification>,

    // 4. Networking Configuration
    pub(crate) socks5: Option<Socks5StreamBuilder>,
    pub(crate) fixed_peer: Option<LocalAddress>,

    // 5. Time and Event Tracking
    pub(crate) inflight: HashMap<InflightRequests, (u32, Instant)>,
    pub(crate) inflight_user_requests:
        HashMap<UserRequest, (u32, Instant, oneshot::Sender<NodeResponse>)>,
    pub(crate) last_tip_update: Instant,
    pub(crate) last_connection: Instant,
    pub(crate) last_peer_db_dump: Instant,
    pub(crate) last_block_request: u32,
    pub(crate) last_get_address_request: Instant,
    pub(crate) last_broadcast: Instant,
    pub(crate) last_send_addresses: Instant,
    pub(crate) block_sync_avg: FractionAvg,
    pub(crate) last_feeler: Instant,
    pub(crate) startup_time: Instant,
    pub(crate) last_dns_seed_call: Instant,

    // 6. Configuration and Metadata
    pub(crate) config: UtreexoNodeConfig,
    pub(crate) datadir: String,
    pub(crate) network: Network,
    pub(crate) kill_signal: Arc<tokio::sync::RwLock<bool>>,
}

pub struct UtreexoNode<Chain: ChainBackend, Context = RunningNode> {
    pub(crate) common: NodeCommon<Chain>,
    pub(crate) context: Context,
}
```

On the other hand, the `Context` generic in `UtreexoNode` will allow the node to implement additional functionality and manage data specific to a particular context. This is explained in the [next section](ch06-01-node-contexts.md).

Although the `Context` generic in the type definition is not constrained by any trait, in the `UtreexoNode` implementation blocks this generic is bound by both a `NodeContext` trait and the `Default` trait.

```rust
# // Path: floresta-wire/src/p2p_wire/node/mod.rs
#
impl<T, Chain> UtreexoNode<Chain, T>
where
    T: 'static + Default + NodeContext,
    Chain: ChainBackend + 'static,
    WireError: From<Chain::Error>,
{
// ...
```

In this implementation block, we also encounter a `WireError`, which serves as the unified error type for methods in `UtreexoNode`. It must implement the `From` trait to convert errors produced by the `Chain` backend via the `BlockchainInterface` trait (i.e., the `Error` associated type of the trait).

> The `WireError` type is defined in the _p2p_wire/error.rs_ file and is the primary error type in `floresta-wire`, alongside `PeerError`, located in _p2p_wire/peer.rs_.

### How to Access Inner Fields

To avoid repetitively calling `self.common.field_name` to access the many inner `NodeCommon` fields, `UtreexoNode` implements the `Deref` and `DerefMut` traits. This means that we can access the `NodeCommon` fields as if they were fields of `UtreexoNode`.

```rust
# // Path: floresta-wire/src/p2p_wire/node/mod.rs
#
impl<Chain: ChainBackend, T> Deref for UtreexoNode<Chain, T> {
    fn deref(&self) -> &Self::Target {
        &self.common
    }
    type Target = NodeCommon<Chain>;
}

impl<T, Chain: ChainBackend> DerefMut for UtreexoNode<Chain, T> {
    fn deref_mut(&mut self) -> &mut Self::Target {
        &mut self.common
    }
}
```

However, the `Context` generic still needs to be accessed explicitly via `self.context`.

## The Role of UtreexoNode

`UtreexoNode` acts as the central task managing critical events like:

- **Receiving new blocks**: Handling block announcements and integrating them into the blockchain.
- **Managing peer connections and disconnections**: Establishing, monitoring, and closing connections with other nodes.
- **Updating peer addresses**: Discovering, maintaining, and sharing network addresses of peers for efficient connectivity.

While the node orchestrates these high-level responsibilities, peer-specific tasks (e.g., handling pings and other peer messages) are delegated to the Floresta `Peer` type, ensuring a clean separation of concerns. The `Peer` component and its networking functionality will be explored in the [next chapter](ch07-00-peer-to-peer-networking.md).

> In networking, a ping is a message sent between peers to check if they are online and responsive, often used to measure latency or maintain active connections.

In the next section we will understand the `NodeContext` trait and the different node `Context`s that Floresta implements.
