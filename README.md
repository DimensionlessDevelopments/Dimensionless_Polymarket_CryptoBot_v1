# Dimensionless_Polymarket_CryptoBot_v1
Polymarket Contrarian Crypto Bot V1 is an interactive Jupyter notebook that walks through prediction market of Polymarket. It keeps the original goal; scan crypto markets, inspect order book structure, compute a contrarian z-score signal, and place trades; but is designed for manual exploration and research rather than unattended execution.


# Polymarket Contrarian Crypto Bot V1 (Notebook)

V1 is an interactive Jupyter notebook that walks through Polymarket's Gamma API and CLOB V2 API step by step. It keeps the original goal—scan crypto markets, inspect order book structure, compute a contrarian z-score signal, and place trades—but is designed for **manual exploration and research** rather than unattended execution.

## Current integration choices

The code targets `py-clob-client-v2` (the official CLOB SDK) alongside raw `requests` calls to the Gamma API for market discovery. The SDK exposes order book reads, midpoint/spread queries, authenticated order placement (FOK market and GTC limit), balance/allowance reads, and open-order management. Analytics use NumPy for computation, Matplotlib for rendering, and Seaborn for styling.

## Install

```bash
python3 -m venv venv
source venv/bin/activate
pip install py-clob-client-v2 requests numpy matplotlib seaborn
jupyter notebook polymarket_bot.ipynb
```

## Quickstart

### Prerequisites

- Python 3.10+
- pip
- Jupyter Notebook
- Windows, macOS, or Linux

### Clone & Run

```bash
# Clone the repository
git clone https://github.com/DimensionlessDevelopments/Dimensionless_Polymarket_CryptoBot_v1.git
cd poly

# Create and activate a virtual environment
python -m venv .venv

# Windows PowerShell
.venv\Scripts\Activate.ps1

# macOS/Linux
source .venv/bin/activate

# Install dependencies
pip install py-clob-client-v2 requests numpy matplotlib seaborn

# Launch the notebook
jupyter notebook poly1.ipynb
```

The notebook opens in your browser. Run cells from top to bottom.

### Usage
1. Run the setup and imports cells.
2. Run market discovery to pull active markets from Gamma API.
3. Run analytics cells to generate charts in the `analysis` folder.
4. Select a market and run order book analysis.
5. For trading actions, fill in wallet credentials in the authentication section before running order cells.

**Example Output:**

```text
Found 80 markets

Question: Will BTC be above $70k at month end?
	Volume 24h: $1,245,000
	Liquidity: $328,000
	Prices: ["0.61", "0.39"]

Best Bid: 0.6000
Best Ask: 0.6200
Midpoint: 0.6100
Spread (abs): 0.0200
Spread (rel): 3.28%

Saved: analysis/volume_liquidity_scatter.png
```

## What the notebook covers

1. **Market discovery** — Gamma API `/markets` endpoint, filtered to crypto keywords, sorted by 24h volume.
2. **Volume & liquidity analytics** — scatter plot and histograms showing which markets have real activity vs. dead liquidity.
3. **Market deep dive** — token ID extraction, outcome parsing, end-date inspection.
4. **Order book analysis** — full bid/ask retrieval, best bid/ask/midpoint/spread computation, cumulative depth chart, top-N level bar chart.
5. **Authentication** — `ClobClient` with private key, signature type, and funder address; `create_or_derive_api_key()` + `set_api_creds()` two-step init.
6. **Market order execution** — `MarketOrderArgs` + `PartialCreateOrderOptions(tick_size="0.01")` + `create_and_post_market_order()` with FOK.
7. **Limit order execution** — `OrderArgs` + `create_and_post_order()` with GTC.
8. **Open orders** — fetch, display, and bar-chart your resting orders by side and price.
9. **Cancel orders** — single order or `cancel_all()`.
10. **Contrarian signal** — rolling midpoint history (deque, window=60), z-score computation, threshold trigger at |z| > 1.5.
11. **Signal analytics** — midpoint history with σ-bands, z-score bar chart with threshold lines.
12. **Multi-market spread comparison** — horizontal bar chart + scatter of spread vs volume across top 10 markets.
13. **Cycle dashboard** — 4-panel summary (volume, spread, midpoint, liquidity vs volume).

## Analysis: What the charts actually tell you

### Order book microstructure (§4a)

The cumulative depth chart is a direct visualisation of **market depth** — a core concept in market microstructure theory. Key insights:

