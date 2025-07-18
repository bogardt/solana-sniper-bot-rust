<p align="center">
  <img src="docs/banner.svg" height="130" alt="Solana Sniper Bot">
</p>

<div align="center">

[![CI](https://img.shields.io/github/actions/workflow/status/bogardt/solana-sniper-bot/ci.yml?label=build)](https://github.com/bogardt/solana-sniper-bot/actions)
![Rust async](https://img.shields.io/badge/Rust-async-93450B?logo=rust)
![Solana 1.18](https://img.shields.io/badge/Solana-1.18-purple)
![Made&nbsp;with&nbsp;❤](https://img.shields.io/badge/Made_with-❤-ff69b4)

</div>

> **Solana Sniper Bot** is a lightning‑fast copy‑trading / MEV engine that
> mirrors Pump.fun launches and whale transfers in **under 150 ms**.  
> Mempool via Yellowstone gRPC ➜ atomic Jito bundles ➜ dynamic TP / SL.

---

## 🗂 Documentation Quick‑Links

| 📄 Doc | 👉 What you’ll find |
|--------|--------------------|
| **[Architecture Overview](docs/architecture.md)** | Async flow, directory tree, bundle pipeline |
| **[Gulf Stream & Priority Primer](docs/gulf-stream.md)** | How Jito BE sorts bundles, tip strategy |**
| **[Advanced Features](docs/advanced-features.md)** | Rate‑limit shield, multi‑wallet rotation, metrics** |
| **[Jito Rust Toolkit](docs/jito-github-libs.md)** | Which crate for which task (`jito-rs`, `mev-protos`, …) |
| **[Price / Risk Settings](docs/price-settings.md)** | Where to tweak TP / SL %, dev‑size mirroring |

---

## ⚡ Quick Demo

```bash
git clone --recurse-submodules https://github.com/bogardt/solana-sniper-bot.git
cd solana-sniper-bot
cp .env.example .env   # fill RPC_URL, KEYPAIR, etc.
cargo run --example demo_swap
````

The demo buys a test token on devnet and shows the bundle pipeline with
logs only (no private heuristics).

---

## ✨ Highlight Features

| 🚀 Speed Edge                | 🌊 Gulf Stream Native   | 🛡 Smart Risk Control                 | 📦 Atomic Bundles              |
| ---------------------------- | ----------------------- | ------------------------------------- | ------------------------------ |
| Rust binary `-C opt-level=3` | gRPC firehose (<200 ms) | trailing TP/SL, per‑wallet rate‑limit | tip‑aware multi‑wallet bundles |

<details>
<summary>More…</summary>

* **Position sizing** – `dev_buy × 0.20` (cap by `MAX_COPY_SOL`) – see [`price-settings.md`](docs/price-settings.md).&#x20;
* **PnL Daemon** polls on‑chain vaults every 15 s.
* **Prometheus metrics** with feature flag `metrics`.
* **Sub‑module patch**: vendored `curve25519‑dalek` accepts `zeroize 1.8` (TLS ready).

</details>

---

## 🏗 Build & Deploy

1. **Build and run**
```
cargo build --release 
RUST_LOG=info ./target/release/sniper_bot_gulf_stream

    ```

2. **Colocate** binary on FRA or NYC metal for <90 ms inclusion.

3. Run with minimal flags:

    ```bash
   RUST_LOG=info ./target/release/sniper_bot_gulf_stream
    ```

---

## 💬 Community & Commercial Edition

* Telegram → **@bogardt**
* Twitter/X → [@toptrendev](https://x.com/toptrendev)
* Discord → `TopTrenDev#146`

**Want the full ultra‑low‑latency engine?**
DM for single‑user licences (private repo + EULA).

---

**License**: MIT (demo edition). Commercial licence available for the full build.

