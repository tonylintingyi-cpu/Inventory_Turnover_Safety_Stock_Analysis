# Veltix Inventory Optimization

A portfolio analysis project that identifies underperforming inventory in a synthetic consumer-electronics distributor, segments slow movers by demand volatility, and simulates differentiated safety-stock policies. The output is an interactive dashboard deployed to GitHub Pages.

**[Live Dashboard →](https://tonylintingyi-cpu.github.io/Inventory_Turnover_Safety_Stock_Analysis/)**

![Safety stock holding cost vs service level by CV group](output.png)

## Key Findings

- **30% of the catalog drags capital efficiency.** Turnover is cleanly bimodal — 15 SKUs sit at 0.81–1.75 turns/yr (well under the 4.5 industry floor) while the other 35 land at 4.56–8.00 with nothing in between. Worst case is MA-009 at 0.81×, roughly 15 months of stock on hand.
- **Slow SKUs split into three distinct demand patterns, one per category.** Mobile Accessories / Computer Peripherals / Smart Home sell at stable but persistently weak volumes (CV 0.14–0.22, "low_cv"); Audio shows moderate swings (CV 0.50–0.56, "mid_cv"); Wearables are highly irregular (CV 0.70–0.75, "high_cv"). The grouping maps 1:1 onto categories.
- **A uniform service-level policy wastes money.** At 95% service level, mean weekly holding cost is $5.4 / $18.5 / $33.5 across the three groups — a 6.2× gap. Each group warrants a different policy: low_cv needs a product/pricing review (not more stock), mid_cv hits a sweet spot at 90% SL, high_cv should cap at 85–90% and invest in better forecasting instead.

## What This Project Does

Veltix is a (fictional) consumer-electronics distributor sitting on slow inventory — some SKUs are genuinely weak sellers, others have demand too erratic to stock efficiently. This project identifies which SKUs underperform, diagnoses *why* by separating "low and stable" from "low and erratic", and simulates safety-stock strategies tailored to each demand profile.

## Analysis Approach

The analysis runs in three phases, each building on the previous:

**Phase 1 — Inventory Turnover.** Calculate annual turnover for all 50 SKUs as total sales ÷ weekly average inventory across the 156-week window. Consumer-electronics distributors typically turn inventory 4.5–9× per year; SKUs well below that floor are flagged as low-performers. In this dataset the cutoff is unambiguous because the distribution is bimodal with a clean gap between 1.75 and 4.56.

**Phase 2 — Demand Volatility Segmentation.** For the low-turnover SKUs, compute the Coefficient of Variation (CV = σ/μ) on weekly sales — including zero-sales weeks, since sporadic demand is exactly the kind of volatility that matters for stocking decisions. Bin into low (CV < 0.3), medium (0.3–0.6), and high (> 0.6) groups. These thresholds are narrower than the conventional 0.2 / 0.7 to keep enough SKUs in each bucket for meaningful per-group analysis.

**Phase 3 — Safety Stock Simulation.** For each volatility group, simulate safety stock and holding cost at 85% / 90% / 95% service levels using the full formula `SS = Z × √(L̄·σ_d² + d̄²·σ_L²)`, which accounts for both demand variability and lead-time variability. Lead-time stats are derived from receipt weeks only (filtering out the zero-receipts weeks that would otherwise drag the mean to zero). Output is a tradeoff curve showing holding cost vs. service level for each group.

## Dataset

Synthetic dataset: 50 SKUs × 156 weeks = 7,800 rows, with ~26% dirty data deliberately injected to exercise the cleaning workflow. The data is designed so that normal SKUs land at 4.5–9× annual turnover and 15 low-turnover SKUs (three per category) fall below 2×. CV segmentation emerges naturally from each category's seasonality profile — Wearables with strong Q4 spikes produce high CV, while stable categories like Mobile Accessories produce low CV.

See [`docs/veltix_data_dictionary.md`](docs/veltix_data_dictionary.md) for field definitions and [`docs/veltix_data_generation_prompt.md`](docs/veltix_data_generation_prompt.md) for generation logic.

## Execution Summary

### Data Cleaning (7,800 → 7,613 rows)

Cleaning followed an SOP organized by processing flow (overview → key/category cleanup → integrity checks → imputation → final validation) rather than by check type. Major issues resolved:

1. **Key & category normalization.** `product_category` had 35 surface variants (`Mob. Acc.`, `Smart Home Devices`, whitespace, mixed case) collapsed to 5 canonical labels via a two-pass mapping. `sku_id` had visual-character corruptions (`O`→`0`, `L`→`1`, `RNA-`→`MA-`) resolved by cross-validating against `unit_cost` (same product = same unit cost). `week_id` format inconsistencies (slashes, reversed `W45-2025`, missing leading zeros) normalized to `YYYY-Www`. → 241 SKU values condensed to 50, 35 categories to 5, 277 week values to 156.
2. **Duplicates.** 234 fully-duplicate rows removed, confirmed by `df.duplicated().sum() == df.duplicated(subset=keys).sum()` (i.e. duplicates were complete, not partial).
3. **Inventory balance equation.** `inv_begin + receipts − sales = inv_end` flagged 219 mismatches. Split into sign errors (68 rows with negative values that simply needed flipping) versus genuine logic errors (151 rows where the equation didn't hold — dropped after confirming no category/week clustering).
4. **Missing-value imputation.** 541 missing values in `sales_qty` and `inventory_begin` (no overlap) back-solved from the now-validated balance equation.
5. **Lead-time logic.** `lead_time_days` should be 0 when `receipts == 0`; found 33 rows violating this rule plus 2 outliers > 56 days. Dropped all 35.

Final: 7,613 clean rows in `data/cleaned/veltix_cleaned.csv`. Full walkthrough in [`notebooks/data_cleaning.ipynb`](notebooks/data_cleaning.ipynb).

### Phase 1 — Turnover Distribution

The distribution is cleanly bimodal: 15 SKUs at 0.81–1.75 turns/yr, 35 at 4.56–8.00, no SKUs in between. Every slow SKU ends in 008–010 (three per category) — a catalog-wide pattern (likely lifecycle decline or deliberate sandbagging) rather than a category-specific problem. The worst case, MA-009 at 0.81 turns/year, sits at roughly 15 months of stock on hand and ~18% of the industry floor.

### Phase 2 — CV Segmentation

The CV distribution splits the 15 slow movers into three patterns of *how* they fail to sell:

- **9 low-CV SKUs** (Mobile Accessories / CP / Smart Home; CV 0.14–0.22) — stable but persistently weak. Demand is easy to forecast, so the problem isn't stocking policy; it's the product itself.
- **3 mid-CV SKUs** (Audio; CV 0.50–0.56) — moderate swings, consistent with early-lifecycle behavior or mild seasonality.
- **3 high-CV SKUs** (Wearables; CV 0.70–0.75) — highly irregular, std exceeding 70% of mean. The inventory pile-up likely stems from over-buffering against stockouts.

The groups align 1:1 with product categories, reflecting that each category carries its own demand profile — a structure that drives the differentiated safety-stock design in Phase 3.

### Phase 3 — Safety Stock & Differentiated Recommendations

Mean weekly holding cost by group:

| Group | 85% SL | 90% SL | 95% SL | vs low_cv (95%) |
|---|---|---|---|---|
| low_cv  (9 SKUs) | $3.4 | $4.2 | $5.4 | 1.0× |
| mid_cv  (3 SKUs) | $11.6 | $14.3 | $18.5 | 3.4× |
| high_cv (3 SKUs) | $21.1 | $26.0 | $33.5 | 6.2× |

The relative 85% → 95% uplift is identical across groups (~+59%, since SS scales linearly with Z); what differentiates them is **absolute level** — high_cv is 6.2× more expensive than low_cv at every service level.

**Low CV (9 SKUs).** 95% SL is essentially free ($5.4/wk incremental). But the prior question is whether to keep these SKUs at all — demand is easy to forecast yet persistently weak, so raising service level won't generate sales. The real lever is revisiting pricing, positioning, or planning a clearance/EOL.

**Mid CV (3 SKUs, Audio).** Set service level at 90% ($14.3/wk) to balance cost and service. Pushing to 95% adds $4.2/wk for limited stockout improvement. Monitor the sales trend — if there's an upward inflection, revisit raising the service level.

**High CV (3 SKUs, Wearables).** Set service level at 85–90% ($21–$26/wk). Chasing 95% pushes weekly cost to $33.5 (6.2× the low_cv group). These SKUs sell irregularly *and* poorly — rather than over-buffering, the priority should be improving demand forecasting or adopting a conservative stocking stance.

**Key takeaway.** A uniform "95% across the board" service-level policy wastes significant holding cost on the high_cv group while masking the opportunity to fix the underlying product problem in the low_cv group. Different patterns of slow-moving inventory call for different policies.

## Project Structure

```
.
├── index.html                       # Static dashboard (deployed to GitHub Pages)
├── data/
│   ├── raw/veltix_raw_data.csv      # 7,800-row dataset with ~26% dirty data injected
│   └── cleaned/veltix_cleaned.csv   # 7,613 rows after cleaning
├── notebooks/
│   ├── data_cleaning.ipynb          # Cleaning SOP walkthrough
│   └── EDA.ipynb                    # 3-phase analysis (turnover / CV / safety stock)
├── src/generate_data.py             # Synthetic data generator
└── docs/                            # Data dictionary, generation prompt, cleaning SOP
```

## Tech Stack

| Layer | Tool |
|---|---|
| Data cleaning & analysis | pandas / NumPy (Jupyter) |
| Visualization | Plotly.js (CDN) |
| Dashboard deploy | GitHub Pages (static, no build step) |
| Synthetic data generation | Python |

## Built With Claude Code

This project was developed in collaboration with Claude Code as a pair-programming partner across three areas:

- **Synthetic data generator** — Python script written from a spec I drafted (`src/generate_data.py`).
- **Cleaning + EDA notebooks** — Claude Code suggested pandas idioms (vectorized operations, chained transforms, idiomatic groupby patterns) while I drove the SOP order, every cleaning decision (drop vs. impute, which mappings to apply, where to set thresholds), and the three-phase analysis framing.
- **Interactive dashboard** — `index.html` generated from the EDA outputs once the analysis was settled; I specified the layout, chart types, KPI framing, and copy.

The analytical decisions, business interpretations, and structural choices are mine — Claude Code accelerated the implementation and helped me learn pandas patterns I wouldn't have reached on my own.
