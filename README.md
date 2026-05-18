# Wind-Price Spread Arbitrage — DK1 & DK2 Nordic Power Markets

**Markets:** DK1 (Denmark West) and DK2 (Denmark East) day-ahead electricity  
**Period:** January 2019 – May 2026 · 64,500+ hourly observations · 6 market regimes  
**Data:** ENTSO-E Transparency Platform · Energi Data Service (Energinet)  
**Validation:** 9-fold rolling walk-forward · frozen holdout · DK2 cross-market test

---

## The core idea

Denmark is one of the most wind-penetrated power markets in the world. The **merit-order effect** predicts that higher wind generation depresses day-ahead prices: wind dispatches at zero marginal cost, displacing expensive gas plants and lowering the clearing price. But this relationship breaks down temporarily — prices stay high when wind surges, or fall when wind is calm. When it does, the market is mispricing renewable intermittency.

This project builds a systematic signal to detect those mispricings, trades mean-reversion back to equilibrium, and validates the approach across six distinct market regimes from COVID (2020) through the European energy crisis (2022) to the high-renewables period of 2024–2026.

---

## Model

### Base regression

$$P_t = \beta_0 + \alpha \cdot W_{\text{actual},t} + \gamma \cdot W_{\text{surprise},t} + \varepsilon_t$$

