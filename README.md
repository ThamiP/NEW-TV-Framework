# NEW-TV-Framework · ICT Fractal Models

A TradingView (Pine Script v6) framework that runs ten ICT entry models on one shared engine:

```
liquidity sweep ─► shift (MSS / CISD) ─► PD array armed ─► tap ─► confirmation close ─► market entry
```

Design rules:

- **No entry limit orders.** A PD array is never an order. Price has to trade into it (the *tap*), and the entry is the **close of the confirmation candle**, taken at market. Stop and target are exit orders only.
- **No SMT, no sessions.** There are no correlated-asset checks, killzones, Silver Bullet windows or clock times anywhere in the logic.
- **Fractal.** Every rule is measured in bars and ATR, never in time. The HTF is chosen relative to the chart (auto ladder), so the same models run unchanged from seconds charts to monthly.

## Files

| File | What it is |
| --- | --- |
| `ICT_Fractal_Models.pine` | The indicator and the single source of truth. Draws structure, PD arrays and signals, tracks every trade, and shows the dashboard and checklist. |
| `ICT_Fractal_Models_Strategy.pine` | Generated strategy build for the TradingView Strategy Tester. Market entry on the signal close, risk-based sizing. |
| `tools/build_strategy.py` | Regenerates the strategy from the indicator (`--check` verifies it is current). |

## Install

1. TradingView → Pine Editor → **New** → paste `ICT_Fractal_Models.pine` → **Add to chart**.
2. For backtests, paste `ICT_Fractal_Models_Strategy.pine` the same way and open the Strategy Tester.
3. Alerts: *Create alert* → condition **ICT Fractal Models** → pick **Any alert() function call** to get a message with the model, entry, stop and target. You can also pick one of the fixed conditions: `Long`, `Short` or `Any signal`.

## The models

Each model is one row in the dashboard and can be toggled on or off. When several models confirm on the same candle they share one label, for example `▲ M1·M3·SB`.

| Tag | Model | Sequence (long; shorts mirror it) | From |
| --- | --- | --- | --- |
| `TS` | **Turtle Soup** | External SSL is swept and reclaimed within the TS window (false break) → MSS → the retrace runs the internal low formed after the shift (`$`) → entry on the close back above `$`. | ICT Turtle Soup |
| `SB` | **Sweep → FVG** (Silver Bullet, no session) | SSL sweep → the displacement off the sweep low prints a bullish FVG → tap → confirmation. | ICT Silver Bullet |
| `UNI` | **Unicorn** | A bearish OB fails after its stop run (within `Unicorn max bars OB → breaker`) and becomes a bullish breaker → a bullish FVG from the same displacement overlaps the breaker → tap the overlap → confirmation. | ICT Unicorn |
| `M1` | **HTF POI + Shift + FVG** | HTF liquidity grab / HTF POI → MSS → FVG of the shift leg reaching into discount → tap → confirmation. | Model 1 |
| `M2` | **HTF POI + Shift + IDM + FVG** | Like M1, plus an internal low (IDM) that forms above the FVG after the shift. The IDM must be swept before the FVG is tapped. | Model 2 |
| `M3` | **HTF POI + Shift + FVG + OTE** | Like M1, where the FVG overlaps the 0.62–0.79 OTE of the shift leg and price reaches 0.62. | Model 3 |
| `M4` | **Box Setup** | Consolidation range → sweep of the range low (EQL) → close back inside → price leaves the range upward → retest of the range → confirmation. | Model 4 |
| `iFVG` | **iFVG Inversion** | SSL sweep → a bearish FVG is closed through and inverts → retest of the iFVG → confirmation. | Inversion FVG / SSL raid charts |
| `OB+FVG` | **OB + FVG Convergence** | After the shift, the OB that launched the displacement sits directly under a shift-leg FVG → price reaches the OB → confirmation. | Bearish OB / FVG convergence chart |
| `CONT` | **FVG Continuation** (off by default) | After the shift leg, each new FVG while the leg delivers toward BSL → tap → confirmation. The stop goes beyond the FVG's first candle. | Continuation FVG → BSL chart |

The "HTF POI" filter is on by default for **M1–M4**, matching their checklists. A setup passes when its sweep leg traded into an HTF FVG / OB on the trade's side, or took the previous HTF candle's high / low. It can be applied to every model, or switched off (**Higher Timeframe → Require HTF POI / liquidity**).

"Good R:R" is enforced for every model: the target has to pay at least **Min R:R** (default 2). With the `Nearest liquidity` target, a setup whose closest pool pays less is skipped.

## Building blocks

