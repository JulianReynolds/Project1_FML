# Empirical Asset Pricing via Machine Learning -- Replication

Replication of **Gu, Kelly & Xiu (2020)**, *"Empirical Asset Pricing via Machine Learning"*, published in the *Review of Financial Studies*.

This project builds the full dataset required for the cross-sectional return prediction framework proposed in the paper, covering **March 1957 -- December 2021** across all NYSE, AMEX, and NASDAQ common stocks.

---

## Important: Data Not Included

The data files are **too large to store in this repository** (the final panel alone is ~11 GB in memory). To reproduce the datasets, follow **Steps 1--3** in the `data_download_&_prosessing.ipynb` notebook, which will download and save everything into the `data/` directory.

---

## Notebook Structure

The notebook `data_download_&_prosessing.ipynb` is divided into two main stages:

### Part I -- Data Download (Sections 1--3)

These sections acquire the three raw data sources needed for the replication:

1. **Section 1 -- Firm-Level Characteristics (94 predictors):** Downloads `datashare.zip` (~3--4 GB) from Dacheng Xiu's website. Contains 94 pre-lagged firm characteristics for ~32,800 unique stocks.
2. **Section 2 -- Monthly Stock Returns (CRSP):** Queries CRSP via WRDS to obtain monthly returns, delisting returns, and the risk-free rate. Computes delisting-adjusted excess returns.
3. **Section 3 -- Macroeconomic Predictors (Welch & Goyal 2008):** Loads 8 aggregate time-series predictors from the Welch & Goyal dataset (requires manual download of `PredictorData.xlsx` from Amit Goyal's website).

### Part II -- Data Processing (Sections 4--6)

These sections merge, transform, and assemble the final analysis-ready panel:

4. **Section 4 -- Merge & Feature Construction:** Merges characteristics with CRSP returns (inner join on `permno` and `date`), adds lagged macro predictors, constructs SIC-2 industry dummies, applies cross-sectional rank transformation, and creates characteristic x macro interaction terms.
5. **Section 5 -- Final Panel Assembly:** Concatenates ranked characteristics, interaction terms, and industry dummies into the GKX-style panel (941 columns, ~3.3M observations).
6. **Section 6 -- Train/Validation/Test Split:** Defines the sample periods following the paper: Training (1957--1974), Validation (1975--1986), Test (1987--2021).

---

## Datasets

### Downloaded (Raw) Data

| File | Source | Description | Key Variables |
|---|---|---|---|
| `datashare.csv` | Dacheng Xiu's website | 94 firm-level characteristics, pre-lagged | `permno`, `date`, `mvel1`, `beta`, `mom12m`, `bm`, `ep`, `turn`, `retvol`, ... (94 characteristics + `sic2`) |
| `Monthly_returns_crps.csv` | CRSP via WRDS | Monthly stock returns for common shares on NYSE/AMEX/NASDAQ | `permno`, `date`, `ret` (total return), `ret_adj` (delisting-adjusted), `ret_excess` (excess over risk-free), `me` (market equity), `siccd`, `exchcd` |
| `risk_free_rate.csv` | CRSP via WRDS | Monthly 30-day T-bill rate | `date`, `rf` |
| `PredictorData.xlsx` | Amit Goyal's website | Monthly macroeconomic predictor variables | `yyyymm`, `D12`, `E12`, `b/m`, `tbl`, `AAA`, `BAA`, `lty`, `ntis`, `svar` |
| `Macro_Predictors.csv` | Derived from `PredictorData.xlsx` | 8 constructed macro predictors used in GKX | `date`, `dp` (dividend-price), `ep` (earnings-price), `bm` (book-to-market), `ntis` (net equity expansion), `tbl` (T-bill rate), `tms` (term spread), `dfy` (default spread), `svar` (stock variance) |

### Processed Data

| File | Shape | Description |
|---|---|---|
| `base_data.parquet` | 3,305,648 x 119 | Merged panel of rank-transformed characteristics + returns + macro variables before feature expansion |
| `industry_dummies.parquet` | 3,305,648 x 90 | One-hot encoded SIC-2 industry group indicators |
| `interactions.parquet` | 3,305,648 x 752 | Interaction terms: 94 ranked characteristics x 8 macro predictors |
| `gkx2020_panel.parquet` | 3,305,648 x 941 | **Final analysis-ready panel** with all 936 features (94 characteristics + 752 interactions + 90 industry dummies) plus identifiers (`permno`, `date`, `ret_excess`, `me`, `exchcd`) |

---

## Processing Methodology

The raw data goes through the following transformations to produce the final panel:

1. **Delisting adjustment:** Monthly returns are adjusted for delisting events. If a delisting return is missing and the delisting code indicates a performance-related event (codes 500, 520, 551, 552, 560, 574), a return of -30% is assumed. Adjusted return = (1 + ret) x (1 + dlret) - 1.

2. **Excess returns:** The risk-free rate (30-day T-bill) is subtracted from the delisting-adjusted return to obtain the monthly excess return, which serves as the target variable.

3. **Macro predictor construction:** Eight aggregate predictors are computed from the Welch & Goyal raw data:
   - `dp`: log(D12) - log(Index)
   - `ep`: log(E12) - log(Index)
   - `bm`: book-to-market ratio (directly from source)
   - `ntis`: net equity expansion (directly from source)
   - `tbl`: Treasury bill rate
   - `tms`: term spread (long-term yield - T-bill)
   - `dfy`: default spread (BAA - AAA)
   - `svar`: stock variance

4. **Macro lagging:** Macro predictors are lagged by one month to ensure all information is available at the time of prediction.

5. **Cross-sectional rank transformation:** Each of the 94 firm characteristics is ranked cross-sectionally within each month, then mapped to the interval [-1, 1] using the formula: 2 x (rank - 1) / (N - 1) - 1. Missing values are filled with 0.

6. **Interaction terms:** Each of the 94 ranked characteristics is multiplied by each of the 8 macro predictors, producing 752 interaction features that capture time-varying factor exposures.

7. **Industry dummies:** SIC-2 industry codes are one-hot encoded, yielding 90 industry group indicators.

8. **Final feature count:** 94 (characteristics) + 752 (interactions) + 90 (industry dummies) = **936 features**.

---

## Sample Splits

Following GKX (2020):

| Split | Period | Observations |
|---|---|---|
| Training | 1957--1974 | 454,192 |
| Validation | 1975--1986 | 714,540 |
| Test (OOS) | 1987--2021 | 2,136,916 |

---

## Requirements

- Python 3.12+
- `pandas`, `numpy`, `requests`, `tqdm`
- `wrds` (for CRSP data download -- requires WRDS account)
- `openpyxl` (for reading the macro predictor Excel file)

---

## Reference

Gu, S., Kelly, B., & Xiu, D. (2020). Empirical Asset Pricing via Machine Learning. *The Review of Financial Studies*, 33(5), 2223--2273.
