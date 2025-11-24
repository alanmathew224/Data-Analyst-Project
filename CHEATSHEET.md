# Customer Segmentation — Cheat Sheet (One page)

**Purpose:** quick reference for data & marketing teams to run the segmentation pipeline and interpret outputs.

---
## 1) Required input columns (invoice-level)
Use exactly these column names in the raw dataframe before running the pipeline:
- `InvoiceNo`
- `StockCode`
- `Description`
- `Quantity`
- `InvoiceDate` (datetime format, e.g. `2009-12-01 07:45:00`)
- `UnitPrice`
- `CustomerID`
- `Country`
- `YearMonth`
- `TotalPrice` (Quantity * UnitPrice; negative values indicate returns)
- `period` (optional — pipeline can assign: `'year1'` / `'year2'`)
- `OrderCount`
- `Month`
- `Season` (one of `Winter`, `Spring`, `Summer`, `Fall`)

---
## 2) Output fields produced (key columns)
- `rev_y1`, `rev_y2` — net revenue in Year1 / Year2 (TotalPrice sum, includes negative returns)  
- `rev_combined` — rev_y1 + rev_y2 (used for VIP/CLTV)
- `seg_y1`, `seg_y2` — revenue tier per year (Mass / Premium / Elite / Hibernating)
- `final_segment` — final label (VIP overrides Elite where applicable)
- `status` — customer status (New / Active / Potential / At Risk / Can't Lose Them / Hibernating)
- `VIP_flag` — boolean marker for VIP customers
- `one_off_flag` — True if a single invoice accounts for ≥ `ONE_OFF_PCT_TH` of that customer's revenue on at least one period
- `seasonal_flag`, `best_season` — whether customer is seasonal + which season
- `repetitive_flag` — buys very few unique products (repetitive behaviour)
- `freq_pos_y1`, `freq_pos_y2`, `freq_neg_y1`, `freq_neg_y2` — invoice-level frequency counts
- `recency_days` — days since last invoice (measured to `cutoff_date`)
- `tenure_days` / `active_days` — first → last purchase span (overall)
- `freq_rate_y1`, `freq_rate_y2` — the result of `freq_pos` divided by `tenure_days`
- `rev_drop_pct` — ((rev_y1 - rev_y2) / rev_y1) * 100 (defined only when rev_y1 > 0)

---
## 3) Segment & behavior definitions (short)
**Segments (by net revenue, year2 primary):**
- **VIP** — top revenue contributors *and* recent & active. (Final override above Elite.)  
  **Rule (project):** `seg_y2 == 'Elite'` **AND** `recency_days <= 91` **AND** `rev_combined >= VIP_threshold` **AND** `freq_pos_y2 >= 12` **AND** `one_off_flag == False`. 
- **Elite** — `rev_y2 >= p98` (98th percentile of active year2 net revenue)
- **Premium** — `rev_y2 >= p80` and `< p98` (80th–98th percentile)
- **Mass** — remaining active customers (rev_y2 > 0)
- **Hibernating** — rev_y2 == 0 (no activity in year2)

**Status (lifecycle / trends):**
- **New** — only in year2 and recent (recency ≤ `NEW_RECENCY_DAYS`) and low freq (`freq_total_y2 < NEW_FREQ_TH`)
- **Active** — present in both years with same segment (or otherwise consistent buyers)
- **Potential** — Mass in y2 but meets Mass→Potential override (monetary > p60, freq, recency, not one-off)
- **At Risk** — customer showing decline (recency or rev drop overrides — especially for Elite/Premium)
- **Can't Lose Them** — was Elite in y1 but dropped to Mass/Hibernating in y2 (highest re‑engagement priority)
- **Hibernating** — seen in y1 only (no year2 activity)

**Behavior flags:**
- **Seasonal Buyer** — ≥ `SEASONAL_THRESH` share of year2 revenue in a single season **AND** tenure > `TENURE_DAYS_THRESHOLD`
- **One-off Buyer** — a single invoice accounts for ≥ `ONE_OFF_PCT_TH` of the customer's revenue (pct_top1)
- **Repetitive Buyer** — very few unique SKUs in year2 (<= percentile defined by `ABS_REPETITIVE_TH`)
- **Frequency groups** — derived from invoice-level frequency (freq_rate thresholds to label Unknown / Low / Regular / High)

**Product category:**
- **Top** — `TotalRevenue >= TopProd_Thresh` (Top ~33% Revenue Share) on at least 1 period
- **Mid** — `TotalRevenue >= MidProd_Thresh` (Mid ~33% Revenue Share)on at least 1 period
- **Bottom** — `TotalRevenue < MidProd_Thresh` (Bottom ~33% Revenue Share) on at least 1 period

---
## 4) Key thresholds (project defaults)
These values are defined in the pipeline and can be tuned centrally.

- `ELITE_PCT = 0.98`  (Elite threshold)
- `PREMIUM_PCT = 0.80` (Premium threshold)
- `VIP`: `rev_combined >= total_2yr_net_revenue * 0.005` AND `recency_days <= 91` AND `seg_y2 == 'Elite'` AND `freq_pos_y2 >= 12` AND `one_off_flag == False`
- `MONETARY_PCT_FOR_POTENTIAL = 0.65`
- `FREQ_FOR_POTENTIAL = 40`
- `RECENCY_FOR_POTENTIAL_DAYS = 90`
- `ONE_OFF_PCT_TH = 0.5` (one-off if single invoice ≥50% of customer rev)
- `NEW_RECENCY_DAYS = 30`
- `NEW_FREQ_TH = 3`
- `SEASONAL_THRESH = 0.5`
- `ABS_REPETITIVE_TH = 50`
- `TENURE_DAYS_THRESHOLD = 273` (used for seasonal detection)
- `RECENCY_OVERRIDE_DAYS = 91` (Elite/Premium override: recency > 91 → At Risk)
- `REV_DROP_PCT = 50` (Elite/Premium override: rev drop ≥50% → At Risk)
- `FREQ_FLAG_THRESH:` Low `> 40`, Reguler `<= 40`, High `<= 18.25`, Unknown (`freq_pos < 5`)
- `TopProd_Thresh= 0.97` (Percentile 0.97)
- `MidProd_Thresh= 0.87` (Percentile 0.87)

---
## 5) Quick implementation checklist
1. Parse `InvoiceDate` as `datetime`. Ensure `TotalPrice = Quantity * UnitPrice`.  
2. Aggregate to invoice-level: `invoice_value` = sum(TotalPrice) for each `InvoiceNo` + `CustomerID`.  
3. Aggregate per customer × period → compute `rev_y1`, `rev_y2`, `num_invoices`, `distinct_skus`, `first/last_invoice_date`.  
4. Compute behavioral flags (pct_top1, seasonal shares, unique products, freq_pos/neg).  
5. Assign `seg_y1`, `seg_y2` using percentiles; assign initial `status`; apply overrides (Mass→Potential, Elite/Premium At‑Risk overrides, VIP override).  
6. Export `customer_segments_light` and `customer_segments_final` CSVs and keep an `order_level_flags` table for auditing.

---
## 6) Notes & recommendations
- Keep a versioned record of threshold changes (Git / README).  
- When presenting to marketing, include counts + CLTV per (Segment × Status) to prioritize quick actions (CSV or one‑pager).  
- Run sensitivity checks for VIP and one‑off thresholds before rolling any automated high-touch outreach.

---
