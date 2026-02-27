## Block Validation

We have arrived at the final part of this chapter! Here we understand the `validate_block_no_acc` method that we used in `connect_block`, and lastly do a recap of everything.

> `validate_block_no_acc` is also used inside the `BlockchainInterface::validate_block` trait method implementation, as it encapsulates the non-utreexo validation logic.
> 
> The difference between `UpdatableChainstate::connect_block` and `BlockchainInterface::validate_block` is that the first is used during IBD, while the latter is a tool that allows the node user to validate blocks without affecting the node state.

The method returns `BlockchainError::BlockValidation`, wrapping many different `BlockValidationErrors`, which is an enum declared in _pruned_utreexo/errors.rs_.

```rust
# // Path: floresta-chain/src/pruned_utreexo/chain_state.rs
#
pub fn validate_block_no_acc(
    &self,
    block: &Block,
    height: u32,
    inputs: HashMap<OutPoint, UtxoData>,
) -> Result<(), BlockchainError> {
    read_lock!(self).consensus.check_block(block, height)?;

    // Validate block transactions
    let subsidy = read_lock!(self).consensus.get_subsidy(height);
    let verify_script = self.verify_script(height)?;

    #[cfg(feature = "bitcoinkernel")]
    let flags = self
        .chain_params()
        .get_validation_flags(height, block.block_hash());
    #[cfg(not(feature = "bitcoinkernel"))]
    let flags = 0;

    Consensus::verify_block_transactions(
        height,
        inputs,
        &block.txdata,
        subsidy,
        verify_script,
        flags,
    )?;
    Ok(())
}
```

First, we call the `check_block` method which performs the following checks:

1. `check_merkle_root`, to ensure that the merkle root in the header commits to all the transactions, and block data has not been tampered with.
2. Check that, if the height is at least that of [BIP 34](https://github.com/bitcoin/bips/blob/master/bip-0034.mediawiki) activation, the coinbase transaction encodes the height as specified.
3. `check_witness_commitment`, to check that the `wtxid` merkle root is included in the coinbase transaction as per [BIP 141](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki).
4. Check that the block weight doesn't exceed the 4,000,000 weight unit limit.

Lastly, we go on to validate the transactions. We retrieve the current subsidy (the newly generated coins) with the `get_subsidy` method on our `Consensus` struct. We also call the `verify_script` method which returns a boolean flag indicating if we are NOT inside the `Assume-Valid` range.

```rust
# // Path: floresta-chain/src/pruned_utreexo/chain_state.rs
#
fn verify_script(&self, height: u32) -> Result<bool, PersistedState::Error> {
    let inner = self.inner.read();
    match inner.assume_valid {
        Some(hash) => {
            match inner.chainstore.get_header(&hash)? {
                // If the assume-valid block is in the best chain, only verify scripts if we are higher
                Some(DiskBlockHeader::HeadersOnly(_, assume_h))
                | Some(DiskBlockHeader::FullyValid(_, assume_h)) => Ok(height > assume_h),
                // Assume-valid is not in the best chain, so verify all the scripts
                _ => Ok(true),
            }
        }
        None => Ok(true),
    }
}
```

We also get the validation flags for the current height with `get_validation_flags`, only if the `bitcoinkernel` feature is active. These flags are used to validate transactions taking into account the different consensus rules that have been added over time.

And lastly we call the `Consensus::verify_block_transactions` associated function.

## Recap

In this chapter we have seen many of the methods for the `ChainState` type, mainly related to the chain validation and state transition process.

We started with `accept_header`, that checked if the header was in the database or not. If it was in the database we just called `maybe_reindex`. If it was not, we validated it and potentially updated the chain tip, either by extending it or by reorging. We also called `ChainStore::save_header` and `ChainStore::update_block_index`.

Then we saw `connect_block`, which validated the next block in the chain with `validate_block_no_acc`. If block was valid, we may invoke `UpdatableChainstate::flush` to persist all data (including the data saved with `ChainStore::save_roots` and `ChainStore::save_height`) and also marked the disk header as `FullyValid`. Finally, we notified the new block.
