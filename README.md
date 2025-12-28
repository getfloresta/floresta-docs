# Floresta Docs

[![CI](https://github.com/JoseSK999/floresta-docs/workflows/CI/badge.svg)](https://github.com/JoseSK999/floresta-docs/actions/workflows/main.yml)
[![Deploy](https://github.com/JoseSK999/floresta-docs/workflows/Deploy/badge.svg)](https://github.com/JoseSK999/floresta-docs/actions/workflows/deploy.yml)

This repository contains the source for the `Floresta Docs` mdBook, an in-depth explanation of the [Floresta](https://github.com/vinteumorg/Floresta) libraries for building a full Bitcoin node.

You can either build the book locally, as explained below, or [read it online](https://josesk999.github.io/floresta-docs/).

> [!Note]
> This book is fully functional and ready to use, though we might add a few appendices or extra chapters down the road.

## Documentation Maintenance

This book heavily uses code snippets that reference actual code from Floresta. We have written a Rust `snippet-checker` script that compares the snippets in the book with the corresponding code in the repository.

This check runs daily at 5 AM UTC, as well as on pull requests and pushes to the main branch, via the `CI` workflow. Whenever the check fails, the script nicely prints the code difference and makes it easy to assess which explanations should be rewritten, if any.

### Deployment

Each time a change is pushed to main, the updated book is automatically deployed online if `CI` completes successfully. In other words, if `CI` and `Deploy` are passing, the deployed book snippets are up-to-date, and all explanations should be consistent with the snippets (although this is a bit harder to ensure).

## Requirements

Building the book requires [mdBook](https://github.com/rust-lang/mdBook). To get it, assuming you have Rust installed, run:

```bash
cargo install mdbook --version 0.4.45 --locked
```

Moreover, this book uses two mdBook plugins:

- For checking internal and external links before building, it uses [mdbook-linkcheck](https://github.com/Michael-F-Bryan/mdbook-linkcheck).

```bash
cargo install mdbook-linkcheck
```

- For rendering interactive quizzes in the book, it uses [mdbook-quiz](https://github.com/cognitive-engineering-lab/mdbook-quiz).

```bash
cargo install mdbook-quiz --locked
```

> **⚠️ Version Compatibility Note**
>
> We currently require **mdbook v0.4.45** because the latest version of mdbook-quiz (v0.4.0) 
> is not compatible with mdbook v0.5.x. The incompatibility stems from breaking API changes 
> introduced in [mdbook v0.5.0](https://github.com/rust-lang/mdBook/blob/master/CHANGELOG.md#mdbook-050).
>
> Once mdbook-quiz releases a version compatible with mdbook v0.5.x, we will update this requirement.

## Building

Finally, you can build the book by typing:

```bash
mdbook build
```

The rendered book will be at the root of this directory, inside `book/html`. You can then just open the `index.html` file in your usual browser.

## Run Snippet Checker

If you want to run the `snippet-checker`, you need to have a local clone of the `Floresta` source code. Then, you should pass a valid path to it via the `CODE_DIR` environment variable.

For instance, if you have cloned `Floresta` into the `~/projects` directory, you would run:

```bash
CODE_DIR=~/projects/Floresta cargo run --release
```
