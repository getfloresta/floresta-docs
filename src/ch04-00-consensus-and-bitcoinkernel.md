# Consensus and bitcoinkernel

In the previous chapter, we saw that the block validation process involves two associated functions from `Consensus`:

- `verify_block_transactions`: The last check performed inside `validate_block_no_acc`, after having validated the two merkle roots and height commitment.
- `update_acc`: Called inside `connect_block`, just after `validate_block_no_acc`, to verify the utreexo proof and get the updated accumulator.

The `Consensus` struct only holds a `parameters` field (as we saw [when we initialized ChainStateInner](ch02-04-initializing-chainstateinner.md#initial-chainstateinner-values)) and provides a few core consensus functions. In this chapter we are going to see the two mentioned functions and discuss the details of how we verify scripts.

Filename: pruned_utreexo/consensus.rs

```rust
# // Path: floresta-chain/src/pruned_utreexo/consensus.rs
#
pub struct Consensus {
    // The chain parameters are in the chainparams.rs file
    pub parameters: ChainParams,
}
```

## bitcoinkernel

`Consensus::verify_block_transactions` is a critical part of Floresta, as it validates all the transactions in a block. One of the hardest parts for validation is checking the **script satisfiability**, that is, verifying whether the inputs can indeed spend the coins. It's also the most resource-intensive part of block validation, as it requires verifying many digital signatures.

Implementing a [Bitcoin script](https://en.bitcoin.it/wiki/Script) interpreter is challenging, and given the complexity of both C++ and Rust, we cannot be certain that it will always behave in the same way as Bitcoin Core. This is problematic because if our Rust implementation rejects a script that Bitcoin Core accepts, our node will fork from the network. It will treat subsequent blocks as invalid, halting synchronization with the chain and being unable to continue tracking the user balance.

Partly because of this reason, in 2015 the script validation logic of Bitcoin Core was extracted and placed into the [libbitcoin-consensus](https://github.com/libbitcoin/libbitcoin-consensus) library. Subsequently, the library API was bound to Rust in [rust-bitcoinconsensus](https://github.com/rust-bitcoin/rust-bitcoinconsensus), which Floresta originally used. Nonetheless, `bitcoinconsensus` handles only script validation and is maintained as a separate project from Bitcoin Core, with limited upkeep.

To address these shortcomings, the Bitcoin Core community has been working to extract the consensus engine into a standalone library. This is known as the [libbitcoinkernel](https://github.com/bitcoin/bitcoin/issues/27587) project.

In late 2025, Floresta started using [rust-bitcoinkernel](https://github.com/sedited/rust-bitcoinkernel), the Rust bindings for the `libbitcoinkernel` library. Thus, Floresta is now able to verify scripts using the same logic as Bitcoin Core.
