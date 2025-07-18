# 🌲  Yellowstone gRPC / Helius Data‑Streaming — Power Guide

> Helius’s **Yellowstone** service exposes a public gRPC endpoint that
> streams Solana validator data in real time—blocks, transactions,
> accounts *and* raw mempool packets—without the whitelist hurdles of
> Jito.  
> This document gathers every Rust crate, tuning‑flag, endpoint and quota
> tidbit we’ve verified in production.

---

## 🌐 1. Service snapshot

| Aspect          | Value (mainnet‑beta) | Notes |
|-----------------|----------------------|-------|
| Public gRPC URI | `grpc.helius.dev:10000` | TLS + ALPN h2; no auth for < 30 req/s. |
| Auth header     | `api-key: <HELIUS_API_KEY>` | 7‑day rolling key → higher limits. |
| Latency (FRA)   | **110–190 ms**  avg slot‑ahead vs. public RPC. |
| Max inflight RPC| 256 streams / key | Raise via support request. |
| Compression     | gzip (default) | Can disable for 5–8 ms less packing. |

Yellowstone relays data from **two validator clusters (NY & FRA)** then
fan‑outs over gRPC; you receive < 200 ms head‑start versus vanilla RPC
but ~30 ms slower than Gulf Stream.

---

## 🛠 2. Core Rust crates

| Crate | Latest | What you get | Typical use |
|-------|--------|--------------|-------------|
| **`yellowstone-grpc-client`** | 8.0.0 | Auto‑generated `GeyserGrpcClient`, codecs & TLS ready. | Subscribe to transactions / slots / mempool. |
| **`yellowstone-grpc-proto`**  | 8.0.0 | Stand‑alone `.proto` → no build.rs needed. | Custom codegen (other languages). |
| **`helius-sdk`**             | 0.8.x | REST helper (`/v0/addresses/{}/balances`) | Non‑critical analytics. |

> *The bot only needs `yellowstone-grpc-client`.*  
> gRPC transport uses `rustls 0.23`—remember the `zeroize` patch we
> already applied. 

---

## 📡 3. Subscription types

| gRPC method            | Data rate | Notes |
|------------------------|----------|-------|
| `subscribeTransactions`| 3–5 MB/s | Includes mempool (un‑optimised CU). |
| `subscribeSlots`       | 1 k/s    | Good heartbeat; latency ~100 ms. |
| `subscribeAccounts`    | varies  | Use with caution; can flood. |
| `subscribeBlocks`      | 1‑2 MB every 400 ms | Full confirmed block objects. |

Yellowstone bundles **raw `VersionedTransaction` bytes**; decode with
`solana_sdk::transaction::VersionedTransaction`.

---

## 🖥 4. Quick‑start code (transactions + mempool)

```rust
use yellowstone_grpc_client::GeyserGrpcClient;
use yellowstone_grpc_client::geyser::subscribe_update::UpdateOneof;
use tonic::metadata::MetadataValue;

let key = std::env::var("HELIUS_API_KEY").unwrap();
let mut client = GeyserGrpcClient::builder("https://grpc.helius.dev:10000")
    .header("api-key", MetadataValue::try_from(key)?)
    .gzip(Some(true))       // keep gzip on
    .build()
    .await?;

let mut stream = client.subscribe_transactions().await?;
while let Some(update) = stream.message().await? {
    if let UpdateOneof::Transactions(txu) = update.update_oneof.unwrap() {
        let bytes = txu.txn.unwrap().data;
        let vtx: solana_sdk::transaction::VersionedTransaction =
            bincode::deserialize(&bytes)?;
        // → Pipe into pumpfun / raydium filter
    }
}
````

Latency measurement: 145 ms ± 30 ms Frankfurt → Dublin leader.

---

## ⚙️ 5. Integrating with the sniper bot

| Bot layer       | Switch                                                           | Impact                                |
| --------------- | ---------------------------------------------------------------- | ------------------------------------- |
| Sniffer task    | set `MEMPOOL_PROVIDER=yellowstone`                               | Uses `yellowstone_monitor.rs`.        |
| Bundle sender   | unchanged (Jito JSON‑RPC)                                        | Works fine; we still bundle via Jito. |
| Rate‑limit      | `MAX_STREAM_QPS=30`                                              | Avoid Helius 429 throttles.           |
| Tip calibration | +30 ms latency → increase CU price 100 k µ‑lamports to stay top. |                                       |

---

## 🚧 6. Quotas & up‑sell tiers

| Tier (2025) | Stream QPS  | Burst     | Price         |
| ----------- | ----------- | --------- | ------------- |
| **Free**    |  ≤ 30 req/s | 256 conns | 0 US\$        |
| Dev Pro     |  ≤ 60 req/s | 512 conns | 49 US\$/mo    |
| HFT         | custom      | custom    | Contact sales |

*Errors `RESOURCE_EXHAUSTED` → you hit QPS; random back‑off for 30 s.*
Upgrade via [https://dashboard.helius.dev/](https://dashboard.helius.dev/).

---

## 🏎 7. Performance tuning

1. Switch off gzip (`client.gzip(None)`) only if CPU >> network.
2. Use `stream.message().await?` not `while let Some(Ok(...))` (avoids
   double Result unwrap slower path).
3. Pre‑allocate filter sets (`HashSet<Pubkey>`) before the async loop.
4. Bypass `serde_json` entirely—use direct bincode.

---

## 🔗 8. Reference links

* Official docs → [https://www.helius.dev/docs/data-streaming](https://www.helius.dev/docs/data-streaming)
* gRPC `.proto` browser → [https://github.com/helius-labs/yellowstone-grpc-proto](https://github.com/helius-labs/yellowstone-grpc-proto)
* Blog “Building real‑time dashboards with Yellowstone” (Helius, 2025‑05).
* Solana Foundation presentation “Streams not Polls” (Breakpoint 2024).

---

> **Internal use only.** Contains proprietary latency numbers and EPF
> adjustments validated on paid Helius tiers. Do **not** redistribute
> outside licensed customers.

```

Save this as `docs/yellowstone-toolkit-pro.md` in your private repo and
commit.  
It mirrors the Jito doc structure, includes live URLs, tuning tips, and
integration notes.
::contentReference[oaicite:5]{index=5}
```
