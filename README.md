# RARB — Crypto Up/Down 5m, 15m Market Maker — High-Level Design (V5.3)

Lɪᴠᴇ Sʜᴏᴡ: https://polymarket.com/@celecula3?tab=activity 

DM ᴍᴇ ꜰᴏʀ ɪɴǫᴜɪʀɪᴇs ʀᴇɢᴀʀᴅɪɴɢ ᴘᴜʀᴄʜᴀsᴇ ᴏʀ ɪɴᴠᴇsᴛᴍᴇɴᴛ. Bᴏᴏᴋ ᴀ ᴄᴀʟʟ ʜᴇʀᴇ: https://calendly.com/endovix/30min

Describes the trading strategy in `RARB/strategy.py` (`run_state_machine`). Older narratives (dynamic hedge rounds, legacy entry constants) are **not** used by the live engine.

**References:** `RARB/config.py` (`BotConfig`), `RARB/settings.py` (`yaml_to_bot_config`), `RARB/state.py` (`BotState`), `RARB/runner.py` (paper loop), `main.py` (live loop).

---

## I. Market model

- **Instrument:** Polymarket **Crypto Up/Down** — two outcome tokens (**Up** / **Down**).
- **Books:** Best bid and ask per side.
- **Time:** Remaining seconds until the market window ends (`t_left`), from chain / resolver.

---

## II. Execution loop

- **Strategy tick:** The runner invokes `run_state_machine` on a fixed cadence (after maker fill handling where applicable).
- **Maker refresh:** The exchange adapter refreshes resting orders / simulates fills on its own interval (paper mode).
- **Concurrency:** `BotState` is updated under a lock; placement uses per–(side, side of book, price) in-flight guards to avoid duplicate orders.

---

## III. Phases (time buckets)

Behavior depends on how much time remains before expiry:

| Phase | Rough meaning |
|-------|----------------|
| **Liquidate** | Final window: stop normal trading and flatten / freeze. |
| **Close** | Pre-liquidation: defensive exits (e.g. tail traps) still allowed by design. |
| **Open** | Main trading window: entries and normal risk rules apply. |

Exact boundaries are **configurable** (death time and close buffer).

---

## IV. Priority stack (strict order)

Each tick runs **at most one** action group that **returns early**; lower priorities do not run in the same tick after a higher priority fires.

| Priority | Role |
|----------|------|
| **Frozen** | If trading is frozen, exit immediately. |
| **P0 — Liquidate** | In the liquidate phase: cancel orders, clear locks, freeze trading, log, **return**. |
| **P1 — Tail trap** | In the **close** phase only: on sides with inventory, optionally place defensive limit sells when the mark is weak enough (parameters control price and “weak” threshold). If any trap is placed, **return**. |
| **P2 — Stop-loss (IOC)** | If position exists and mark vs. average exceeds the stop rule, IOC sell that side (size and price floor from config). If any stop fires, **return**. |
| **P3 — Take-profit (IOC)** | If bids are strong enough and there is position, sell down using a **house-money** size rule tied to deployed capital; IOC with a bid floor slack. If any TP fires, **return**. |
| **P4 — Grid sweep** | If nothing above returned: when spread, risk, cooldown, reference band, and locks allow, place a **ladder of limit buys** on each side within total risk caps. |

Logged actions use **`V53_P0` … `V53_P4`**.

---

## V. P4 dual-sided sweep (core logic)

 Preconditions (conceptually):

1. Spreads on both books must be tight enough vs. a max-spread gate.
2. Aggregate exposure (`total_invest`) must stay under a global risk cap.
3. A cooldown must have elapsed since the last sweep that actually placed orders.

 Per **Up** and **Down** (independent):

- Skip invalid asks.
- Only sweep if the ask sits inside a configurable **reference band** (cheap-enough entry zone).
- **Base size** is jittered slightly for camouflage, then scaled by **tier**.
- For each tier: compute a limit buy price stepping down from the ask, skip prices that are too low, respect the risk cap per tier, acquire the in-flight lock for that price, place the buy. Tier size grows with tier index.

---

## VI. State (`BotState`)

- **Positions / accounting:** Per-side position and average; `total_invest`, costs, proceeds; **net capital deployed** (used for P3 sizing).
- **Control:** `trading_frozen`, per-price in-flight locks, last successful sweep timestamp.
- **Logging:** `cycle_events` (JSONL with `V53_P0`–`V53_P4`).

---

## VII. Configuration

All thresholds, bands, sizes, timings, and phase lengths are **YAML / `BotConfig`** — see `RARB/config.py` and per-pair files under `strategies/`. Merge order: global `config.yaml` `strategy:` then `strategies/{ASSET}_{TIMEFRAME}.yaml`.

---

## VIII. Programmer checklist

1. **Evaluation order** in code must stay **P0 → P1 (close only) → P2 → P3 → P4**.
2. **Per tick:** Higher priority can return before lower; P2/P3 may hit multiple IOC paths; P4 can emit many limits in one sweep.
3. **Live vs paper:** IOC behavior depends on the exchange adapter; see repo `README.md` for live caveats.

---

*Aligned with `RARB/strategy.py` (V5.3).*
