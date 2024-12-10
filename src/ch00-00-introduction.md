# Introduction

Floresta is a collection of Rust libraries designed for building a Bitcoin full node, alongside the assembled daemon and RPC client binaries, all developed by Davidson Souza.

A key feature of Floresta is its use of the [utreexo accumulator](https://eprint.iacr.org/2019/611) to maintain the UTXO set in a highly compact format. It also incorporates innovative techniques to significantly reduce Initial Block Download (IBD) times with minimal security tradeoffs.

> The Utreexo accumulator consists of a forest of merkle trees where leaves are individual UTXOs, thus the name U-Tree-XO. The UTXO set at any moment is represented as the **merkle roots of the forest**.
>
> The novelty in this cryptographic system is a mechanism to update the forest, both adding new UTXOs and deleting existing ones from the set. When a transaction spends UTXOs we can verify it with an inclusion proof, and then delete those specific UTXOs from the set.

Currently, the node can only operate in pruned mode, meaning it deletes block data after validation. Combined with utreexo, this design keeps storage requirements exceptionally low (< 1 GB).

The ultimate vision for Floresta is to deliver a reliable and ultra-lightweight node implementation capable of running on low-resource devices, democratizing the access to the Bitcoin blockchain.

⚠️ Keep in mind that Floresta is still highly experimental. **We do not recommend using it for transactions involving meaningful amounts of satoshis.** If you notice a bug or anything that seems incorrect, please open an issue in the [official Floresta repository](https://github.com/getfloresta/Floresta).

## About This Book

This documentation provides an overview of the process involved in creating and running a Floresta node (i.e., a `UtreexoNode`). We start by examining the project's overall structure, which is necessary to build a foundation for understanding its internal workings.

The book will contain plenty of code snippets, which are identical to Floresta code sections. Each snippet includes a commented path referencing its corresponding file, visible by clicking the _eye icon_ in the top-right corner of the snippet (e.g., `// Path: floresta-chain/src/lib.rs`).

You will also find some interactive quizzes to test and reinforce your understanding of Floresta!

### Note on Types

To avoid repetitive explanations, this documentation follows a simple convention: **unless otherwise stated** any mentioned type is assumed to come from [the `bitcoin` crate](https://github.com/rust-bitcoin/rust-bitcoin/tree/master), which provides many of the building blocks for Floresta. If you see a type, and we don't mention its source, you can assume it's a `bitcoin` type.
