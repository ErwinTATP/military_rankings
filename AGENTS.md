# AGENTS.md

## What this repo is

Wikipedia scraper → battle dataset → WAR (Wins Above Replacement) for military units and generals.
Currently being rewritten: Python 2 → Python 3, English-only → multilingual Wikipedia.

## State of the code

- **All code lives in two Jupyter notebooks** — no `.py` files exist.
  - `battle_scraper.ipynb` — original Python 2 notebook (has `print` statements without parens, `% matplotlib inline` syntax, `html_table_parser` import).
  - `battle_project.ipynb` — partially migrated Python 3 notebook; most cells still have Python 2 `print` syntax and won't execute as-is.
- **The notebooks are NOT runnable end-to-end today.** They were run cell-by-cell and intermediate results were saved to CSV/Excel files manually.
- `rows_1.csv` through `rows_18.csv` + `battles_dirty.csv` + `battles_deduped.csv` etc. are cached scrape outputs — do not delete them.

## Python 2 → 3 migration gotchas (what to fix)

- `print x` → `print(x)` throughout.
- `html_table_parser` (`parse.make2d(bell)`) is Python 2 only. Replacement: parse with `pd.read_html(str(bell))` or `BeautifulSoup` directly.
- `df.sort('col')` → `df.sort_values('col')`.
- `.iteritems()` on dict → `.items()`.
- `BeautifulSoup(content)` without parser arg raises warning; use `BeautifulSoup(content, 'lxml')`.
- `requests.get(url).content` returns bytes in Python 3; pass `.text` or decode before passing to BeautifulSoup.
- `global` variable mutation inside functions (e.g., `global df_battle_all`, `global count`) — anti-pattern, return values instead.

## Multilingual Wikipedia

- Wikipedia infobox structure varies by language — field names like `"Belligerents"`, `"Commanders and leaders"`, `"Strength"`, `"Casualties and losses"` will differ.
- Use `https://{lang}.wikipedia.org/wiki/{title}` pattern; the existing code already handles some non-English links via `ref[8:11]` prefix check (fragile, replace with URL parsing).
- The Wikipedia REST API (`https://{lang}.wikipedia.org/api/rest_v1/`) or `wikipedia-api` PyPI package are cleaner alternatives to raw `requests.get`.
- Battle list pages also differ by language — the English structure splits into table-based (pre-1600) and list-based (1601+) pages; other languages may use different structures entirely.

## Data model

Each row in the dataset represents one **belligerent-side per battle**:
- `pos`: `'L'` (left/attacker in Wikipedia infobox) or `'R'` (right/defender)
- `VorD`: `'V'` (victory) or `'D'` (defeat)
- `belligerent`: commander/unit name (resolved via Wikipedia page title)
- `own` / `opp`: troop strength strings (raw, unparsed — contain free text)
- `taken` / `inflicted`: casualty strings (raw)
- Strength breakdown columns: `Infantry`, `Cavalry`, `Artillery`, `Ships`, `Airforce`, `Special` — manually filled in Excel.

## Scraping conventions

- `sleep(1)` was used between requests but is commented out in places — re-enable to avoid rate-limiting.
- Wikipedia `infobox vevent` is the CSS class targeted for battle infoboxes.
- `omit_list` in `table_scrape()` filters non-person links (e.g., "Killed in action", "Prisoner of war") from commander extraction.
- Redirect detection uses `soup.find('ul', {'class':'redirectText'})` — this may no longer work with current Wikipedia HTML.

## No build system

No `requirements.txt`, `pyproject.toml`, `Makefile`, or tests exist. Environment: Python 3.14, `pandas` 3.x, `requests` 2.x, `beautifulsoup4` 4.x installed globally. `html_table_parser` is NOT installed (Python 2 artifact).

## WAR model

The WAR (Wins Above Replacement) score is computed per general by summing per-battle values derived from a logistic regression win-probability model.

**Features** — For each battle, four normalized force-ratio differences are fed to the model:

