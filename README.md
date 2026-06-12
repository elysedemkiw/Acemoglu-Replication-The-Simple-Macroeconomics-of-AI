# Acemoglu-Replication-The-Simple-Macroeconomics-of-AI
# Realized AI Productivity Gains
### Anchoring Acemoglu's Task-Based TFP Ceiling to BTOS Firm-Adoption Data

Acemoglu (2024) estimates the TFP gain AI will eventually deliver *at full adoption*; this project a) reproduces that result given the author'soriginal parameters and b) anchors that ceiling to real adoption data from US Census Business Trends and Outlooks Survey to ask **how much of that gain has actually been realized so far, and on what trajectory.**

**Main finding:** at today's approx 10% firm adoption, the US economy has realized only about a **tenth** of AI's eventual TFP gain, and how large that fraction is depends almost entirely on where adoption ultimately **saturates**, which the data cannot yet pin down. *Speed is well-identified; the ceiling is a scenario dependent on those four parameters (part 1).*

---

## 1. The core idea

| Piece | Supplies | Source |
|---|---|---|
| **Ceiling** | the *size* of the eventual TFP gain at full adoption (approx 0.66% over a decade) | Acemoglu's task model[^acemoglu] |
| **Trajectory** | the *status & speed* of adoption or how far we've actually climbed | BTOS firm-adoption rate[^btos] |

The whole model is one multiplication:

```
realized TFP gain(t)  =  ceiling  ×  adoption progress(t)
                         └ Acemoglu ┘   └──── BTOS ─────┘
```

Acemoglu sets *how big*; BTOS sets *how much of it is switched on yet*. The ceiling is a constant; the time dimension comes entirely from adoption.

---

## 2. Data sources

| Source | Used for | Notes |
|---|---|---|
| **U.S. Census BTOS**[^btos] | the adoption series `A_t` (share of firms using AI, biweekly) | **narrow** "produce goods or services" wording, 09/2023–10/2025; see definition-break caveat[^break] |
| **Acemoglu (2024)**[^acemoglu] | the ceiling formula + its three input parameters | back-of-envelope via Hulten's theorem |
| ↳ Eloundou et al. (2023)[^eloundou] | task **exposure** (~20%) | "GPTs are GPTs" |
| ↳ Svanberg et al. (2024)[^svanberg] | **profitable-automation** share (~23%) | "Beyond AI Exposure" |
| ↳ Noy–Zhang (2023); Brynjolfsson et al. (2023)[^savings] | **cost savings** per task (~27% of labor cost) | RCT estimates |
| **Anthropic Economic Index**[^aei] | the (abandoned) Section-3 adoption-speed proxy | open on Hugging Face, CC-BY |

---

## 3. Variables & parameters

| Name | Meaning | Value / how obtained |
|---|---|---|
| `exposure` | share of tasks exposed to AI | 0.20[^eloundou] |
| `profitable_share` | of those, share worth automating in 10y | 0.23[^svanberg] |
| `cost_savings` | labor-cost saving per automated task | 0.27[^savings] |
| `labor_share` | labor's share of total cost (α in Acemoglu eq. 14) | 0.53 |
| `ceiling` | eventual TFP gain at full adoption | `exposure·profitable_share·cost_savings·labor_share` = **0.66%** |
| `A["rate"]` | firm AI-adoption rate over time (`A_t`) | BTOS[^btos] |
| `L` | adoption **saturation** (eventual share of firms) | **fitted but fragile**[^L]; or set as a scenario |
| `k` | diffusion **speed** (logistic steepness) | least-squares fit; well-identified[^L] |
| `t0` | inflection year of the S-curve | least-squares fit |
| `progress` | how far along, in [0,1] | `rate / L` |
| `tfp_path` | realized TFP gain to date | `ceiling × progress` |
| `q_growth` | per-quarter productivity shock | `tfp_path.resample("QE").last().diff()` |

---

## 4. Notebook structure — the seven sections

### 1 · Parameters
Defines the four Acemoglu scalars and computes the **ceiling** via the Hulten chain `exposure × profitable_share × cost_savings × labor_share`.
*Functions:* `Params` (dataclass of knobs), `tfp_gain_decade(P)`, `annualize()`.

### 2 · Sensitivity — how the estimate depends on its assumptions
Sweeps **exposure × cost-savings** on a grid and plots the resulting annual TFP gain, marking Acemoglu's calibration. The fan shape shows the result is **multiplicative** in two contested inputs — the entire modest-vs-transformative debate reduces to *where on this chart you sit*.

### 3 · Extension: adoption speed from the Anthropic Economic Index *(a flop — kept for honesty)*
Attempted to build the adoption trajectory from EI task-share data (`A_breadth = 1 − top-10-task share`). **It failed, instructively:** the measure came out flat and noisy, confounded by changing task counts and methodology breaks across releases. **Conclusion:** the EI measures the *composition* of AI use, not the *level* of adoption — so it can't supply a diffusion trajectory. This motivated the pivot to BTOS.[^flop]

### 4 · Anchoring Acemoglu's ceiling to BTOS adoption *(the core)*
Treats the BTOS adoption rate as **progress toward full uptake** and multiplies it by the ceiling: `tfp_path = ceiling × (rate / L)`. A logistic curve is fit to the rate to recover the speed `k` and (fragile) saturation `L`, and the path is projected forward.
*Functions:* `logistic()` + `scipy.optimize.curve_fit`.