- **Wall detection** — a sudden spike in cumulative depth at a specific price reveals a large resting order (institutional or market-maker). In traditional markets this is "iceberg" or "hidden liquidity" detection. In Polymarket it reveals whether a large player is defending a price level.
- **Book imbalance** — if bid depth is 3× ask depth within the top 5 levels, the book is "bid-heavy." This is the same metric used in HFT order flow imbalance (OFI) models. A persistent imbalance predicts short-horizon price pressure in the direction of the heavier side.
- **Spread as a cost of capital** — the absolute spread is the minimum cost of crossing the market. In quant finance terms, this is your **implementation shortfall** before any slippage. Markets with spreads > 4% are effectively illiquid for any strategy with a sub-10% edge.

### Signal validation (§10a)

The z-score chart is a **statistical process control** chart (SPC), borrowed directly from quality control and adapted for price series:

- **Regime detection** — if the z-score stays within ±1σ for 40+ samples then suddenly breaks to 2σ, that's a regime shift. The chart makes this visible in a way a single number cannot.
- **Signal frequency calibration** — at z_threshold = 1.5, you expect ~13% of samples to trigger (two-tailed). If the bar chart shows signals firing on 50% of samples, your window is too short or the market is in high volatility. If it never fires, the window is too long or the market is too stable. The chart lets you calibrate this *visually* before committing capital.
- **Mean-reversion half-life** — by looking at how many samples it takes for the z-score to decay back to zero after a trigger, you get a rough estimate of the **mean-reversion half-life**. This directly informs your `max_hold_minutes` parameter in V2. If the half-life is ~5 samples at 30s intervals, holding for 30 minutes (60 samples) is 6 half-lives — you've almost certainly captured the full reversion.

### Cross-market comparison (§10b)

The spread-vs-volume scatter is a **market quality map**. In quant finance:

- **Liquidity premium** — markets with high volume but wide spreads are "illiquid in a liquid market." This is analogous to the liquidity premium in bond markets: you need a larger edge to compensate for the cost of execution. The chart lets you visually identify which markets offer the best risk-adjusted execution.
- **Cross-sectional alpha** — if one crypto market has a 0.5% spread and another has a 6% spread, the signal-to-noise ratio is dramatically different. A z-score of 1.5 in a 0.5% spread market is a 3× edge; the same z-score in a 6% spread market is likely noise. The comparison chart lets you **rank markets by tradeability** before deploying the same signal across all of them.

### Volume & liquidity distributions (§2a)

The histograms reveal the **power-law distribution** typical of prediction markets:

- A few markets dominate volume (the "head"), the long tail is illiquid.
- The median is far below the mean — most markets are not tradeable.
- This is the same structure as equity market capitalisation distributions. The practical implication: your strategy's universe is effectively the top 5–10% of markets by volume. The chart confirms this empirically rather than assuming it.

### Cycle dashboard (§10c)

The 4-panel dashboard is a **state-of-the-market snapshot** — the equivalent of a quant's daily pre-market brief:

- Panel 1 (volume): Is the market active today?
- Panel 2 (spread): Are execution costs acceptable?
- Panel 3 (midpoint): Is the signal in a firing zone?
- Panel 4 (liquidity vs volume): Are the conditions consistent?

If any panel looks wrong (e.g., volume collapsed, spreads widened), you skip trading for that cycle. This is **regime-aware execution** — the same principle that stops a market-making strategy when volatility spikes.

## Connection to quantitative finance

The notebook implements several concepts directly from the quant finance literature, adapted to a prediction market context:

| Concept | Where in V1 | Quant finance origin |
|---------|-------------|---------------------|
| Rolling z-score signal | §10 | Statistical arbitrage / mean-reversion (e.g., Pairs Trading, Gatev et al. 2006) |
| Order book imbalance | §4a | Order Flow Imbalance (OFI) — Cont, Kukanov & Stoikov (2014) |
| Spread as execution cost | §4a | Implementation Shortfall — Perold (1988) |
| Depth/wall detection | §4a | Market microstructure — Hasbrouck (2007) |
| Signal frequency calibration | §10a | Statistical process control / false discovery rate |
| Mean-reversion half-life | §10a | Ornstein-Uhlenbeck process estimation |
| Cross-sectional market ranking | §10b | Liquidity-adjusted alpha — Amihud (2002) illiquidity measure |
| Regime-aware execution | §10c | Volatility targeting / regime switching (Hamilton 1989) |
| Power-law volume distribution | §2a | Fat-tailed returns / Pareto distributions in market data |
| Append-only telemetry (charts) | All sections | Event-driven backtesting infrastructure (Zipline, QuantConnect) |

