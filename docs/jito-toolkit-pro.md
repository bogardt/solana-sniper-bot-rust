# 📚 The **Ultimate Jito Rust Toolkit** (internal edition)
> A deep reference for every publicly available Jito crate, gRPC
> service, helper repo, and block‑engine trick we’ve found.  
> Use this as your “one‑stop” lookup when extending or debugging the
> commercial sniper bot.
---

## 🌐 1. Overview of the Jito stack

| Layer | Service / Repo | Notes |
|-------|----------------|-------|
| **Block Engine** | `block_engine` (closed‑source) | Validates packets, runs state & global auctions, returns bundle status. |
| **Gulf Stream gRPC** | Exposed via `SearcherService` (see `jito-rs`) | Firehose of mempool packets before RPC nodes see them. |
| **JSON‑RPC façade** | `/api/v1`“Bundle RPC” (see `jito-rust-rpc`) | Thin wrapper around gRPC; easier to call from browsers or scripts. |
<br>

All public Rust crates are **thin clients** to one of those layers:
<br>
`jito-sdk-rust` ↔ JSON‑RPC; `jito-rs` ↔ gRPC; `mev-protos` ↔ schema
---
<br>

## 🛠 2. Core Rust crates

| Crate | Latest | What you get | Typical latency |
|-------|--------|--------------|-----------------|
| **`jito-sdk-rust`** | 0.2.1 | JSON‑RPC helpers: `sendBundle`, `getTipAccounts`, status calls. | ~250 ms (HTTP + JSON) |
| **`jito-rs`** | Git‑master | gRPC `SearcherServiceClient`, `BlockEngineClient` (tokio/tonic). | 120–160 ms |
| **`mev-protos`** | Git‑master | Raw `.proto` schemas → generate any language. | depends |
| **`shredstream-proxy`** | 0.1.x | Streams raw Solana shreds over WebSocket/gRPC. | < 100 ms | :contentReference[oaicite:1]{index=1}
<br>

> **Tip** — You can compile `mev-protos` with
> ```rust
> tonic_build::configure().compile(&["protos/mev/searcher.proto"], &["protos"])?;
> ```
---
<br>

## 🧩 3. Support repos & playgrounds

| Repo | Why it’s useful |
|------|-----------------|
| **`searcher-examples`** | Auth flow, bundle loop, CLI playground. |
| **`block_engine_simple`** | Mock block‑engine for local integration tests. :contentReference[oaicite:2]{index=2} |
| **`jito-js-rpc`** | TypeScript SDK; good for front‑end dashboards. :contentReference[oaicite:3]{index=3} |
---
<br>

## ⚙️ 4. Implementation patterns

### 4.1 Subscribe to Gulf Stream (gRPC)

```rust
use mev_protos::searcher::{
    searcher_service_client::SearcherServiceClient,
    SubscribeMempoolTransactionsRequest,
};
let mut cli = SearcherServiceClient::connect(BE_ENDPOINT).await?;
let req = SubscribeMempoolTransactionsRequest::default();
let mut stream = cli.subscribe_mempool_transactions(req).await?.into_inner();
while let Some(pkt) = stream.message().await? {
    let tx: solana_sdk::transaction::VersionedTransaction =
        bincode::deserialize(&pkt.transaction.unwrap().transaction)?;
    // → inspect instructions, compute_units, etc.
}
````

<br>
> 🚀 **Speed delta**  
> 
> ```
>  Public RPC  ▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒  0 ms
>  Gulf Stream ████████████▌  –160 ms
> ```
> *A typical Packet from Gulf Stream reaches us **≈ 160 ms** before it
> shows up on public RPC nodes.*
> 
> [Low‑Latency Transaction Send ▸ Jito Docs](https://docs.jito.wtf/lowlatencytxnsend/)


<br>
See Jito docs → *Low Latency Transaction Send*. ([docs.jito.wtf][1])
<br>


### 4.2 Build & send a bundle (JSON‑RPC)

```rust
let sdk = jito_sdk_rust::JitoJsonRpcSDK::new(BE_RPC, None);
let txs = json!([ [b64_swap_tx, b64_tip_tx], { "encoding":"base64" } ]);
sdk.send_bundle(Some(txs), None).await?;
```

<br>
Up to 5 transactions, executed sequentially & atomically in one slot.
<bt>
Jito rejects bundles >0.1 SOL total tip. ([docs.jito.wtf][1])
<bt>


---
<br>

## 💸 5. Effective Priority Fee (EPF) formula

```
EPF = (compute_unit_price × compute_units) + tip_lamports
```

*State‑auction* sorts by EPF. Recommended starting values:

| Scenario        | CU price (µ‑lamports) | Tip (lamports)    |
| --------------- | --------------------- | ----------------- |
| Pump.fun launch | 800 000–1 200 000     | 500 000–3 000 000 |
| Copy whale buy  | 500 000–800 000       | 400 000–2 000 000 |
| TP / SL exit    | 200 000–400 000       | 0                 |

---

## 🚧 6. Common errors & fixes

| gRPC `BundleResult`       | Meaning                      | Fix                                                                          |
| ------------------------- | ---------------------------- | ---------------------------------------------------------------------------- |
| `StateAuctionBidRejected` | Someone out‑tipped globally. | Increase tip 25–50 %.                                                        |
| `SimulationFailure`       | Slippage / CU exceeded.      | Check raydium price; add `ComputeBudgetInstruction::set_compute_unit_limit`. |
| No result in 5 s          | Channel idle.                | Ensure API key valid; Gulf Stream firewall may block.                        |

*Reference: Jito BE error codes (docs)…* ([docs.jito.wtf][1])

---

## 🏎 7. Latency optimisation checklist

1. **Co‑locate** VM in `fra1.digitalocean` or `ny5.equinix`.
2. Use **UDP QUIC** when Jito exposes it (beta Q3 2025).
3. Pre‑sign transactions; only mutate tip lamports field per bundle.
4. Run `RUSTFLAGS="-C target-cpu=native -C opt-level=3"`.

---

## 🔗 8. Further reading & bookmarks

* Jito Docs → [https://docs.jito.wtf/](https://docs.jito.wtf/) ([docs.jito.wtf][2])
* Low‑Latency guide → [https://docs.jito.wtf/lowlatencytxnsend/](https://docs.jito.wtf/lowlatencytxnsend/) ([docs.jito.wtf][1])
* QuickNode tutorial on Rust bundles (good code snippets) → [https://www.quicknode.com/guides/solana-development/transactions/jito-bundles-rust](https://www.quicknode.com/guides/solana-development/transactions/jito-bundles-rust) ([QuickNode][3])
* Shredstream intro talk (Solana Breakpoint 2024) → *YouTube “Shreds for Searchers”*
* State‑auction design paper (Jito Foundation, Apr 2025) → PDF in `/docs/papers/`.

---

> **Internal use only.** This document aggregates public information but
> also includes proprietary tuning values tested in production. Do **not**
> redistribute outside licensed clients.

```