| Block | Definition (all bar / ATR based) |
| --- | --- |
| Liquidity | External swing pivots (`External swing length`). Pools closer than the tolerance merge into EQH / EQL. The previous HTF candle's high / low is also a pool. |
| Sweep | A bar trades through an untaken pool. The deepest pool taken becomes the swept level. The sweep extreme keeps extending until the shift. |
| False break | A close back inside the swept level within `Turtle Soup reclaim window` bars. |
| MSS | A close through the internal swing (`Internal swing length`) that delivered price into the sweep extreme. |
| CISD | A close through the open of the run of opposite candles that delivered price into the extreme. |
| FVG / iFVG | A 3-candle gap ≥ `Min FVG size × ATR` with a displacement middle candle. An FVG closed through becomes an iFVG of the opposite side. |
| OB / Breaker | The last opposite-close candle at the extreme of a move that broke internal structure. An OB closed through becomes a breaker; the stop run it made first becomes the Unicorn stop. |
| Consolidation | The last `M4 consolidation length` bars fit inside `M4 max height × ATR`. |
| HTF POI | Confirmed HTF FVGs and HTF OBs (the down / up candle that starts the HTF FVG). Requested with `[1]` offsets and `lookahead_on`, so they never repaint. |

### Auto HTF ladder

| Chart | HTF |
| --- | --- |
| seconds | 5m |
| 1m | 15m |
| ≤ 5m | 1H |
| ≤ 30m | 4H |
| ≤ 1H | D |
| ≤ 4H | W |
| ≤ D | M |
| ≤ W | 3M |
| above | 12M |

Switch to **Manual** to pick your own. A manual HTF that is not higher than the chart falls back to the ladder.

## Entries, stops and targets

- **Tap depth:** `Zone edge` (price touches the near edge) or `CE (50%)`.
- **Confirmation** (after the tap; the entry is that candle's close):
  - `CISD` *(default)*: closes through the open of the delivery into the tap.
  - `Rejection close`: closes back out of the zone in the trade direction.
  - `Close in direction`: any candle closing in the trade direction without closing through the zone.
  - Turtle Soup always confirms on its reclaim close.
- **Confirmation window:** if nothing confirms within `Confirmation window` bars of the tap, the tap is discarded. The next touch starts a fresh one, and CISD is measured from the new delivery. This stops stale taps from entering far from the PD array.
- **Invalidation:** the plan dies if price trades through its sweep extreme, closes through the zone, reaches the zone before its precondition (M2 IDM), or expires after `Plan expiry` bars.
- **Stop:** `Sweep extreme` *(default)* or `PD array edge` (beyond the zone or the tap wick, whichever is further), plus `Stop buffer × ATR`.
- **Target:** `Liquidity ≥ min R:R` *(default: the closest untaken pool that pays at least the min R:R)*, `Nearest liquidity` (the closest pool; the trade is skipped when it pays less than the min R:R), or `Fixed R`. When no pool exists beyond the entry, the leg extreme or `Fixed R` is used.

## Dashboard

- **Per model:** trades, win %, Σ R and live armed plans (▲ / ▼).
- **Live checklist** per side: liquidity grab, false break, HTF POI / liquidity, shift, PD array armed, tapped and awaiting the confirmation close.

The indicator's stats are an approximation. If stop and target print on the same candle it counts a loss, and it ignores spread and slippage. Use the strategy build in the Strategy Tester for fills, commission and slippage.

## Drawings

TradingView keeps at most 500 labels, lines and boxes of each type, and deletes the oldest first. History drawings go through capped queues instead: the last 200 signals, 150 structure labels, 200 structure lines and 250 history boxes are kept. That way live PD arrays, HTF POIs and recent signals are never the ones removed.

## Repainting

- The engine runs only on **confirmed bars** (`barstate.isconfirmed`), so signals and alerts never appear intrabar.
- Pivots are used only once they are confirmed. Pool lines are drawn back to the pivot for readability, but a pool can only be swept after it exists.
- HTF data uses closed HTF candles only.

## Strategy build

The indicator carries two markers:

- `//@S <code>`: strategy-only line (uncommented in the build).
- `<code> // @ind`: indicator-only line (commented out in the build).

After editing the indicator:

```bash
python3 tools/build_strategy.py          # regenerate ICT_Fractal_Models_Strategy.pine
python3 tools/build_strategy.py --check  # fail if the strategy file is stale
```

The strategy enters only when flat: one position at a time, and when longs and shorts confirm on the same candle it skips. It uses `process_orders_on_close = true`, so fills match the indicator's close-based entries. Position size risks `Risk per trade (% of equity)` between entry and stop.
