## Address Manager

Before diving into the details of P2P networking, let's understand the crucial address manager module.

In the last section, we saw that the `create_connection` method uses the `self.address_man.get_address_to_connect` method to obtain a suitable address for connection. This method belongs to the `AddressMan` struct (a field from `NodeCommon`), which is responsible for maintaining a record of known peer addresses and providing them to the node when needed.

Filename: p2p_wire/address_man.rs

```rust
# // Path: floresta-wire/src/p2p_wire/address_man.rs
#
pub struct AddressMan {
    /// A map of all peers we know, mapping the address id to the actual address.
    addresses: HashMap<usize, LocalAddress>,

    good_addresses: Vec<usize>,
    good_peers_by_service: HashMap<ServiceFlags, Vec<usize>>,
    peers_by_service: HashMap<ServiceFlags, Vec<usize>>,

    /// The maximum number of entries this address manager can hold
    max_size: usize,
    /// The networks we can reach (e.g., IPv4, IPv6, TorV3, etc.)
    reachable_networks: HashSet<ReachableNetworks>,
}
```

This struct is straightforward: it keeps known peer addresses in a `HashMap` (as `LocalAddress`), the indexes of good peers in the map, and links peers with the network services they support (including the `UTREEXO` service bit).

### Local Address

We have also encountered the `LocalAddress` type a few times, which is implemented in this module. This type encapsulates all the information our node knows about each peer, effectively serving as our local representation of the peer's details.

```rust
# // Path: floresta-wire/src/p2p_wire/address_man.rs
#
pub struct LocalAddress {
    /// An actual address
    address: AddrV2,
    /// Last time we successfully connected to this peer, in secs, only relevant if state == State::Tried
    last_connected: u64,
    /// Our local state for this peer, as defined in AddressState
    state: AddressState,
    /// Network services announced by this peer
    services: ServiceFlags,
    /// Network port this peers listens to
    port: u16,
    /// Random id for this peer
    pub id: usize,
}
```

The actual address is stored in the form of an `AddrV2` enum, which is implemented by the `bitcoin` crate. `AddrV2` represents various network address types supported in Bitcoin's P2P protocol, as defined in [BIP155](https://github.com/bitcoin/bips/blob/master/bip-0155.mediawiki).

> Concretely, `AddrV2` includes variants for `IPv4`, `IPv6`, `Tor` (v2 and v3), `I2P`, `Cjdns` addresses, and an `Unknown` variant for unrecognized address types. This design allows the protocol to handle a diverse set of network addresses.

