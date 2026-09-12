# note.md — Customer Segmentation with RFM and Clustering

## Dataset
Online Retail (UCI): 541,909 transaction lines, Dec 2010 – Dec 2011, UK-based online store.
Columns: `InvoiceNo`, `StockCode`, `Description`, `Quantity`, `InvoiceDate`, `UnitPrice`, `CustomerID`, `Country`.

## 1. Feature preparation

### Cleaning (before RFM aggregation)
Transaction-level data required cleaning before it could be aggregated to the customer level:

| Step | Rows removed | Reason |
|---|---|---|
| Drop missing `CustomerID` | 135,080 | Can't attribute a transaction to a customer |
| Drop cancellations (`InvoiceNo` starts with `'C'`) | 8,905 | Represent returns, not purchases |
| Drop non-positive `Quantity`/`UnitPrice` | 40 | Data errors / free items |
| Drop non-product stock codes (`POST`, `D`, `M`, `DOT`, `CRUK`, `BANK CHARGES`, `PADS`) | 1,414 | Postage, fees, manual adjustments — not products |

**Result:** 396,470 clean transaction lines, 4,334 unique customers, matching the dataset's documented scale (~4,000 customers).

### RFM construction
- **Recency** — `(snapshot_date - last_purchase_date).days`, where `snapshot_date` = one day after the last transaction in the data (2011-12-10).
- **Frequency** — count of distinct `InvoiceNo` per customer.
- **Monetary** — sum of `Quantity × UnitPrice` per customer.
- Extra behavioral features for richer profiling: `AvgBasketValue`, `CategoryDiversity` (distinct `StockCode`s bought), `TotalItems`.

### Handling skew
Raw RFM skewness: Recency 1.24, Frequency 11.98, Monetary 19.54 — Frequency and Monetary are extremely right-skewed because a small number of customers place a disproportionate number of orders / spend disproportionately. A `log1p` transform was applied to all three (and the extra features), reducing skewness to Recency −0.38, Frequency 1.21, Monetary 0.40.

### Scaling
After the log transform, features were standardized with `StandardScaler` (zero mean, unit variance) so K-Means/DBSCAN's Euclidean distance isn't dominated by whichever feature happens to have the largest raw range.

## 2. How the number of clusters was chosen

K-Means was evaluated for k = 2..10 using inertia (elbow), silhouette score, and Davies-Bouldin index.

- Silhouette score is **maximized at k=2** (0.433), but this only splits customers into "active" vs "inactive" — not granular enough to build differentiated marketing campaigns around.
- **Decision rule:** choose the smallest k ≥ 4 whose silhouette score is within 75% of the global maximum. This landed on **k=4** (silhouette 0.338), which also sits right where the elbow curve visibly flattens (diminishing inertia reduction beyond k=4).
- This is an explicit, documented trade-off between statistical optimality and business usability — not a value chosen "by eye."

k=4 was cross-checked with:
- **Hierarchical clustering (Ward linkage)** at k=4 — silhouette 0.267, and the dendrogram on a 300-customer sample shows a natural 4-branch structure, independently confirming the K-Means result isn't an artifact of that specific algorithm.
- **DBSCAN** (eps chosen via k-distance plot, eps=0.4, min_samples=5) — also converges on 4 density-based clusters with only 2.3% of points as noise, though with a lower silhouette (0.129 on non-noise points) since DBSCAN optimizes for density rather than the same objective as K-Means/silhouette.

K-Means (k=4) was selected as the primary segmentation for its stability, interpretability, and independent confirmation from Hierarchical clustering.

## 3. What the segments mean for the business

| Cluster | Size | Segment | Recency | Frequency | Monetary (avg) | Revenue share |
|---|---|---|---|---|---|---|
| 3 | 15.8% | **Champions / VIP** | 12 days | 14.0 orders | £8,196 | **63.9%** |
| 2 | 27.0% | **Loyal / Core** | 67 days | 4.2 orders | £1,819 | 24.3% |
| 0 | 19.5% | **Recent / Promising** | 19 days | 2.1 orders | £539 | 5.2% |
| 1 | 37.7% | **At risk / Lapsed** | 183 days | 1.3 orders | £352 | 6.6% |

**Headline finding:** a classic long-tail / Pareto pattern. The top 15.8% of customers (Champions/VIP) generate nearly two-thirds of revenue, while the largest group by headcount (At risk/Lapsed, 37.7% of the base) contributes only 6.6%.

**Suggested actions:**
- **Champions/VIP** — protect this segment: loyalty perks, early access, dedicated outreach. Highest leverage per customer.
- **Loyal/Core** — upsell/cross-sell to graduate them toward VIP status.
- **Recent/Promising** — onboarding nurture and second-purchase incentives to build a habit before they drift toward "at risk."
- **At risk/Lapsed** — structured win-back campaigns; given the segment's size, even a modest reactivation rate matters at scale.

## 4. Bonus work completed
- **Outlier detection:** DBSCAN noise points (2.3%, 98 customers) and Isolation Forest (contamination=0.03, 130 customers) were compared; 62 customers were flagged by both, a meaningful overlap worth investigating individually (possible wholesalers/resellers rather than typical consumers).
- **t-SNE visualization** as a second, non-linear 2D projection alongside PCA, to sanity-check the cluster separation visually.
- **Extended behavioral features:** `CategoryDiversity` was added beyond core RFM and shown to track strongly with segment health (157 distinct categories bought on average for Champions/VIP vs. 22 for At risk/Lapsed).
- An interactive dashboard was out of scope for this pass but the cleaned `customer_segments_final.csv` (customer-level table with cluster, segment name, and outlier flag) is ready to plug into one.

## 5. Files produced
- `segmentation_notebook.ipynb` — full, executed, end-to-end pipeline (cleaning → RFM → prep → clustering comparison → validation → profiling → bonus)
- `Cluster_Profile_Report.docx` — business-facing report with charts and segment write-ups
- `note.md` — this file
- `rfm_table.csv`, `cluster_labels.csv`, `cluster_profile.csv`, `customer_segments_final.csv` — supporting data outputs
