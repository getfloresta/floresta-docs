## Floresta Daemon

The `florestad` binary is the entry point for running a Floresta node. It parses the CLI arguments, builds a `Config`, and optionally daemonizes the process if the `--daemon` flag is set. Once initialization is complete, it hands control to `Florestad::start`, which actually launches the node and all its subsystems.

We'll first look at how the binary sets things up, and then we'll dive into `Florestad::start` itself to see how the node comes alive.

### Florestad

The very first thing we do in the `florestad` main function is parsing any CLI argument and create the actual data directory, if not already present. Then, we build the `Config` from the CLI arguments and try to daemonize the process.

Also, if logging to `stdout` or a file is enabled, we call `init_logging` to initialize a tracing subscriber. This subscriber may also include [tokio-console](https://github.com/tokio-rs/console) (a utility from the Tokio project that lets you inspect all running async tasks in real time) if you run `florestad` with the `tokio-console` feature. You can learn more about tokio console usage in the [Floresta doc folder](https://github.com/getfloresta/Floresta/blob/master/doc/run.md#using-tokio-console).

After that, we spawn a process that waits for the `Ctrl+C` signal, and when it's read this task writes `true` to the signal variable (an `Arc<RwLock<bool>>`).

```rust
# // Path: ../bin/florestad/src/main.rs
#
fn main() {
    let params = Cli::parse();

    // If not provided defaults to `$HOME/.floresta`. Uses a subdirectory for non-mainnet networks.
    let data_dir = data_dir_path(params.data_dir, params.network);

    // Create the data directory if it doesn't exist
    fs::create_dir_all(&data_dir).unwrap_or_else(|e| {
        eprintln!("Could not create data dir {data_dir:?}: {e}");
        exit(1);
    });

    let config = Config {
        data_dir,
        // Setting the config from the CLI arguments
        // ...
        # disable_dns_seeds: params.connect.is_some() || params.disable_dns_seeds,
        # network: params.network,
        # debug: params.debug,
        # cfilters: !params.no_cfilters,
        # proxy: params.proxy,
        # assume_utreexo: !params.no_assume_utreexo,
        # connect: params.connect,
        # wallet_xpub: params.wallet_xpub,
        # config_file: params.config_file,
        # #[cfg(unix)]
        # log_to_file: params.log_to_file || params.daemon,
        # #[cfg(not(unix))]
        # log_to_file: params.log_to_file,
        # assume_valid: params.assume_valid,
        # log_to_stdout: true,
        # #[cfg(feature = "zmq-server")]
        # zmq_address: params.zmq_address,
        # #[cfg(feature = "json-rpc")]
        # json_rpc_address: params.rpc_address,
        # generate_cert: params.generate_cert,
        # wallet_descriptor: params.wallet_descriptor,
        # filters_start_height: params.filters_start_height,
        # user_agent: env!("USER_AGENT").to_owned(),
        # assumeutreexo_value: None,
        # electrum_address: params.electrum_address,
        # enable_electrum_tls: params.enable_electrum_tls,
        # electrum_address_tls: params.electrum_address_tls,
        # tls_cert_path: params.tls_cert_path,
        # tls_key_path: params.tls_key_path,
        # allow_v1_fallback: params.allow_v1_fallback,
        # backfill: !params.no_backfill,
    };

    #[cfg(unix)]
    if params.daemon {
        // Daemonizing the process
        // ...
        # let mut daemon = Daemonize::new();
        # if let Some(pid_file) = params.pid_file {
            # daemon = daemon.pid_file(pid_file);
        # }
        # daemon.start().expect("Failed to daemonize");
    }

    let mut _log_guard: Option<WorkerGuard> = None;
    if config.log_to_stdout || config.log_to_file {
        let dir = &config.data_dir;

        // Initialize logging (stdout/file) per config and retain the file guard
        _log_guard = init_logging(dir, config.log_to_file, config.log_to_stdout, config.debug)
            .unwrap_or_else(|e| {
                eprintln!("Logging file couldn't be created at {dir:?}: {e}");
                exit(1);
            });
    }

    let _rt = tokio::runtime::Builder::new_multi_thread()
        // Setting the runtime
        // ...
        # .enable_all()
        # .worker_threads(4)
        # .max_blocking_threads(2)
        # .thread_keep_alive(Duration::from_secs(60))
        # .thread_name("florestad")
        # .build()
        # .unwrap();

    let signal = Arc::new(RwLock::new(false));
    let _signal = signal.clone();

    _rt.spawn(async move {
        // This is used to signal the runtime to stop gracefully.
        // It will be set to true when we receive a Ctrl-C or a stop signal.
        tokio::signal::ctrl_c().await.unwrap();
        let mut sig = signal.write().await;
        *sig = true;
    });

    let florestad = Florestad::from(config);
    _rt.block_on(async {
        florestad.start().await.unwrap_or_else(|e| {
            eprintln!("Failed to start florestad: {e}");
            exit(1);
        });

        // wait for shutdown
        loop {
            if florestad.should_stop().await || *_signal.read().await {
                info!("Stopping Floresta");
                florestad.stop().await;
                let _ = timeout(Duration::from_secs(10), florestad.wait_shutdown()).await;
                break;
            }

            sleep(Duration::from_secs(5)).await;
        }
    });

    # // drop them outside the async block, so we won't cause a nested drop of the runtime
    # // due to the rpc server, causing a panic.
    drop(florestad);
    drop(_rt);
    drop(_log_guard); // flush file logs on exit
}
```

Last of all, we build the `Florestad` instance, and call `Florestad::start` to start the node. From this point on, we keep checking for the stop signal. If it's found to be `true`, we call `Florestad::stop`, which simply replays the signal to the `UtreexoNode`, so it will "know" when to shut down (we saw how `UtreexoNode::new` required a kill signal in [Chapter 6.2](ch06-02-utreexonode-config-and-builder.md)).

### CLI Options

You can take a look at the `florestad/cli.rs` file to see the exact CLI arguments that are supported, or run `cargo run --bin florestad -- --help` to see the help message.

One particularly useful option is `--proxy`, which routes all P2P traffic, as well as DNS seed lookups, through a SOCKS5 proxy such as Tor:

```bash
# start the daemon with the Tor proxy
florestad --proxy 127.0.0.1:9050
```

You may also want to disable `Assume-Utreexo`, the UTXO set snapshot sync we discussed in [Chapter 5](ch05-00-advanced-chain-validation-methods.md#trusted-utxo-set-snapshots), which is enabled by default:

```bash
# start syncing from the genesis block
florestad --no-assume-utreexo
```

It's highly recommended to take a look at the [Floresta doc folder](https://github.com/getfloresta/Floresta/blob/master/doc), where many more details about building, running and testing Floresta are available.

### Florestad::start

As we have just mentioned, `Florestad` and its methods are implemented in the `floresta-node` "assembler" library. The `Florestad::start` method is where the node actually comes alive. It wires up the subsystems (watch-only wallet, the `UtreexoNode`, Electrum server, JSON-RPC server, etc.), and then spawns their asynchronous loops.

At a high level, the method:

1. Validates the data directory.
2. Loads the watch-only wallet.
3. Loads the blockchain database and optional compact filters.
4. Builds a `UtreexoNodeConfig` and starts the `UtreexoNode`.
5. Optionally starts ZMQ, JSON-RPC, and Electrum servers.
6. Spawns background tasks and returns.

Let’s walk through the important steps.

First, the data directory is ensured to exist and be writable, or else this library function returns error (note the `florestad` binary creates such directory).

```rust
# // Path: floresta-node/src/florestad.rs
#
/// Actually runs florestad, spawning all modules and waiting until
/// someone asks to stop.
///
/// This function will return an error if the configured data directory path is not an
/// **existing and writable directory**, or cannot be validated as such.
pub async fn start(&self) -> Result<(), FlorestadError> {
    let data_dir = &self.config.data_dir;

    // Check that the directory exists and is writable
    Florestad::validate_data_dir(data_dir)?;
```

Then the watch-only wallet is loaded, and most importantly, the blockchain database (as an `Arc<ChainState<FlatChainStore>>`):

```rust
let blockchain_state = Arc::new(Self::load_chain_state(
    data_dir.clone(),
    self.config.network,
    assume_valid,
)?);
```

If the compact filters feature is enabled, the filter store is initialized. With all these `Config` values, the `UtreexoNodeConfig` we saw at [Chapter 6.2](ch06-02-utreexonode-config-and-builder.md#utreexonodeconfig) is assembled and the `UtreexoNode` is created:

```rust
// Chain Provider (p2p)
let chain_provider = UtreexoNode::<_, RunningNode>::new(
    config,
    blockchain_state.clone(),
    Arc::new(tokio::sync::Mutex::new(Mempool::new(acc, 300_000_000))),
    cfilters.clone(),
    kill_signal.clone(),
    AddressMan::default(),
)
.map_err(|e| FlorestadError::CouldNotCreateChainProvider(format!("{e}")))?;
```

The `UtreexoNode` instance we create uses the `RunningNode` context. As we explained in [Chapter 6.1](ch06-01-node-contexts.md#default-floresta-contexts):

> `RunningNode` is the top-level context. When a `UtreexoNode` is first created, it will internally switch to the other contexts as needed, and then return to `RunningNode`. In practice, this makes `RunningNode` the default context used by `florestad` to run the node.

After that, optional subsystems like ZMQ, JSON-RPC, and Electrum are instantiated. Finally, we spawn the task that runs `UtreexoNode<_, RunningNode>::run`, which is the main loop of the node.

#### Summary

At this point, we've seen how the `florestad` binary works: it parses configuration, optionally daemonizes, and then calls `Florestad::start`, which orchestrates the watch-only wallet, the actual `UtreexoNode`, and optional servers.

In the next section, we'll look at the other binary in the Floresta project, `floresta-cli`: a lightweight command-line client that interacts with the daemon over JSON-RPC.
