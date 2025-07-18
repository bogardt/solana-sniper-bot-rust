<!-- generated 2025‑07‑18  -->

# Understanding **Jito Gulf Stream** & Solana Priority Mechanics (2025)

> This primer explains **how Gulf Stream works**, why **priority fees**  
> (tips) matter, and how different Solana RPC providers expose—or
> throttle—the mempool.  
> Every concept is mapped to concrete Rust APIs (`jito-rs`,
> `jito-sdk-rust`, `yellowstone‑grpc-client`) so you can build and
> benchmark real‑world bots.

---

## Choosing your mempool provider 🚀

The bot lets you swap the ingest layer at runtime—just edit `.env`:

```dotenv
# "jito"        → Gulf Stream gRPC (default, sub‑200 ms)
# "yellowstone" → Helius Yellowstone gRPC feed (≈ +30 ms, no whitelist)
MEMPOOL_PROVIDER=jito
````

| Provider      | Transport / Auth             | Code module                         |
| ------------- | ---------------------------- | ----------------------------------- |
| `jito`        | gRPC + API key (rustls)      | `src/engine/sniffer.rs`             |
| `yellowstone` | gRPC TLS (public port 10000) | `src/engine/yellowstone_monitor.rs` |

Both providers output raw `VersionedTransaction` packets, so the rest of
the copy‑engine remains identical.

---

## 1 ⃣ What *is* Gulf Stream?

|                       | Classic Solana                | **Jito Gulf Stream**                             |
| --------------------- | ----------------------------- | ------------------------------------------------ |
| **Push path**         | Client ➜ Leader’s TPU         | Client ➜ **Block Engine** ➜ Leader TPU swarm     |
| **Packet validation** | on leader                     | *pre‑validated* in Block Engine (signature + CU) |
| **Searcher access**   | no direct mempool             | gRPC firehose: `subscribe_mempool_transactions`  |
| **Latency**           | 250‑700 ms (region‑dependent) | **80‑160 ms** median (FRA/NY)**\***              |

**\*** *Median RTT measured June 2025, Frankfurt BE ➜ Dublin validator.*

### Key components

| Component         | What it does                                                            |
| ----------------- | ----------------------------------------------------------------------- |
| **Block Engine**  | Aggregates txs, deduplicates, sorts by **Effective Priority Fee**.      |
| **Tip accounts**  | Random PDAs receiving lamport tips; rewards validators.                 |
| **Searcher gRPC** | Low‑latency stream of `Packet`s with full `VersionedTransaction` bytes. |
| **Bundle API**    | Allows multiple transactions (multi‑wallet) to land atomically.         |

---

## 2 ⃣ Priority Mechanics on Solana (2025 refresh)

| Priority lever              | Field                                                      | How to set in Rust                                                                                        |
| --------------------------- | ---------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| **Compute‑unit price**      | `ComputeBudgetInstruction::set_compute_unit_price(u64)`    | `rust\nlet ix = solana_sdk::compute_budget::ComputeBudgetInstruction::set_compute_unit_price(500_000);\n` |
| **Compute‑unit limit**      | `ComputeBudgetInstruction::set_compute_unit_limit(u32)`    | idem                                                                                                      |
| **Tip account lamports**    | 2nd tx in a **Jito bundle** transferring to random tip PDA | see `core::tx::send_bundle()`                                                                             |
| **Priority fee (JSON‑RPC)** | `sendTransaction(..., {"computeUnitPrice":X})`             | handled by `jito-sdk-rust`                                                                                |

> **Effective Priority Fee (EPF)** = `(compute_unit_price × compute_units) + tip_lamports`.

Block Engine sorts bundles by **slot**, then by **EPF**. Higher EPF
pretty much guarantees inclusion—provided it doesn’t exceed the
leader’s CU budget.

---

## 3 ⃣ Provider Landscape

| Provider                 | Gulf Stream?                           | Native WS Logs | Free TPS | Notes                                           |
| ------------------------ | -------------------------------------- | -------------- | -------- | ----------------------------------------------- |
| **Jito Labs**            | ✅ FRA/NY/TYO                           | limited        | 60       | Highest consistency; tip accounts auto‑rotated. |
| **Helius (Yellowstone)** | 🚧 (planned) but provides mempool gRPC | ✅              | 40       | Public gRPC port 10000; good backup.            |
| Triton One               | ❌                                      | ✅              | 30       | Fast WS, still 250 ms slower than BE.           |
| QuickNode                | ❌                                      | ✅              | 10       | Strict rate limits.                             |
| Ankr / Alchemy           | ❌                                      | ✅              | 10       | Good for token‑metadata fetch, not sniping.     |

**Takeaway** → Use **Gulf Stream** for sub‑200 ms; keep Yellowstone as a
public fallback.

---

## 4 ⃣ Connecting in Rust

### gRPC firehose (Jito variant)

```rust
use mev_protos::searcher::{
    searcher_service_client::SearcherServiceClient,
    SubscribeMempoolTransactionsRequest,
};
let mut client = SearcherServiceClient::connect(
    "https://frankfurt.mainnet.block-engine.jito.wtf"
).await?;
let mut stream = client
    .subscribe_mempool_transactions(SubscribeMempoolTransactionsRequest::default())
    .await?
    .into_inner();
while let Some(pkt) = stream.message().await? {
    let tx: solana_sdk::transaction::VersionedTransaction =
        bincode::deserialize(&pkt.transaction.unwrap().transaction)?;
    // analyse tx...
}
```

### gRPC firehose (Yellowstone variant)

```rust
use yellowstone_grpc_client::geyser::subscribe_update::UpdateOneof;
use yellowstone_grpc_client::GeyserGrpcClient;

let mut client = GeyserGrpcClient::build_from_static("http://grpc.helius.dev:10000")
    .build()
    .await?;
let mut stream = client.subscribe_slot().await?;

while let Some(update) = stream.message().await? {
    if let UpdateOneof::Transactions(tx_update) = update.update_oneof.unwrap() {
        // tx_update.txn contains VersionedTransaction bytes
    }
}
```

---

### Submitting a bundle with tip

```rust
use jito_sdk_rust::JitoJsonRpcSDK;
let sdk = JitoJsonRpcSDK::new(
    "https://mainnet.block-engine.jito.wtf/api/v1",
    None
);
let bundle = json!([
    [base64_swap_tx, base64_tip_tx],
    { "encoding":"base64" }
]);
sdk.send_bundle(Some(bundle), None).await?;
```

---

## 5 ⃣ Recommended Bot Workflow

1. **Subscribe** to mempool (`Sniffer` for Jito, `YellowstoneMonitor` for Helius).
2. **Filter** packets: Pump.fun `InitializeMint2`, watched wallets, Raydium pool init.
3. **Validate** mint authority, liquidity.
4. **Build bundle**: swap + tip.
5. **Submit** via `JitoJsonRpcSDK` or raw gRPC.
6. **Track** status via `getBundleStatuses` or fallback confirmation.

---

### Further Reading

* [Jito Docs → Block Engine](https://docs.jito.wtf/block-engine/)
* [“Low Latency Transaction Send”](https://docs.jito.wtf/lowlatencytxnsend/)
* [Yellowstone gRPC Docs](https://www.helius.dev/docs/data-streaming)
* [Priority Fee Markets on Solana (Anza Labs, 2025‑04)](https://anza.so/blog/priority-fee-markets)

---

Happy sniping — and remember: **tip responsibly**! 🚀

```

*Replace the entire content of `docs/gulf-stream.md` with this merged
version.*
::contentReference[oaicite:0]{index=0}
```
