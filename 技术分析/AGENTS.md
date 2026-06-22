# Agent Notes

## Workspace Intent

This folder is a vertical knowledge base for technical analysis inside the broader `量化` workspace.
Use the LLM Wiki vertical knowledge base branch here, but follow this topic's three-layer material structure instead of creating separate map files by default.

The goal is to turn technical-analysis material into a reusable domain judgment system for market interpretation, strategy filtering, timing, and replay.

In this public repository, first-layer raw materials are intentionally excluded. The index may mention those sources, but PDF files, raw reports, original books, and backtest outputs stay outside the repo.

## Required Read Order

1. `README.md`
2. `【第三层】期货市场技术分析.md`
3. `【索引】技术分析资料索引.md`
4. The minimum relevant files under `02-第二层-类型整理/`

## Three Material Layers

- First layer: raw or bottom-layer material. This includes reports, original sources, backtests, outputs, cases, and future source documents. Preserve provenance and avoid over-summarizing at the folder-maintenance stage. Public copies should only keep source indexes, not the raw files.
- Second layer, `02-第二层-类型整理/`: already compressed material organized by type, such as book notes, method notes, and self-built strategy experiments.
- Third layer, topic root or future `03-第三层-二次整理/`: recomposed domain knowledge. `【第三层】期货市场技术分析.md` is the current main body and should be strengthened before parallel summaries are created.

`【索引】技术分析资料索引.md` is a reference and navigation file. It may mention first-layer sources, but its index sections must stay separate from the third-layer body.

## Build Mode

Initial build:

- establish or repair the three-layer folder structure
- keep the third-layer main note as the first reading target
- create or update `【索引】技术分析资料索引.md`
- write local README/AGENTS instructions that explain first read, fallback read, and material boundaries

Supplemental build:

- read the third-layer main note and index first
- decide whether new material belongs to first, second, or third layer
- merge durable ideas into the third-layer body when they change the domain framework
- update the index when new sources or notes are added
- avoid parallel one-off summaries unless a standalone topic file is explicitly useful

## Current Boundary

In scope:

- technical-analysis definitions, assumptions, schools, and failure modes
- K-line and candlestick patterns
- price-volume, Wyckoff-style reasoning, and order-flow reading
- trend following, reversal indicators, and cycle reasoning
- Bolling, RSI, RSRS, moving averages, MACD, MFI, KDJ, CCI, and related timing signals
- technical-analysis use in stock index, futures, options CTA, and crypto examples

Adjacent but not automatically in scope:

- broad momentum-factor research
- fundamental, macro, industry, and valuation analysis
- code-framework maintenance
- commodity supply-demand research
- digital-currency domain research that is not primarily technical-analysis driven

## Editing Rules

- Do not add code or runnable source files to this topic.
- Keep spreadsheets and source PDFs outside the public repository unless explicitly converted into user-authored notes.
- Use `content-master` for one-report-at-a-time source understanding, then update the index. Do not create a second-layer report note unless the report has reusable standalone value.
- Strengthen `【第三层】期货市场技术分析.md` before creating another third-layer peer file.
