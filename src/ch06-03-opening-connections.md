## Opening Connections

In this section we are finally going to understand how Floresta connects to peers. The highest-level method for opening connections is `maybe_open_connection`, which will redirect us to other lower-level functionality. Remember that these methods are context-independent.

### When to Open Connections

The `maybe_open_connection` method determines whether the node should establish a new connection to a peer and, if so, calls `create_connection`.

```rust
# // Path: floresta-wire/src/p2p_wire/node/conn.rs
#
pub(crate) fn maybe_open_connection(
    &mut self,
    required_service: ServiceFlags,
) -> Result<(), WireError> {
    // Try to connect with manually added peers
    self.maybe_open_connection_with_added_peers()?;
    if self.connected_peers() >= T::MAX_OUTGOING_PEERS {
        return Ok(());
    }

    let connection_kind = ConnectionKind::Regular(required_service);

    // If the user passes in a `--connect` cli argument, we only connect with
    // that particular peer.
    if self.fixed_peer.is_some() {
        if self.peers.is_empty() {
            self.create_connection(connection_kind)?;
        }
        return Ok(());
    }

    // If we've tried getting some connections, but the addresses we have are not
    // working. Try getting some more addresses from DNS
    self.maybe_ask_dns_seed_for_addresses();
    let needs_utreexo = required_service.has(service_flags::UTREEXO.into());
    self.maybe_use_hardcoded_addresses(needs_utreexo);

    for _ in 0..T::NEW_CONNECTIONS_BATCH_SIZE {
        // Ignore the error so we don't break out of the loop
        let _ = self.create_connection(connection_kind);
    }

    Ok(())
}
```

If the user has specified a fixed peer via the `--connect` command-line argument (`self.fixed_peer.is_some()`), the method tries to establish that fixed connection if needed and returns.

Else, if we didn't specify a fixed peer and the number of peers is below the maximum allowed (`self.connected_peers() < T::MAX_OUTGOING_PEERS`), we call the `create_connection` method a few times to establish new 'regular' connections.

The `ConnectionKind` struct that `create_connection` takes as argument is shown below.

### Connection Kinds

```rust
# // Path: floresta-wire/src/p2p_wire/node/mod.rs
#
/// The kind of connection we see this peer as.
///
/// Core's counterpart: <https://github.com/bitcoin/bitcoin/blob/bf9ef4f0433551e850a11c2da8baae0ec6439a99/src/node/connection_types.h#L18>.
pub enum ConnectionKind {
    /// A feeler connection is a short-lived connection used to check whether this peer is alive.
    ///
    /// After handshake, we ask for addresses and when we receive an answer we just disconnect,
    /// marking this peer as alive in our address manager.
    Feeler,

    /// A regular peer, used to send requests to and learn about transactions and blocks.
    Regular(ServiceFlags),

    /// An extra peer specially created if our tip hasn't moved for too long.
    ///
    /// If more than [`NodeContext::ASSUME_STALE`] seconds have passed since the
    /// last processed block, we use this to make sure we are not in a partitioned subnet,
    /// unable to learn about new blocks.
    Extra,

    /// A connection that was manually requested by our user. This type of peer won't be banned on
    /// misbehaving, and won't respect the [`ServiceFlags`] requirements when creating a
    /// connection.
    Manual,
}
```

The only two long-lived connection kinds are `Regular`, for peers that satisfy certain service flags (e.g., support for Utreexo or compact block filters), and `Manual`, which are added by the user via `--connect` (fixed peer) or `--addnode`. These two connection kinds handle the bulk of the node operations, such as exchanging blocks, headers, transactions, and keeping the node in sync.

`Feeler` and `Extra` connections, on the other hand, are short-lived connection kinds. `Feeler` connections serve as a probe to verify if a peer is still alive, and also to ask for new peer addresses. `Extra` peers are used in case our tip hasn't changed in a while, to temporarily increase our peer count and try to find new blocks. We create `Feeler` connections every few seconds, while `Extra` connections are less common and reserved for potential stale tip situations.

### Create Connection

`create_connection` takes the required services for the connection kind, gets a peer address (prioritizing the fixed peer if specified), and ensures the peer isn't already connected.

If no fixed peer is specified, we get a suitable peer address (or `LocalAddress`) for connection by calling `self.address_man.get_address_to_connect`. This method takes the required services and a boolean indicating whether a feeler connection is desired. We will explore this method in the next section.

