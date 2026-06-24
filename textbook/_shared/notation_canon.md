# Notation Canon

Canonical symbol -> meaning list for the USE4 lecture-note library. Every set draws its Notation table from this file; add a row here before introducing a new symbol in any set.

| Symbol | Meaning | First used in |
|---|---|---|
| $\Var$ | variance operator | (operators) |
| $\Cov$ | covariance operator | (operators) |
| $\Corr$ | correlation operator | (operators) |
| $\diag$ | diagonal-matrix operator | (operators) |
| $\std$ | standard-deviation operator | (operators) |
| $\avg$ / $E$ | expectation / mean operator | (operators) |
| $\T$ | matrix/vector transpose | (operators) |
| $\R$ | the real numbers | (operators) |
| $\half$ | one-half | (operators) |
| $\wvec$ ($w$) | weight vector | (operators) |
| $\Xmat$ ($X$) | exposure (design) matrix | (operators) |
| $\Fmat$ ($F$) | factor covariance matrix | (operators) |
| $\Vmat$ ($V$) | asset covariance matrix | (operators) |
| $\Umat$ | specific-risk / orthogonal matrix | (operators) |
| $\Rmat$ ($R$) | returns / correlation matrix | (operators) |
| $\Lam$ | eigenvalue / loading matrix | (operators) |
| $\Del$ ($\Delta$) | difference / increment | (operators) |
| $\fvec$ ($f$) | factor-return vector | (operators) |
| $t$ | a monthly signal date (rebalance date) | set2_estimation_universe |
| $U_t$ | coverage universe: all stocks alive at $t$ | set2_estimation_universe |
| $E_t$ | estimation universe (ESTU) at $t$, $E_t \subseteq U_t$ | set2_estimation_universe |
| $m_{i,t}$ | total market capitalisation of stock $i$ at $t$ (USD) | set2_estimation_universe |
| $w_{i,t}$ | cap weight $m_{i,t}/\sum_{j\in E_t} m_{j,t}$ | set2_estimation_universe |
| $d_{i,t}$ | daily dollar volume, closeunadj $\times$ volume | set2_estimation_universe |
| $\widetilde d_{i,t}$ | trailing $\approx$63-day median dollar volume (days $\le t$) | set2_estimation_universe |
| $\text{ATVR}_{i,t}$ | annualised traded value ratio (liquidity screen) | set2_estimation_universe |
| $r_{i,t}$ | rank of $m_{i,t}$ among hard-filter- and retain-eligible names (1 = largest) | set2_estimation_universe |
| $\mu_{\text{cw}}$ | cap-weighted mean of a descriptor over $E_t$ | set2_estimation_universe |
| $\sigma_{\text{ew}}$ | equal-weighted std. of a descriptor over $E_t$ | set2_estimation_universe |
| $x_i$ | a raw descriptor | set2_estimation_universe |
| $z_i$ | standardised descriptor $(x_i-\mu_{\text{cw}})/\sigma_{\text{ew}}$ | set2_estimation_universe |
| $r$, $r_i$ | excess (log) return vector; for stock $i$ | overview_risk_model |
| $X_{ik}$ | exposure of stock $i$ to factor $k$ (entry of $\Xmat$) | overview_risk_model |
| $f_k$ | return of factor $k$ (entry of $\fvec$) | overview_risk_model |
| $u$, $u_i$ | specific (idiosyncratic) return vector; for stock $i$ | overview_risk_model |
| $\sigma_p$ | portfolio volatility (risk) | overview_risk_model |
| $N$ | number of assets ($\approx 2{,}500$) | overview_risk_model |
| $K$ | number of factors ($68$ = country + 55 industries + 12 styles) | overview_risk_model |
| $W$ | WLS regression weight matrix, $\diag(\sqrt{m_{i,t}})$ | overview_risk_model |
| $R^2$ | per-period cross-sectional coefficient of determination | overview_risk_model |
| $\ell_n$ | raw Size descriptor (LNCAP), $\ell_n=\ln(M_n)$ | set2_size |
| $s_n$ | standardised SIZE score (the exposure) | set2_size |
| $M_n$ | total market capitalisation of stock $n$ (dollars) | set2_size |
| $\mu_{\text{ew}}$ | equal-weighted mean of $\ell$ over the ESTU (trim center) | set2_size |
| $[\ell_{\text{lo}},\ell_{\text{hi}}]$ | trim band on $\ell$, centered on $\mu_{\text{ew}}$ | set2_size |
