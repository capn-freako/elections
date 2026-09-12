# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository purpose

This is the Election Truth Alliance's data repository: public election data (raw and cleaned) plus
Python tooling to clean that data and generate charts analyzing it (vote share vs. turnout, vote
share vs. precinct size, etc.), mirroring analyses on the ETA's public dashboard
(https://data.electiontruthalliance.org/). It is not a software application — there is no build,
lint, or test suite. Most of the repo is data files (raw source exports and cleaned CSVs) and
tutorials documenting how that data was produced.

## Repository layout

- `data/raw/<STATE>/<County or state-level>/<year>/` — original, unmodified exports from state/county
  election sites (Clarity XML/XLS/TXT detail exports, precinct-level results, voter stats), each with
  its own `README.md` noting provenance.
- `data/clean/<STATE>/<County>/<year>/*.csv` — cleaned, per-contest CSVs derived from the raw data,
  with standardized columns (see naming conventions below).
- `data/scripts/clean_clarity_to_csv.py` — CLI that compiles a per-precinct CSV for one contest out of
  a Clarity "detail" XML export (`--contest-key` or `--choice-key` to select the contest).
- `tutorials/2024_G_NC_Data_Cleaning_Spreadsheet/` — spreadsheet-based walkthrough (`CLEANING.md`,
  `SOURCING.md`) of manually cleaning NC data; source-of-truth for the manual cleaning methodology.
- `tutorials/analysis/` — the chart-generation tooling (see below).

Large/binary raw data files (`*.txt`, `*.pdf`, `*.xlsx` anywhere in the repo, plus everything under
`data/raw/**`) are tracked via Git LFS (`.gitattributes`); ensure `git lfs` is set up before expecting
these to have real content on checkout.

## Chart generation tooling (`tutorials/analysis/scripts/`)

This is the only "code" in the repo. Structure:

- `parameters.py` — the single source of truth for every chart. It's a dict `params` keyed by
  `election_key` → `race_key` → race config. Each race config points at a data file
  (`file`, relative to `scripts/`) and maps that file's columns to standard names
  (`candidate_a_column`, `candidate_b_column`, `total_column`, `registration_column`,
  `min_registered_voters`, candidate colors), then nests one sub-dict per chart type the race
  supports (`scatter_plot`, `turnout_scatter_plot`, `turnout_bar_chart`, `turnout_heatmap`, etc.)
  holding that chart's title/axis labels/chart-specific options.
- `utils.py` — shared helpers used by every chart script:
  - `load_data_frame(path)` — loads CSV or Excel by extension.
  - `clean_num(series)` — strips commas/`%` and coerces to numeric.
  - `get_voter_stats(df, ...)` — the standard data-cleaning pipeline: coerces key columns numeric,
    drops rows with non-positive registration/total, applies `min_registered_voters` filtering,
    computes `turnout_percent` and `<candidate>_share` columns, drops inf/NaN.
  - `find_races_with_chart` / `prompt_user_to_choose` / `choose_race_for_chart` — scan `parameters.params`
    for races that define a given chart key and interactively prompt the user (via stdin) to pick one.
  - `get_dot_size(min_size, max_size, data)` — normalizes a data series into a marker-size range.
- Chart scripts (`scatter_plot.py`, `turnout_scatter_plot.py`, `turnout_bar_chart.py`,
  `turnout_heatmap.py`, `turnout_histogram.py`, `vote_share_histogram.py`) — each is standalone and
  run directly (`python turnout_bar_chart.py`); at import time each is only usable for its own
  chart key. Every script follows the same `__main__` pattern:
  1. `utils.choose_race_for_chart(parameters.params, chart_key="...")` to interactively pick a race.
  2. `utils.load_data_frame(...)` to load the race's data file.
  3. `utils.get_voter_stats(...)` to clean it.
  4. Call the script's own `create_*` function with the race's chart-specific config from `parameters.py`.
  All plotting uses matplotlib with `matplotlib.use("MacOSX")` hardcoded — this assumes running on
  macOS with a display; adjust the backend if running headless or on another OS.
  `turnout_bar_chart.py` and `turnout_heatmap.py` intentionally replicate the visual behavior of
  companion TypeScript/D3 charts on the live dashboard (see comments referencing D3 scale/contour
  behavior and component names like `DensityHeatmapByTurnout.tsx`) — when changing bucketing, binning,
  or color logic, keep that visual parity in mind.
- `requirements.txt` lists the Python dependencies (pandas, numpy, matplotlib, scipy).

### Adding a new chart for a race

1. Add/extend the race's entry in `parameters.py` with a new `<chart_key>: {...}` sub-dict (title,
   axis labels, chart-specific options).
2. If it's a new chart type, add a new script following the existing `create_*` + `__main__` pattern,
   using `utils.choose_race_for_chart(parameters.params, chart_key="<chart_key>")` to drive selection.
3. Run it directly with `python <chart_name>.py`; there is no test harness — verify visually via the
   matplotlib window that pops up.

## Data conventions

- Cleaned CSVs (`data/clean/**`) use per-candidate columns often split by vote method, e.g.
  `Donald J. Trump - Election Day`, plus `total_votes_cast` and `registered_voters` columns — the
  exact column names vary by source/contest and are declared explicitly in `parameters.py`, not
  inferred.
- `min_registered_voters` in a race config filters out precincts below that registration count before
  computing shares/turnout — useful for excluding tiny/aggregate precincts that skew scatter plots.
