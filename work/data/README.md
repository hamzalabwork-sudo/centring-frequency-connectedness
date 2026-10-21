# Data

## Crypto data (not redistributed)
These are one-minute klines from the exchanges' public market-data interfaces:
- BTC and ETH spot (USDT) and USDT-margined perpetual contracts;
- Binance, OKX, Bitget, Bitfinex and Bybit;
- January 2021 to September 2026.

Download them from `work/`:

```bash
for ex in binance okx bitget bitfinex bybit; do python ../code/full_download.py --exchange $ex; done
python ../code/check_download.py data/raw
```

The download is resumable. To halve the time on the rate-limited exchanges (OKX, Bitget), run a second process with `--reverse`. The files land in `data/raw/<exchange>/<exchange>_<asset>_<spot|perp>_<YYYY-MM>.csv.gz`, about 0.9 GB in total. `code/build_rv.py` turns them into hourly log realised variance in `data/processed/`.

## Example datasets of ConnectednessApproach
The 18 datasets distributed with the R package are `.rda` files. Each comes from a published study documented in the package. Convert them to CSV with:

```bash
Rscript ../code/convert_package_data.R pkgdata/rda pkgdata/csv
```

The script downloads the `.rda` files from the CRAN mirror on GitHub if they are not already present.
