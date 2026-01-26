## UtreexoNode Config and Builder

In this section and the next ones within this chapter, we are going to understand some methods implemented generically for `UtreexoNode`, that is, methods implemented for `UtreexoNode` under `T: NodeContext` rather than a specific context type.

While `ChainSelector`, `SyncNode`, and `RunningNode` provide `impl` blocks for `UtreexoNode`, those methods can only be used in a single context. Here, we will explore the functionality shared across all contexts.

### UtreexoNode Builder

Let's start with the builder function, taking a `UtreexoNodeConfig`, the `Chain` backend, a floresta mempool type, optional compact block filters from the `floresta-compact-filters` crate, a kill signal to stop the node, and the address manager backend (that we will see [later in this chapter](ch06-04-address-manager.md)).

```rust
# // Path: floresta-wire/src/p2p_wire/node/mod.rs
#
impl<T, Chain> UtreexoNode<Chain, T>
where
    T: 'static + Default + NodeContext,
    Chain: ChainBackend + 'static,
    WireError: From<Chain::Error>,
{
    pub fn new(
        config: UtreexoNodeConfig,
        chain: Chain,
        mempool: Arc<Mutex<Mempool>>,
        block_filters: Option<Arc<NetworkFilters<FlatFiltersStore>>>,
        kill_signal: Arc<tokio::sync::RwLock<bool>>,
        address_man: AddressMan,
    ) -> Result<Self, WireError> {
        let (node_tx, node_rx) = unbounded_channel();
        let socks5 = config.proxy.map(Socks5StreamBuilder::new);

        let fixed_peer = config
            .fixed_peer
            .as_ref()
            .map(|address| Self::resolve_connect_host(address, Self::get_port(config.network)))
            .transpose()?;

        Ok(UtreexoNode {
            common: NodeCommon {
                // Initialization of many fields :P
                # last_dns_seed_call: Instant::now(),
                # startup_time: Instant::now(),
                # block_sync_avg: FractionAvg::new(0, 0),
                # last_filter: chain.get_block_hash(0).unwrap(),
                # block_filters,
                # inflight: HashMap::new(),
                # inflight_user_requests: HashMap::new(),
                # peer_id_count: 0,
                # peers: HashMap::new(),
                # last_block_request: chain.get_validation_index().expect("Invalid chain"),
                # chain,
                # peer_ids: Vec::new(),
                # peer_by_service: HashMap::new(),
                # mempool,
                # network: config.network,
                # node_rx,
                # node_tx,
                # address_man,
                # last_tip_update: Instant::now(),
                # last_connection: Instant::now(),
                # last_peer_db_dump: Instant::now(),
                # last_broadcast: Instant::now(),
                # last_feeler: Instant::now(),
                # blocks: HashMap::new(),
                # last_get_address_request: Instant::now(),
                # last_send_addresses: Instant::now(),
                # datadir: config.datadir.clone(),
                # max_banscore: config.max_banscore,
                # socks5,
                # fixed_peer,
                # config,
                # kill_signal,
                # added_peers: Vec::new(),
            },
            context: T::default(),
        })
    }
// ...
```

The `UtreexoNodeConfig` type outlines all the customizable options for running a `UtreexoNode`. It specifies essential settings like the network, connection preferences, and resource management options. It also has a `UtreexoNodeConfig::default` implementation. This type is better explained below.

Then, the `Mempool` type that we see in the signature is a simple mempool implementation for broadcasting transactions to the network.

The first line creates an unbounded **multi-producer, single-consumer** (mpsc) channel, allowing multiple tasks in the `UtreexoNode` to send messages (via the sender `node_tx`) to a central task that processes them (via the receiver `node_rx`). If you are not familiar with channels, there's [a section from the Rust book](https://doc.rust-lang.org/book/ch16-02-message-passing.html) that covers them. Here, we use `tokio` channels instead of Rust's standard library channels.

> The `unbounded_channel` has no backpressure, meaning producers can keep sending messages without being slowed down, even if the receiver is behind. While this is convenient, it comes with the risk of excessive memory use if messages accumulate faster than they are processed.

The second line sets up a SOCKS5 proxy connection if a proxy address (`config.proxy`) is provided. SOCKS5 is a protocol that routes network traffic through a proxy server, useful for privacy or bypassing network restrictions. The `Socks5StreamBuilder` is a wrapper for `core::net::SocketAddr`, implemented in _p2p_wire/socks.rs_. If no proxy is configured, the node will connect directly to the network.

```rust
# // Path: floresta-wire/src/p2p_wire/socks.rs
#
impl Socks5StreamBuilder {
    pub fn new(address: SocketAddr) -> Self {
        Self { address }
    }
    // ...
```

Thirdly, we take the `config.fixed_peer`, which is an `Option<String>`, and convert it into an `Option<LocalAddress>` by using the `UtreexoNode::resolve_connect_host` function. `LocalAddress` is the local representation of a peer address in Floresta, which we will explore in the [Address Manager](ch06-04-address-manager.md) section.

