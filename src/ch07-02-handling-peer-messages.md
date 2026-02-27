## Handling Peer Messages

As we can see below, the only thing the `Peer::create_peer` method does is initializing the `Peer` and running its `read_loop` method.

```rust
# // Path: floresta-wire/src/p2p_wire/peer.rs
#
pub fn create_peer<W: AsyncWrite + Unpin + Send + Sync + 'static>(
    // ...
    # id: u32,
    # mempool: Arc<Mutex<Mempool>>,
    # node_tx: UnboundedSender<NodeNotification>,
    # node_requests: UnboundedReceiver<NodeRequest>,
    # address_id: usize,
    # kind: ConnectionKind,
    # actor_receiver: UnboundedReceiver<ReaderMessage>,
    # writer: WriteTransport<W>,
    # our_user_agent: String,
    # cancellation_sender: tokio::sync::oneshot::Sender<()>,
    # transport_protocol: TransportProtocol,
) {
    let peer = Peer {
        // Initializing the many Peer fields :P
        # address_id,
        # blocks_only: false,
        # current_best_block: -1,
        # id,
        # mempool,
        # last_ping: None,
        # last_message: Instant::now(),
        # last_addrv2: Instant::now() - Duration::from_secs(ADDRV2_MESSAGE_INTERVAL_SECS),
        # node_tx,
        # services: ServiceFlags::NONE,
        # messages: 0,
        # start_time: Instant::now(),
        # user_agent: "".into(),
        # state: State::None,
        # send_headers: false,
        # node_requests,
        # kind,
        # wants_addrv2: false,
        # shutdown: false,
        # actor_receiver, // Add the receiver for messages from TcpStreamActor
        # writer,
        # our_user_agent,
        # cancellation_sender,
        # transport_protocol,
    };

    spawn(peer.read_loop());
}
```

This `read_loop` method will in turn call a `peer_loop_inner` method:

```rust
# // Path: floresta-wire/src/p2p_wire/peer.rs
#
pub async fn read_loop(mut self) -> Result<()> {
    let err = self.peer_loop_inner().await;
    // Check any errors returned by the loop, shutdown the stream writer
    // and send cancellation signal to close the stream reader (actor task)
    // ...
    # if err.is_err() {
        # debug!("Peer {} connection loop closed: {err:?}", self.id);
    # }
    # let now = Instant::now();
    # self.send_to_node(PeerMessages::Disconnected(self.address_id), now);
    # // force the stream to shutdown to prevent leaking resources
    # if let Err(shutdown_err) = self.writer.shutdown().await {
        # debug!(
            # "Failed to shutdown writer for Peer {}: {shutdown_err:?}",
            # self.id
        # );
    # }
    #
    # if let Err(cancellation_err) = self.cancellation_sender.send(()) {
        # debug!(
            # "Failed to propagate cancellation signal for Peer {}: {cancellation_err:?}",
            # self.id
        # );
    # }
    #
    # if let Err(err) = err {
        # debug!("Peer {} connection loop closed: {err:?}", self.id);
    # }
    #
    # Ok(())
}
```

### The Peer Loop

The `peer_loop_inner` method is the main loop execution that handles all communication between the `Peer` component, the actual peer over the network, and the `UtreexoNode`. It sends P2P messages to the peer, processes requests from the node, and manages responses from the peer.

1. **Initial Handshake and Main Loop**: At the start, the method sends a version message to the peer using `peer_utils::build_version_message`, which initiates the handshake. Then the method enters an asynchronous loop where it handles node requests, peer messages, and ensures the peer connection remains healthy.

```rust
# // Path: floresta-wire/src/p2p_wire/peer.rs
#
async fn peer_loop_inner(&mut self) -> Result<()> {
    // send a version
    let version = peer_utils::build_version_message(self.our_user_agent.clone());
    self.write(version).await?;
    self.state = State::SentVersion(Instant::now());
    loop {
        tokio::select! {
            request = tokio::time::timeout(Duration::from_secs(2), self.node_requests.recv()) => {
                match request {
                    Ok(None) => {
                        return Err(PeerError::Channel);
                    },
                    Ok(Some(request)) => {
                        self.handle_node_request(request).await?;
                    },
                    Err(_) => {
                        // Timeout, do nothing
                    }
                }
            },
            message = self.actor_receiver.recv() => {
                match message {
                    None => {
                        return Err(PeerError::Channel);
                    }
                    Some(ReaderMessage::Error(e)) => {
                        return Err(e);
                    }
                    Some(ReaderMessage::Message(msg, time)) => {
                        self.handle_peer_message(msg, time).await?;
                    }
                }
            }
        }
        // ...
        #
        # if self.shutdown {
            # return Ok(());
        # }
        #
        # // If we send a ping and our peer doesn't respond in time, disconnect
        # if let Some(when) = self.last_ping {
            # if when.elapsed().as_secs() > PING_TIMEOUT {
                # return Err(PeerError::PingTimeout);
            # }
        # }
        #
        # // Send a ping to check if this peer is still good
        # let last_message = self.last_message.elapsed().as_secs();
        # if last_message > SEND_PING_TIMEOUT {
            # if self.last_ping.is_some() {
                # continue;
            # }
            # let nonce = rand::random();
            # self.last_ping = Some(Instant::now());
            # self.write(NetworkMessage::Ping(nonce)).await?;
        # }
        #
        # // divide the number of messages by the number of seconds we've been connected,
        # // if it's more than 10 msg/sec, this peer is sending us too many messages, and we should
        # // disconnect.
        # let msg_sec = self
            # .messages
            # .checked_div(Instant::now().duration_since(self.start_time).as_secs())
            # .unwrap_or(0);
        #
        # if msg_sec > 10 {
            # error!(
                # "Peer {} is sending us too many messages, disconnecting",
                # self.id
            # );
            # return Err(PeerError::TooManyMessages);
        # }
        #
        # if let State::SentVersion(when) = self.state {
            # if Instant::now().duration_since(when) > Duration::from_secs(10) {
                # return Err(PeerError::UnexpectedMessage);
            # }
        # }
    # }
}
```

