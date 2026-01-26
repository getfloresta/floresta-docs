## Transaction Validation

Let's now dive into `Consensus::verify_block_transactions`, to see how we verify the transactions in a block. As we saw in the [Block Validation](ch03-04-block-validation.md) section from last chapter, this function takes the height, the UTXOs to spend, the spending transactions, the current subsidy, the `verify_script` boolean (which was only true when we are not in the `Assume-Valid` range) and the validation flags.

```rust
# // Path: floresta-chain/src/pruned_utreexo/chain_state.rs
#
pub fn validate_block_no_acc(
    &self,
    block: &Block,
    height: u32,
    inputs: HashMap<OutPoint, UtxoData>,
) -> Result<(), BlockchainError> {
    # if !block.check_merkle_root() {
        # return Err(BlockValidationErrors::BadMerkleRoot)?;
    # }
    #
    # let bip34_height = self.chain_params().params.bip34_height;
    # // If bip34 is active, check that the encoded block height is correct
    # if height >= bip34_height && Consensus::get_bip34_height(block) != Some(height) {
        # return Err(BlockValidationErrors::BadBip34)?;
    # }
    #
    # if !block.check_witness_commitment() {
        # return Err(BlockValidationErrors::BadWitnessCommitment)?;
    # }
    #
    # if block.weight().to_wu() > 4_000_000 {
        # return Err(BlockValidationErrors::BlockTooBig)?;
    # }
    #
    # // Validate block transactions
    # let subsidy = read_lock!(self).consensus.get_subsidy(height);
    # let verify_script = self.verify_script(height)?;
    // ...
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

### Validation Flags

The validation flags were returned by `get_validation_flags` based on the current height and block hash, and they are of type `core::ffi::c_uint`: a foreign function interface type used for the C++ bindings.

```rust
# // Path: floresta-chain/src/pruned_utreexo/chainparams.rs
#
// Omitted: impl ChainParams {

#[cfg(feature = "bitcoinkernel")]
/// Returns the validation flags for a given block hash and height
pub fn get_validation_flags(&self, height: u32, hash: BlockHash) -> c_uint {
    if let Some(flag) = self.exceptions.get(&hash) {
        return *flag;
    }

    // From Bitcoin Core:
    // BIP16 didn't become active until Apr 1 2012 (on mainnet, and
    // retroactively applied to testnet)
    // However, only one historical block violated the P2SH rules (on both
    // mainnet and testnet).
    // Similarly, only one historical block violated the TAPROOT rules on
    // mainnet.
    // For simplicity, always leave P2SH+WITNESS+TAPROOT on except for the two
    // violating blocks.
    let mut flags = bitcoinkernel::VERIFY_P2SH
        | bitcoinkernel::VERIFY_WITNESS
        | bitcoinkernel::VERIFY_TAPROOT;

    if height >= self.params.bip65_height {
        flags |= bitcoinkernel::VERIFY_CHECKLOCKTIMEVERIFY;
    }
    if height >= self.params.bip66_height {
        flags |= bitcoinkernel::VERIFY_DERSIG;
    }
    if height >= self.csv_activation_height {
        flags |= bitcoinkernel::VERIFY_CHECKSEQUENCEVERIFY;
    }
    if height >= self.segwit_activation_height {
        flags |= bitcoinkernel::VERIFY_NULLDUMMY;
    }

    flags
}
```

The flags cover the following consensus rules added to Bitcoin over time:

- P2SH ([BIP 16](https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki)): Activated at height 173,805
- Enforce strict DER signatures ([BIP 66](https://github.com/bitcoin/bips/blob/master/bip-0066.mediawiki)): Activated at height 363,725
- CHECKLOCKTIMEVERIFY ([BIP 65](https://github.com/bitcoin/bips/blob/master/bip-0065.mediawiki)): Activated at height 388,381
- CHECKSEQUENCEVERIFY ([BIP 112](https://github.com/bitcoin/bips/blob/master/bip-0112.mediawiki)): Activated at height 419,328
- Segregated Witness ([BIP 141](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki)) and Null Dummy ([BIP 147](https://github.com/bitcoin/bips/blob/master/bip-0147.mediawiki)): Activated at height 481,824

### Verify Block Transactions

Now, the `Consensus::verify_block_transactions` function has this body, which in turn calls `Consensus::verify_transaction`:

Filename: pruned_utreexo/consensus.rs

```rust
# // Path: floresta-chain/src/pruned_utreexo/consensus.rs
#
// Omitted: impl Consensus {

