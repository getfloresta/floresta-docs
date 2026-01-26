# Peer-to-Peer Networking

In the previous chapter, we learned how `UtreexoNode` opens connections, although we didn't dive into the low-level networking details. We mentioned that each peer connection is handled by the `Peer` type, keeping the peer networking logic separate from `UtreexoNode`.

In this chapter, we will explore the details of `Peer` operations, beginning with the low-level logic for opening connections (that is, the `Peer` creation).

## Peer Creation

Recall that in [the `open_connection` method](ch06-03-opening-connections.md#open-connection) on `UtreexoNode` we call either `UtreexoNode::open_proxy_connection` or `UtreexoNode::open_non_proxy_connection`, depending on the `self.socks5` proxy option. It's within these two functions that the `Peer` is created. Let's first learn how the direct TCP connection is opened!

The `open_non_proxy_connection` function will first retrieve the peer's network address and port from the provided `LocalAddress`. Then it will attempt to establish a TCP connection using `transport::connect` (implemented in the `transport` module, which handles both the v1 and v2 transport protocols).

If successful, we get the transport reader and writer, which are of type `ReadTransport` and `WriteTransport`, defined in _p2p_wire/transport.rs_. Respectively, these two types wrap a `tokio` `ReadHalf<TcpStream>`, for receiving data, and a `WriteHalf<TcpStream>`, for sending data to the peer. We also get the transport protocol enum, with two possible variants, `V1` or `V2`.

It then sets up an 'actor', that is, an independent component that reads incoming messages and communicates them to the 'actor receiver'. The actor is effectively a transport reader wrapper.

```rust
# // Path: floresta-wire/src/p2p_wire/node/conn.rs
#
pub(crate) async fn open_non_proxy_connection(
    kind: ConnectionKind,
    peer_id: usize,
    address: LocalAddress,
    requests_rx: UnboundedReceiver<NodeRequest>,
    peer_id_count: u32,
    mempool: Arc<Mutex<Mempool>>,
    network: Network,
    node_tx: UnboundedSender<NodeNotification>,
    user_agent: String,
    allow_v1_fallback: bool,
) -> Result<(), WireError> {
    let address = (address.get_net_address(), address.get_port());

    let (transport_reader, transport_writer, transport_protocol) =
        transport::connect(address, network, allow_v1_fallback).await?;

    let (cancellation_sender, cancellation_receiver) = oneshot::channel();
    let (actor_receiver, actor) = create_actors(transport_reader);
    tokio::spawn(async move {
        tokio::select! {
            _ = cancellation_receiver => {}
            _ = actor.run() => {}
        }
    });

    // Use create_peer function instead of manually creating the peer
    Peer::<WriteHalf>::create_peer(
        peer_id_count,
        mempool,
        node_tx.clone(),
        requests_rx,
        peer_id,
        kind,
        actor_receiver,
        transport_writer,
        user_agent,
        cancellation_sender,
        transport_protocol,
    );

    Ok(())
}
```

This actor is obtained via the `create_actors` function, implemented in _p2p_wire/peer.rs_, and is of type `MessageActor`. **The actor is spawned as a separate asynchronous task**, ensuring it runs independently to handle incoming data.

Very importantly, the actor for a peer must be closed when the connection finalizes, and this is why we have an additional one-time-use channel, used by the `Peer` type to send a cancellation signal (essentially saying "_the peer connection is closed, so there’s nothing left to listen to_"). The `tokio::select` macro ensures that the async actor task is dropped whenever a cancellation signal is received from `Peer`.

Finally, the `Peer` instance is created using the `Peer::create_peer` function. The communication channels (internal and over the P2P network) that the `Peer` uses are:

- The node sender (`node_tx`): to send messages to `UtreexoNode`.
- The requests receiver (`requests_rx`): to receive requests from `UtreexoNode` that will be sent to the peer.
- The `actor_receiver`: to receive peer messages.
- The `transport_writer`: to send messages to the peer.
- The `cancellation_sender`: to close the reader actor task.

By the end of this function, a fully initialized `Peer` is ready to manage communication with the connected peer via TCP (writing side) and via `MessageActor` (reading side), as well as communicating with `UtreexoNode`.

### Proxy Connection

The `open_proxy_connection` is pretty much the same, except we get the transport reader and writer from the proxy connection instead, handled by `transport::connect_proxy`.

```rust
# // Path: floresta-wire/src/p2p_wire/node/conn.rs
#
pub(crate) async fn open_proxy_connection(
    proxy: SocketAddr,
    // ...
    # kind: ConnectionKind,
    # mempool: Arc<Mutex<Mempool>>,
    # network: Network,
    # node_tx: UnboundedSender<NodeNotification>,
    # peer_id: usize,
    # address: LocalAddress,
    # requests_rx: UnboundedReceiver<NodeRequest>,
    # peer_id_count: u32,
    # user_agent: String,
    # allow_v1_fallback: bool,
) -> Result<(), WireError> {
    let (transport_reader, transport_writer, transport_protocol) =
        transport::connect_proxy(proxy, address, network, allow_v1_fallback).await?;

    let (cancellation_sender, cancellation_receiver) = oneshot::channel();
    let (actor_receiver, actor) = create_actors(transport_reader);
    tokio::spawn(async move {
        tokio::select! {
            _ = cancellation_receiver => {}
            _ = actor.run() => {}
        }
    });

    Peer::<WriteHalf>::create_peer(
        // Same as before
        # peer_id_count,
        # mempool,
        # node_tx,
        # requests_rx,
        # peer_id,
        # kind,
        # actor_receiver,
        # transport_writer,
        # user_agent,
        # cancellation_sender,
        # transport_protocol,
    );

    Ok(())
}
```

## Recap of Channels

Let's do a brief recap of the channels we have opened for internal node message passing:

- **Node Channel** (`Peer` -> `UtreexoNode`)
  - `Peer` sends via `node_tx`
  - `UtreexoNode` receives via `NodeCommon.node_rx`

- **Requests Channel** (`UtreexoNode` -> `Peer`)
  - `UtreexoNode` sends via each `LocalPeerView.channel`, stored in `NodeCommon.peers`
  - `Peer` receives via its `requests_rx`

- **Message Actor Channel** (`MessageActor` -> `Peer`)
  - `MessageActor` sends via `actor_sender`
  - `Peer` receives via `actor_receiver`

- **Cancellation Signal Channel** (`Peer` -> `UtreexoNode`)
  - `Peer` sends the signal via `cancellation_sender` at the end of the connection
  - `UtreexoNode` receives it via `cancellation_receiver`

`UtreexoNode` sends requests via the **Request Channel** to the `Peer` component (which then forwards them to the peer via TCP), `Peer` receives the result or other peer messages via the **Actor Channel**, and then it notifies `UtreexoNode` via the **Node Channel**. When the peer connection is closed, `Peer` uses the **Cancellation Signal Channel** to allow the TCP actor listening to the peer to be closed as well.

Next, we'll explore how messages are read and sent in the P2P network!
