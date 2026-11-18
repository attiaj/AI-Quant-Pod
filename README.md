# AI Quant Pod

A systematic trading research desk run by Claude Code agents. Each agent holds one seat of a quant "pod" (scout, reader, ideator, economist, critic, data, developer, analyst, red team, portfolio manager), and the main Claude Code session acts as the head of the pod (the "desk chief"). The agents never talk to each other directly: they read and write notes in a shared Obsidian vault, and they call a small Python library (`lab/`) for data loading, backtesting, diagnostics, trial counting and PDF reports.

The method behind everything is the one of *The Systematic Trading Lab*: **write what you expect before you look at the result**. The agents automate the work; they do not relax the method. Every strategy gets a written design (Pass 1) before any code, every backtest run is saved and counted, and the test period of each strategy stays sealed until a human approves opening it, once.

---

## Contents

1. [Where this repository comes from](#where-this-repository-comes-from)
2. [The pod: agents, skills, hooks and settings](#the-pod-agents-skills-hooks-and-settings)
3. [Repository map](#repository-map)
4. [Capabilities](#capabilities)
5. [How it works, step by step](#how-it-works-step-by-step)
6. [Recommended workflow for a team member](#recommended-workflow-for-a-team-member)
7. [Disclaimer](#disclaimer)
8. [Original README (Quantreo)](#original-readme-quantreo)

---

## Where this repository comes from

This is Jacob Attia's copy ([attiaj/ai-quant-pod](https://github.com/attiaj/ai-quant-pod)) of the course repository **ai-quant-pod** by Lucas Inglese ([Quantreo/ai-quant-pod](https://github.com/Quantreo/ai-quant-pod)). The course builds the pod seat by seat, and the commit history follows its sections:

| Commit | Course section | What it added |
| --- | --- | --- |
| `017ba20` | Section 0 | Project setup and the data download scripts |
| `59b6eda` | Section 2 | Library team: scout, reader, technique cards |
| `06b4e36` | (starter library) | 65 reading notes and 101 technique cards |
| `fefbdfd` | Section 3 | Quant researchers: ideator, economist, critic |
| `23c4616` | Section 4 | Data engineer and quant developer: the lab, the data check, the train gate |
| `851ca97` | Section 5 | Risk team: analyst, red team, sealed test, strategy report |
| `8cc1985` | Section 6 | Portfolio manager: book construction, the desk book on paper |
| `d7458b0` | Section 7 | Head of the pod: the desk chief skill, first ideas from the book's needs |
| `8f6da4e` | Section 8 | Case studies: funding carry, thin losers, SPY vol managed, the graveyard |
| `6b8bbdf` | (this fork) | Hook command in `.claude/settings.json` changed to a relative path (`python3 .claude/hooks/test_guard.py`) |

Remotes in this copy:

| Remote | URL | Role |
| --- | --- | --- |
| `origin` | https://github.com/attiaj/ai-quant-pod | The team's copy. Push here. |
| `upstream` | https://github.com/Quantreo/ai-quant-pod | Lucas Inglese's course repository. Pull new sections from here. |

To bring in new course material:

```bash
git checkout main
git pull upstream main     # fetch and merge Quantreo's latest commits
git push                   # publish them to origin (attiaj/ai-quant-pod)
```

If you have never added the upstream remote on your machine: `git remote add upstream https://github.com/Quantreo/ai-quant-pod`.

---

## The pod: agents, skills, hooks and settings

### The seats (agents)

Agents live in [`.claude/agents/`](.claude/agents). Each file sets the seat's job, its allowed tools (all of them have `Bash, Read, Write, Edit, Glob, Grep`), its model, and which skill it must follow. Each seat may write only in specific places, and its final reply is kept to a few lines so the desk chief can route cheaply.

| Seat (file) | Model | Responsibility | Skill it follows | What it produces |
| --- | --- | --- | --- | --- |
| Scout ([`scout.md`](.claude/agents/scout.md)) | haiku | Searches arXiv and OpenAlex for strategy papers, triages them, downloads the open-access PDFs picked | `paper-scout` | PDFs in `vault/0 Library/Sources/`, lines in `vault/0 Library/Inbox/Scout log.md` |
| Reader (no agent file) | sonnet, as a general-purpose agent | Reads one paper and writes a faithful reading note with page references | `reading-source` | `vault/0 Library/Notes/<Year> - <Authors> - <Title>.md` |
| Ideator ([`ideator.md`](.claude/agents/ideator.md)) | sonnet | Combines the library into up to 5 argued ideas | `ideation` | Entries marked **new** in `vault/1 Ideas/Idea Journal.md` |
| Economist ([`economist.md`](.claude/agents/economist.md)) | opus | Judges ideas (Go, Go with a check, Park, Kill), designs the strategy for a Go, answers objection rounds | `pass-one`, `revise-strategy` | Strategy folder: index `<Name>.md`, `Pass 1.md`, `Changelog.md`; answers in `Review.md` |
| Critic ([`critic.md`](.claude/agents/critic.md)) | sonnet | Attacks a Pass 1 after a passed train gate: checks figures against the library, builds the strongest rival explanation, ranks objections. Never sees results | `pass-one-review` | `Review.md` |
| Data ([`data.md`](.claude/agents/data.md)) | sonnet | Checks that the strategy can be tested honestly with the desk's data, before code. Never changes data, never computes a return | `data-check` | `Data check.md` (ready, ready with reserves, blocked) |
| Developer ([`developer.md`](.claude/agents/developer.md)) | sonnet | Codes the strategy exactly as Pass 1 says, runs the look-ahead test, the train gate, then the validation runs. Saves and logs every run | `build-strategy` | `strategies/<slug>.py`, runs in `results/`, `Build.md`, Research Log lines |
| Analyst ([`analyst.md`](.claude/agents/analyst.md)) | sonnet | Reads saved runs against the rules Pass 1 wrote before any result; after the test, applies the three-zone rule. Never runs a backtest | `analyze-backtest` | `Pass 2.md` (kill, candidate for the test, inconclusive), `Test.md` (passed or killed), Trials rows, backtest report page |
| Red team ([`red-team.md`](.claude/agents/red-team.md)) | sonnet | Tries to break the backtest before the test: code audit, wider truncation test, permutation placebo, cost stress, raw-data check | `red-team` | `Red team.md` (open the test, fix first, kill) |
| Portfolio manager ([`pm.md`](.claude/agents/pm.md)) | sonnet | Decides what a tested strategy adds to the book, at which weight; or which strategies make a book. Never edits the book | `portfolio-manager` | Candidate review or book construction PDF in `vault/4 Book/Reviews/` |
| Desk chief (the main session) | your session model | Runs the whole pipeline, calls each seat as its own agent, keeps the logs, asks you for one approval | `desk` | Desk log lines, Lessons, `Desk economics.csv`, the final desk report |

The Reader is not in `.claude/agents/`: [`desk/reference/pipeline.md`](.claude/skills/desk/reference/pipeline.md) says it runs as "a general-purpose agent with `model: sonnet`, told to follow `.claude/skills/reading-source/SKILL.md`", only when the user asks. The pipeline also mentions a later "Monitor" seat that does not exist yet.

### The skills

Skills live in [`.claude/skills/`](.claude/skills). Each has a `SKILL.md` (the procedure), and usually `reference/` (rules, examples), `assets/` (note templates) and `scripts/` (helpers).

| Skill | Used by | What it does | Helper scripts |
| --- | --- | --- | --- |
| [`desk`](.claude/skills/desk/SKILL.md) | Main session (head of the pod) | Runs the desk end to end from a theme or an idea, through every seat, to one batch of sealed tests and the PM; writes the final report | none (reads `reference/pipeline.md`) |
| [`paper-scout`](.claude/skills/paper-scout/SKILL.md) | Scout | Queries arXiv (q-fin) and OpenAlex, triages by title and abstract, downloads picks | `search.py`, `fetch.py` (standard library) |
| [`reading-source`](.claude/skills/reading-source/SKILL.md) | Reader | One paper, one faithful note: every fact with a page, no opinions, chart values marked with ≈ | `pdf_text.py` (PyMuPDF: text per page, figure pages as PNG) |
| [`technique-card`](.claude/skills/technique-card/SKILL.md) | Main session, after a reading note | Turns techniques in a note into reusable cards filed by brick, merged when a card already exists | none |
| [`ideation`](.claude/skills/ideation/SKILL.md) | Ideator | Builds about 10 candidates with six operators (transpose, swap, condition, test a claim, fix a weakness, combine), keeps 5 | `library_index.py` (compact index of notes, cards, journal, strategies) |
| [`pass-one`](.claude/skills/pass-one/SKILL.md) | Economist | Judges the mechanism (who pays, why it lasts, predictions), writes a short Pass 1 (500 words) for a Go, completes it (1500 words) after a passed gate | `budget.py` (luck line, power, cost budget) |
| [`pass-one-review`](.claude/skills/pass-one-review/SKILL.md) | Critic | One review round: fidelity of cited figures, rival explanation, design checks, ranked objections, spec gaps | reuses `pass-one/scripts/budget.py` |
| [`revise-strategy`](.claude/skills/revise-strategy/SKILL.md) | Economist | Decides each objection (fixed, partly fixed, refused), edits Pass 1 minimally, logs a new version | reuses `budget.py` |
| [`data-check`](.claude/skills/data-check/SKILL.md) | Data | Lists needs, checks presence and quality, builds the point-in-time universe, looks at the data by hand | `check_panel.py`, `universe_report.py` |
| [`build-strategy`](.claude/skills/build-strategy/SKILL.md) | Developer | Spec table, strategy file from the template, truncation test, phase gate (one run), phase validation | `assets/strategy-template.py`, `reference/conventions.md` |
| [`analyze-backtest`](.claude/skills/analyze-backtest/SKILL.md) | Analyst | Criterion by criterion verdict, trial counting, luck line, controls, report page; post-test three-zone verdict | `run_report.py`, `regress.py` (Newey-West factor regression) |
| [`red-team`](.claude/skills/red-team/SKILL.md) | Red team | Look-ahead audit, wider truncation test, placebo, 2x cost stress, raw data of the top days and names | `placebo.py` |
| [`strategy-report`](.claude/skills/strategy-report/SKILL.md) | Main session | LaTeX PDF research note on one strategy: numbers by script, reading by the agent | `lab/strategy_report.py` |
| [`portfolio-manager`](.claude/skills/portfolio-manager/SKILL.md) | PM | Candidate review or book construction PDF | `lab/book_report.py` |

### How the rules are enforced

The four non-negotiables in [`CLAUDE.md`](CLAUDE.md) are:

1. Pass 1 of a strategy is written before any code is written for it.
2. Every backtest run is saved and logged, especially the bad ones. Every try counts.
3. The test period is opened once, only through `lab/run_test.py`, after the user approves it.
4. Entries in the book at half weight, exits on the written rule only. The book is the user's decision.

They are enforced in layers. Only the sealed test has a hard technical guard; the train gate and the trial count are enforced by procedure and by the library.

| Mechanism | Where | What it does |
| --- | --- | --- |
| Data cut | [`lab/data.py`](lab/data.py) | `load(name, test_start)` drops every row at or after the test start unless `open_test=True` is passed. |
| Strategy template | [`strategy-template.py`](.claude/skills/build-strategy/assets/strategy-template.py) | `run(period="test")` raises unless `open_test=True`, which only `lab/run_test.py` passes. |
| Test lock | [`lab/run_test.py`](lab/run_test.py) | Runs each strategy's own `run("test", variant, open_test=True)`, then writes `results/<strategy>/TEST_DONE` with the run, variant, code hash and lab hash. A second opening is refused. |
| Permission "ask" | [`.claude/settings.json`](.claude/settings.json) | `poetry run python lab/run_test.py` and `python3 lab/run_test.py` always prompt you. That click is the human gate. |
| Permission "deny" | `.claude/settings.json` | Agents may never run `rm -rf` or `git push`. |
| Permission "allow" | `.claude/settings.json` | Read, Edit, Write, Glob, Grep, WebSearch, WebFetch, and Bash for `poetry run python`, `python3`, `ls`, `cat`, `head`, `tail`, `wc`, `grep`, `find`, `mkdir`, `mv`, `cp`. |
| PreToolUse hook | [`.claude/hooks/test_guard.py`](.claude/hooks/test_guard.py) | Runs before every Bash, Write, Edit, MultiEdit and NotebookEdit call of every agent. Blocks any command or file containing `open_test=True` (except inside `lab/strategy_report.py` and `lab/book_report.py`, which read the test only once the lock exists), any touch of a `TEST_DONE` lock, any call to `run_test` other than the single allowed form, and any edit of `lab/run_test.py`, the hook itself or `.claude/settings.json`. Its own docstring says it "stops mistakes and shortcuts, not a determined attacker"; the real protection is the lock, the code hash and git. |
| Train gate | `pass-one`, `build-strategy`, `desk` skills | The Developer makes exactly one train run of the base version at the central cost. The gate is the level: net Sharpe above the luck line plus cost, with a t-stat bar of 1 if the train is shorter than 5 years and 2 otherwise ([Kill Policy](vault/Toolbox/Kill%20Policy.md)). Most ideas are meant to die here, cheaply. |
| Trial count | [`lab/trials.py`](lab/trials.py) | Every run read by the Analyst is appended to `Trials.csv` as a try (`--counts yes`) or not (stress tests, controls). The luck line rises with the effective number of tries. |
| Lab hashes | [`lab/runs.py`](lab/runs.py) | Each run records `code_sha` (strategy file) and `lab_sha` (`data.py`, `universe.py`, `backtest.py`). Runs compared together must share the same `lab_sha`. |
| Defaults | [`vault/Toolbox/Defaults.md`](vault/Toolbox/Defaults.md) | The desk never waits for you: questions with a default take it and are logged as assumptions; questions without one ("real decisions": new data source, test window change, `lab/` change, money, a rule of the method) are listed for you. |

---

## Repository map

```
ai-quant-pod/
├── CLAUDE.md                 House rules every agent reads first
├── README.md                 This file
├── pyproject.toml            Python >= 3.12, dependencies (Poetry, package-mode = false)
├── poetry.lock               Locked versions (generated by Poetry 2.2.1)
├── .gitignore                Keeps data, results and paper PDFs out of git
├── .claude/
│   ├── settings.json         Permissions (allow / ask / deny) and the PreToolUse hook
│   ├── hooks/test_guard.py   Guard of the sealed test
│   ├── agents/               9 seat definitions (scout, ideator, economist, critic, data, developer, analyst, red-team, pm)
│   └── skills/               14 skills (procedures, templates, reference rules, helper scripts)
├── lab/                      The desk's Python library (agents call it, never modify it)
├── data/                     Download scripts; the downloaded data lands here but is never committed
├── strategies/               One Python file per strategy, written by the Developer
├── case-studies/             The author's own strategies from Section 8, to read, not to run
├── vault/                    The Obsidian vault: the pod's shared memory
└── results/                  (created locally, git-ignored) every saved run, report work folders, PDF text extracts
```

### `lab/` (the library)

| File | What it does |
| --- | --- |
| [`data.py`](lab/data.py) | `load(name, test_start, open_test=False)`: loads a dataset strictly before the test start. Datasets: `crypto_daily`, `crypto_funding`, `crypto_mark_hourly`, `crypto_1h`, `crypto_1m`, `fx_daily`, `rates`, `etf_daily`. |
| [`universe.py`](lab/universe.py) | `crypto_universe(daily, top, volume_window, min_age_days, min_names, underlying)`: point-in-time universe ranked by average quote volume, with a minimum age, coins only by default (Binance also lists stock, metal and index perps). |
| [`backtest.py`](lab/backtest.py) | The engine. Target weights in, P&L per bar out, at daily, hourly or minute bars. A weight dated d trades at the close of d. Mark price with last-price fallback, cost per unit traded, funding paid per print, no-trade band, isolated-margin liquidation on bar highs and lows, delisting exits. Returns P&L by source and leg, positions, trades, liquidations, P&L by name. |
| [`checks.py`](lab/checks.py) | `truncation_test` (recomputes weights on data cut at sampled trade days; any difference is a look-ahead) and `delay_test` (same weights traded a day later, a control). |
| [`runs.py`](lab/runs.py) | `save_run`: writes `results/<strategy>/<run id>/` with `daily.parquet`, `positions.parquet`, `trades.parquet`, `liquidations.csv`, `by_name.csv`, `summary.json` (parameters, engine notes, code and lab hashes). |
| [`stats.py`](lab/stats.py) | Standard summary: gross and net Sharpe, net t-stat, volatility, max drawdown, P&L by source, funding share, turnover, break-even cost, liquidations. |
| [`trials.py`](lab/trials.py) | `add` a run to `Trials.csv`; `report` writes `Trials summary.md`: tries by strategy, family and asset, effective N, luck line, best Sharpe against it, and the budget of tries (10 per family and asset). |
| [`attribution.py`](lab/attribution.py) | P&L by name and concentration (share of the best 1% of days, best 10 days, top 5 names, Sharpe without the best days). |
| [`report.py`](lab/report.py) | Markdown backtest report next to the strategy note, with PNG charts: cumulative return in % (linear), drawdown, P&L by source, by year, by leg, runs compared. |
| [`strategy_report.py`](lab/strategy_report.py) + [`strategy_report.tex`](lab/strategy_report.tex) | The strategy PDF: `compute` writes `numbers.md`/`numbers.json` and figures, the agent writes `text.md`, `build` compiles LaTeX into `<Strategy> - Report.pdf`. |
| [`book_report.py`](lab/book_report.py) | The PM's PDFs: `review` (candidate against the current book) and `construct` (a book from a set of strategies). |
| [`portfolio.py`](lab/portfolio.py) | Older command-line book check (one candidate against a set of strategies, block bootstrap, halve and cut levels). `book_report.py` reuses its run reader and sizing; `Book.md` cites its halve/cut rule. |
| [`fx.py`](lab/fx.py) | `fx_bars` (nine currencies in dollars per unit, for the engine) and `fx_carry` (daily carry prints from the monthly rate differential, lagged one month, stale after three months). |
| [`sheets.py`](lab/sheets.py) | Finds a strategy's folder in the vault whatever its status, and concatenates its section files. |
| [`selftest.py`](lab/selftest.py) | Checks the engine on cases with known answers. Run after installing and after any change to `lab/`. |
| [`run_test.py`](lab/run_test.py) | The only door to the test period (see above). |
| [`desk.mplstyle`](lab/desk.mplstyle) | Figure style for the PDF reports (vendored SciencePlots-like settings and the house palette). |
| [`README.md`](lab/README.md) | One paragraph: "Agents call it, they never modify it." |

### `data/` (download scripts)

| Script | Source | Output (`data/<folder>/`, never committed) |
| --- | --- | --- |
| [`get_crypto_daily.py`](data/get_crypto_daily.py) | Binance Vision public archive (data.binance.vision), CC BY-NC-SA 4.0 | `crypto/perps_daily.parquet`, `perps_funding.parquet`, `perps_meta.csv`, `quality_report.md`; raw monthly zips cached in `crypto/raw/` |
| [`get_etf_daily.py`](data/get_etf_daily.py) | Yahoo Finance public chart endpoint | `etf/etf_daily.parquet`, `etf_liquidity.csv`, `quality_report.md` |
| [`get_fx_daily.py`](data/get_fx_daily.py) | FRED, Federal Reserve H.10 (noon New York) | `fx/fx_daily.parquet`, `quality_report.md` |
| [`get_rates.py`](data/get_rates.py) | FRED (OECD and Federal Reserve series) | `rates/rates_monthly.parquet`, `cpi.parquet`, `policy.parquet`, `tbill.parquet`, `ust_curve.parquet`, and a quality report for each |
| [`README.md`](data/README.md) | | The four download commands |

Each dataset has a data card in [`vault/Toolbox/Data/`](vault/Toolbox/Data): source, licence, period, known gaps. The Data seat reads it before every check.

### `strategies/`

One file per strategy, written by the Developer from Pass 1 on top of the lab. The file decides target weights; the lab loads data, builds the universe, computes P&L and saves runs. The file is frozen once its test is run. Current files:

| File | Strategy | Data | Train / validation / test start | Status in the vault |
| --- | --- | --- | --- | --- |
| [`crypto_thin_losers.py`](strategies/crypto_thin_losers.py) | Among 14-day losers on Binance perps, long thin names, short liquid names, BTC beta hedge | `crypto_daily`, `crypto_funding` | 2021-2023 / 2024 / 2025-01-01 | Passed (ambiguous zone, paper only) |
| [`etf_tsmom_12m.py`](strategies/etf_tsmom_12m.py) | 12-month time-series momentum on 22 ETFs, risk-balanced | `etf_daily`, T-bill | 2006-2017 / 2018-2020 / 2021-01-01 | Passed (zone confirms, full-sample t 2.35) |
| [`spy_volatility_managed.py`](strategies/spy_volatility_managed.py) | SPY weight scaled by last month's realized variance, rest in T-bills | `etf_daily`, T-bill | 2005-02 to 2017 / 2018-2020 / 2021-01-01 | Passed (zone confirms, paper only) |
| [`bond_trend_demeaned.py`](strategies/bond_trend_demeaned.py) | Trend on six government bond ETFs, net of an expanding drift | `etf_daily`, T-bill | 2008-2018 / 2019-2021 / 2022-01-01 | Killed at the train gate |
| [`crypto_low_max_long_leg.py`](strategies/crypto_low_max_long_leg.py) | Long the 6 perps with the lowest 7-day maximum daily return, beta-hedged | `crypto_daily`, `crypto_funding` | 2021-2023 / 2024 / 2025-01-01 | Killed at the train gate |

### `case-studies/`

The author's own strategies shown in Section 8, kept **outside** `vault/` on purpose: the desk never reads this folder, so these strategies do not count as tries in your `Trials.csv`, do not spend your budget of tries, and do not stop your desk from trying the same ideas ([`case-studies/README.md`](case-studies/README.md)).

- [`Crypto Funding Carry/`](case-studies/Crypto%20Funding%20Carry): a complete strategy folder (Pass 1 and a v4 of it, Review, Data check, Build, Pass 2, Red team, Test, Changelog, backtest report, charts, report PDF). Its index records it as killed and parked by the user because the P&L came in bursts and was a fast price effect, not the weekly carry Pass 1 described.
- [`Slides/`](case-studies/Slides): one-page PDFs for videos 8.1 (the path), 8.2 (thin losers), 8.3 (SPY vol managed), 8.4 (cost per idea, and the graveyard).
- Crypto Thin Losers (8.2) and SPY Volatility Managed (8.3) live in `vault/2 Strategies/Passed/`.

### `vault/` (the Obsidian vault)

Open it in Obsidian with *File → Open folder as vault* and choose the `vault` folder. Start with [`vault/README.md`](vault/README.md).

| Folder | What it holds | Written by |
| --- | --- | --- |
| [`0 Library/Sources/`](vault/0%20Library/Sources) | Paper PDFs (git-ignored, copyrighted; each user downloads them) | Scout, Reader |
| [`0 Library/Notes/`](vault/0%20Library/Notes) | 65 reading notes, one per paper, with frontmatter (authors, year, tier, tags `[family, asset, horizon]`) and page references | Reader |
| [`0 Library/Techniques/`](vault/0%20Library/Techniques) | 101 technique cards by brick: Diagnostics 44, Signal 36, Sizing 7, Risk 6, Rebalancing 4, Exit 2, Universe 2, Entry 0 | `technique-card` skill |
| [`0 Library/Inbox/`](vault/0%20Library/Inbox) | `Inbox.md` (papers waiting to be read, with `[x]`, `[>]`, `[-]`, `[?]` statuses) and `Scout log.md` (every candidate the scout triaged) | Economist, Scout, you |
| [`1 Ideas/Idea Journal.md`](vault/1%20Ideas/Idea%20Journal.md) | One entry per idea, newest first: mechanism sentence, operator, what's new, bricks, evidence, first kill, data, status (new, reading, derived, sheet, dropped, parked) | Ideator, Economist (status) |
| [`2 Strategies/`](vault/2%20Strategies) | One folder per strategy; the folder is the status: `Pipeline/` (in progress), `Passed/` (passed the sealed test), `Killed/` (with the reason and a `revisit:` line). Register: `_Index.md` | All seats |
| [`3 Research Log/`](vault/3%20Research%20Log) | `Lessons.md` (what the desk learned, by Markets, Design, Data, Method), `Desk reports/` (final reports of desk runs, plus the 2026-10-25 gate power diagnostic), `_desk/` (agent logs: `Desk log.md`, `Research Log.md`, `Trials.csv`, `Trials summary.md`, `Desk economics.csv`) | Desk chief, Developer, Analyst, Red team |
| [`4 Book/`](vault/4%20Book) | `Book.md` (the book you actually run, weights in risk; only you edit it) and `Reviews/` (PM PDFs) | You; PM writes only the PDFs |
| [`Toolbox/`](vault/Toolbox) | The desk's reference: `Defaults.md`, `Kill Policy.md`, `Cost Assumptions.md`, `Data Sources.md`, `Data/` (8 data cards), `Tags.md`, `Strategy Families.md`, `Source Registry.md`, `Instrument Traps.md`, `Deflated Sharpe.md`, `Reading the Diagnostic.md` | Humans (agents read) |

A strategy folder, as the agents write it:

| File | Written by | Content |
| --- | --- | --- |
| `<Name>.md` | Economist, then every seat adds a row | Index: frontmatter (`family`, `asset`, `horizon`, `status`, `version`, `stage`), one-line mechanism, one row per seat `| seat | date | verdict | [[file]] |` |
| `Pass 1.md` | Economist | 1.1 Mechanism and predictions, 1.2 Sources and crowding, 1.3 Expected behaviour, 1.4 Universe, 1.5 Base version, 1.5b Free parameters, 1.6 Power, 1.7 Cost and capacity budget, 1.8 Instrument traps, 1.9 Variants, 1.10 Test plan (periods), 1.11 Kill criterion |
| `Changelog.md` | Economist | One row per version |
| `Data check.md` | Data | Needs, presence, checks, issues with instructions for the Developer, verdict |
| `Build.md` | Developer | Code hash, spec table, assumptions, runs table, turnover, gate result |
| `Review.md` | Critic, Economist, Developer | Objections and answers; Developer rounds for spec gaps |
| `Pass 2.md` | Analyst | Criterion by criterion verdict before the test |
| `Red team.md` | Red team | Findings by severity and recommendation |
| `Test.md` | Analyst | Three-zone verdict after the test |
| `<Name> - Backtest report.md`, `- Report.pdf`, PNGs | `lab/report.py`, `lab/strategy_report.py` | Human-readable reports |

---

## Capabilities

### Data, by asset class

| Asset class | Script | Coverage | Notes |
| --- | --- | --- | --- |
| Crypto perpetuals | `get_crypto_daily.py` | Every Binance USDT-margined perp ever listed, dead ones included, 2020-01-01 to 2026-09-30 | OHLC last price, mark price OHLC, quote volume, daily funding sum, every funding print. Zero-volume days dropped, missing days filled from daily files, a ticker silent for more than 30 days becomes a new symbol (`LITUSDT.2`). `--symbols BTCUSDT,ETHUSDT` for a quick test. The data card says 15 to 45 minutes the first time, cached after. |
| ETFs | `get_etf_daily.py` | About 220 US-listed ETFs probed (a little over 200 kept), 2005-01 to 2026-09 | Equity indexes, sectors, size and style, factors, countries, bonds of every maturity, commodities, real estate, currencies. `adj_close` for returns. Survivors only (Yahoo does not serve dead ETFs). About 2 minutes. |
| FX | `get_fx_daily.py` | Nine majors against USD, 2008-01 to 2026-09 | One noon New York rate a day (FRED H.10), no bid, ask, high or low. `lab/fx.py` turns it into engine bars and carry prints. |
| Rates and macro | `get_rates.py` | G10 3-month interbank rates (from 2009), G10 CPI (from 2007), ECB and Fed policy rates (from 2009), 3-month T-bill (from 2004), US Treasury curve 1Y to 30Y (from 2004) | Monthly series from FRED. Stale series are flagged, never filled. |

Intraday crypto datasets (`crypto_1h`, `crypto_1m`, `crypto_mark_hourly`) are declared in `lab/data.py` and supported by the engine, and the build conventions mention `data/get_crypto_intraday.py`, but that script is not in this repository. Anything beyond the four scripts above (a new source, a paid source) is a "real decision" for a human, except that the desk may add a new FRED series to `get_rates.py` in its style, with a data card.

### Research and library

- **Paper search**: arXiv (q-fin) and OpenAlex only, never Google Scholar or SSRN scraping; triage with keep, maybe, reject; open-access PDFs downloaded with a standard file name; paywalled papers listed with a landing link for manual download.
- **Reading notes**: faithful, page-referenced summaries (finding, mechanism, where it works, weak spots, caveats, techniques table). No opinions, no outside knowledge.
- **Technique cards**: one card per reusable technique, filed by brick (signal, sizing, risk, exit, rebalancing, universe, entry, diagnostic), merged across sources, with `aliases` and `tested_on` tags.
- **Ideation**: ideas built from the library with six operators, each with a labeled mechanism sentence (`[source: Author Year, p. X]` or `[hypothesis]`), a first kill test, and a data check against what the desk can download.

### Strategy design and review

- **Pass 1**: verdict (Go, Go with a check, Park, Kill), a named payer and predictions that could fail, a design where every choice has a basis, at most 3 pre-registered variants (one brick each), a cost budget, periods, and at most three kill criteria.
- **`budget.py`**: the luck line (expected best Sharpe of N skill-less tries on T years, Bailey and López de Prado), the power of the gate, and the cost budget from turnover.
- **Critic review**: one round per strategy, objections ranked blocking, major, minor, each checkable, plus spec gaps.

### Backtesting

- **Engine** (`lab/backtest.py`): books of perpetual futures at any bar size; also used for ETFs and FX by passing bars and carry prints. Mark-price P&L, cost per unit traded each way, funding by print (a long pays positive funding), no-trade band, isolated-margin liquidation with maintenance margin and forced-trade cost, delisting exits, and P&L split into price, funding, cost and liquidation, by leg and by name.
- **Look-ahead checks**: truncation test (mandatory before the gate) and one-day delay control.
- **Point-in-time universe** for crypto; ETFs enter on their first trading day; ETF cost tiers by prior-year dollar volume ([Cost Assumptions](vault/Toolbox/Cost%20Assumptions.md): T1 2 bp to T4 15 bp, T5 excluded).
- **Every run saved** with its code and lab hashes, and logged in the Research Log.

### Checks and red teaming

- **Data checks** (`check_panel.py`, `universe_report.py`): coverage, generic quality checks before the test start, the point-in-time universe with Pass 1 parameters, and market-specific traps documented in [`data-check/reference/checks.md`](.claude/skills/data-check/reference/checks.md) (zombie bars, relisted tickers, non-coin perps, funding intervals and gaps, multipliers, FX quote direction and holidays, and more).
- **Red team**: line-by-line code audit, a wider truncation test with another seed, a permutation placebo (`placebo.py`, shuffling weights or signs among the same names), a 2x cost stress, and a look at the raw data of the 5 names and 10 days that carry the most P&L.

### Statistics and multiple testing

- **Trial count** (`lab/trials.py`): every try in one file, with family, asset, horizon, universe, period, variant and Sharpes. The report computes N, the mean correlation of the tries' gross daily returns, N_eff = N / (1 + (N - 1)ρ), the luck line for N_eff, and the best gross Sharpe above it, for the desk, each family, and each family and asset.
- **Budget of tries**: 10 per family and asset. Past it, the desk parks ideas on that data with "budget spent: new data needed".
- **Strategy report statistics**: stationary block bootstrap (20-day blocks, 2000 paths) ranges for return, volatility, Sharpe, drawdown; Newey-West t-stats; benchmark beta and alpha (BTC for crypto, SPY otherwise, none for FX); deflated Sharpe probability against the luck line.
- **Three-zone test rule**: confirms (test mean above half the earlier mean), contradicts (below minus half, with test t under -1.5), ambiguous otherwise. Money also needs the full-sample net t-stat at 2 or more ([Kill Policy](vault/Toolbox/Kill%20Policy.md)).

### Portfolio and book construction

- **Sizing**: each strategy scaled to 15% volatility with trailing 60-day volatility known the day before; equal risk weights by family; the book scaled to a 10% target through the diversification multiplier (IDM), capped at 2.5.
- **Candidate review** (`book_report.py review`): the marginal rule Sharpe(candidate after luck haircut) > correlation × Sharpe(book), correlation on the book's worst days and months, a weight curve, alternatives (drop the weakest, swap it for the candidate), and a block bootstrap share of histories where the book improves, on the whole window and after the candidate's train. 90% or more enters at half weight (only with full-sample t at 2 or more and never for imported returns), 60 to 90% is paper, below 60% stays out.
- **Book construction** (`book_report.py construct`): by default every passed strategy with an opened test; a strategy earns its place if dropping it lowers the book's Sharpe in 60% of bootstrap histories or more; money allowed only with status passed, full-sample t of 2 or more and a positive Sharpe after the luck haircut.
- **Both reports** read rolling correlations, risk and return shares, factor regressions on SPY, TLT, UUP, GLD and BTC, suspect days, and stress episodes from the 2008 crisis to the 2025 tariff shock.
- External returns can be imported (`--returns`, `--import`) for comparison, capped at paper.

### Attribution and reports

- **Backtest report page** (`lab/report.py`): Markdown in the strategy folder with charts (cumulative return in %, linear scale, never "growth of $1", never log scale).
- **Strategy report PDF** (`lab/strategy_report.py`): sections abstract, hypothesis, method, performance, exposure, sources, concentration, implementation, robustness, luck, risks, decision. The script computes every number; the agent writes the reading in `text.md`, and the abstract carries one bold sentence with the worst news.
- **PM PDFs**: written to `vault/4 Book/Reviews/`.

### Current state of the vault (as committed)

- Strategies: 3 passed (Crypto Thin Losers, ETF TSMOM 12-Month, SPY Volatility Managed), 2 killed (Bond Trend Demeaned, Crypto Low-MAX Long Leg), `Pipeline/` empty. `_Index.md` currently lists only three of the five (the two Section 8 strategies are missing from it).
- [`Book.md`](vault/4%20Book/Book.md): Crypto Thin Losers and SPY Volatility Managed, both on paper (0% money), target 50% risk each; ETF TSMOM 12-Month left out. Source: `vault/4 Book/Reviews/Desk book - Book construction.pdf`.
- [`Trials summary.md`](vault/3%20Research%20Log/_desk/Trials%20summary.md): 44 rows, 9 counted tries over 5 strategies.
- Idea Journal: four **new** ideas from the 2026-11-10 run on the book's needs (G10 carry with a volatility switch, FX value, Treasury curve carry, commodity ETF momentum).
- `Lessons.md` and `Desk economics.csv` have their headers only.

---

## How it works, step by step

The pipeline, from [`CLAUDE.md`](CLAUDE.md):

```
Library (run by hand):  Scout → Reader → Technique cards

The desk (/desk):       Ideator → Economist → Data → Developer (train gate)
                        → Critic → Economist → Developer (validation)
                        → Analyst + Red team → [you approve the tests]
                        → sealed test → Analyst → report → Portfolio manager
```

You step in twice: the theme at the start, and one approval of the batch of sealed tests near the end. Everything else runs on [`Defaults.md`](vault/Toolbox/Defaults.md). The lane puts the cheap test first: no review, validation or analysis is spent on an idea before it passes its train gate. The desk skill targets 60k to 80k tokens for an idea that dies at the gate, so a run fits a Pro plan.

| # | Stage | Who | What happens | Where the vault records it |
| --- | --- | --- | --- | --- |
| 0 | Library (optional, by hand) | Scout, Reader, `technique-card` | Find papers, read them into notes, extract technique cards | `0 Library/Sources/`, `Notes/`, `Techniques/`, `Inbox/Scout log.md` |
| 1 | Ideas | Ideator | From a theme (or, with no theme, the PM's last "What the book needs"), writes up to 5 ideas. Stops and asks for papers if the library has no note on the theme | `1 Ideas/Idea Journal.md` (status **new**) |
| 2 | Budget check | Desk chief | An idea whose family and asset have no try left is parked | `Trials summary.md`, Desk log |
| 3 | Judge | Economist (one call for the batch) | Verdict per idea; for a Go, a short Pass 1 (500 words) with the train gate | New folder `2 Strategies/Pipeline/<Name>/` with `<Name>.md`, `Pass 1.md`, `Changelog.md`; journal status **sheet**; Park or Kill sets **parked** or **dropped** |
| 4 | Data | Data | Needs, presence, quality checks before the test start, point-in-time universe, hand inspection | `Data check.md` |
| 5 | Strategy code and train gate | Developer (one call for the batch) | Writes `strategies/<slug>.py` from the template, passes the truncation test, makes exactly one train run of the base at central cost, computes the gate. Spec gaps become written **Assumptions** | `Build.md`, `results/<slug>/<run>/`, Research Log line |
| 5b | Gate failed | Desk chief | Run added to Trials as a try, folder moved to `Killed/`, journal and `_Index.md` updated, lesson written. No more tokens on it | `Killed/<Name>/`, `Trials.csv`, `Lessons.md` |
| 6 | Complete Pass 1 | Economist (resumed) | Fills the after-the-gate sections (1500 words), records the train result as data already seen, never changes the base because of it | `Pass 1.md` (`stage: full`), Changelog |
| 7 | Review | Critic (fresh, never hears the gate result) | One round of ranked objections | `Review.md` |
| 8 | Revise | Economist (resumed) | Fixed, partly fixed or refused, minimal edits. A change to the base is a new try and reruns the gate | `Pass 1.md`, `Review.md`, Changelog, index `status: reviewed` |
| 9 | Validation | Developer (resumed) | Runs what Pass 1 lists (validation period, variants, stress tests) plus a one-day delay control | `Build.md`, more runs |
| 10 | Analysis and red team (in parallel) | Analyst, Red team (fresh) | Analyst: each kill criterion, each prediction, luck, controls; verdict kill, candidate for the test, or inconclusive. Red team: tries to break it; open the test, fix first, or kill (only on evidence against) | `Pass 2.md`, `<Name> - Backtest report.md`, `Trials.csv`; `Red team.md`, `results/<slug>/redteam/` |
| 11 | Strategy report | Desk chief (`strategy-report`) | PDF built from the train and validation runs | `<Name> - Report.pdf` in the folder |
| 12 | Sealed test | You approve; `lab/run_test.py --batch` | One command for every waiting strategy and its frozen variant; Claude Code asks you first. Writes `TEST_DONE` locks | `results/<slug>/TEST_DONE`, test run |
| 13 | Post-test | Analyst (fresh, one per strategy) | Three-zone rule and full-sample t: **passed** (confirms or ambiguous) or **killed** (contradicts, or a Kill Policy kill). Folder moved. Report rebuilt with the test | `Test.md`, folder in `Passed/` or `Killed/`, updated PDF |
| 14 | Book | PM | Candidate review if `Book.md` holds strategies, otherwise a book construction from every passed strategy. Its "What the book needs" is the default theme of the next run | `4 Book/Reviews/*.pdf` |
| 15 | Head of pod decision and final report | Desk chief, then you | Desk report with three blocks first: tests (zone, t, PM decision, paths), assumptions taken (numbered, overturn one by replying "2: <new answer>"), decisions for the user. Then what moved, what died, tries and luck by family, tokens by seat. You alone edit `Book.md`, in the monthly slot | `3 Research Log/Desk reports/<YYYY-MM-DD HHMM>.md`, `Lessons.md`, `Desk economics.csv`, `Book.md` (by you) |

Routing rules worth knowing:

- **A twin beats the base**: the simpler twin becomes the candidate, one more counted try, then the gate again on the twin.
- **Inconclusive or fix first**: back to the seat whose job it is (Data, Developer or Economist); a rewrite of the mechanism counts as a try; the Analyst and Red team run again in fresh agents.
- **Kill on evidence, never on absence of evidence**: a positive but under-powered control is a label on the strategy, not a kill ([Kill Policy](vault/Toolbox/Kill%20Policy.md)).
- **If you refuse the test approval**, the tests stay sealed and the report lists the command to run them later.
- **Optional Gate 1**: if you ask to approve Pass 1 before code, the desk stops after the judgement and shows the verdicts.

---

## Recommended workflow for a team member

### 1. Setup

Requirements, from [`pyproject.toml`](pyproject.toml) and the code:

- **Python 3.12 or newer** (`requires-python = ">=3.12,<4"`).
- **Poetry** (the lock file was generated by Poetry 2.2.1; `package-mode = false`, so Poetry installs dependencies only). Dependencies: numpy, pandas 3.x, pyarrow, requests, certifi, pymupdf, matplotlib, tabulate, reportlab.
- **A LaTeX engine** for the PDF reports only: `lab/strategy_report.py` and `lab/book_report.py` look for `pdflatex` or `tectonic` on your `PATH`. The repository's messages suggest `brew install tectonic` (macOS) or MacTeX/BasicTeX. On Windows, any distribution that provides `pdflatex` (for example MiKTeX or TeX Live) or a `tectonic` binary should work. Everything except the PDFs runs without LaTeX.
- **Claude Code**, opened at the repository root so it picks up `CLAUDE.md`, `.claude/agents`, `.claude/skills` and `.claude/settings.json`.
- **Obsidian** (optional but recommended) to browse the vault.

```bash
git clone https://github.com/attiaj/ai-quant-pod.git
cd ai-quant-pod
git remote add upstream https://github.com/Quantreo/ai-quant-pod
poetry install
poetry run python lab/selftest.py     # every line must say OK
```

`lab/selftest.py` checks, on synthetic prices: buy and hold earns the price return; a weight dated d earns nothing on day d; a round trip at 10 bp costs 20 bp; a long pays positive funding; the truncation test passes an honest signal and catches one that reads tomorrow. Its last check (loaded data stops before the test start) is skipped until you download the crypto data. Run it again after any change to `lab/`.

Windows notes (not tested by the course, which was built on macOS): the hook and several skills call `python3`, so make sure `python3` resolves to a Python 3.12+ interpreter in the shell Claude Code uses. The scripts read and write vault notes with Python's default text encoding, and the notes contain non-ASCII characters, so setting `PYTHONUTF8=1` in your environment is a sensible precaution.

### 2. Download the data

From the repository root (see [`data/README.md`](data/README.md)):

```bash
poetry run python data/get_crypto_daily.py     # Binance USDT perps, daily -> data/crypto/
poetry run python data/get_fx_daily.py         # 9 FX majors vs USD (FRED H.10) -> data/fx/
poetry run python data/get_rates.py            # G10 rates, T-bill, policy rates, CPI, US curve (FRED) -> data/rates/
poetry run python data/get_etf_daily.py        # about 220 ETFs (Yahoo Finance) -> data/etf/
```

Each script writes a quality report next to its files: read it once. The date ranges are fixed in the scripts so every team member builds the same data. The data is never committed (licences): each person downloads it for their own research. Then run `poetry run python lab/selftest.py` again; the sealed-test check should now say OK.

### 3. Work with the pod in Claude Code

Skills are picked by Claude Code from their descriptions, so plain requests work; naming the skill or the seat makes it explicit. Seats can be called as subagents (the `/agents` command in Claude Code lists them). Example prompts, taken from the skill and agent descriptions:

| Goal | Example prompt | What runs |
| --- | --- | --- |
| Run the whole desk on a theme | "Run the desk: find me a crypto trend strategy." | `desk` skill; you approve the test batch at the end |
| Push one strategy forward | "Run the desk on Crypto Funding Carry: push it forward." | `desk` skill from the strategy's current stage |
| Ask where things stand | "Where does the desk stand?" | `desk` skill reads the index, trials summary and Desk log |
| Use the PM's last needs as theme | "Run the desk." (no theme) | `desk` takes "What the book needs" from the newest PM PDF |
| Approve Pass 1 before any code | "Run the desk on FX carry, and let me approve Pass 1 before code." | `desk` stops after the judgement (Gate 1) |
| Find papers | "Use the scout to find papers on FX carry crash risk." | `scout` agent, `paper-scout` skill; it shows up to 10 picks and waits for you |
| Read a paper | "Read `vault/0 Library/Sources/<file>.pdf` and write a reading note." | `reading-source` skill (run it as a general-purpose sonnet agent to save context) |
| Make technique cards | "Make technique cards from the note on Moreira Muir 2016." | `technique-card` skill; shows a triage table first |
| Get ideas | "Give me ideas on carry in FX from the library." | `ideator` agent, `ideation` skill |
| Judge an idea | "Have the economist judge the FX value idea in the Idea Journal." | `economist` agent, `pass-one` skill |
| Check data | "Check the data for `vault/2 Strategies/Pipeline/<Name>/`." | `data` agent, `data-check` skill |
| Build and gate | "Build `<Name>`, phase gate." | `developer` agent, `build-strategy` skill |
| Analyse | "Analyse the results of `<Name>`." | `analyst` agent, `analyze-backtest` skill |
| Attack | "Red-team `<Name>` before its test." | `red-team` agent |
| Report | "Write the strategy report for ETF TSMOM 12-Month." | `strategy-report` skill |
| Portfolio | "Build a book from every passed strategy." / "Review `<Name>` as a candidate for the book." | `pm` agent, `portfolio-manager` skill |

When a desk run reaches the test batch, Claude Code will ask permission for a single command of the form:

```bash
poetry run python lab/run_test.py --batch strategies/<a>.py:<variant> strategies/<b>.py:<variant>
```

Approve it only if you have read the strategies' reports in their folders. An opened test is final for that strategy. Afterwards, read the desk report in `vault/3 Research Log/Desk reports/`, overturn any assumption by number, and decide on `Book.md` yourself.

Useful manual commands:

```bash
poetry run python lab/trials.py report                                   # refresh the luck lines and the budget of tries
poetry run python .claude/skills/pass-one/scripts/budget.py --tries 4 --years 12 --expected-sharpe 0.6
python3 .claude/skills/ideation/scripts/library_index.py --family carry  # what the library holds on a theme
```

### 4. Add a new strategy

The intended route is through the desk (idea, Pass 1, data check, then the Developer writes the file). If you write or edit one by hand, follow the same conventions ([`build-strategy/reference/conventions.md`](.claude/skills/build-strategy/reference/conventions.md)):

1. **Pass 1 first.** Create the strategy folder `vault/2 Strategies/Pipeline/<Name>/` with its index, `Pass 1.md` and `Changelog.md` (templates in [`pass-one/assets/`](.claude/skills/pass-one/assets)). Get luck lines, power and cost figures from `budget.py`.
2. **Start from the template**: copy [`strategy-template.py`](.claude/skills/build-strategy/assets/strategy-template.py) to `strategies/<strategy_slug>.py`.
3. **Constants**: set `NAME` (it must match the vault folder name), `TRAIN`, `VALIDATION`, `TEST_START` as plain string literals (the report script finds them by pattern), and every parameter with a comment naming its Pass 1 section.
4. **Data**: always `lab.data.load(name, test_start=TEST_START)`. Never read a date at or after the test start.
5. **Weights**: `build_weights` computes everything that depends on data, the universe included, using only data up to the close of each trade day (a weight dated d trades at the close of d). For crypto, use `lab.universe.crypto_universe`; for FX, `lab.fx.fx_bars` and `fx_carry`.
6. **One `run(period, variant, cost_bp, delay, open_test)` function**: every frozen choice (costs, leverage, band, tiers) must live inside it, because `lab/run_test.py` calls exactly this function for the test and refuses a file without it. Never pass `open_test=True` yourself; the hook blocks it.
7. **Check, then run**:

   ```bash
   poetry run python strategies/<file>.py --period train --variant base --check-lookahead   # must PASS first
   poetry run python strategies/<file>.py --period train --variant base                    # the gate: one run
   poetry run python strategies/<file>.py --period validation --variant base               # after review
   poetry run python strategies/<file>.py --period train --variant base --delay 1          # control, not a try
   poetry run python strategies/<file>.py --period train --variant base --cost-bp 20       # stress test
   ```

8. **Log and count every run**: a Research Log line per run, and `poetry run python lab/trials.py add results/<slug>/<run id> --sheet "vault/2 Strategies/Pipeline/<Name>/<Name>.md" --counts yes|no --why "..."`.
9. **Do not change `lab/`** while a strategy is being built. A change to `lab/` is a team decision; run `lab/selftest.py` after it, and rerun any runs that must be compared, since their `lab_sha` will differ.

### 5. Git tips for the team

- **Branch per piece of work**, for example `git checkout -b desk/fx-carry` for a desk run or `strategy/<slug>` for a hand-built strategy, and open a pull request into `main` on `origin`.
- **What gets committed**: vault notes, strategy files, PM and strategy PDFs. **What never does**: `data/` downloads, `results/` (runs, locks, report work folders), paper PDFs in `vault/0 Library/Sources/`, and `vault/4 Book/Returns/*.csv` (all in `.gitignore`). Because `results/` is local, a teammate cannot see your runs or `TEST_DONE` locks; the vault notes (`Build.md`, `Test.md`, `Trials.csv`) are the shared record.
- **Agents cannot push**: `git push` is denied in `.claude/settings.json`. Commit and push from your own terminal after reviewing what the agents wrote.
- **The trial count is shared**: every team member's tries land in the same `Trials.csv` and raise the same luck lines. Pull before a desk run, and expect merge conflicts in shared logs (`Trials.csv`, `Desk log.md`, `Research Log.md`, `_Index.md`, `Idea Journal.md`). Keep both sides' rows when you resolve them; deleting a row would hide a try.
- **Pulling course updates**: `git checkout main`, `git pull upstream main`, resolve any conflicts, then `git push`. Upstream may change the same vault files the team has edited.
- **Never rewrite the history of a sealed test**: the strategy file and Pass 1 are frozen once its test is opened.

---

## Disclaimer

This repository is educational research material from a course. Nothing in it, including the strategies, backtests, reports, book proposals and agent outputs, is investment advice or a recommendation to trade. Backtested results are hypothetical, depend on cost and data assumptions that may be wrong, and do not predict future returns. The agents can make mistakes. Market data downloaded by the scripts is subject to its providers' licences (Binance Vision CC BY-NC-SA 4.0, Yahoo Finance and FRED terms of use) and is for your own non-commercial research only.

---

## Original README (Quantreo)

### ai-quant-pod