2. **Handling Node Requests**: The method uses a `futures::select!` block to listen for requests from `UtreexoNode` via `self.node_requests`, with a 10-second timeout for each operation.
    - If a request is received, it is passed to the `handle_node_request` method for processing.
    - If the channel is closed (`Ok(None)`), the method exits with a `PeerError::Channel`.
    - If the timeout expires without receiving a request, the method simply does nothing, allowing the loop to continue.

3. **Processing Peer Messages**: Simultaneously, the loop listens for messages from the TCP actor via `self.actor_receiver`. Depending on the type of message received:
    - Error: If an error is reported (closed channel or `ReaderMessage::Error`), the loop exits with the error.
    - Message: Peer messages are processed by the `handle_peer_message` method.

```rust
# // Path: floresta-wire/src/p2p_wire/peer.rs
#
async fn peer_loop_inner(&mut self) -> Result<()> {
    # // send a version
    # let version = peer_utils::build_version_message(self.our_user_agent.clone());
    # self.write(version).await?;
    # self.state = State::SentVersion(Instant::now());
    # loop {
        # tokio::select! {
            # request = tokio::time::timeout(Duration::from_secs(2), self.node_requests.recv()) => {
                # match request {
                    # Ok(None) => {
                        # return Err(PeerError::Channel);
                    # },
                    # Ok(Some(request)) => {
                        # self.handle_node_request(request).await?;
                    # },
                    # Err(_) => {
                        # // Timeout, do nothing
                    # }
                # }
            # },
            # message = self.actor_receiver.recv() => {
                # match message {
                    # None => {
                        # return Err(PeerError::Channel);
                    # }
                    # Some(ReaderMessage::Error(e)) => {
                        # return Err(e);
                    # }
                    # Some(ReaderMessage::Message(msg, time)) => {
                        # self.handle_peer_message(msg, time).await?;
                    # }
                # }
            # }
        # }
        // ...
        if self.shutdown {
            return Ok(());
        }

        // If we send a ping and our peer doesn't respond in time, disconnect
        if let Some(when) = self.last_ping {
            if when.elapsed().as_secs() > PING_TIMEOUT {
                return Err(PeerError::PingTimeout);
            }
        }

        // Send a ping to check if this peer is still good
        let last_message = self.last_message.elapsed().as_secs();
        if last_message > SEND_PING_TIMEOUT {
            if self.last_ping.is_some() {
                continue;
            }
            let nonce = rand::random();
            self.last_ping = Some(Instant::now());
            self.write(NetworkMessage::Ping(nonce)).await?;
        }
        // ...
        #
        # // divide the number of messages by the number of seconds we've been connected,
        # // if it's more than 10 msg/sec, this peer is sending us too many messages, and we should
        # // disconnect.
        # let msg_sec = self
            # .messages
            # .checked_div(Instant::now().duration_since(self.start_time).as_secs())
            # .unwrap_or(0);
        #
        # if msg_sec > 10 {
            # error!(
                # "Peer {} is sending us too many messages, disconnecting",
                # self.id
            # );
            # return Err(PeerError::TooManyMessages);
        # }
        #
        # if let State::SentVersion(when) = self.state {
            # if Instant::now().duration_since(when) > Duration::from_secs(10) {
                # return Err(PeerError::UnexpectedMessage);
            # }
        # }
    # }
}
```

4. **Shutdown Check**: The loop continually checks if the `shutdown` flag is set. If it is, the loop exits gracefully.

5. **Ping Management**: To maintain the connection, the method sends periodic `NetworkMessage::Ping`s. If the peer fails to respond within a timeout (`PING_TIMEOUT`), the connection is terminated. Additionally, if no messages have been exchanged for a period (`SEND_PING_TIMEOUT`), a new ping is sent, and the timestamp is updated.