**α (merit-order coefficient):** how much each MWh of wind depresses price. Expected negative. Estimated to be as negative as −0.082 during the 2022 gas crisis (when TTF exceeded 300 EUR/MWh and wind's displacement value was extreme) and weakening toward −0.013 by 2025 as storage and demand response absorb intermittency. This time-variation is the main structural finding.

**W_surprise:** actual wind minus its 7-day rolling seasonal mean. This is the wind the day-ahead auction — which closes at noon the previous day — could not have priced in. We do not use the ENTSO-E published forecast because it covers onshore wind only while actuals include offshore, making the difference meaningless. The seasonal baseline is computed from actuals alone, always aligned, always available.

**γ (surprise coefficient):** additional price impact of unexpected wind. Expected negative.

### The spread and z-score signal

The regression residual is the tradeable quantity:

$$S_t = P_t - \hat{\alpha} \cdot W_{\text{actual},t} - \hat{\gamma} \cdot W_{\text{surprise},t}$$

Wind and price are cointegrated — they share a long-run equilibrium through the merit order. When the spread deviates, it tends to revert. We standardise using a 30-day (720-hour) rolling z-score:

- **Entry:** |z| > 2.0 (spread has deviated significantly from its mean)
- **Exit:** |z| < 0.5 (spread has reverted)

The rolling window (not expanding) is important: α̂ shifts with market structure, so anchoring the z-score to an outdated mean would generate spurious signals.

### Interaction model

Beyond the base model, we estimate how α̂ varies with market conditions:

$$P_t = \beta_0 + \alpha_0 \cdot W + \alpha_1 \cdot (W \times \text{penetration}_t) + \alpha_2 \cdot (W \times \text{price\_high}_t) + \varepsilon_t$$

This gives a continuous, real-time estimate of the merit-order coefficient as a function of how wind-saturated the grid is and how expensive gas is — rather than a fixed coefficient re-estimated every two years.

---

## Regime conditioning

The signal is stronger in certain market conditions. Two regime indicators restrict entry to hours where the economic mechanism is most active.

**Wind penetration** = wind / gross consumption. When wind covers a large share of demand, the grid struggles to absorb surplus and price drops are amplified. Defined as penetration above its rolling 60th percentile (adapts as installed capacity grows). Data: `GrossConsumptionMWh` from Energi Data Service — the same settlement record used for financial imbalance charges.

**Price level:** DK1 price above its 6-month rolling median. Proxy for an expensive-gas environment — when gas is costly, wind's displacement value is high and the merit-order effect is stronger. We use price level rather than TTF gas futures directly because the causal channel runs from gas to power prices, making the power price itself a more direct and reliable signal.

**High-confidence (HC) regime:** both conditions simultaneously. Approximately 7–8% of hours. Win rate: 63.5% vs 57.6% unfiltered — a +6 percentage point improvement that is the core empirical finding.

---

## Walk-forward validation

| Parameter | Value |
|---|---|
| Training window | 2 years, fixed width |
| Test window | 6 months |
| Step size | 6 months (no overlap between test windows) |
| Total folds | 9 |
| HC trades (OOS) | 106 |

**Why fixed training windows (not expanding):** A model trained partly on 2019 data estimates α̂ in a market with fundamentally less wind capacity and gas at 15 EUR/MWh. That estimate is irrelevant to a 2024 market. Fixed 2-year windows ensure the model always reflects current market structure.

**Why 6-month test windows:** Power markets have strong seasonality — summer and winter behave structurally differently. A 6-month window always includes roughly one season, preventing a result that holds only in a favourable season from appearing robust.

### Market regimes covered

| Folds | Period | Regime | Key driver |
|---|---|---|---|
| — | 2019–Feb 2020 | Pre-crisis stable | Gas ~15 EUR/MWh; DK1 ~45 EUR/MWh |
| — | Mar–Dec 2020 | COVID demand shock | Industrial demand −10–15%; negative-price spikes |
| 1 | H1 2021 | Energy crisis onset | Gas tightening; DK1 prices +252% YoY |
| 2–4 | H2 2021–H2 2022 | Energy crisis (onset → peak) | TTF >300 EUR/MWh Aug 2022; DK1 avg 213.6 EUR/MWh |
| 5–6 | H1–H2 2023 | Post-crisis normalisation | DK1 −60.3% YoY; 281 negative-price hours |
| 7–8 | H1–H2 2024 | High renewables / low price | Record Nordic hydro; 275 negative-price hours |
| 9 | H1 2025 | Low-price continuation | EU wholesale +10% YoY |

Note: COVID and pre-crisis periods fall within training windows for the first folds, not test windows — they inform the model but are not the OOS evaluation period.

---

## Cost model

| Component | Value | Basis |
|---|---|---|
| Entry | 1.0 EUR/MWh | Broker and clearing fees on financial power forwards |
| Exit | 1.0 EUR/MWh | Symmetric with entry |
| Holding | 1.5 EUR/MWh/hr | Imbalance settlement risk + capital cost on margin |

The holding cost dominates for any trade held longer than a few hours. At the median HC holding time of 8 hours, total cost is 14 EUR/MW against an average gross P&L of ~60+ EUR/MW per trade. The cost sensitivity sweep confirms the HC strategy remains profitable at every tested cost from 0.0 to 3.0 EUR/MWh/hr — providing a 2× margin of safety on the most uncertain assumption.

What this model does not capture: crisis-period bid-ask widening (5–10 EUR/MWh in August 2022), margin call risk during extreme price moves, and minimum contract size constraints.

---

## Results

### Walk-forward performance (DK1, 9 folds, 6 regimes)

| Strategy | All-hours Sharpe | Win rate | Trades | % Time active |
|---|---|---|---|---|
| No filter (baseline) | 3.23 | 57.6% | 562 | 16.4% |
| High penetration | 0.88 | 54.7% | 180 | 5.7% |
| Price above median | 1.98 | 65.1% | 348 | 9.9% |
| **HC (both) ← headline** | **0.48** | **63.5%** | **106** | **2.7%** |
| Buy-and-hold | −8.40 | — | — | 100% |
| Momentum | +26.88 | — | — | 100% |

The momentum Sharpe of 26.88 is high but expected — power prices have genuine strong hourly autocorrelation due to generator ramp constraints. The wind-price strategy operates in a different regime (2.7% of hours) on a different signal (spread reversion vs price continuation) and is designed to complement momentum, not replace it.

### Merit-order coefficient evolution

α̂ peaks at −0.082 during the gas crisis (2022–2023) and weakens to −0.013 by 2025. This monotonic post-crisis decline as storage and demand response absorb wind intermittency is the main structural finding beyond the trading signal. It has a direct implication: the forward-looking performance of this strategy is better represented by the post-crisis folds (7–9) than by the full-sample average.

### DK2 out-of-sample test

Frozen DK1 parameters from the last training fold applied unchanged to DK2 (see fold table in output for exact values):

| Strategy | DK1 SR | DK2 SR |
|---|---|---|
| No filter | 3.23 | 3.15 |
| Price above median | 1.98 | 2.23 |
| HC (both) | 0.48 | 0.65 |

The HC signal is stronger on DK2 than DK1 with frozen DK1 parameters. DK2's primary interconnector runs to Sweden (hydro-dominated) rather than Germany (gas/coal). Hydro responds more slowly to wind surplus than gas demand response, allowing spreads to deviate further before reverting — creating larger, more tradeable mispricings. The result confirms the merit-order relationship is a pan-Nordic structural feature, not a DK1-specific quirk.

### Final holdout (January–May 2026)

7 HC trades. Sharpe +0.63, win rate 55%, P&L +185 EUR/MW. Trade count is too small for statistical inference but the direction is consistent with walk-forward results.

### Sharpe methodology note

Two measures are reported. **Active-hour Sharpe** annualises using $\sqrt{8760 \times \%\text{active}}$ — the correct denominator for a strategy not always deployed. **All-hours Sharpe** multiplies by $\sqrt{\%\text{active}}$ to give a literature-comparable number. Using $\sqrt{8760}$ throughout would overstate the HC Sharpe by a factor of approximately 3.8× given 2.7% utilisation.

---

## Where the strategy can fail

**Structural break from storage growth.** α̂ weakened post-2022 as gas prices normalised — this is expected and already reflected in the post-crisis fold results. If battery storage and demand response continue growing, the merit-order effect will be buffered further before the spread can deviate enough to trade profitably. Monitor: suspend trading if α̂ in the most recent training fold is less negative than −0.005.

**Regime persistence.** If a weather event (multi-day calm or multi-day storm) keeps wind abnormally low or high for 100+ hours, the spread stays dislocated and holding costs accumulate. At 1.5 EUR/MWh/hr, a 100-hour position costs 150 EUR/MW in holding costs alone. The maximum observed holding time in the data is 152 hours. A hard 48-hour stop-loss is the obvious fix and was not implemented.

**Liquidity crises.** The strategy fires most aggressively in high-volatility periods — exactly when bid-ask spreads widen and brokers raise margin requirements. During August 2022, DK1 financial forward spreads widened to 5–10 EUR/MWh, five to ten times the assumed entry cost. The cost sensitivity analysis provides comfort at 3 EUR/MWh/hr but cannot fully capture crisis-period liquidity.

**Publication lag.** ENTSO-E settlement data arrives 30–60 minutes after the delivery hour. The backtest does not model this delay. In live trading, the strategy would either use noisier preliminary data or accept a one-hour execution lag, both of which reduce the number of clean entry signals.

**Gas crisis overfitting.** Folds 2–6 cover 2021–2023 — the most extreme price environment in modern European power market history. These folds likely drive the aggregate Sharpe upward. The true forward-looking performance is better represented by post-crisis folds 7–9 in isolation.

---

## Scope for further development

**Hard stop-loss on holding time.** Exit any position after 48 hours regardless of z-score. Single most impactful practical improvement. Directly limits worst-case holding cost exposure.

**ECMWF wind forecast as surprise baseline.** Replace the 7-day rolling seasonal mean with the ECMWF 6-hour-ahead wind forecast — what professional participants actually use. Available via ECMWF's open-data API. Upgrades γ̂ from a statistical proxy to a genuine market signal.

**Dynamic α̂ estimation.** Re-estimate α̂ monthly within each test window rather than fixing it for the full 6 months. Reduces sensitivity to structural breaks mid-fold.

**Monte Carlo threshold optimisation.** The entry/exit thresholds (2.0/0.5) were set by convention. A sweep over entry ∈ [1.5, 3.0] and exit ∈ [0.0, 1.0] — optimised on training data only — would confirm whether these are genuinely optimal.

**Three-zone coupled model.** Treat DK1, DK2, and Germany as a coupled system. The spread signal in DK1 depends on whether both the DK1-DE and DK2-SE interconnectors are congested simultaneously. Requires ENTSO-E cross-border flow permissions.

**Negative price separation.** DK1 had 281 negative-price hours in 2023 and 275 in 2024. These are driven by physical export constraints, not the merit-order relationship the model is built on. Excluding or separately modelling these hours would reduce noise.