### 5 · Aggregate to a quarterly productivity shock
Resamples the biweekly `tfp_path` to end-of-quarter levels and differences them, yielding the **per-quarter TFP increment** — the productivity shock the gap model consumes.

### 6 · Saturation scenarios — bracket the unknown ceiling
Re-fits the speed under **low / medium / high** assumed saturation `L`, since `L` is the one parameter the still-rising data can't identify.[^L] The wide spread in "% of the way realized" is the project's honest headline.
*Functions:* `fit_fixed_L(L)`.

### 7 · Feeding the shock into a compact gap model *(exclusively illustrative)*
Pours the quarterly shock into a small backward-looking 3-equation New-Keynesian model (Phillips / IS / Taylor) to *visualize* the channel: AI lowers costs → inflation dips → the central bank eases → output booms → reverts as adoption saturates. **Not robust by design**[^gap] parameters are assumed and `lam` is cranked for visibility; only the *direction* is meaningful.

---

## 5. Reusable functions

| Function | What it does |
|---|---|
| `tfp_gain_decade(P)` | Hulten/Acemoglu ceiling from the four scalars |
| `annualize(g, years=10)` | geometric per-year rate from a cumulative gain |
| `clean_btos(path)` | BTOS Excel → filter Q7/"Yes" → `melt` wide→long → parse dates → tidy `(date, rate)` |
| `logistic(t, L, k, t0)` | the diffusion S-curve fit by `curve_fit` |
| `fit_fixed_L(L)` | fits speed `k`, `t0` with saturation `L` held fixed (for scenarios) |
| `Config` / `run(A, cfg)` / `summary(res)` | **plug-and-play** module (`ai_tfp_pipeline.py`): change any knob, re-run the whole pipeline |

---

## 6. Extensions / future work

- **Task-level Hulten weighting** — replace the scalar chain with O\*NET tasks weighted by BLS employment shares, rather than economy-wide averages
- **Split the cost-savings channel** — separate automation from task-complementarity (same TFP, opposite wage effects) to recover Acemoglu's distributional results; requires moving beyond a representative-agent frame
- **Ground `cost_savings` in data** — use the EI "primitives" (`human_with_ai_time` vs `human_only_time` per task) instead of Acemoglu's 27%
- **Better adoption series** — chain-link the BTOS narrow→broad break (if an overlap period exists), or SOC-aggregate the EI breadth measure to defeat the granularity confound
- **Real Stage 3** — replace the toy gap model with the forward-looking QPM (EViews/Dynare) for credible magnitudes

---

## 7. Caveats & honest limits

1. **Speed known, ceiling unknown.** `k` is identified from the rise; `L` is not (no plateau in the data), and `L` drives "how far along" we are [^L]
2. **The ceiling itself is contested.** Acemoglu's 0.66% is a conservative back-of-envelope; the optimists assume far higher exposure and savings (see Section 2)
3. **BTOS definition break.** The AI question switched from "produce goods or services" (~10%) to "any business function" (~18%) in late 2025; this project uses the **narrow** series for consistency and does **not** splice the two [^break]
4. **The gap model is illustrative only**[^gap]

---

## Footnotes

[^acemoglu]: Acemoglu, D. (2024). *The Simple Macroeconomics of AI.* NBER WP 32487 / *Economic Policy* (2025). Headline: ~0.5–0.7% TFP over a decade (~0.06–0.07%/yr).
[^btos]: U.S. Census Bureau, *Business Trends and Outlook Survey (BTOS)*, AI core question (Q7), national, biweekly. https://www.census.gov/hfp/btos · Public domain. Begin the series at 09/11/2023 (BTOS v2); v1 (pre-09/2023) is not comparable.
[^break]: The Q7 wording changed from "use AI in *producing goods or services*" (narrow, ~10%) to "use AI in *any of its business functions*" (broad, ~18%) around late 2025. The level jump is a definition change, not real adoption — the two series are not spliced (no overlap period for empirical chain-linking).
[^eloundou]: Eloundou, T. et al. (2023). *GPTs are GPTs: An Early Look at the Labor Market Impact Potential of LLMs.*
[^svanberg]: Svanberg, M. et al. (2024). *Beyond AI Exposure: Which Tasks are Cost-Effective to Automate with Computer Vision?* (MIT FutureTech).
[^savings]: Noy, S. & Zhang, W. (2023), *Science*; Brynjolfsson, E., Li, D. & Raymond, L. (2023), *Generative AI at Work* (NBER). Both imply ~27% labor-cost savings on exposed tasks.
[^aei]: Anthropic Economic Index. https://huggingface.co/datasets/Anthropic/EconomicIndex · CC-BY. Releases 2025-02 through 2026-03; usage data mapped to O\*NET tasks.
[^L]: Saturation `L` is poorly identified because the adoption series is still on the rising portion of the S-curve (no observed plateau). The speed `k`, read from the slope, is reliable. Treat `L` as a scenario, not an estimate (Section 6).
[^flop]: The EI measures *shares* of usage (task composition), all summing to 100% within a release, not a *level* of economy-wide adoption — so it structurally cannot trace an adoption trajectory. Documenting the null is the point.
[^gap]: Parameters are assumed (not estimated), the disinflation coefficient `lam` is set high purely for visibility, and the model is backward-looking rather than forward-looking. The *direction* of the channel is meaningful; the *magnitudes* are not and must not be read as a forecast. The production analogue is the IMF Quarterly Projection Model (QPM).
