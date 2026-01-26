## Chain Backend API

We will now take a look at the API that the `Chain` backend (from `UtreexoNode`) is required to expose, as part of the `BlockchainInterface` and `UpdatableChainstate` traits.

The lists below are only meant to provide an initial sense of the expected chain API.

### The BlockchainInterface Trait

The `BlockchainInterface` methods are mainly about _getting information_ from the current view of the blockchain and state of validation.

It defines a generic associated error type bounded by the `std::error::Error` trait—or, in `no-std` environments, by Floresta's own minimal `Error` marker trait—so each `BlockchainInterface` implementation can pick its own error type.

The list of required methods:

- `get_block_hash`, given a u32 height.
- `get_tx`, given its txid.
- `get_height` of the chain.
- `broadcast` a transaction to the network.
- `estimate_fee` for inclusion in usize target blocks.
- `get_block`, given its hash.
- `get_best_block` hash and height.
- `get_block_header`, given its hash.
- `is_in_ibd`, whether we are in Initial Block Download (IBD) or not.
- `get_unbroadcasted` transactions.
- `is_coinbase_mature`, given its block hash and height (on the mainchain, coinbase transactions mature after 100 blocks).
- `get_block_locator`, i.e., a compact list of block hashes used to efficiently identify the most recent common point in the blockchain between two nodes for synchronization purposes.
- `get_block_locator_for_tip`, given the hash of the tip block. This can be used for tips that are not canonical or best.
- `get_validation_index`, i.e., the height of the last block we have validated.
- `get_block_height`, given its block hash.
- `get_chain_tips` block hashes, including the best tip and non-canonical ones.
- `get_fork_point`, to get the block hash where a given branch forks (the branch is represented by its tip block hash).
- `get_params`, to get the parameters for chain consensus.
- `acc`, to get the current utreexo accumulator.

Also, we have a `subscribe` method which allows other components to receive notifications of new validated blocks from the blockchain backend.

Filename: floresta-chain/src/pruned_utreexo/mod.rs

```rust
# // Path: floresta-chain/src/pruned_utreexo/mod.rs
#
pub trait BlockchainInterface {
    type Error: Error + Send + Sync + 'static;
    // ...
    #
    # fn get_block_hash(&self, height: u32) -> Result<bitcoin::BlockHash, Self::Error>;
    #
    # fn get_tx(&self, txid: &bitcoin::Txid) -> Result<Option<bitcoin::Transaction>, Self::Error>;
    #
    # fn get_height(&self) -> Result<u32, Self::Error>;
    #
    # fn broadcast(&self, tx: &bitcoin::Transaction) -> Result<(), Self::Error>;
    #
    # fn estimate_fee(&self, target: usize) -> Result<f64, Self::Error>;
    #
    # fn get_block(&self, hash: &BlockHash) -> Result<Block, Self::Error>;
    #
    # fn get_best_block(&self) -> Result<(u32, BlockHash), Self::Error>;
    #
    # fn get_block_header(&self, hash: &BlockHash) -> Result<BlockHeader, Self::Error>;
    #
    fn subscribe(&self, tx: Arc<dyn BlockConsumer>);
    // ...
    #
    # fn is_in_ibd(&self) -> bool;
    #
    # fn get_unbroadcasted(&self) -> Vec<Transaction>;
    #
    # fn is_coinbase_mature(&self, height: u32, block: BlockHash) -> Result<bool, Self::Error>;
    #
    # fn get_block_locator(&self) -> Result<Vec<BlockHash>, Self::Error>;
    #
    # fn get_block_locator_for_tip(&self, tip: BlockHash) -> Result<Vec<BlockHash>, BlockchainError>;
    #
    # fn get_validation_index(&self) -> Result<u32, Self::Error>;
    #
    # fn get_block_height(&self, hash: &BlockHash) -> Result<Option<u32>, Self::Error>;
    #
    # fn update_acc(
        # &self,
        # acc: Stump,
        # block: &Block,
        # height: u32,
        # proof: Proof,
        # del_hashes: Vec<sha256::Hash>,
    # ) -> Result<Stump, Self::Error>;
    #
    # fn get_chain_tips(&self) -> Result<Vec<BlockHash>, Self::Error>;
    #
    # fn validate_block(
        # &self,
        # block: &Block,
        # proof: Proof,
        # inputs: HashMap<OutPoint, UtxoData>,
        # del_hashes: Vec<sha256::Hash>,
        # acc: Stump,
    # ) -> Result<(), Self::Error>;
    #
    # fn get_fork_point(&self, block: BlockHash) -> Result<BlockHash, Self::Error>;
    #
    # fn get_params(&self) -> bitcoin::params::Params;
    #
    # fn acc(&self) -> Stump;
}
```