### Why prediction markets are a unique testbed

Traditional quant strategies trade equities, futures, or crypto spot. Polymarket binary options introduce properties that don't exist in those markets:

1. **Bounded prices [0, 1]** — no fat tails from the price process itself. The "return" is bounded, which makes z-score signals more stable and mean-reversion more reliable than in unbounded asset classes.
2. **Convergence guarantee** — every market resolves to 0 or 1. This is a **hard anchor** that doesn't exist in equities. A "fair value" of 0.70 means the event has a 70% probability of occurring — it's a probability, not a model output. This makes the mean-reversion hypothesis more grounded: if the crowd overprices an event at 0.85 when the "true" probability is 0.70, the price *must* revert as new information arrives or as the resolution date approaches.
3. **No shorting** — you can only buy YES or NO. This is equivalent to a **long-only constraint** in equity quant. The contrarian strategy must work within this constraint, which is a realistic and important limitation.
4. **Discrete information events** — crypto markets on Polymarket resolve on specific dates or price levels. The information flow is event-driven, not continuous. This is closer to **earnings-driven alpha** in equities than to momentum or carry.
5. **Small market, transparent order flow** — the entire order book is public. In equities, you see a fraction of the flow (your exchange's book). In Polymarket, the book *is* the market. This makes microstructure analysis (imbalance, depth, spread) more informative because you're seeing 100% of the data.

### How to use the notebook as a quant research tool

1. **Hypothesis generation** — run the notebook daily, collect the z-score charts, and look for patterns. Do signals fire more often before resolution dates? During high-volatility hours? After large volume spikes? These are **alpha hypotheses** that you can then formalise.

2. **Parameter sensitivity** — the notebook makes it trivial to test `Z_THRESHOLD = 1.0, 1.25, 1.5, 1.75, 2.0` and `HISTORY_WINDOW = 30, 60, 120, 240`. Plot the signal frequency and (manually) the quality of the reversion for each combination. This is a **parameter grid search** — the first step in strategy optimisation.

3. **Regime mapping** — over weeks of use, the dashboard charts build a visual history. You can identify regimes: "high volume + tight spread + stable midpoint" (tradeable) vs "low volume + wide spread + volatile midpoint" (untradeable). This is **market regime classification** without needing a GARCH model.

4. **Execution cost modelling** — the spread charts give you empirical execution costs per market. Combine this with the depth charts to estimate **slippage as a function of order size**. This is the input you need for a proper **cost-aware optimisation** (e.g., Almgren-Chriss optimal execution).

5. **Forward validation** — the z-score chart with σ-bands is a **walk-forward test**: you can visually confirm that past signals (red bars) were followed by reversion (the line returning toward the mean). If you see 10 signals and 8 reverted, that's a 80% hit rate — a useful prior before building a formal backtest in V2.

## What V1 does NOT include

- No automated execution loop — every trade is a manual cell run.
- No risk manager (no per-market or total exposure caps, no cooldown, no cash buffer check).
- No position tracking or state persistence.
- No automated exits (take-profit, stop-loss, time-based).
- No error handling or retry logic — API exceptions crash the cell.
- No logging framework — output is `print()` only.
- No telemetry file — charts are saved as PNGs but there is no machine-readable event log.
- No single-instance locking or crash-safe state.
- No dynamic fee estimation.
- V2 does all the above that V1 does not

## Important modeling limitation

The contrarian signal is a **mean-reversion hypothesis**. A rolling midpoint z-score is not an independently estimated probability or fair value. The signal can be statistically interesting and still lose money. Treat the charts as research evidence, not proof of edge. However V2 is more calibrated and built for the market in live trading. Email Dimensionless Developments for it

In quant finance terms: you have a **signal**, not a **model**. A model estimates parameters from data and produces calibrated probabilities. A signal is a heuristic that *may* correlate with future returns. The notebook helps you determine whether the correlation is real, persistent, and large enough to survive transaction costs. That determination is the entire job of alpha research.

## Install

```bash
python3 -m venv venv
source venv/bin/activate
pip install py-clob-client-v2 requests numpy matplotlib seaborn
jupyter notebook polymarket_bot.ipynb   


**Built with ❤️ for quantitative trading research**
# Dimensionless Polymarket Crypto Bot

---


## Contact
**Made by Dimensionless Developments**
**Head to our website https://quant.dimensionlessdevelopments.com/**
  **↓   ↓   ↓   ↓   ↓   ↓   ↓   ↓   ↓   ↓**
**Email: contactus@dimensionlessdevelopments.com**