/// Verify if all transactions in a block are valid. Here we check the following:
/// - The block must contain at least one transaction, and this transaction must be coinbase
/// - The first transaction in the block must be coinbase
/// - The coinbase transaction must have the correct value (subsidy + fees)
/// - The block must not create more coins than allowed
/// - All transactions must be valid, as verified by [`Consensus::verify_transaction`]
#[allow(unused)]
pub fn verify_block_transactions(
    height: u32,
    mut utxos: HashMap<OutPoint, UtxoData>,
    transactions: &[Transaction],
    subsidy: u64,
    verify_script: bool,
    flags: c_uint,
) -> Result<(), BlockchainError> {
    // Blocks must contain at least one transaction (i.e., the coinbase)
    if transactions.is_empty() {
        return Err(BlockValidationErrors::EmptyBlock)?;
    }

    // Total block fees that the miner can claim in the coinbase
    let mut fee = Amount::ZERO;

    for (n, transaction) in transactions.iter().enumerate() {
        if n == 0 {
            if !transaction.is_coinbase() {
                return Err(BlockValidationErrors::FirstTxIsNotCoinbase)?;
            }
            Self::verify_coinbase(transaction)?;
            // Skip next checks: coinbase input is exempt, coinbase reward checked later
            continue;
        }

        // Actually verify the transaction
        let (in_value, out_value) =
            Self::verify_transaction(transaction, &mut utxos, height, verify_script, flags)?;

        // Fee is the difference between inputs and outputs. In the above function call we have
        // verified that `out_value <= in_value` (no underflow risk).
        fee = fee
            .checked_add(in_value - out_value)
            .ok_or(BlockValidationErrors::TooManyCoins)?;
    }

    // Check coinbase output values to ensure the miner isn't producing excess coins
    let allowed_reward = fee
        .checked_add(Amount::from_sat(subsidy))
        .ok_or(BlockValidationErrors::TooManyCoins)?;

    let coinbase_total = transactions[0]
        .output
        .iter()
        .try_fold(Amount::ZERO, |acc, out| acc.checked_add(out.value))
        .ok_or(BlockValidationErrors::TooManyCoins)?;

    if coinbase_total > allowed_reward {
        return Err(BlockValidationErrors::BadCoinbaseOutValue)?;
    }

    Ok(())
}

/// Verifies a single, non-coinbase transaction. To verify (the structure of) a coinbase
/// transaction, use [`Consensus::verify_coinbase`].
///
/// This function checks that the transaction:
///   - Has at least one input and one output
///   - Doesn't have null PrevOuts (reserved only for coinbase transactions)
///   - Doesn't spend more coins than it claims in the inputs
///   - Doesn't "move" more coins than allowed (at most 21 million)
///   - Spends mature coins, in case any input refers to a coinbase transaction
///   - Has valid scripts (if we don't assume them), and within the allowed size
pub fn verify_transaction(
    transaction: &Transaction,
    utxos: &mut HashMap<OutPoint, UtxoData>,
    height: u32,
    _verify_script: bool,
    _flags: c_uint,
) -> Result<(Amount, Amount), BlockchainError> {
    let txid = || transaction.compute_txid();

    let out_value = Self::check_transaction_context_free(transaction)?;

    let mut in_value = Amount::ZERO;
    for input in &transaction.input {
        // Null PrevOuts already checked in the previous step

        let utxo = Self::get_utxo(input, utxos, txid)?;
        let txout = &utxo.txout;

        // A coinbase output created at height n can only be spent at height >= n + 100
        if utxo.is_coinbase && (height < utxo.creation_height + 100) {
            return Err(tx_err!(txid, CoinbaseNotMatured))?;
        }

        // Check script sizes (spent txo pubkey, inputs are covered already)
        Self::validate_script_size(&txout.script_pubkey, txid)?;

        in_value = in_value
            .checked_add(txout.value)
            .ok_or(BlockValidationErrors::TooManyCoins)?;
    }

    // Sanity check
    if in_value > Amount::MAX_MONEY {
        return Err(BlockValidationErrors::TooManyCoins)?;
    }

    // Value in should be greater or equal to value out. Otherwise, inflation.
    if out_value > in_value {
        return Err(tx_err!(txid, NotEnoughMoney))?;
    }

    // Verify the tx script
    #[cfg(feature = "bitcoinkernel")]
    if _verify_script {
        Self::verify_input_scripts(transaction, utxos, _flags)?;
    };

    Ok((in_value, out_value))
}
```

In general, the function behavior is well explained in the comments. Something to note is that we need the `bitcoinkernel` feature set in order to verify the transaction scripts. Because such validation is performed with the Bitcoin Core library behind the scenes, we make it optional for some architectures without C++ support to compile the Floresta crates.

We also don't validate if `verify_script` is false, but this is because the `Assume-Valid` process has already assessed the scripts as valid.