The `LocalAddress` also stores the last connection date or time, measured as seconds since the [UNIX_EPOCH](https://en.wikipedia.org/wiki/Unix_time), an `AddressState` struct, the network services announced by the peer, the port that the peer listens to, and its identifier.

Below is the definition of `AddressState`, which tracks the current status and history of our interactions with this peer:

```rust
# // Path: floresta-wire/src/p2p_wire/address_man.rs
#
pub enum AddressState {
    /// We never tried this peer before, so we don't know what to expect. This variant
    /// also applies to peers that we tried to connect, but failed, or we didn't connect
    /// to for a long time.
    NeverTried,
    /// We tried this peer before, and had success at least once, so we know what to expect
    Tried(u64),
    /// This peer misbehaved and we banned them
    Banned(u64),
    /// We are connected to this peer right now
    Connected,
    /// We tried connecting, but failed
    Failed(u64),
}
```

### Get Address to Connect

Let's finally inspect the `get_address_to_connect` method on the `AddressMan`, which we use to [create connections](ch06-03-opening-connections.md#create-connection).

This method selects a peer address for a new connection based on required services and whether the connection is a feeler. First of all, we will return `None` if the address manager doesn't have any peers. Otherwise:

- For feeler connections, it randomly picks an address, or returns `None` if the peer is `Banned` or already `Connected`.
- For regular connections, it prioritizes peers supporting the required services or falls back to a random address. Peers in the `NeverTried` and `Tried` states are considered valid, `Connected` and `Banned` peers are skipped, and `Failed` ones are only accepted if enough time has passed. If no suitable address is found, it returns `None`.

```rust
# // Path: floresta-wire/src/p2p_wire/address_man.rs
#
/// Returns a new random address to open a new connection, we try to get addresses with
/// a set of features supported for our peers
pub fn get_address_to_connect(
    &mut self,
    required_service: ServiceFlags,
    feeler: bool,
) -> Option<(usize, LocalAddress)> {
    if self.addresses.is_empty() {
        return None;
    }

    // Feeler connection are used to test if a peer is still alive, we don't care about
    // the features it supports or even if it's a valid peer. The only thing we care about
    // is that we haven't banned it.
    if feeler {
        let idx = rand::random::<usize>() % self.addresses.len();
        let peer = self.addresses.keys().nth(idx)?;
        let address = self.addresses.get(peer)?.to_owned();

        // don't try to connect to a peer that is banned or already connected
        if matches!(address.state, AddressState::Banned(_))
            | matches!(address.state, AddressState::Connected)
        {
            return None;
        }

        return Some((*peer, address));
    };

    for _ in 0..10 {
        let (id, peer) = self
            .get_address_by_service(required_service)
            .or_else(|| self.get_random_address(required_service))?;

        match peer.state {
            AddressState::NeverTried | AddressState::Tried(_) => {
                return Some((id, peer));
            }

            AddressState::Connected => {
                // if we are connected to this peer, don't try to connect again
                continue;
            }

            AddressState::Failed(when) => {
                let now = Self::time_since_unix();
                if when + RETRY_TIME < now {
                    return Some((id, peer));
                }

                if let Some(peers) = self.good_peers_by_service.get_mut(&required_service) {
                    peers.retain(|&x| x != id)
                }

                self.good_addresses.retain(|&x| x != id);
            }

            AddressState::Banned(_) => {}
        }
    }

    None
}
```

### Dump Peers

Another key functionality implemented in this module is the ability to write the current peer data to a `peers.json` file, enabling the node to resume peer connections after a restart without repeating the initial peer discovery process.

To save each `LocalAddress` we use a slightly modified type called `DiskLocalAddress`, similar to how we used [the `DiskBlockHeader` type](ch02-02-bestchain-and-diskblockheader.md#diskblockheader) to persist `BlockHeader`s.

```rust
# // Path: floresta-wire/src/p2p_wire/address_man.rs
#
pub fn dump_peers(&self, datadir: &str) -> std::io::Result<()> {
    let peers: Vec<_> = self
        .addresses
        .values()
        .cloned()
        .map(Into::<DiskLocalAddress>::into)
        .collect::<Vec<_>>();
    let peers = serde_json::to_string(&peers);
    if let Ok(peers) = peers {
        std::fs::write(datadir.to_owned() + "/peers.json", peers)?;
    }
    Ok(())
}
```

Similarly, there's a `dump_utreexo_peers` method for persisting the utreexo peers into an `anchors.json` file. Peers that support utreexo are very valuable for our node; we need their utreexo proofs for validating blocks, and they are rare in the network.

```rust
# // Path: floresta-wire/src/p2p_wire/address_man.rs
#
/// Dumps the connected utreexo peers to a file on dir `datadir/anchors.json` in json format `
/// inputs are the directory to save the file and the list of ids of the connected utreexo peers
pub fn dump_utreexo_peers(&self, datadir: &str, peers_id: &[usize]) -> std::io::Result<()> {
    // ...
    # let addresses: Vec<DiskLocalAddress> = peers_id
        # .iter()
        # .filter_map(|id| Some(self.addresses.get(id)?.to_owned().into()))
        # .collect();
    # let addresses: Result<String, serde_json::Error> = serde_json::to_string(&addresses);
    # if let Ok(addresses) = addresses {
        # std::fs::write(datadir.to_owned() + "/anchors.json", addresses)?;
    # }
    # Ok(())
}
```

Great! This concludes the chapter. In the next chapter, we will dive into P2P communication and networking, focusing on the `Peer` type.