```rust
# // Path: floresta-wire/src/p2p_wire/node/conn.rs
#
pub(crate) fn create_connection(&mut self, mut kind: ConnectionKind) -> Result<(), WireError> {
    let is_fixed = self.fixed_peer.is_some();

    // Connection with fixed peers should be marked as `manual`, rather than `regular`
    if is_fixed && matches!(kind, ConnectionKind::Regular(_)) {
        kind = ConnectionKind::Manual;
    }

    let required_services = match kind {
        ConnectionKind::Regular(services) => services,
        _ => ServiceFlags::NONE,
    };

    let address = self
        .fixed_peer
        .as_ref()
        .map(|addr| (0, addr.clone()))
        .or_else(|| {
            self.address_man.get_address_to_connect(
                required_services,
                matches!(kind, ConnectionKind::Feeler),
            )
        });

    let Some((peer_id, address)) = address else {
        // No peers with the desired services are known, load hardcoded addresses
        let net = self.network;
        self.address_man.add_fixed_addresses(net);

        return Err(WireError::NoAddressesAvailable);
    };
    #
    # debug!("attempting connection with address={address:?} kind={kind:?}",);

    let now = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap()
        .as_secs();

    // Defaults to failed, if the connection is successful, we'll update the state
    self.address_man
        .update_set_state(peer_id, AddressState::Failed(now));

    // Don't connect to the same peer twice
    let is_connected = |(_, peer_addr): (_, &LocalPeerView)| {
        peer_addr.address == address.get_net_address() && peer_addr.port == address.get_port()
    };

    if self.peers.iter().any(is_connected) {
        return Err(WireError::PeerAlreadyExists(
            address.get_net_address(),
            address.get_port(),
        ));
    }

    // We allow V1 fallback only if the cli option was set, it's a --connect peer
    // or if we are connecting to a utreexo peer, since utreexod doesn't support V2 yet.
    let allow_v1 = self.config.allow_v1_fallback
        || kind == ConnectionKind::Regular(UTREEXO.into())
        || is_fixed;

    self.open_connection(kind, peer_id, address, allow_v1)?;

    Ok(())
}
```

Then we use the obtained `LocalAddress` and the peer identifier as arguments for `open_connection`, as well as the connection kind and whether we should fall back to the V1 P2P protocol if the V2 handshake fails.

Both the `LocalAddress` type and the `get_address_to_connect` method are implemented in the address manager module (_p2p_wire/address_man.rs_) that we will see in the next section.

### Open Connection

Moving on to `open_connection`, we create a new `unbounded_channel` for sending requests to the `Peer` instance. Recall that the `Peer` component is in charge of actually connecting to the respective peer over the network.

Then, depending on the value of `self.socks5` we will call `UtreexoNode::open_proxy_connection` or `UtreexoNode::open_non_proxy_connection`. Each one of these functions will create a `Peer` instance with the provided data and the channel receiver.

```rust
# // Path: floresta-wire/src/p2p_wire/node/conn.rs
#
pub(crate) fn open_connection(
    &mut self,
    kind: ConnectionKind,
    peer_id: usize,
    address: LocalAddress,
    allow_v1_fallback: bool,
) -> Result<(), WireError> {
    let (requests_tx, requests_rx) = unbounded_channel();
    if let Some(ref proxy) = self.socks5 {
        spawn(timeout(
            Duration::from_secs(10),
            Self::open_proxy_connection(
                // Arguments omitted for brevity :P
                # proxy.address,
                # kind,
                # self.mempool.clone(),
                # self.network,
                # self.node_tx.clone(),
                # peer_id,
                # address.clone(),
                # requests_rx,
                # self.peer_id_count,
                # self.config.user_agent.clone(),
                # allow_v1_fallback,
            ),
        ));
    } else {
        spawn(timeout(
            Duration::from_secs(10),
            Self::open_non_proxy_connection(
                // Arguments omitted for brevity :P
                # kind,
                # peer_id,
                # address.clone(),
                # requests_rx,
                # self.peer_id_count,
                # self.mempool.clone(),
                # self.network,
                # self.node_tx.clone(),
                # self.config.user_agent.clone(),
                # allow_v1_fallback,
            ),
        ));
    }

    let peer_count: u32 = self.peer_id_count;

    self.inflight.insert(
        InflightRequests::Connect(peer_count),
        (peer_count, Instant::now()),
    );

    self.peers.insert(
        peer_count,
        LocalPeerView {
            // Fields omitted for brevity :P
            # message_times: Ema::with_half_life_50(),
            # address: address.get_net_address(),
            # port: address.get_port(),
            # user_agent: "".to_string(),
            # state: PeerStatus::Awaiting,
            # channel: requests_tx,
            # services: ServiceFlags::NONE,
            # _last_message: Instant::now(),
            # kind,
            # address_id: peer_id as u32,
            # height: 0,
            # banscore: 0,
            # // Will be downgraded to V1 if the V2 handshake fails, and we allow fallback
            # transport_protocol: TransportProtocol::V2,
        },
    );

    match kind {
        ConnectionKind::Feeler => self.last_feeler = Instant::now(),
        ConnectionKind::Regular(_) => self.last_connection = Instant::now(),
        _ => {}
    }
    self.peer_id_count += 1;

    Ok(())
}
```

Last of all, we simply insert the new inflight request (via the `InflightRequests` type) to our tracker `HashMap`, as well as the new peer (via the `LocalPeerView`). Both types are defined in _p2p_wire/node/mod.rs_, along with `UtreexoNode`, `NodeCommon`, `ConnectionKind`, and a few other types.

### Recap

In this section, we have learned how Floresta establishes peer-to-peer connections, starting with the `maybe_open_connection` method. This method initiates a connection if we aren't already connected to the optional fixed peer and either have fewer connections than `Context::MAX_OUTGOING_PEERS` or lack a peer offering utreexo services.

We explored the three connection types: `Feeler` (peer availability check), `Regular` (core communication), and `Extra` (specialized services). The `create_connection` method selects an appropriate peer address while preventing duplicate connections, and `open_connection` handles the network setup, either via a proxy or directly (internally creating a new `Peer`). Finally, we examined how new connections are tracked using inflight requests and a peer registry, both fields of `NodeCommon`.