```
infantry_diff  = (Infantry_x  - Infantry_y)  / (Infantry_x  + Infantry_y)
cavalry_diff   = (Cavalry_x   - Cavalry_y)   / (Cavalry_x   + Cavalry_y)
ships_diff     = (Ships_x     - Ships_y)     / (Ships_x     + Ships_y)
airforce_diff  = (Airforce_x  - Airforce_y)  / (Airforce_x  + Airforce_y)
```

`_x` = attacker (pos='L'), `_y` = defender (pos='R'). Artillery and Special were dropped before fitting (unexplained in the notebook). Missing values are filled with 0.

**Model** — `sklearn.linear_model.LogisticRegression()` with all defaults (L2, C=1.0, liblinear solver). Trained on battles with a binary label (V=1, D=0); inconclusive battles excluded from training. Training accuracy ~54.4%. A RandomForest was also tried but achieved only ~50.4% cross-validated accuracy (no better than chance), so LR was kept.

**Coefficients** (infantry, cavalry, ships, airforce): `[0.414, 0.108, 0.569, 0.126]` — ships is the strongest predictor.

**Per-battle WAR value** — `lr.predict_proba(pred_diff)` returns `[P(loss), P(win)]`:

| Outcome | value |
|---|---|
| Win | `+P(loss)` — reward scales with how unexpected the win was |
| Loss | `-P(win)` — penalty scales with how expected a win was |
| Inconclusive | `0.5 - P(win)` |
| No troop data | `±0.5` flat fallback |

Replacement level is explicitly 0.51 for a win / 0.49 for a loss with equal forces. WAR is the sum of per-battle values; `War_perbat` is WAR divided by number of battles. No probability calibration is applied.

**Notebook differences** — `battle_scraper.ipynb` (original) drops only `special_diff` and requires `len(battles) > 4` to include a general. `battle_project.ipynb` (migrated) also drops `artillery_diff` and removes the minimum-battle filter.

## Visualization

Bokeh is used for the WAR scatter plots (output to `gh-pages/`). The `docs/` directory mirrors `gh-pages/` for GitHub Pages serving.

## Manual steps to automate

These are the parts of the current pipeline that required human intervention and are blockers for a fully automated re-run.

- **Troop strength parsing** — `Infantry`, `Cavalry`, `Artillery`, `Ships`, `Airforce`, `Special` were hand-entered from the raw `own`/`opp` free-text strings into Excel files (`strength_entry.xlsx`, `last_strength_entry.xlsx`, `all_strength_probably.xlsx`). These strings need a parser (regex + heuristics, or an LLM) to extract numeric values per unit type. This is the single largest manual bottleneck.

- **VorD labeling** — Wikipedia `Result` is free-text ("German victory", "Jordanian partial victory", "Inconclusive"). Many rows were patched directly by integer index in the notebook. Needs a classifier or keyword-match rules that map the raw `Result` string to `V`/`D`/`I` per side, keyed on which side is recorded as `pos='L'` vs `pos='R'`.

- **Non-battle URL filtering** — The list scrapers pull in stray links (e.g. "English East India Company", "Afghanistan", Wikipedia nav links). Currently filtered by ad-hoc string checks. Should be tightened to only accept links whose Wikipedia page is actually a battle infobox (`infobox vevent`), dropping the rest automatically.

- **Special character / encoding fixes** — `special_character_fix.xlsx` was created for rows where Unicode in battle names caused merge keys to fail (null `Date` after join). Needs explicit UTF-8 normalization and Unicode slug matching instead of exact string equality.

- **One-off index-level data patches** — Specific cells were corrected by hard-coded integer index (e.g. `df_strength_all.loc[9988, 'Infantry'] = 2000`, `df_model.loc[2378, 'Infantry_y'] = 25000`). These need to be replaced with named lookups (by `Battle` + `pos`) so they survive any re-ordering of the DataFrame.

- **Manual battle additions** — A handful of battles missing from the scraped lists were added as inline dicts (e.g. `arras_1`). Should become a small `manual_additions.csv` that is merged in at a defined pipeline stage rather than hardcoded in the notebook.

- **`omit_list` maintenance** — The list of non-person anchor titles filtered during commander extraction (`"Killed in action"`, `"Prisoner of war"`, etc.) is hardcoded. Should be driven by a config file or Wikidata property check (is the link target a human/Q5?).