Finally, the function returns the `UtreexoNode` with all the `NodeCommon` fields initialized and the default value of the passed `NodeContext`.

### UtreexoNodeConfig

Let's now check what are the customizable options for the node, that is, the `UtreexoNodeConfig` struct.

It starts with the `network` field (i.e. `Bitcoin`, `Testnet`, `Regtest` or `Signet`). Next we find the key options to enable `pow_fraud_proofs` for a very fast node sync, which we learned about in the [PoW Fraud Proofs Sync](ch05-00-advanced-chain-validation-methods.md#pow-fraud-proofs-sync) section from last chapter, and `compact_filters` for lightweight blockchain rescans.

A fixed peer can be specified via `fixed_peer`, and settings like `max_banscore`, `max_outbound`, and `max_inflight` allow fine-tuning of peer management, connection limits, and parallel requests.

Filename: p2p_wire/mod.rs

```rust
# // Path: floresta-wire/src/p2p_wire/mod.rs
#
pub struct UtreexoNodeConfig {
    # /// The blockchain we are in, defaults to Bitcoin. Possible values are Bitcoin,
    # /// Testnet, Regtest and Signet.
    pub network: Network,
    # /// Whether to use PoW fraud proofs. Defaults to false.
    # ///
    # /// PoW fraud proof is a mechanism to skip the verification of the whole blockchain,
    # /// but while also giving a better security than simple SPV.
    pub pow_fraud_proofs: bool,
    # /// Whether to use compact filters. Defaults to false.
    # ///
    # /// Compact filters are useful to rescan the blockchain for a specific address, without
    # /// needing to download the whole chain. It will download ~1GB of filters, and then
    # /// download the blocks that match the filters.
    pub compact_filters: bool,
    # /// Fixed peers to connect to. Defaults to None.
    # ///
    # /// If you want to connect to a specific peer, you can set this to a string with the
    # /// format `ip:port`. For example, `localhost:8333`.
    pub fixed_peer: Option<String>,
    /// If a peer misbehaves, we increase its ban score. If the ban score reaches this value,
    /// we disconnect from the peer. Defaults to 100.
    pub max_banscore: u32,
    /// Maximum number of outbound connections. Defaults to 8.
    pub max_outbound: u32,
    /// Maximum number of inflight requests. Defaults to 10.
    ///
    /// More inflight requests means more memory usage, but also more parallelism.
    pub max_inflight: u32,
    // ...
    # /// Data directory for the node. Defaults to `.floresta-node`.
    # pub datadir: String,
    # /// A SOCKS5 proxy to use. Defaults to None.
    # pub proxy: Option<SocketAddr>,
    # /// If enabled, the node will assume that the provided Utreexo state is valid, and will
    # /// start running from there
    # pub assume_utreexo: Option<AssumeUtreexoValue>,
    # /// If we assumeutreexo or pow_fraud_proof, we can skip the IBD and make our node usable
    # /// faster, with the tradeoff of security. If this is enabled, we will still download the
    # /// blocks in the background, and verify the final Utreexo state. So, the worse case scenario
    # /// is that we are vulnerable to a fraud proof attack for a few hours, but we can spot it
    # /// and react in a couple of hours at most, so the attack window is very small.
    # pub backfill: bool,
    # /// If we are using network-provided block filters, we may not need to download the whole
    # /// chain of filters, as our wallets may not have been created at the beginning of the chain.
    # /// With this option, we can make a rough estimate of the block height we need to start
    # /// and only download the filters from that height.
    # ///
    # /// If the value is negative, it's relative to the current tip. For example, if the current
    # /// tip is at height 1000, and we set this value to -100, we will start downloading filters
    # /// from height 900.
    # pub filter_start_height: Option<i32>,
    # /// The user agent that we will advertise to our peers. Defaults to `floresta:<version>`.
    # pub user_agent: String,
    # /// Whether to allow fallback to v1 transport if v2 connection fails.
    # /// Defaults to true.
    # pub allow_v1_fallback: bool,
    # /// Whether to disable DNS seeds. Defaults to false.
    # pub disable_dns_seeds: bool,
# }
#
# impl Default for UtreexoNodeConfig {
    # fn default() -> Self {
        # UtreexoNodeConfig {
            # disable_dns_seeds: false,
            # network: Network::Bitcoin,
            # pow_fraud_proofs: false,
            # compact_filters: false,
            # fixed_peer: None,
            # max_banscore: 100,
            # max_outbound: 8,
            # max_inflight: 10,
            # datadir: ".floresta-node".to_string(),
            # proxy: None,
            # backfill: false,
            # assume_utreexo: None,
            # filter_start_height: None,
            # user_agent: format!("floresta:{}", env!("CARGO_PKG_VERSION")),
            # allow_v1_fallback: true,
        # }
    # }
# }
```

Additional configurations include `datadir` for specifying the node's data directory and an optional `proxy` for connecting through a SOCKS5 proxy. Then we see the `assume_utreexo` option, which we also explained in the [Trusted UTXO Set Snapshots](ch05-00-advanced-chain-validation-methods.md#trusted-utxo-set-snapshots) section from last chapter.

```rust
# // Path: floresta-wire/src/p2p_wire/mod.rs
#
# pub struct UtreexoNodeConfig {
    # /// The blockchain we are in, defaults to Bitcoin. Possible values are Bitcoin,
    # /// Testnet, Regtest and Signet.
    # pub network: Network,
    # /// Whether to use PoW fraud proofs. Defaults to false.
    # ///
    # /// PoW fraud proof is a mechanism to skip the verification of the whole blockchain,
    # /// but while also giving a better security than simple SPV.
    # pub pow_fraud_proofs: bool,
    # /// Whether to use compact filters. Defaults to false.
    # ///
    # /// Compact filters are useful to rescan the blockchain for a specific address, without
    # /// needing to download the whole chain. It will download ~1GB of filters, and then
    # /// download the blocks that match the filters.
    # pub compact_filters: bool,
    # /// Fixed peers to connect to. Defaults to None.
    # ///
    # /// If you want to connect to a specific peer, you can set this to a string with the
    # /// format `ip:port`. For example, `localhost:8333`.
    # pub fixed_peer: Option<String>,
    # /// If a peer misbehaves, we increase its ban score. If the ban score reaches this value,
    # /// we disconnect from the peer. Defaults to 100.
    # pub max_banscore: u32,
    # /// Maximum number of outbound connections. Defaults to 8.
    # pub max_outbound: u32,
    # /// Maximum number of inflight requests. Defaults to 10.
    # ///
    # /// More inflight requests means more memory usage, but also more parallelism.
    # pub max_inflight: u32,
    # /// Data directory for the node. Defaults to `.floresta-node`.
    // ...
    pub datadir: String,
    # /// A SOCKS5 proxy to use. Defaults to None.
    pub proxy: Option<SocketAddr>,
    # /// If enabled, the node will assume that the provided Utreexo state is valid, and will
    # /// start running from there
    pub assume_utreexo: Option<AssumeUtreexoValue>,
    # /// If we assumeutreexo or pow_fraud_proof, we can skip the IBD and make our node usable
    # /// faster, with the tradeoff of security. If this is enabled, we will still download the
    # /// blocks in the background, and verify the final Utreexo state. So, the worse case scenario
    # /// is that we are vulnerable to a fraud proof attack for a few hours, but we can spot it
    # /// and react in a couple of hours at most, so the attack window is very small.
    pub backfill: bool,
    # /// If we are using network-provided block filters, we may not need to download the whole
    # /// chain of filters, as our wallets may not have been created at the beginning of the chain.
    # /// With this option, we can make a rough estimate of the block height we need to start
    # /// and only download the filters from that height.
    # ///
    # /// If the value is negative, it's relative to the current tip. For example, if the current
    # /// tip is at height 1000, and we set this value to -100, we will start downloading filters
    # /// from height 900.
    pub filter_start_height: Option<i32>,
    /// The user agent that we will advertise to our peers. Defaults to `floresta:<version>`.
    pub user_agent: String,
    /// Whether to allow fallback to v1 transport if v2 connection fails.
    /// Defaults to true.
    pub allow_v1_fallback: bool,
    /// Whether to disable DNS seeds. Defaults to false.
    pub disable_dns_seeds: bool,
}
#
# impl Default for UtreexoNodeConfig {
    # fn default() -> Self {
        # UtreexoNodeConfig {
            # disable_dns_seeds: false,
            # network: Network::Bitcoin,
            # pow_fraud_proofs: false,
            # compact_filters: false,
            # fixed_peer: None,
            # max_banscore: 100,
            # max_outbound: 8,
            # max_inflight: 10,
            # datadir: ".floresta-node".to_string(),
            # proxy: None,
            # backfill: false,
            # assume_utreexo: None,
            # filter_start_height: None,
            # user_agent: format!("floresta:{}", env!("CARGO_PKG_VERSION")),
            # allow_v1_fallback: true,
        # }
    # }
# }
```

If one of `pow_fraud_proofs` or `assume_utreexo` is set, the `backfill` option enables background and full validation of the chain, which is recommended for security since the node skipped the IBD.

For block filters, `filter_start_height` helps optimize downloads by starting from a specific height rather than the chain’s beginning. In relation to networking, `user_agent` allows nodes to customize their identifier when advertising to peers. Also, `allow_v1_fallback` is used for specifying if our node should use the version 1 P2P transport protocol as a fallback if we encounter a problem when connecting via the [version 2 transport](https://github.com/bitcoin/bips/blob/master/bip-0324.mediawiki).

Lastly, if `disable_dns_seeds` is set, our node will not perform any DNS seed lookups to find peers. Instead, it will use hardcoded peer addresses, found at the `floresta-wire` [seeds subdirectory](https://github.com/getfloresta/Floresta/tree/master/crates/floresta-wire/src/p2p_wire/seeds).

This flexible configuration ensures adaptability for various use cases and security levels, from development to production.
