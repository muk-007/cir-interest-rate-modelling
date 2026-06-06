# Stochastic Interest Rate Modelling: CIR Implementation & Extension

A complete implementation of the **Cox-Ingersoll-Ross (1985)** short-rate model — calibrated via Kalman Filter Maximum Likelihood Estimation, extended with CIR++, and evaluated on real zero-coupon yield curve data.


---



## Project Overview

Interest rates evolve over time in a complex, seemingly random manner. This project builds a stochastic short-rate model from scratch to capture that evolution, reconstruct the full yield curve from a single observable input, and critically analyse where such models succeed and fail against real market dynamics.

The **CIR model** describes the instantaneous short rate $r_t$ via the SDE:

$$dr_t = \kappa(\theta - r_t)\,dt + \sigma\sqrt{r_t}\,dW_t$$

where $\kappa$ is the speed of mean reversion, $\theta$ is the long-run equilibrium rate, $\sigma$ is the volatility coefficient, and $W_t$ is a standard Brownian motion. The square-root diffusion ensures rates stay positive when the **Feller condition** $2\kappa\theta \geq \sigma^2$ is satisfied.

---

## Repository Structure

```
.
├── ratemodellingproject.ipynb     # Main notebook (all code + markdown analysis)
├── train_data.csv           # Training data: 9 maturities, May 2016 – Apr 2024 (1,976 days)
├── test_data.csv            # Test data:     5 maturities, Apr 2024 – Apr 2026 (495 days)
├── test_data_3M.csv         # 3M yield only — sole input for out-of-sample prediction
└── README.md
```

---

## Dataset

Daily zero-coupon bond yields across nine maturity tenors: **3M, 6M, 9M, 1Y, 2Y, 5Y, 10Y, 20Y, 30Y**.

| Split | Period | Rows | Maturities available |
|---|---|---|---|
| Training | May 2016 – Apr 2024 | 1,976 | All 9 |
| Test | Apr 2024 – Apr 2026 | 495 | 3M – 2Y |

The underlying market and geography are intentionally undisclosed; the model relies purely on mathematical relationships within the data.

---

## Methodology

### A · Data Preprocessing

Raw yield data required four cleaning steps before calibration:

1. **Weekend removal** — non-trading day rows dropped (`dayofweek < 5`)
2. **Missing value imputation** — forward-fill → backward-fill → time interpolation
3. **Outlier replacement** — values beyond ±3σ of a 20-day rolling median are replaced by that median (not dropped, to preserve time-series length for the Kalman Filter)
4. **Positivity floor** — all yields clipped to ≥ 0.0001 (1 bp) to prevent domain errors in $\sigma\sqrt{r_t}$

### B · CIR Model & Calibration

The closed-form bond price under CIR is:

$$P(t,T) = A(\tau)\,e^{-B(\tau)\,r_t}, \qquad \tau = T - t$$

with:

$$B(\tau) = \frac{2(e^{h\tau}-1)}{2h + (\kappa+h)(e^{h\tau}-1)}, \qquad h = \sqrt{\kappa^2 + 2\sigma^2}$$

$$A(\tau) = \left[\frac{2h\,e^{(\kappa+h)\tau/2}}{2h + (\kappa+h)(e^{h\tau}-1)}\right]^{2\kappa\theta/\sigma^2}$$

yielding the continuously compounded yield:

$$y(r_t,\tau) = \frac{B(\tau)\,r_t - \ln A(\tau)}{\tau}$$

**Calibration — Kalman Filter MLE:** Since $r_t$ is latent, the model is cast as a state-space system. The affine yield formula is linear in $r_t$, so a standard linear Kalman Filter applies exactly. The Gaussian log-likelihood accumulated across all observations is maximised over $(\kappa, \theta, \sigma)$ using L-BFGS-B.

KF-MLE was chosen over OLS because OLS on first differences treats each tenor independently, conflating measurement noise with diffusion variance and systematically overestimating σ while underestimating κ. KF-MLE jointly processes all nine tenors simultaneously, separating the latent $r_t$ signal from noise.

### C · Yield Curve Prediction

For every day in the test period, **only the 3M yield** is used as input — serving as a proxy for the instantaneous short rate $r_t$. The affine CIR formula then deterministically reconstructs yields at all other maturities. No other market data is used.

The short end (3M–1Y) is reproduced with high accuracy. The 2Y is the structural weak point: during the test period, 2Y yields were partly shaped by expectations of future rate cuts — a second factor absent from the single-factor CIR framework that the 3M rate alone cannot encode.

### D · Extension: CIR++ (Brigo-Mercurio)

CIR++ adds a maturity-dependent deterministic shift $\varphi(\tau)$ to exactly fit the observed term structure:

$$y^{\text{CIR++}}(r_t, \tau) = y^{\text{CIR}}(r_t, \tau) + \varphi(\tau)$$

where $\varphi(\tau)$ is calibrated as the mean residual over the last 20 training days. This absorbs any systematic bias the base model leaves at each tenor without adding stochastic complexity.

**Limitation demonstrated:** The shift, anchored to a training period with an average 3M rate of 4.91%, became directionally incorrect in the test period where the 3M rate averaged 3.04% (a 186 bp drop). This amplified rather than corrected errors at longer maturities, confirming that CIR++ is sensitive to regime shifts between calibration and deployment.

### E · Critical Analysis

**Theoretical limitations:**
- Single-factor structure constrains yield curves to monotone or single-humped shapes — cannot reproduce butterfly movements or slope/level dynamics independently
- Time-homogeneous parameters: a fixed $\theta$ acts as a permanent gravitational anchor, causing systematic yield underestimation when $r_t > \theta$ in the test period
- Continuous sample paths: no jump component, so the model is blind to discrete rate moves following central bank announcements

**Practical implications for trading and risk management:**
- Systematic yield underestimation at 1Y–2Y implies consistent bond overpricing at those maturities (mark-to-market risk)
- Single-factor DV01: hedging a 2Y exposure with a 3M instrument based on CIR sensitivities leaves the position systematically under-hedged against slope moves
- Swaption/cap mispricing: affine structure dampens long-end volatility faster than observed, leading to underpriced long-dated optionality
- VaR underestimation: absence of jumps understates tail risk on central bank announcement days

---



## References

- Cox, J.C., Ingersoll, J.E., Ross, S.A. (1985). *A Theory of the Term Structure of Interest Rates.* Econometrica, 53(2), 385–407.
- Brigo, D., Mercurio, F. (2001). *Interest Rate Models — Theory and Practice.* Springer Finance.
- Duffie, D., Pan, J., Singleton, K. (2000). *Transform Analysis and Asset Pricing for Affine Jump-Diffusions.* Econometrica, 68(6), 1343–1376.
- Longstaff, F.A., Schwartz, E.S. (1992). *Interest Rate Volatility and the Term Structure.* Journal of Finance, 47(4), 1259–1282.
- Harvey, A.C. (1990). *Forecasting, Structural Time Series Models and the Kalman Filter.* Cambridge University Press.
