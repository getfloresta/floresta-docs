## Building the ChainState

The next step is building the `ChainState` struct, which validates blocks and updates the `ChainStore`.

Filename: pruned_utreexo/chain_state.rs

```rust
# // Path: floresta-chain/src/pruned_utreexo/chain_state.rs
#
pub struct ChainState<PersistedState: ChainStore> {
    inner: RwLock<ChainStateInner<PersistedState>>,
}
```

Note that the `RwLock` that we use to wrap `ChainStateInner` is not the one from the standard library but from the `spin` crate, thus allowing `no_std`.

> `std::sync::RwLock` relies on the OS to block and wake threads when the lock is available, while `spin::RwLock` uses a [spinlock](https://en.wikipedia.org/wiki/Spinlock) which does not require OS support for thread management, as the thread simply keeps running (and checking for lock availability) instead of sleeping.

The builder for `ChainState` has this signature:

```rust
# // Path: floresta-chain/src/pruned_utreexo/chain_state.rs
#
pub fn new(
    mut chainstore: PersistedState,
    network: Network,
    assume_valid: AssumeValidArg,
) -> ChainState<PersistedState> {
```

The first argument is our `ChainStore` implementation, the second one is [the `Network` enum](https://docs.rs/bitcoin/latest/bitcoin/enum.Network.html) from the `bitcoin` crate, and thirdly the `AssumeValidArg` enum.

The `Network` enum acknowledges four kinds of networks: `Bitcoin` (mainchain), `Testnet` (version 3), `Testnet4`, `Signet` and `Regtest`.

### The Assume-Valid Lore

The `assume_valid` argument refers to a Bitcoin Core option that allows nodes during IBD to assume the validity of scripts (mainly signatures) up to a certain block.

Nodes with this option enabled will still choose the most PoW chain (the best tip), and will only skip script validation if the `Assume-Valid` block is in that chain. Otherwise, if the `Assume-Valid` block is not in the best chain, they will validate everything.

> When users use the default `Assume-Valid` hash, hardcoded in the software, they aren't blindly trusting script validity. These hashes are reviewed through the same open-source process as other security-critical changes in Bitcoin Core, so the trust model is unchanged.

In Bitcoin Core, the hardcoded `Assume-Valid` block hash is included in _src/kernel/chainparams.cpp_.

Filename: pruned_utreexo/chain_state.rs

```rust
# // Path: floresta-chain/src/pruned_utreexo/chain_state.rs
#
pub enum AssumeValidArg {
    Disabled,
    Hardcoded,
    UserInput(BlockHash),
}
```

`Disabled` means the node verifies all scripts, `Hardcoded` means the node uses the default block hash that has been hardcoded in the software (and validated by maintainers, developers and reviewers), and `UserInput` means using a hash that the node runner provides, although the validity of the scripts up to this block should have been externally validated.

### Genesis and Assume-Valid Blocks

The first part of the body of `ChainState::new` (let's omit the `impl` block from now on):

```rust
# // Path: floresta-chain/src/pruned_utreexo/chain_state.rs
#
pub fn new(
    mut chainstore: PersistedState,
    network: Network,
    assume_valid: AssumeValidArg,
) -> ChainState<PersistedState> {
    let parameters = network.into();
    let genesis = genesis_block(&parameters);

    chainstore
        .save_header(&DiskBlockHeader::FullyValid(genesis.header, 0))
        .expect("Error while saving genesis");

    chainstore
        .update_block_index(0, genesis.block_hash())
        .expect("Error updating index");

    let assume_valid = ChainParams::get_assume_valid(network, assume_valid);
    // ...
    # ChainState {
        # inner: RwLock::new(ChainStateInner {
            # chainstore,
            # acc: Stump::new(),
            # best_block: BestChain {
                # best_block: genesis.block_hash(),
                # depth: 0,
                # validation_index: genesis.block_hash(),
                # alternative_tips: Vec::new(),
            # },
            # subscribers: Vec::new(),
            # fee_estimation: (1_f64, 1_f64, 1_f64),
            # ibd: true,
            # consensus: Consensus { parameters },
            # assume_valid,
        # }),
    # }
}
```

First, we use the `genesis_block` function from `bitcoin` to retrieve the genesis block based on the specified parameters, which are determined by the `Network` kind.

Then we save the genesis header into `chainstore`, which of course is `FullyValid` and has height 0. We also link the index 0 with the genesis block hash.

Finally, we get an `Option<BlockHash>` by calling the `ChainParams::get_assume_valid` function, which takes a `Network` and an `AssumeValidArg`.

Filename: pruned_utreexo/chainparams.rs

```rust
# // Path: floresta-chain/src/pruned_utreexo/chainparams.rs
#
// Omitted: impl ChainParams {

pub fn get_assume_valid(network: Network, arg: AssumeValidArg) -> Option<BlockHash> {
    match arg {
        AssumeValidArg::Disabled => None,
        AssumeValidArg::UserInput(hash) => Some(hash),
        AssumeValidArg::Hardcoded => match network {
            Network::Bitcoin => Some(bhash!(
                "00000000000000000001ff36aef3a0454cf48887edefa3aab1f91c6e67fee294"
            )),
            Network::Testnet => Some(bhash!(
                "000000007df22db38949c61ceb3d893b26db65e8341611150e7d0a9cd46be927"
            )),
            Network::Testnet4 => Some(bhash!(
                "0000000000335c2895f02ebc75773d2ca86095325becb51773ce5151e9bcf4e0"
            )),
            Network::Signet => Some(bhash!(
                "000000084ece77f20a0b6a7dda9163f4527fd96d59f7941fb8452b3cec855c2e"
            )),
            Network::Regtest => Some(bhash!(
                "0f9188f13cb7b2c71f2a335e3a4fc328bf5beb436012afca590b1a11466e2206"
            )),
        },
    }
}
```

The final part of `ChainState::new` just returns the instance of `ChainState` with the `ChainStateInner` initialized. We will see this initialization next.

{{#quiz ../quizzes/ch02-03-building-the-chainstate.toml}}