> Currently, we disconnect if a peer doesn't respond to a ping within 30 seconds, and we send a ping 60 seconds after the last message.
> 
> ```rust
> # // Path: floresta-wire/src/p2p_wire/peer.rs
> #
> const PING_TIMEOUT: u64 = 30;
> const SEND_PING_TIMEOUT: u64 = 60;
> ```

```rust
# // Path: floresta-wire/src/p2p_wire/peer.rs
#
async fn peer_loop_inner(&mut self) -> Result<()> {
    # // send a version
    # let version = peer_utils::build_version_message(self.our_user_agent.clone());
    # self.write(version).await?;
    # self.state = State::SentVersion(Instant::now());
    # loop {
        # tokio::select! {
            # request = tokio::time::timeout(Duration::from_secs(2), self.node_requests.recv()) => {
                # match request {
                    # Ok(None) => {
                        # return Err(PeerError::Channel);
                    # },
                    # Ok(Some(request)) => {
                        # self.handle_node_request(request).await?;
                    # },
                    # Err(_) => {
                        # // Timeout, do nothing
                    # }
                # }
            # },
            # message = self.actor_receiver.recv() => {
                # match message {
                    # None => {
                        # return Err(PeerError::Channel);
                    # }
                    # Some(ReaderMessage::Error(e)) => {
                        # return Err(e);
                    # }
                    # Some(ReaderMessage::Message(msg, time)) => {
                        # self.handle_peer_message(msg, time).await?;
                    # }
                # }
            # }
        # }
        #
        # if self.shutdown {
            # return Ok(());
        # }
        #
        # // If we send a ping and our peer doesn't respond in time, disconnect
        # if let Some(when) = self.last_ping {
            # if when.elapsed().as_secs() > PING_TIMEOUT {
                # return Err(PeerError::PingTimeout);
            # }
        # }
        #
        # // Send a ping to check if this peer is still good
        # let last_message = self.last_message.elapsed().as_secs();
        # if last_message > SEND_PING_TIMEOUT {
            # if self.last_ping.is_some() {
                # continue;
            # }
            # let nonce = rand::random();
            # self.last_ping = Some(Instant::now());
            # self.write(NetworkMessage::Ping(nonce)).await?;
        # }
        #
        // ...
        // divide the number of messages by the number of seconds we've been connected,
        // if it's more than 10 msg/sec, this peer is sending us too many messages, and we should
        // disconnect.
        let msg_sec = self
            .messages
            .checked_div(Instant::now().duration_since(self.start_time).as_secs())
            .unwrap_or(0);

        if msg_sec > 10 {
            error!(
                "Peer {} is sending us too many messages, disconnecting",
                self.id
            );
            return Err(PeerError::TooManyMessages);
        }

        if let State::SentVersion(when) = self.state {
            if Instant::now().duration_since(when) > Duration::from_secs(10) {
                return Err(PeerError::UnexpectedMessage);
            }
        }
    }
}
```

6. **Rate Limiting**: The method calculates the rate of messages received from the peer. If the peer sends more than 10 messages per second on average, it is deemed misbehaving, and the connection is closed.

7. **Handshake Timeout**: If the peer does not respond to the version message within 10 seconds, the loop exits with an error, as the expected handshake flow was not completed.

### Handshake Process

In this `Peer` execution loop we have also seen a `State` type, stored in the `Peer.state` field. This represents the state of the handshake with the peer:

```rust
# // Path: floresta-wire/src/p2p_wire/peer.rs
#
enum State {
    None,
    SentVersion(Instant),
    SentVerack,
    Connected,
}
```

`None` is the initial state when the `Peer` is created, but shortly after that it will be updated with `SentVersion`, when we initiate the handshake by sending our `NetworkMessage::Version`.

If the peer is responsive, we will hear back from her within the next 10 seconds, via her `NetworkMessage::Version`, which will be handled by the `handle_peer_message` (that we saw in the third step). This method will internally save data from the peer, send her a `NetworkMessage::Verack` (i.e., the acknowledgment of her message), and update the state to `SentVerack`.

Finally, when we receive the `NetworkMessage::Verack` from the peer, we can switch to the `Connected` state, and communicate the new peer data with `UtreexoNode`.

### Node Communication Lifecycle

Once connected to a peer, `UtreexoNode` can send requests and receive responses.

1. It interacts with a specific peer through `NodeCommon.peers` and uses `LocalPeerView.channel` to send requests.

2. `Peer` receives the request message and handles it via `handle_node_request` (that we saw in the second step). This method will perform the write operation.

3. When the peer responds with a message, it is received via the message `actor_receiver` and handled by the `handle_peer_message` method, which likely passes new data back to `UtreexoNode`, continuing the communication loop.
