file - [bitcoin.ipynb](https://github.com/user-attachments/files/33660331/bitcoin.ipynb)


STAGE 1: Data Ingestion and Feature Engineering1. 
MethodologyThe objective of Stage 1 was to build a robust, institutional-grade historical database pulling directly from the Binance Futures API. 
To ensure data integrity and prevent look-ahead bias or missing information, 
the ingestion pipeline executed the following steps:Asset Universe Selection: Ingested 30-minute ($30m$) interval data across a diversified basket of major cryptocurrency perpetual futures pairs 
(BTCUSDT, ETHUSDT, BNBUSDT, LTCUSDT, XMRUSDT, XLMUSDT, ADAUSDT, DOGEUSDT, SOLUSDT, with DOTUSDT set for a quick rerun).   
Multi-Layer Data Synchronization: For every asset, the pipeline fetched and aligned five critical data streams:Price Action (Klines): Open, high, low, close, and volume bars spanning back to 2019/2020.   Derivative Metrics: Premium index, mark price, and index price series.   
Funding Rates: Periodic perpetual swap funding payments.   
Global Metrics & Snapshots: Daily high-frequency metrics files tracking open interest and long/short ratio snapshots.   
Caching & Robustness: Implemented a local caching mechanism (*_30m_features.csv) checking daily files to ensure fault tolerance, allowing interrupted downloads to resume seamlessly without re-fetching existing data.  
Feature Engineering Output: Compiled each asset into a standardized dataframe containing 54 engineered features ready for volatility modeling.   2. Outcomes & Data Quality ReportAs shown in your ingestion console output, 
the pipeline executed successfully with exceptional data hygiene:Zero Missing Bars: Across all ingested assets, missing_bars stood strictly at 0, ensuring a continuous, unbroken time-series timeline from inception through October 2026. High Data Coverage Completeness:Funding Rate Coverage: 100.0% across all coins.
Premium / Mark / Index Price Coverage: Between 95.9% and 100.0%.   Open Interest & Long/Short Ratios: Robust coverage averaging between 70% and 86% across the history.   
<img width="708" height="200" alt="image" src="https://github.com/user-attachments/assets/bb212c12-4055-4319-8bf3-f0378e6ca567" />

STAGE 2
Part 1: Methodology (How Stage 2 Works)Stage 2 classifies every 30-minute market bar into different "market weather conditions" (regimes) using a Hidden Markov Model (HMM) with 3 distinct states: Calm (State 0), Trending (State 1), and Volatile/Stormy (State 2).To ensure the model is 100% realistic and free of look-ahead bias (cheating),
it uses strict institutional constraints:Walk-Forward Expanding Window: The model is initially trained only on the first 2 years of data (the anchor), and then refitted every quarter (~91 days) using an expanding historical window. It never uses future data to judge the past.
Causal Forward Filtering: Probabilities are calculated strictly using past and present information (the forward algorithm), verified by a causality check showing zero future leakage (max change = 0.00e+00).   
State Sorting: States are automatically sorted by volatility after every fit so that State 0 is always the calmest and State 2 is always the most volatile.
Two Model Paths:Market Regime: Built from Bitcoin (BTCUSDT) to represent the broader crypto market health.   
Per-Coin Regimes: Learned using shared parameters across all 10 coins, giving each individual asset its own independent regime timeline.   Part 2: Key Insights & Important Metrics from the Output1. The Market Regime BreakdownLooking at the Market Regime Report, 
the three market states behave exactly as expected:   State 0 (Calm): Accounts for 44.3% of the time with an annualized volatility of 22.5% and an average duration of 7.8 hours.   State 1 (Trending): Accounts for 37.5% of the time with moderate volatility (36.5%).   State 2 (Volatile/Crash): Rare and explosive.
It only happens 18.2% of the time, but volatility spikes massively to 104.0% annualized, and these storms burn out fast (lasting only 1.7 hours on average).   2. Universal Validation Across All 10 CoinsThe output confirms that "Volatility increases with state index, per symbol: True" for all 10 coins (BTC, ETH, SOL, ADA, DOGE, etc.).
This proves the HMM successfully categorized market states identically across the entire altcoin universe without breaking.
High-beta altcoins like DOGE and DOT show extreme State 2 volatilities (over 200% annualized) during panics, whereas BTC stays comparatively contained (105.7% in State 2).   
3. Regime Persistence (Transition Matrix)The transition matrices show very high diagonal numbers (e.g., staying in calm or volatile states is sticky), meaning once the market enters a calm or stormy phase, it tends to stay there for multiple bars before transitioning.
4. The Causality Check PassedBoth Market and Per-Coin runs passed the causality check with a filtered max change of 0.00e+00, proving that stripping away future bars does not alter past regime probabilities. No data leakage occurred.
<img width="611" height="260" alt="image" src="https://github.com/user-attachments/assets/f514865b-5465-48e3-9533-8ae111790b3d" />
<img width="560" height="363" alt="image" src="https://github.com/user-attachments/assets/64a5dbd8-808b-44d1-b265-a9941634eefa" />
<img width="648" height="395" alt="image" src="https://github.com/user-attachments/assets/cea39745-65c4-44db-8169-b55af42c2ae7" />


Stage 3: asks a ranking question, not a forecasting one: can a simple signal tell us which coins will do better than the others over the next few hours or days?

Think of it as a talent test:

Every 4 hours, we look at all 10 coins and compute about 18 simple signals (past returns, volatility, funding, premium, order flow, open interest, volume).
For each signal we rank the coins from "highest signal" to "lowest".
We wait 4, 12, 24 or 72 hours and rank the coins again by what they actually earned.
We then check how similar the two rankings are.
The important metrics
Metric	Plain meaning
IC (rank correlation)	How well the signal's ranking matches the real ranking. +1 is perfect, 0 is noise, and negative means the signal ranks coins backwards (a reversal). In practice, anything beyond ±0.05 is a decent signal.
Spread (bps)	Return of the top 30% of coins minus the bottom 30%, in basis points (1 bp = 0.01%). This is the closest thing to a profit.
Cost hurdle (22 bps)	The cost of rebalancing both legs: 4 trades × 5.5 bps. A signal is only tradable if its spread beats this.
Market regime split	The same numbers for calm, mid and volatile markets (using the Stage 2 HMM, which is past-only).
Weekly block bootstrap	Resamples whole weeks to test significance. This is needed because overlapping windows make neighbouring data points near-duplicates.
False-discovery correction	We ran 288 tests, so about 14 would look significant by luck alone. The correction removes results that could be luck.
Same sign in both halves	A stability check. Does the signal point the same way in the first and second half of the sample?
What the output shows

1. The signals are real statistically, not luck.
Of 288 tests, 138 were significant before correction (about 14 expected by luck) and 105 survived the correction. Many survive because the signals are close cousins, so 105 survivors is not 105 findings.
2. It really boils down to three findings.
Finding	Evidence
Volatility (strongest)	vol_1d has the strongest IC: −0.10 at 72 hours in calm markets, and −0.085 across all markets. Higher-volatility coins tend to rank lower over the next 1 to 3 days. The effect strengthens as the horizon lengthens.
Short-term reversal	Past returns (6 hours to 7 days) have IC around −0.03 to −0.05, strongest at 4 to 24 hours. Coins that jumped recently tend to lag over the next few hours.
Rich-premium reversal	Premium and basis (perpetual priced above spot) show about −0.03, again at short horizons.
Funding rate, order flow, open interest and volume are weak or noise (IC under about 0.03, only a few survivors).
3. The volatility effect disappears in stormy markets.
The IC difference between volatile and calm regimes is +0.07 at 72 hours for vol_1d, and it survives the correction at all four horizons. This is the one place where the HMM regime matters: the "avoid the wild coins" tendency works in calm and mid markets, but not in volatile ones. Only 4 of 72 regime comparisons survived, all the same signal, so treat it as one finding.
4. None of it is tradable after costs.
This is the main outcome. The read-out says 0 tests are both stable across halves and above the 22 bps hurdle. The best spreads are tiny: −6.8 bps for vol_1d at 72 hours in calm, and about −3 bps for the return signals. Even the best are 5 to 20 bps short of the hurdle.
<img width="768" height="417" alt="image" src="https://github.com/user-attachments/assets/dddebead-937a-409d-a1ec-bed7dd0b1693" />
<img width="711" height="335" alt="image" src="https://github.com/user-attachments/assets/8c3b7981-f232-44d7-9920-430f3a523436" />
<img width="691" height="349" alt="image" src="https://github.com/user-attachments/assets/9f366a85-5371-4454-9fa7-3592c4dbc989" />

STAGE 4
The question
Which is the better volatility tool, the HMM regime model or the classic GARCH model? Does either one improve a simple trend-following strategy after costs?
Stage 4 is a fair fight between the tools, judged on two things:
Part A (forecasting): how well each predicts tomorrow's volatility.
Part B (trading): whether using that forecast to reduce risk improves a strategy.

It runs on BTC (the main test) and ETH (a repeat with the same settings, to see if it transfers).

Methodology in plain words
The contestants (all forecast the same thing: how wild the next 24 hours will be):
Model	Idea
Persistence	Tomorrow will be as wild as today. This is the "do nothing clever" baseline.
GARCH	Big moves are followed by big moves, and calm slowly returns.
HMM	The Stage 2 regime model says "calm, mid or stormy". The HMM only outputs probabilities, so we learn from past data how much volatility each state usually brings, then convert them into a forecast.
HMM-prop	Same idea, but it pushes the probabilities forward in time using an estimated transition matrix.
HAR-S	The textbook regression using yesterday, last week and last month of volatility.

The strategy being tested (Part B). A basic trend rule: hold the coin when the 20-day average is above the 50-day average, otherwise sit out. Each volatility tool then shrinks that position when it expects stormy times.

The fairness rules:

Every model forecasts the same target and is scored with the same yardsticks.
All models are refitted on the same dates, using only past data.
The "truth" for "volatility is high" does not come from the HMM, so it can't grade itself.
Settings are chosen on the first 60% of the data and judged on the last 40%.
The key control: a constant position of the same average size. Any overlay lowers risk simply by holding less, so it counts only if it beats this control.
Costs of 22 bps per round trip, plus funding.
Pass/fail rules were written before any run.
The metrics
Metric	Plain meaning
QLIKE	The standard error score for volatility forecasts. Lower is better. This is the main score.
MSE	A simpler error score that is dominated by the biggest spikes.
Diebold-Mariano test	Tests whether one model's errors are really smaller than another's, or just luck.
AUC	Can the model tell "unusually high volatility" bars from normal ones? 0.5 is a coin flip and 1.0 is perfect.
Top-20% precision	Of the bars the model worries about most, how many were really high-volatility? About 48% is the base rate.
Sharpe / max drawdown	Return per unit of risk; worst peak-to-trough fall.
Sharpe minus control	The real test in Part B: does the overlay beat just holding a smaller constant position?
Turnover and cost drag	How much the strategy trades per year, and the % of return lost to fees.
Break-even cost	The highest trading cost at which the overlay still beats its control.
Key insights
Part A: forecasting BTC	ETH
QLIKE vs persistence: GARCH	−22.9%	−23.9%
HMM	−26.8%	−7.5%
HMM-prop	−5.0%	+13.5% (worse)
HAR-S	−47.4%	−40.2%
AUC: persistence / GARCH / HMM / HAR-S	0.73 / 0.68 / 0.69 / 0.77	0.73 / 0.70 / 0.65 / 0.76
HMM versus GARCH has no stable winner. The HMM won on BTC (a significant difference), but GARCH won clearly on ETH. The HMM used here is the market-wide (BTC-based) regime, so it fits BTC better than other coins. This matches Stage 5, where the market-regime HMM was the weakest model. (That link is my interpretation.)
HAR-S beats both by a wide margin on both coins. It has the best QLIKE and the best AUC, with a significant Diebold-Mariano win against every rival. The built-in "forecast winner" line only compares HMM and GARCH, so it hides this. This is the main result of Part A and sets up Stage 5.
Naive persistence ranks high-volatility days better than GARCH or the HMM (AUC 0.73 against 0.68 and 0.69 on BTC). Sophisticated is not the same as better.
The HMM carries real but modest information. Volatility was about 1.4 to 1.5 times higher when the stormy probability was high than when it was low. That is a real signal, but weaker than simple volatility memory.
The "propagated" HMM is weak, probably because its transition matrix is estimated from noisy probabilities, as the script notes. It was no better than persistence on ETH.
The two error scores disagree. GARCH and the HMM beat persistence on QLIKE but score worse on MSE (GARCH +76% on BTC). QLIKE is the pre-declared main yardstick, but MSE shows these models miss extreme spikes badly.
Part B: strategy

1. The trend rule is the real performer. On the test period it earned a Sharpe of 0.88 (BTC) and 1.05 (ETH), against 0.38 and 0.23 for buy-and-hold. It also cut the worst loss from −54% to −31% on BTC, and from −70% to −46% on ETH. No overlay improved on it.

2. The HMM rule as specified loses money through over-trading.

Full period (2021 to 2026)	Return	Trades per year	Fee drag per year
BTC: trend baseline	+0.8%	9	about 1%
BTC: HMM rule	−11.3%	114	12.5%
ETH: HMM rule	−9.8%	160	17.6%

The regime probability flickers, so the position keeps flipping. Fees ate most of the return.

3. It is not only a cost problem. Even at a tiny cost of 2.5 bps, the HMM and GARCH rules still scored worse than their constant-size controls. The signal itself did not time the market well. Year by year, the overlay hurt most in rallies. In 2024 on BTC it lost 18.8%, while the constant control gained 27.2%. It helped only in 2025.

4. Properly tuned versions only tie.

Test period	Sharpe minus control (BTC / ETH)
HMM, tuned	−0.21 / −0.26
GARCH	−0.02 / +0.02
HAR-S	−0.02 / 0.00
Persistence	−0.10 / 0.00

Every 90% interval includes zero. The best case, GARCH on ETH (+0.02), is not significant.

5. Selection collapsed to "do nothing". For the HMM-variance and HMM-propagated versions, the exact-zero results mean the chosen setting never changed the position, so they are identical to the baseline. Across all tested settings, only 0% to 22% beat the control on test.

6. Final verdict: all 16 overlay tests (8 per coin) were classified "NEITHER". None qualified as an alpha driver (Sharpe gain with the interval above zero) or as a risk filter (at least 15% shallower drawdown than the control, keeping at least 60% of the return).

Stage 5: which volatility model fits which coin?

Question. Of eight ways to forecast next-24-hour volatility, which is best for each coin? Is it worth using a different model per coin, or is one model enough?

Method in plain words.

Every model forecasts the same thing: the average variance of the next 24 hours.
The eight models are persistence, EWMA, GARCH, GJR-GARCH, HAR-S, HMM-own, HMM-mkt and HAR-HMM. HAR-HMM is HAR-S plus the coin's own regime probabilities as extra inputs.
All models are refitted on the same dates, using only earlier data.
The out-of-sample bars are split in time: the first 60% (DEV) choose a champion per coin, and the last 40% (TEST) judge it.
A champion counts only if it beats HAR-S on TEST with Diebold-Mariano p < 0.05 after a Holm correction across the 10 coins. Otherwise the coin falls back to HAR-S.
Per-coin tailoring is recommended only if it beats one-model-for-all by at least 1 percentage point.

Metrics.

QLIKE relative to HAR-S is the score. Negative means better than HAR-S.
Diebold-Mariano with Holm tests whether a gain is real or luck, after allowing for many comparisons.

Insights from the output.

HAR-HMM was the best model on DEV for all 10 coins, and it beat HAR-S on TEST for all 10 (between −0.8% and −4.6%).
The gain is small. It averages about 2%, and only four coins are proven after correction: BTC, ETH, DOGE and SOL.
Regime models alone are weak. HMM-own is on average 35% worse than HAR-S, and HMM-mkt is 68% worse. The regime helps only on top of HAR-S.
One universal model won. One model for all scored −1.98%, against −0.81% for per-coin confirmed champions.
GARCH is unstable. It looked best on TEST for 4 coins (XMR, XLM, ADA, DOT), by as much as 19% on XMR, but was 19% to 42% worse than HAR-S on DEV. It could be a lucky stretch or a change in market behaviour. 
Either way, it is why the DEV champion matched the TEST best on only 6 of 10 coins, so coin-level winners are mostly noise.
Persistence is clearly the worst (about 52% worse than HAR-S).

Conclusion. Use one universal HAR-HMM for all coins. It gives a small, consistent improvement over the strongest simple baseline, and it is statistically proven on four coins.
<img width="670" height="404" alt="image" src="https://github.com/user-attachments/assets/a819e9b1-6f5d-46a3-ba24-e01d8b2949ee" />
<img width="695" height="299" alt="image" src="https://github.com/user-attachments/assets/b1a25902-dec8-4949-a7f0-d87ab02f86ca" />
<img width="684" height="277" alt="image" src="https://github.com/user-attachments/assets/6ff75e7f-d64b-4743-b1bd-67f43e800a90" />

Stage 6: Tail risk (VaR and Expected Shortfall)
The questions it answers

Earlier stages showed that the forecast is only slightly better than HAR-S and does not make trading money. Stage 6 asks whether it is useful for measuring risk:

Is the Value-at-Risk (VaR) right? Does the 99% line get crossed about 1% of the time?
Is the Expected Shortfall (ES) right? When a bad day happens, is the real loss about as big as the model said?
Is HAR-HMM better than the other models at this? And if so, is it better by a provable margin?
Do the coins break the line together?
What shape of "bad day" should we assume? Bell curve or fat-tailed?
Methodology in plain words
Forecast to risk numbers. Every day at 00:00 UTC, each model's volatility forecast is multiplied by a "tail multiplier". That gives a VaR 99% ("a 1-in-100 bad day loses at least this much") and an ES 97.5% ("on the worst 2.5% of days, the average loss").
Choose the tail shape on DEV only. Normal, Student-t and empirical were tried, using HAR-S so the champion isn't favoured. The shape was fitted on early DEV and judged on late DEV. It was then refitted on all DEV and frozen. Each model's tail multipliers were calibrated on DEV, so a model that is uniformly too high or too low isn't punished for scale alone.
Test once on unseen data. The TEST period (Sept 2024 to Oct 2026) lies inside every coin's Stage 5 TEST period. There is one observation per coin per day, with no overlapping windows, and results are pooled over 10 coins (7,429 coin-days).
Five models were compared: persistence, EWMA, GARCH, HAR-S and HAR-HMM.
Tests used:
Test	Plain meaning
Kupiec	Is the number of breaches what it should be?
Christoffersen	Are breaches bunched together in time?
ES ratio (bootstrap)	Real average loss on bad days ÷ predicted ES. 1.00 is perfect and above 1 means risk is understated.
FZ0 score	One joint score for VaR and ES that a model can't game by being too cautious or too bold.
Diebold-Mariano with Holm	Is one model's score really better, after allowing for several comparisons?
Cluster count	Days when 3 or more coins breach together.
Pass/fail rules were written before running. C1: breach rate consistent with 1%. C2: ES ratio interval contains 1. C3: lower joint score than HAR-S. C4: that improvement is proven.
Deep insights from the output

1. The risk numbers were accurate.

HAR-HMM	Result	Target
99% VaR breaches	71 (0.96%)	about 74 (1%)
Kupiec p-value	0.70	at least 0.05 to pass
ES ratio	0.995 (95% interval 0.89 to 1.12)	1.00

C1, C2 and C3 passed. Settings chosen on 2021 to 2024 data still worked on 2024 to 2026, which is the strongest sign the calibration is reliable.

2. The tail shape matters a lot. The best shape was Student-t with about 4 degrees of freedom, which means very fat tails. On the late-DEV check, a normal bell curve with the same forecasts gave 1.41% breaches (41% too many) and an ES ratio of 1.16, so it understated tail losses by about 16%.
3. The t-shape was slightly cautious there (0.76% breaches). This is a concrete argument against "normal returns" in a crypto risk report.
The lead over HAR-S (+0.008) is about 14 times smaller than the gap to persistence (0.110).
Only the win over persistence is proven (p after Holm = 0.004). The edge over HAR-S has p = 0.24, so C4 failed. Over EWMA and GARCH it is also not proven (p = 0.24).
HAR-HMM beat HAR-S on the joint score for all 10 coins, but by tiny margins (0.002 to 0.016), and the coins move together, so this is not 10 independent confirmations.

4. Right count is not the same as good. Persistence had almost exactly the right number of breaches (73) but the worst score. My reading is that its risk line jumps around with yesterday's noise, so the counts balance out while the timing is poor. GARCH had too many breaches (90, 1.21%), a borderline miss with p = 0.076.
5. EWMA and persistence slightly overstate ES (ratios 0.94 and 0.96), so they are mildly over-cautious.

6. Coins break the line together. If the 10 coins failed independently, we would expect about 0.08 days with 3 or more simultaneous breaches. We saw 7, which means at least 21 of the 71 breaches (about 30%) came from just 7 market-wide days. Every model had this problem (8 to 13 days), so a better forecast does not fix it.
7. <img width="712" height="391" alt="image" src="https://github.com/user-attachments/assets/83ea1567-20e3-4b77-a757-b403c983deed" />
<img width="703" height="320" alt="image" src="https://github.com/user-attachments/assets/729caf23-1b6c-4c4f-a789-fb674d5eb55a" />
<img width="747" height="332" alt="image" src="https://github.com/user-attachments/assets/01663ff5-f500-4990-ba4e-f0bc02fa8683" />
<img width="690" height="121" alt="image" src="https://github.com/user-attachments/assets/8619eb70-4027-4eb7-bf44-16aa39da6205" />
<img width="1181" height="790" alt="image" src="https://github.com/user-attachments/assets/ce820c72-f950-49ac-8f5b-f07a75a7219a" />

Conclusion of Stage 6

HAR-HMM is a well-calibrated tail-risk engine (VaR and ES both accurate on unseen data). It is the best of five models, but HAR-S was almost as good, so the case is "reliable and slightly better", not "uniquely superior".
The model is suitable for risk limits and margin, with the warning that shared shocks need portfolio-level handling.