```rust
# // Path: floresta-chain/src/pruned_utreexo/mod.rs
#
pub enum Notification {
    NewBlock((Block, u32)),
}
```

Any type that implements the `BlockConsumer` trait can `subscribe` to our `BlockchainInterface` by passing a reference of itself, and receive notifications of new blocks (including block data and height). In the future this can be extended to also notify transactions.

#### Validation Methods

Finally, there are two validation methods that do NOT update the node state:

- `update_acc`, to get the new accumulator after applying a new block. It requires the current accumulator, the new block data, the inclusion proof for the spent UTXOs, and the hashes of the spent UTXOs.
- `validate_block`, which instead of only verifying the inclusion proof, validates the whole block (including its transactions, for which the spent UTXOs themselves are needed).

### The UpdatableChainstate Trait

On the other hand, the methods required by `UpdatableChainstate` are expected to update the node state.

These methods use the `BlockchainError` enum, found in _pruned_utreexo/error.rs_. Each variant of `BlockchainError` represents a kind of error that is expected to occur (block validation errors, invalid utreexo proofs, etc.). The `UpdatableChainstate` methods are:

#### Very important
- `connect_block`: Takes a block and utreexo data, validates the block and adds it to our chain.
- `accept_header`: Checks a header and saves it in storage. This is called before `connect_block`, which is responsible for accepting or rejecting the actual block.

```rust
# // Path: floresta-chain/src/pruned_utreexo/mod.rs
#
pub trait UpdatableChainstate {
    fn connect_block(
        &self,
        block: &Block,
        proof: Proof,
        inputs: HashMap<OutPoint, UtxoData>,
        del_hashes: Vec<sha256::Hash>,
    ) -> Result<u32, BlockchainError>;
    // ...
    #
    # fn switch_chain(&self, new_tip: BlockHash) -> Result<(), BlockchainError>;
    #
    fn accept_header(&self, header: BlockHeader) -> Result<(), BlockchainError>;
    // ...
    #
    # fn handle_transaction(&self) -> Result<(), BlockchainError>;
    #
    # fn flush(&self) -> Result<(), BlockchainError>;
    #
    # fn toggle_ibd(&self, is_ibd: bool);
    #
    # fn invalidate_block(&self, block: BlockHash) -> Result<(), BlockchainError>;
    #
    # fn mark_block_as_valid(&self, block: BlockHash) -> Result<(), BlockchainError>;
    #
    # fn get_root_hashes(&self) -> Vec<BitcoinNodeHash>;
    #
    # fn get_partial_chain(
        # &self,
        # initial_height: u32,
        # final_height: u32,
        # acc: Stump,
    # ) -> Result<PartialChainState, BlockchainError>;
    #
    # fn mark_chain_as_assumed(&self, acc: Stump, tip: BlockHash) -> Result<bool, BlockchainError>;
    #
    # fn get_acc(&self) -> Stump;
}
```

> Usually, in IBD we **fetch a chain of headers with sufficient PoW first**, and only then do we ask for the block data (i.e., the transactions) in order to verify the blocks. This way we ensure that DoS attacks sending our node invalid blocks, with the purpose of wasting our resources, are costly because of the required PoW.

#### Others
- `switch_chain`: Reorg to another branch, given its tip block hash.
- `handle_transaction`: Process transactions that are in the mempool.
- `flush`: Writes pending data to storage. Should be invoked periodically.
- `toggle_ibd`: Toggle the IBD process on/off.
- `invalidate_block`: Tells the blockchain backend to consider this block invalid.
- `mark_block_as_valid`: Overrides a block that was marked as invalid, considering it as fully validated.
- `get_root_hashes`: Returns the root hashes of our utreexo accumulator.
- `get_partial_chain`: Returns a `PartialChainState` (a Floresta type allowing to validate parts of the chain in parallel, explained in [Chapter 5](ch05-00-advanced-chain-validation-methods.md)), given the height range and the initial utreexo state.
- `mark_chain_as_assumed`: Given a block hash and the corresponding accumulator, assume every ancestor block is valid.
- `get_acc`: Returns the current accumulator.

{{#quiz ../quizzes/ch01-02-chain-backend-api.toml}}
