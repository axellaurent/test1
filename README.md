# RSI Recovery Scanner — TradingView Pine Script v5

Automated scanner that detects a stateful RSI(14) recovery pattern on the 12H timeframe across 120 crypto and stock symbols. Sends an alert only when the full pattern completes on confirmed candles.

---

## Signal Logic

The system detects this exact four-stage sequence, per symbol:

1. RSI(14) closes **at or below 30** on a completed 12H candle → *oversold triggered*
2. RSI(14) closes **above 30** on the next completed 12H candle → *first recovery*
3. RSI(14) closes **above 30** on the following 12H candle → **alert fires**

After an alert fires, the symbol is in a "confirmed" state and will not alert again until RSI drops back to or below 30, at which point the full sequence resets.

### State Machine

| State | Meaning | Exits when… |
|---|---|---|
| **0** | Waiting | RSI ≤ 30 → State 1 |
| **1** | Oversold | RSI > 30 → State 2 ; RSI ≤ 30 → stay 1 |
| **2** | First recovery | RSI > 30 → **alert + State 3** ; RSI ≤ 30 → State 1 |
| **3** | Confirmed; waiting for reset | RSI ≤ 30 → State 1 ; otherwise stay 3 |

State 3 resets to State 1 (not 0) because RSI dropping back below 30 immediately satisfies the oversold condition.

### Table Color Guide

| Color | State |
|---|---|
| Gray | 0 — Waiting |
| Red | 1 — Oversold |
| Orange | 2 — Recovering (first candle) |
| Green | 3 — Confirmed / completed |

---

## Files

| File | Purpose | Symbols | Requests |
|---|---|---|---|
| `phase1_single_symbol.pine` | Validate state machine on one symbol | Chart symbol | 1 |
| `phase2_basket.pine` | Validate multi-symbol independence | 5 hardcoded | 5 |
| `phase3_scanner_template.pine` | Configurable template (up to 35 inputs) | up to 35 | up to 35 |
| `scanner_binance_majors.pine` | Binance major USDT pairs | 20 | 20 |
| `scanner_binance_alts.pine` | Binance alt USDT pairs | 20 | 20 |
| `scanner_okx.pine` | OKX USDT pairs | 15 | 15 |
| `scanner_kraken.pine` | Kraken USD pairs | 15 | 15 |
| `scanner_stocks_1.pine` | US tech stocks | 15 | 15 |
| `scanner_stocks_2.pine` | US finance/health/industrial stocks | 15 | 15 |

**Total coverage: 120 symbols across 6 production scanners.**
All scripts stay within the TradingView free plan limit of 40 `request.security()` calls per script.

---

## Recommended Build Sequence

Follow this order. Validate each phase before proceeding.

### Phase 1 — Single symbol validation

1. Open TradingView → navigate to `BINANCE:BTCUSDT` on any chart (e.g. 1H or 4H).
2. Open the Pine Script editor → paste the contents of `phase1_single_symbol.pine`.
3. Click **Add to chart**.
4. Use **Bar Replay** to rewind to a date when BTC RSI was below 30 (e.g. June 2022).
5. Step forward bar by bar and confirm:
   - Background turns **red** when RSI ≤ 30 (State 1)
   - Background turns **orange** on the first recovery candle (State 2)
   - Background turns **green** on the second consecutive recovery candle (State 3)
   - An alert message appears in the Data Window on the State 2→3 bar
   - No alert fires on the State 1→2 bar
6. Check edge cases using the table below.

#### Edge cases to verify in Bar Replay

| Scenario | RSI sequence | Expected states |
|---|---|---|
| Clean recovery | 45→28→32→35 | 0→1→2→3 (alert on 35 bar) |
| Double dip (State 2 reversal) | 45→28→32→27→33→36 | 0→1→2→1→2→3 |
| Extended oversold | 28→25→22→31→34 | 1→1→1→2→3 |
| Post-alert reset | (alert fired)→45→25→32→35 | 3→3→1→2→3 |
| RSI exactly at 30 | RSI=30.00 | treated as ≤30; stays in State 1 |

### Phase 2 — Multi-symbol basket

