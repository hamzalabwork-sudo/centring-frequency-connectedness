# Centring and band labelling shape frequency connectedness

Replication code for:

> Zhang, Z., Ibrahim, H. *Centring and band labelling shape frequency connectedness.* Manuscript submitted to *Finance Research Letters*.

The frequency decomposition of TVP-VAR connectedness (Diebold–Yilmaz; Baruník–Křehlík; Antonakakis et al.) is widely used to attribute spillovers to short- and long-term horizons. We show that two defaults of the R package [ConnectednessApproach](https://cran.r-project.org/package=ConnectednessApproach) shape that decomposition:
- full-sample centring before a zero-intercept TVP-VAR;
- band labels given as π/ω rather than the cycle length 2π/ω.

The main case study re-estimates the Bitcoin–Ethereum spot–perpetual networks of Shi and Lin (2026, *Finance Research Letters* 113, 110724). The generality check uses the 17 example datasets distributed with the package. The repository also contains:
- simulations with a known truth;
- a pre-specified out-of-sample forecasting exercise.

## Layout
```
code/            all scripts (Python 3.10+, one R script for validation, one for data conversion)
work/            working directory: run every command from here
  data/          downloaded and processed data (not in the repository; see work/data/README.md)
  results/       small result summaries behind every table (large intermediate files are regenerated)
notes/           pre-specified forecasting design and its SHA-256 hash log
```

## Requirements
```
pip install -r requirements.txt     # numpy, pandas, scipy, torch, matplotlib
```
- **R** (with `zoo`, `progress`, `MASS`) is needed only for the validation against the package and to convert the package's example datasets.
- **A GPU** is optional. The decompositions run on CPU, and each 4-variable network takes about 30 seconds.

## Reproducing the results
Run from `work/`:

```bash
cd work

# 1. Data: 1-minute klines from public exchange APIs (Jan 2021 - Sep 2026; resumable)
for ex in binance okx bitget bitfinex bybit; do python ../code/full_download.py --exchange $ex; done
python ../code/check_download.py data/raw

# 2. Hourly log realised variance (cleaning rules in the script header)
python ../code/build_rv.py                                   # panel used for the connectedness results
python ../code/build_rv.py --max-gap 0 --tag causal          # strictly causal panel used for forecasting
python ../code/build_rv.py --sample-minutes 15 --tag rv15    # 15-minute sampling (Bitfinex staleness)

# 3. Replication of Shi & Lin (package settings) and causal EWMA centring
python ../code/run_connectedness.py --mode package --name shilin_package
python ../code/run_connectedness.py --mode realtime --prior-hours 720 --demean ewma --demean-halflife 360 --name shilin_ewma15d
python ../code/compare_shilin.py --runs shilin_package shilin_ewma15d

# 4. Robustness grid (19 runs x 8 networks), mechanism checks, staleness
bash ../code/robustness_grid.sh
python ../code/mechanism_checks.py
python ../code/staleness.py

# 5. The package's example datasets (convert .rda to CSV first, see work/data/README.md)
python ../code/package_datasets.py --data pkgdata/csv --out results/pkgdata

# 6. Simulations (see the header of sim_centering.py for the designs)
python ../code/sim_centering.py --panel data/processed/logrv_ds.csv.gz --out results/sim/centering_level30d.csv
python ../code/sim_centering.py --panel data/processed/logrv_ds.csv.gz --dgp harvar --scales 0 --seeds 5 --out results/sim/centering_harvar.csv
python ../code/sim_centering.py --panel data/processed/logrv_ds.csv.gz --level-mode random --scales 0 1 --seeds 5 --out results/sim/centering_randomlevel.csv

# 7. Pre-specified forecasts (real-time connectedness on the causal panel)
for spec in "rtc_ewma15d_true --mode realtime --prior-hours 720 --demean ewma --demean-halflife 360" \
            "rtc_window_true --mode realtime --prior-hours 720" \
            "rtc_tvint_true --mode realtime --prior-hours 720 --intercept" \
            "pkgc_full_true --mode package"; do
  set -- $spec; name=$1; shift
  python ../code/run_connectedness.py --panel data/processed/logrv_ds_causal.csv.gz --end "2026-09-30 23:00" --true-periods "$@" --name $name
done
C="--panel data/processed/logrv_ds_causal.csv.gz --conn rtc_ewma15d_true rtc_window_true rtc_tvint_true pkgc_full_true --out results/forecast_causal"
python ../code/forecast.py --split validation $C
FRL_UNSEAL_TEST=1 python ../code/forecast.py --split test $C
python ../code/evaluate.py --root results/forecast_causal --split test
python ../code/test_stage.py --root results/forecast_causal --split test
python ../code/bounds_and_flips.py

# 8. Tables and figures
python ../code/make_tables.py --out tables
python ../code/make_figures.py --out figs
```

## Validation against the R package
Our `connectedness.py` ports the package's `TVPVAR`, `BayesPrior`, `FEVD` and `FrequencyConnectedness` step by step. It reproduces the package's R code to within 3×10⁻¹⁰ in the coefficients and 6×10⁻⁶ in all connectedness measures.

```bash
bash ../code/fetch_rsrc.sh rsrc          # downloads the package's R sources from the CRAN mirror
python ../code/validate_r.py --panel data/processed/logrv_ds.csv.gz \
    --cols BTCS_BN BTCP_BN ETHS_BN ETHP_BN --hours 1000 --nlag 2 --nfore 100 --rsrc rsrc --out results/validate/a
```

## Pre-specified forecasting design
`notes/preregistration.md` was written and hashed before any forecast was estimated. Every later amendment is appended to its change log. `notes/preregistration.sha256` lists the hash of each version, and `notes/unseal.log` records when the test period was opened.

The file in this repository is the final version. Check it with:
```bash
sha256sum notes/preregistration.md      # equals the last hash in notes/preregistration.sha256
```

The design history is disclosed in the paper's supplementary material:
- one look-ahead correction after the test period was opened;
- analyses added after the validation year.

`work/results/forecast_original/` holds the results before the correction, and `work/results/forecast_causal/` the reported ones.

## Data
- **Crypto data:** collected from the public market-data interfaces of Binance, OKX, Bitget, Bitfinex and Bybit, using `code/full_download.py`. Raw data are not redistributed.
- **Other datasets:** distributed with ConnectednessApproach (see `work/data/README.md`).

## License
MIT (see `LICENSE`).
