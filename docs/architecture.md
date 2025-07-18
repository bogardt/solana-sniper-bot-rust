
<!-- banner line keeps SEO consistent -->
<!-- generated 2025‑07‑18 -->
<p align="center">
  <img src="assets/banner.svg" height="130" alt="Solana Sniper Bot">
</p>
# 🏗 Architecture Overview

**sniper_bot_gulf_stream** combines **Jito Gulf Stream** (mempool feed)
with a Rust async pipeline to copy trades in < 200 ms.

---

## 🔍 High‑Level Flow

```text
Gulf Stream gRPC  ─┐
▼
┌──────────────────────────────────────────┐
│  Sniffer (tokio task)                    │
│  • InitializeMint2 detector              │
│  • Dev‑wallet watcher                    │
└──────────────┬───────────────────────────┘
▼
┌──────────────────────────────────────────┐
│  Copy Engine                             │
│  • Pump.fun / Raydium swap helpers       │
│  • Adaptive tip calculation              │
│  • TradeState recorder                   │
└──────────────┬───────────────────────────┘
▼
┌──────────────────────────────────────────┐
│  PnL Monitor (per‑trade task)            │
│  • On‑chain price check every 15 s       │
│  • Trigger TP/SL sell bundles            │
└──────────────────────────────────────────┘
````

---

## 📁 Directory Tree  (2025‑07)

```text
src/
├─ main.rs            # entrypoint: config + tasks
├─ core/              # low‑level primitives
│  ├─ token.rs        # SPL helpers
│  ├─ tx.rs           # bundle sender
│  └─ price.rs        # on‑chain price derivation
├─ engine/            # where the magic happens
│  ├─ copier.rs
│  ├─ pumpfun.rs
│  ├─ raydium.rs
│  ├─ pnl_monitor.rs
│  └─ sniffer.rs
└─ monitor/           # Prometheus / tracing glue
```

*For deeper dives see **[Gulf Stream Primer](gulf-stream.md)**. *

````
