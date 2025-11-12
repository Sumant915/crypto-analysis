# Crypto Analysis DASHBOARD

This repository contains an analysis of the cryptocurrency market. The goal of the project is to ingest live or historic crypto data from the CoinGecko public API, clean and transform it using Power Query (Power BI / Power Query Editor), and produce interactive analyses and dashboards in Power BI Desktop. All cleaned outputs and exported visuals are stored in the `Result/` folder.

## Overview

- Data source: CoinGecko public API (the repo may also contain exported raw snapshots in `raw data/` as `rawdataapi.txt` — these are optional offline copies). When possible prefer calling the API directly from Power Query so you can refresh the dataset.
- Cleaning & ETL: Power Query (in Power BI Desktop or Excel Power Query).
- Editor / Analysis: Power BI Desktop (Power Query for transformations, Report view for visuals).
- Results: Exported CSVs, images, or Power BI reports are saved under `Result/`.

## Repository structure

- `raw data/` — original raw data files (example: `rawdataapi.txt`). Do not modify these files; keep them as the source-of-truth.
- `Result/` — cleaned datasets, exported visuals, and final report artifacts.
- `README.md` — this document.

## What I did (high-level)

1. Acquired cryptocurrency data using the CoinGecko public API (or used an exported snapshot placed in `raw data/`).
2. Loaded the API data into Power BI Desktop using Get Data -> Web (or used `Web.Contents` from Power Query) and opened Power Query Editor.
3. Performed data cleaning and transformation in Power Query (detailed, step-by-step guidance below).
4. Built dashboards and visualizations in Power BI (time-series charts, volume analysis, correlations, volatility, aggregated returns, etc.).
5. Exported cleaned datasets and final visualizations into `Result/` for sharing and downstream use.

## Data cleaning (Power Query) — reproducible steps

Use Power BI Desktop > Get Data > Web (or Get Data -> Blank Query + Advanced Editor) and then click Transform Data to open Power Query Editor. The common sequence of transformations I applied when pulling data from CoinGecko's API:

- Inspect and set types: ensure Date/Time, numeric, and text columns have correct types.
- Parse dates and times: convert timestamps to UTC/local timezone as needed.
- Trim and clean text columns: remove leading/trailing whitespace.
- Remove completely empty rows and columns.
- Remove duplicates (Home -> Remove Rows -> Remove Duplicates) using the appropriate key (e.g., timestamp + symbol).
- Handle missing values: either remove rows with critical missing fields, or fill/replace missing numeric values (e.g., with 0 or with the previous value) depending on the column semantics.
- Expand JSON records: when you call CoinGecko endpoints you receive JSON — expand record columns and transform arrays into tables using Expand/Record and Expand/List operations.
- Create derived columns:
	- Daily returns and log returns (Power Query formulas shown below)
	- Rolling windows (7 / 30-day moving averages) — implementable in Power Query using grouping and List functions or calculated measures in Power BI after load
	- Volatility measures (rolling standard deviation)
	- UTC-normalized date columns (Date, Year, Month)
- Filter outliers or erroneous entries if present (e.g., negative prices where not expected).
- Sort rows by timestamp and symbol.
- Promote headers / clean top-level JSON structure.

Once transformations are complete, use Close & Apply. To persist a cleaned dataset for other tools, export the table to CSV using the Data view or use "Export data" from the visuals. Save exported CSV(s) into `Result/` and commit them to the repo when appropriate.

Notes / assumptions about cleaning:
- Source: this README assumes you are using the CoinGecko public API which returns JSON. If you instead have offline CSV snapshots in `raw data/`, those work too.
- Rate limits and best practice: respect CoinGecko's API rate limits and terms of service — if you plan to fetch many coins or long histories, batch requests and add throttling (for large bulk fetches, export snapshots and store in `raw data/`).
- For timezone-sensitive analyses, ensure timestamps are normalized to a single timezone (I recommend UTC).

### Example: fetching market data from CoinGecko in Power Query (single page)
Below is a compact example Power Query M snippet that calls the CoinGecko "coins/markets" endpoint for a single request and converts the JSON to a table. Use Get Data -> Blank Query -> Advanced Editor and paste this as a starting point.

let
	baseUrl = "https://api.coingecko.com/api/v3/coins/markets?vs_currency=usd&order=market_cap_desc&per_page=250&page=1&sparkline=false",
	raw = Json.Document(Web.Contents(baseUrl)),
	table = Table.FromList(raw, Splitter.SplitByNothing(), null, null, ExtraValues.Error),
	expanded = Table.ExpandRecordColumn(table, "Column1", {"id","symbol","name","current_price","market_cap","total_volume","last_updated"}, {"id","symbol","name","current_price","market_cap","total_volume","last_updated"}),
	changedTypes = Table.TransformColumnTypes(expanded, {{"current_price", type number}, {"market_cap", Int64.Type}, {"total_volume", Int64.Type}, {"last_updated", type datetimezone}})
in
	changedTypes

Notes: this returns one page (up to `per_page`). For full coverage you need to page through `page=1..N` and combine results (example pattern below).

### Example: basic pagination pattern (conceptual)
You can implement a paging loop in Power Query using a small generator function that calls the API until an empty page is returned. Example (conceptual):

let
	base = "https://api.coingecko.com/api/v3/coins/markets?vs_currency=usd&order=market_cap_desc&per_page=250&page=",
	getPage = (p as number) => Json.Document(Web.Contents(base & Text.From(p) & "&sparkline=false")),
	pages = List.Generate(()=>1, each List.Count(getPage(_))>0, each _ + 1),
	all = List.Combine(List.Transform(pages, each getPage(_))),
	table = Table.FromRecords(all)
in
	table

Replace the conceptual snippet with a tested implementation before running at scale; add throttling (Delay) if you encounter rate limits.

## Analysis (Power BI) — what was produced

Typical analyses and visuals included in this project:

- Time-series price charts (line charts) for selected cryptocurrencies (single coin and multi-coin comparison).
- Volume analysis (bar charts) and correlation with price movements.
- Market-cap trends (if market cap present in raw data).
- Daily / monthly returns and cumulative returns.
- Volatility and rolling volatility (7/30/90-day) charts.
- Drawdown charts and top N movers (largest gains / losses over a window).
- Correlation matrix between coin returns.
- Heatmaps (volatility or correlation) and scatter plots for cross-asset comparisons.

All Power BI report files, exported images, and any CSVs produced from the cleaned model can be found in `Result/`.

## How to reproduce the analysis (short guide)

1. Open Power BI Desktop.
2. Get Data -> Web (or Blank Query) and use the Power Query M examples above to call CoinGecko endpoints or load an offline snapshot from `raw data/rawdataapi.txt`.
3. Transform Data -> follow the Power Query steps described above (expand JSON, type casts, dedupe, derive returns and rolling metrics).
4. Close & Apply.
5. Build visuals in Report view (use standard visuals or custom visuals if needed).
6. To share the cleaned dataset, select the table in Data view -> right-click -> Copy Table, or export a visual's data to CSV (More options -> Export data). Save exports into `Result/`.

## Files to check

- `raw data/rawdataapi.txt` — optional raw snapshot (if you exported a CoinGecko API snapshot to disk). If you have a different raw filename, update this readme to reflect it.
- `Result/` — look here for outputs (CSV, images, exported data). If you don't see PBIX files here, add your `.pbix` (Power BI report) for easier sharing.



Last updated: November 12, 2025