1. Load `phase2_basket.pine` on any chart.
2. Confirm the status table in the top-right shows all 5 symbols with independent states.
3. Cross-reference BTC and ETH states by also loading `phase1_single_symbol.pine` on those charts separately. States must match.
4. Confirm alerts identify the correct symbol name.

### Phase 3 — Template

Load `phase3_scanner_template.pine`, fill in Symbol 01–05 with symbols you have verified in Phase 2, and confirm the table matches Phase 2 output. This validates the input-string approach used by the template.

### Phase 4 — Production scanners

1. Load each of the six production scanner scripts on any chart (1H recommended).
2. Check that no "too many security calls" error appears in the Pine editor.
3. Compare BTC/ETH states against Phase 2 results — they must match.
4. Set up live alerts (see below).

---

## Setting Up Alerts in TradingView

Each script uses `alert()` (not `alertcondition()`), which means:

1. Load the script on a chart.
2. Click the **Alerts** clock icon in the toolbar, or press `Alt+A`.
3. In the **Condition** dropdown, select the script name.
4. Under the condition, select **"Any alert() function call"**.
5. Set **Once Per Bar Close** as the trigger (this is enforced in code too, but belt-and-suspenders).
6. Set **Expiration** to **Open-ended alert**.
7. Choose notification method: app push notification, email, or webhook URL.
8. Click **Create**.

Repeat this for each of the six production scanner scripts. You will have 6 active alerts total — one per scanner batch.

### Webhook (optional)

To route alerts to Telegram, Discord, or a custom system:

1. Set up a webhook endpoint (e.g. using a free service like ntfy.sh, or a custom server).
2. In the TradingView alert dialog, select **Webhook URL** and paste the endpoint.
3. The alert message body will be sent as a JSON payload to the endpoint.

---

## Alert Message Format

```
RSI Recovery | BINANCE:BTCUSDT | 12H | RSI: 31.47 | Vol(12H): $2.34B | SIGNAL: Confirmed Recovery | Bar: 2024-03-15 12:00 UTC
```

Fields:

| Field | Description |
|---|---|
| Symbol | Full TradingView identifier (EXCHANGE:TICKER) |
| Timeframe | Always "12H" |
| RSI | RSI(14) value at the moment the alert fires, formatted to 2 decimal places |
| Vol(12H) | USD volume of the triggering 12H candle (price × volume), display only |
| SIGNAL | Always "Confirmed Recovery" — the second consecutive candle above 30 |
| Bar | UTC timestamp of the bar open (= close of the confirmed 12H candle) |

Volume is informational only. It is not used as an alert gate.

---

## Symbol Lists

### Binance Majors (scanner_binance_majors.pine)
BTCUSDT, ETHUSDT, BNBUSDT, SOLUSDT, XRPUSDT, ADAUSDT, DOGEUSDT, AVAXUSDT, TRXUSDT, DOTUSDT, MATICUSDT, LTCUSDT, BCHUSDT, UNIUSDT, ATOMUSDT, ETCUSDT, XLMUSDT, NEARUSDT, FILUSDT, AAVEUSDT

### Binance Alts (scanner_binance_alts.pine)
LINKUSDT, APTUSDT, ARBUSDT, OPUSDT, SUIUSDT, SEIUSDT, INJUSDT, TIAUSDT, RNDRUSDT, FTMUSDT, ALGOUSDT, VETUSDT, ICPUSDT, HBARUSDT, EGLDUSDT, THETAUSDT, SANDUSDT, AXSUSDT, CHZUSDT, GALAUSDT

### OKX (scanner_okx.pine)
BTCUSDT, ETHUSDT, SOLUSDT, XRPUSDT, DOGEUSDT, ADAUSDT, AVAXUSDT, DOTUSDT, LINKUSDT, UNIUSDT, ATOMUSDT, NEARUSDT, APTUSDT, ARBUSDT, OPUSDT

### Kraken (scanner_kraken.pine)
XBTUSD, ETHUSD, SOLUSD, XRPUSD, ADAUSD, DOTUSD, AVAXUSD, ATOMUSD, LINKUSD, UNIUSD, LTCUSD, BCHUSD, DOGEUSD, NEARUSD, FILUSD

