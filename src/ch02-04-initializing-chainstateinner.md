## Initializing ChainStateInner

This is the struct that holds the meat of the matter (or should we say the root of the issue). As it's inside the `spin::RwLock` it can be read and modified in a thread-safe way. Its fields are:

- `acc`: The accumulator, of type `Stump` (which comes from the `rustreexo` crate).
- `chainstore`: Our implementation of `ChainStore`.
- `best_block`: Of type `BestChain`.
- `broadcast_queue`: Holds a list of transactions to be broadcast, of type `Vec<Transaction>`.
- `subscribers`: A vector of trait objects (different types allowed) that implement the `BlockConsumer` trait, indicating they want to get notified when a new valid block arrives.
- `fee_estimation`: Fee estimation for the next 1, 10 and 20 blocks, as a tuple of three f64.
- `ibd`: A boolean indicating if we are in IBD.
- `consensus`: Parameters for the chain validation, as a `Consensus` struct (a Floresta type that will be explained in detail in [Chapter 4](ch04-00-consensus-and-bitcoinkernel.md)).
- `assume_valid`: As an `Option<BlockHash>`.

Note that the accumulator and the best block data are kept in our `ChainStore`, but we cache them in `ChainStateInner` for faster access, avoiding potential disk reads and deserializations (e.g., loading them from the `meta` bucket if we use `KvChainStore`).

Let's next see how these fields are accessed with an example.

### Adding Subscribers to ChainState

As `ChainState` implements `BlockchainInterface`, it has a `subscribe` method to allow other types receive notifications.

The subscribers are stored in the `ChainStateInner.subscribers` field, but we need to handle the `RwLock` that wraps `ChainStateInner` for that.

Filename: pruned_utreexo/chain_state.rs

```rust
# // Path: floresta-chain/src/pruned_utreexo/chain_state.rs
#
// Omitted: impl<PersistedState: ChainStore> BlockchainInterface for ChainState<PersistedState> {

fn subscribe(&self, tx: Arc<dyn BlockConsumer>) {
    let mut inner = self.inner.write();
    inner.subscribers.push(tx);
}
```

This is the `BlockchainInterface::subscribe` implementation. We use the `write` method on the `RwLock` which then gives us exclusive access to the `ChainStateInner`.

When `inner` is dropped after the push, the lock is released and becomes available for other threads, which may acquire `write` access (one thread at a time) or `read` access (multiple threads simultaneously).

### Initial ChainStateInner Values

In `ChainState::new` we initialize `ChainStateInner` like so:

```rust
# // Path: floresta-chain/src/pruned_utreexo/chain_state.rs
#
pub fn new(
    mut chainstore: PersistedState,
    network: Network,
    assume_valid: AssumeValidArg,
) -> ChainState<PersistedState> {
    # let parameters = network.into();
    # let genesis = genesis_block(&parameters);
    #
    # chainstore
        # .save_header(&DiskBlockHeader::FullyValid(genesis.header, 0))
        # .expect("Error while saving genesis");
    #
    # chainstore
        # .update_block_index(0, genesis.block_hash())
        # .expect("Error updating index");
    #
    # let assume_valid = ChainParams::get_assume_valid(network, assume_valid);
    // ...
    ChainState {
        inner: RwLock::new(ChainStateInner {
            chainstore,
            acc: Stump::new(),
            best_block: BestChain {
                best_block: genesis.block_hash(),
                depth: 0,
                validation_index: genesis.block_hash(),
                alternative_tips: Vec::new(),
            },
            broadcast_queue: Vec::new(),
            subscribers: Vec::new(),
            fee_estimation: (1_f64, 1_f64, 1_f64),
            ibd: true,
            consensus: Consensus { parameters },
            assume_valid,
        }),
    }
}
```

The TLDR is that we move `chainstore` to the `ChainStateInner`, initialize the accumulator (`Stump::new`), initialize `BestChain` with the genesis block (being the best block and best validated block) and depth 0, initialize `broadcast_queue` and `subscribers` as empty vectors, set the minimum fee estimations, set `ibd` to true, use the `Consensus` parameters for the current `Network` and move the `assume_valid` optional hash in.

{{#quiz ../quizzes/ch02-04-initializing-chainstateinner.toml}}

## Recap

In this chapter we have understood the structure of `ChainState`, a type implementing `UpdatableChainstate + BlockchainInterface`; a blockchain backend. This type required a `ChainStore` implementation, expected to save state data to disk, and we examined the provided `KvChainStore`.

Finally, we have seen the `ChainStateInner` struct, which keeps track of the `ChainStore` and more data.

We can now build a `ChainState` as simply as:

```rust
fn main() {
    let chain_store =
        KvChainStore::new("./epic_location".to_string())
            .expect("failed to open the blockchain database");

    let chain = ChainState::new(
        chain_store,
        Network::Bitcoin,
        AssumeValidArg::Disabled,
    );
}
```
