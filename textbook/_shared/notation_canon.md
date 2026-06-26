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
| $\Del$ ($\Delta$) | specific-variance diagonal, $\Del=\diag(\sigma^{\text{spec},2}_n)$ | overview_risk_model |
| $\fvec$ ($f$) | factor-return vector | (operators) |
| $t$ | a monthly signal date (rebalance date) | estimation_universe |
| $U_t$ | coverage universe: all stocks alive at $t$ | estimation_universe |
| $E_t$ | estimation universe (ESTU) at $t$, $E_t \subseteq U_t$ | estimation_universe |
| $m_{i,t}$ | total market capitalisation of stock $i$ at $t$ (USD) | estimation_universe |
| $w_{i,t}$ | cap weight $m_{i,t}/\sum_{j\in E_t} m_{j,t}$ | estimation_universe |
| $d_{i,t}$ | daily dollar volume, closeunadj $\times$ volume | estimation_universe |
| $\widetilde d_{i,t}$ | trailing $\approx$63-day median dollar volume (days $\le t$) | estimation_universe |
| $\text{ATVR}_{i,t}$ | annualised traded value ratio (liquidity screen) | estimation_universe |
| $r_{i,t}$ | rank of $m_{i,t}$ among hard-filter- and retain-eligible names (1 = largest) | estimation_universe |
| $\mu_{\text{cw}}$ | cap-weighted mean of a descriptor over $E_t$ | estimation_universe |
| $\sigma_{\text{ew}}$ | equal-weighted std. of a descriptor over $E_t$ | estimation_universe |
| $x_i$ | a raw descriptor | estimation_universe |
| $z_i$ | standardised descriptor $(x_i-\mu_{\text{cw}})/\sigma_{\text{ew}}$ | estimation_universe |
| $r$, $r_i$ | excess (log) return vector; for stock $i$ | overview_risk_model |
| $X_{ik}$ | exposure of stock $i$ to factor $k$ (entry of $\Xmat$) | overview_risk_model |
| $f_k$ | return of factor $k$ (entry of $\fvec$) | overview_risk_model |
| $u$, $u_i$ | specific (idiosyncratic) return vector; for stock $i$ | overview_risk_model |
| $\sigma_p$ | portfolio volatility (risk) | overview_risk_model |
| $N$ | number of assets ($\approx 2{,}500$) | overview_risk_model |
| $K$ | number of factors ($68$ = country + 55 industries + 12 styles) | overview_risk_model |
| $W$ | WLS regression weight matrix, $\diag(\sqrt{m_{i,t}})$ | overview_risk_model |
| $R^2$ | per-period cross-sectional coefficient of determination | overview_risk_model |
| $\ell_n$ | raw Size descriptor (LNCAP), $\ell_n=\ln(M_n)$ | size |
| $s_n$ | standardised SIZE score (the exposure) | size |
| $M_n$ | total market capitalisation of stock $n$ (dollars) | size |
| $\mu_{\text{ew}}$ | equal-weighted mean of $\ell$ over the ESTU (trim center) | size |
| $[\ell_{\text{lo}},\ell_{\text{hi}}]$ | trim band on $\ell$, centered on $\mu_{\text{ew}}$ | size |
| $B_i$ | book value of common equity for stock $i$ (PIT, latest filing) | bp |
| $E_i$ | total stockholders' equity (reported balance-sheet line) | bp |
| $P_i$ | preferred equity; $B_i=E_i-P_i$, with $P_i=0$ in the build | bp |
| $\mathrm{BTOP}_i$ | raw Book-to-Price descriptor, $B_i/M_i$ | bp |
| $W$ | staleness window: max admissible filing age ($\approx$18 months) | bp |
| $D_{i,t}$ | trailing-12-month dividends per share, stock $i$ at $t$ | dyld |
| $d_{i,q}$ | per-share dividend from the $q$-th most recent quarterly filing | dyld |
| $s_{i,q}$ | cumulative split factor (puts a past per-share figure on the current basis) | dyld |
| $P_{i,t}$ | point-in-time price per share | dyld |
| $y_{i,t}$ | raw dividend yield, $D_{i,t}/P_{i,t}$ | dyld |
| $\text{ETOP}_i$ | trailing earnings-to-price view, net income $/$ price | eyld |
| $\text{CETOP}_i$ | cash earnings-to-price view, $\approx$ (net income $+$ D\&A) $/$ price | eyld |
| $\text{EPFWD}_i$ | forward earnings-to-price view (USE4's heaviest; dropped in this build) | eyld |
| $e_{i,\tau},\ s_{i,\tau}$ | annual per-share earnings / sales of stock $i$ in window-year $\tau$ | gro |
| $\beta^{E}_i,\ \beta^{S}_i$ | OLS slope of $e_{i,\tau}$ / $s_{i,\tau}$ on the year index ($\bar e_i,\bar s_i$ their means) | gro |
| $\text{EGRO}_i$ | normalised earnings-growth slope, $\beta^{E}_i/\bar e_i$ | gro |
| $\text{SGRO}_i$ | normalised sales-growth slope, $\beta^{S}_i/\bar s_i$ | gro |
| $\text{EGRLF}_i$ | forward analyst earnings-growth forecast (USE4's heaviest GROWTH view; unavailable here) | gro |
| $g_i$ | raw GRO descriptor, $\approx 0.90\,\text{EGRO}_i + 0.10\,\text{SGRO}_i$ | gro |
| $\text{ME}_i,\ \text{BE}_i$ | market equity / book (common) equity of stock $i$ | lev |
| $\text{LD}_i,\ \text{TD}_i,\ \text{TA}_i$ | long-term debt / total debt / total assets of stock $i$ | lev |
| $\text{MLEV}_i$ | market leverage, $(\text{ME}_i+\text{LD}_i)/\text{ME}_i$ | lev |
| $\text{DTOA}_i$ | debt-to-assets, $\text{TD}_i/\text{TA}_i$ | lev |
| $\text{BLEV}_i$ | book leverage, $(\text{BE}_i+\text{LD}_i)/\text{BE}_i$ | lev |
| $\widetilde{(\cdot)}$ | a sub-descriptor standardised over the ESTU before compositing | lev |
| $c_i$ | raw LEV composite, $0.75\,\widetilde{\text{MLEV}}_i+0.15\,\widetilde{\text{DTOA}}_i+0.10\,\widetilde{\text{BLEV}}_i$ | lev |
| $V_{i,d},\ S_{i,d}$ | daily volume / shares outstanding of stock $i$ on day $d$ (turnover $=V/S$) | liq |
| $\text{STOM}_i,\ \text{STOQ}_i,\ \text{STOA}_i$ | log share turnover over 1 / 3 / 12 months | liq |
| $r_{i,\tau},\ r_{m,\tau}$ | excess daily return of stock $i$ / of the cap-weighted market on day $\tau$ | beta |
| $\omega_\tau$ | exponential time-decay weight $\exp(-\ln2\,(t-\tau)/h)$, half-life $h$ | beta |
| $\beta_i$ | raw market beta: time-weighted regression slope $\Cov_\omega(r_i,r_m)/\Var_\omega(r_m)$ | beta |
| $h$ | half-life of an exponential time-decay weight | beta |
| $L,\ T,\ W$ | momentum lag ($\approx 21$d), window length ($\approx 504$d), full-window weight sum | mom |
| $m_i$ | raw momentum: lagged exp-weighted mean of excess log returns, $(\sum_\tau\omega_\tau r_{i,\tau})/W$ | mom |
| $\sigma^{D}_i,\ \text{CMRA}_i,\ \sigma^{H}_i$ | RESVOL sub-descriptors: daily-return std / cumulative-path range / market-residual std | resvol |
| $v_i$ | raw RESVOL composite $0.75\sigma^{D}_i+0.15\text{CMRA}_i+0.10\sigma^{H}_i$, then orthogonalised to beta | resvol |
| $\gamma_t$ | per-date slope from orthogonalising one standardised exposure on another (the removed component) | resvol |
| $\beta^{z}_i,\ (\beta^{z}_i)^3$ | standardised Beta exposure and its cube | nlb |
| $n_i$ | raw NLB: orthogonalisation residual $(\beta^{z}_i)^3-\gamma_t\beta^{z}_i$ | nlb |
| $s^{z}_i,\ (s^{z}_i)^3$ | standardised Size exposure and its cube | nls |
| $n_i$ | raw NLS: orthogonalisation residual $(s^{z}_i)^3-\gamma_t s^{z}_i$, then winsorised | nls |
| $X^{\text{ind}}_{ij}$ | industry exposure: $1$ if stock $i$ is in industry $j$, else $0$ (dummy) | industries |
| $f^{\text{ind}}_j,\ w_j$ | industry $j$'s factor return and cap weight; constraint $\sum_j w_j f^{\text{ind}}_j=0$ | industries |
| $J$ | number of industry factors ($=55$ in this build) | industries |
| $v_i$ | cross-sectional regression weight $\propto\sqrt{\text{mcap}_i}$ (normalised per date) | csr |
| $f_c$ | country (market) factor return; country exposure $\equiv 1$ for all stocks | csr |
| $\Vmat_\ell$ | lag-$\ell$ autocovariance of daily factor returns (Newey--West input) | fcov |
| $b_k$ | bias statistic: realised $/$ predicted risk of factor/eigenfactor $k$ (target $\approx 1$) | fcov |
| $B_t,\ \lambda_{\text{vol}}$ | daily cross-sectional bias; volatility-regime multiplier $\sqrt{\operatorname{EWMA}(B_t)}$ | fcov |
| $\sigma^{\text{spec}}_n$ | forecast specific (idiosyncratic) volatility; $\Del=\diag(\sigma^{\text{spec},2}_n)$ | specific risk |
| $\gamma_n$ | specific-risk coverage coefficient (history credibility: time-series vs structural) | specific risk |
| $\text{MCTR}_i,\ \text{CTR}_i$ | marginal / component contribution to risk; $\sum_i\text{CTR}_i=\sigma_p$ (Euler) | risk decomp |
| $\wvec_a=\wvec_p-\wvec_b$ | active weights vs the cap-weighted ESTU benchmark; gives tracking error | risk decomp |