### US Tech Stocks — Batch 1 (scanner_stocks_1.pine)
AAPL, MSFT, NVDA, GOOGL, AMZN, META, TSLA, AVGO, AMD, QCOM, INTC, CRM, TSM, ORCL, ASML

### US Stocks — Batch 2 (scanner_stocks_2.pine)
JPM, BAC, GS, MS, WFC, JNJ, UNH, PFE, MRNA, ABT, BA, CAT, GE, HON, LMT

---

## Exchange Naming Conventions

| Exchange | BTC ticker | Quote currency | Example |
|---|---|---|---|
| Binance | BTCUSDT | USDT | `BINANCE:BTCUSDT` |
| OKX | BTCUSDT | USDT | `OKX:BTCUSDT` |
| Kraken | **XBTUSD** | **USD** (not USDT) | `KRAKEN:XBTUSD` |
| NASDAQ | — | USD | `NASDAQ:AAPL` |
| NYSE | — | USD | `NYSE:JPM` |

**Kraken gotcha**: Kraken lists Bitcoin as XBT (not BTC) and uses USD, not USDT. If any Kraken symbol shows "n/a" in the table, open TradingView's symbol search, select Kraken as the exchange, and verify the exact ticker string.

---

## TradingView Plan Requirements

All six production scanners use ≤ 20 `request.security()` calls each, well below TradingView's limit of 40 per script on all plan tiers including free.

| Plan | Security call limit | Sufficient for this project? |
|---|---|---|
| Free | 40 | Yes |
| Pro | 40 | Yes |
| Pro+ | 40 | Yes |
| Premium | 40 (64 on v6 professional) | Yes |

TradingView Premium supports up to 400 price alerts and 400 technical alerts, so 6 active alerts is not a constraint.

**Stock real-time data**: TradingView's base plan includes delayed stock data (15 min delay for US stocks). Real-time data for NASDAQ and NYSE requires adding the relevant exchange data subscriptions (available from your TradingView account settings under "Additional data").

---

## Known Limitations

1. **State is not persistent across script reloads.** If you remove and re-add a scanner script, state resets to 0 for all symbols. The script will not know about oversold conditions that occurred before the current chart session. This is a Pine Script platform limitation — state lives in memory, not on disk.

2. **12H bar alignment.** Crypto 12H bars align at 00:00 UTC and 12:00 UTC. Stock 12H bars follow exchange session calendars. The alert timestamp reflects the UTC open of the new 12H bar, which equals the confirmed close of the previous one.

3. **Kraken symbol names.** Kraken uses non-standard ticker symbols (XBT for BTC, no USDT suffix). Verify each Kraken symbol exists in TradingView before relying on it.

4. **Intra-bar chart updates.** The scripts gate all state transitions on `ta.change(time("720")) != 0`, which is only true on the first tick of a new 12H bar. Visuals on the chart (e.g. the status table RSI values) update continuously, but alerts and state changes only happen at the confirmed bar boundary.

5. **`str.format_time()` requires a recent Pine Script v5 version.** This function was added to Pine Script v5 in a 2022–2023 update. TradingView auto-updates Pine, so this should not be an issue on any active account.

---

## Customizing the Symbol List

To add or swap symbols in a production scanner:

1. Open the scanner script in the Pine editor.
2. Change the string literal in the matching `request.security()` call to the new symbol.
3. Update the same symbol string in the matching `state_transition(...)` call.
4. Update the display ticker in the `tickers` array in the Status Table section.
5. Save and re-add to chart.

To use the template (`phase3_scanner_template.pine`) for a custom list, simply fill in the input fields when the indicator is loaded on the chart — no code editing required.

---

## Quick Reference: Adding a New Scanner Batch

1. Copy any existing production scanner file (e.g. `scanner_binance_majors.pine`).
2. Replace all symbol strings with your new list (up to 20 symbols before adjusting array sizes).
3. Adjust the `var int s01...` declarations to match the count.
4. Adjust the `for i = 0 to N` loop and `tickers`/`sts`/`rsis` arrays.
5. Update the table row count in `table.new(...)` (header row + N data rows).
6. Load on chart, verify no errors, set up alert.
