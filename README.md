# Malaysian SnR + Storyline Indicator

A TradingView (Pine Script v6) indicator that automates the **Malaysian Support & Resistance (SnR)** methodology — A-levels, V-levels, and Open-Close/Gap levels — filters them by freshness and higher-timeframe confluence, reads the market's **storyline** (trend structure, BOS/CHoCH), and prints suggested **entry, stop loss, and take profit** when a high-probability setup lines up.

> **Disclaimer:** this is a discretionary-trading concept translated into rules a script can follow. It is not financial advice, and no filter here guarantees a level will hold. Backtest and forward-test on demo before risking capital.

---

## 1. What it plots

| Element | Meaning |
|---|---|
| **Red line — A-level** | Resistance, formed at a peak in the close-to-close ("line chart") price path |
| **Lime line — V-level** | Support, formed at a valley in the close-to-close price path |
| **Gap level** | Formed between two same-colour consecutive candles, where candle 1's close doesn't meet candle 2's open |
| **Solid line** | Level is *fresh* — no wick has touched it yet |
| **Dashed line** | Level is *unfresh* — a wick has already tested it once |
| **Gray line** | Level is *broken* — a candle body closed through it (flipped SnR) |
| **BOS label** | Break of Structure — price closed beyond the last swing point *in the current trend's direction* (continuation) |
| **CHoCH label** | Change of Character — price closed beyond the last swing point *against* the current trend (possible reversal) |
| **Storyline label** (top-right, updates live) | Current read of market structure: `BULL`, `BEAR`, or `RANGE` |
| **Green/red triangle + label** | Long/short signal, with suggested Entry / SL / TP values |
| **Dotted lines** | Visual SL (red) and TP (green) projection from the signal bar |

---

## 2. How each level type is detected

### A-levels & V-levels
Malaysian SnR is drawn from a **line chart** (close-to-close), which ignores wicks. The script mirrors this by running pivot detection on `close` rather than `high`/`low`:

- `ta.pivothigh(close, pivotLen, pivotLen)` → a swing peak in the close series → **A-level**
- `ta.pivotlow(close, pivotLen, pivotLen)` → a swing trough in the close series → **V-level**

`pivotLen` bars are required on both sides before a pivot confirms — this is the standard trade-off between responsiveness (small value) and reliability (larger value).

### Open-Close / Gap levels
Formed when two **same-coloured** candles in a row leave a gap between candle 1's close and candle 2's open:

- Two bullish candles with a gap up → treated as a **support-type** gap level
- Two bearish candles with a gap down → treated as a **resistance-type** gap level

The level itself is placed at the midpoint between the two prices.

---

## 3. Fresh / Unfresh / Broken lifecycle

Every level is tracked as a stateful object (Pine `Level` type) for as long as it stays on the chart:

1. **Fresh** — created, never touched. Theory says fresh levels have the highest odds of a reaction.
2. **Unfresh** — a candle's wick has touched the level (`high >= price` and `low <= price` on the same bar), without a body close through it. Still tradeable, but considered a weaker level than the first touch.
3. **Broken** — a candle's **body** closes through the level. The level stops extending forward and is drawn in gray; it can act as a *flipped* level (old resistance → new support) if you want to extend the script to trade that.

This state machine runs every bar for every stored level (`updateArray()` in the code).

---

## 4. Storyline: how the trend/structure read works

The script tracks the two most recent confirmed swing highs and swing lows:

- **Higher High + Higher Low** → `trend = "bull"`
- **Lower High + Lower Low** → `trend = "bear"`
- Anything mixed → trend carries over from its last confirmed state (avoids flip-flopping on noise)

From there:
- **BOS (Break of Structure):** price closes beyond the last swing *in the trend's own direction* → confirms continuation.
- **CHoCH (Change of Character):** price closes beyond the last swing *against* the trend → the script flips `trend` to the new direction.

This is deliberately simple (2-swing structure). It's meant as a first-pass filter for "which side of the SnR levels should I even be looking to trade" — not a full replacement for discretionary structure reading (e.g. it doesn't currently weigh *how* a level formed or prior reaction quality at it).

---

## 5. Probability filter — what makes a level "high probability"

A level is only considered for a trade signal if:

1. **It's still fresh and unbroken.**
2. *(optional, `useHtf`)* **It has higher-timeframe confluence** — the script pulls the higher timeframe's own A/V pivots via `request.security()` and checks whether this level sits within `htfTolAtr × ATR` of one of them. A level agreed on by both timeframes is weighted as stronger, in line with how the methodology treats HTF-aligned levels.
3. **It's on the correct side of the current storyline** — only V-levels are considered for longs (in a bull storyline), only A-levels for shorts (in a bear storyline).

Levels that fail any of these are still drawn (so you can see the full map), but won't trigger a signal.

---

## 6. Entry / Stop Loss / Take Profit logic

**Long setup** (only evaluated while `trend == "bull"`):
- Trigger: the bar's low wicks into a fresh, high-probability V-level, and the candle closes back above it (a reaction, not a break).
- **Entry:** the closing price of the reaction candle.
- **Stop Loss:** the level's price minus `slBufferAtr × ATR` (a small buffer so a fresh level's ordinary wick noise doesn't stop you out immediately).
- **Take Profit:** the nearest unbroken A-level above price. If none is stored yet, it falls back to a fixed `rr` (reward:risk) multiple of the resulting stop distance.

**Short setup** mirrors this using A-levels, `trend == "bear"`, and the nearest V-level below price as the TP target.

Both are advisory labels/lines the script draws — it does **not** place trades. Pair it with `strategy.entry()`/`strategy.exit()` if you want to backtest it (see below).

---

## 7. Inputs reference

| Group | Input | Default | Purpose |
|---|---|---|---|
| SnR Levels | Pivot Lookback | 3 | Bars required each side to confirm a pivot (A/V level) |
| | Max levels stored per type | 25 | Caps memory/drawing objects; oldest levels drop off first |
| | Show A/V/Gap Levels | on | Toggle each level type |
| | Hide unfresh levels | off | Declutter the chart to only fresh levels |
| Probability Filter | Min distance from price (×ATR) | 0.5 | Reserved for filtering levels too close to be tradeable |
| | Require HTF confluence | on | Only flag levels confirmed on the higher timeframe |
| | Higher timeframe | 60 | Which HTF to pull pivots from |
| | HTF confluence tolerance (×ATR) | 0.5 | How close an LTF level must sit to a HTF level to count as confluent |
| Entry/SL/TP | ATR length | 14 | Used throughout for buffers and distance checks |
| | SL buffer (×ATR) | 0.25 | Extra room beyond the level for the stop loss |
| | Reward:Risk | 2.0 | Fallback TP multiple if no opposing level exists yet |
| | Plot entry signals | on | Master toggle for signal labels/shapes |

---

## 8. Known limitations / next steps

- Storyline detection uses only the last two swings — doesn't yet score *quality* of structure (e.g. equal highs, liquidity sweeps before BOS).
- Freshness is binary (fresh/unfresh) rather than a retest count — a level touched once and a level touched five times are currently treated the same once "unfresh."
- No position sizing, no strategy-mode backtest stats (win rate, R-expectancy) — this is an overlay indicator, not a `strategy()` script.
- Gap levels are simplified to the midpoint between the two candles; some variants of the methodology treat the full gap as a zone rather than a single line.

Natural next steps: convert to a `strategy()` script for backtesting, add a retest counter to strengthen the probability score, and extend the fresh→broken transition to auto-generate "flipped level" trades.
