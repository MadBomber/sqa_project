# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Workspace Overview

`sqa_project/` is not a single repository — it's a workspace containing the independent repos that make up the **SQA (Simple Qualitative Analysis)** ecosystem, a Ruby toolset for stock market technical analysis, trading strategy backtesting, and portfolio optimization. SQA is explicitly an **educational/learning tool, not production trading software** — financial risk warnings in each repo's README must be preserved.

Each subdirectory is its own git repo with its own `Gemfile`, tests, and (usually) its own `CLAUDE.md`. When working inside one, that sub-repo's `CLAUDE.md` takes precedence for project-specific guidance — this file covers cross-repo structure only.

This workspace has [asgard](https://github.com/MadBomber/asgard) `.loki` task files set up (root `.loki`, `repo_dev.loki`, `ws_*.loki`, plus one `.loki` per repo) — run `asgard help` from the workspace root or any repo for the available commands instead of memorizing raw `rake`/`bundle` invocations.

## Repos in this workspace

- **`sqa/`** — the **core `sqa` gem**: `SQA::Stock`, `SQA::DataFrame` (Polars-backed), trading strategies, backtesting, portfolio management, genetic programming, KBS (RETE rule engine), real-time streaming, risk management, and more. This is the foundation every other repo depends on. A single `main`-branch checkout — there is no `develop` worktree anymore (it was collapsed back into `main` at some point; if you see references to `sqa/main` or `sqa/develop` elsewhere, e.g. in stale docs or `Gemfile`/`CLAUDE.md` text, they're outdated). See `sqa/CLAUDE.md` for its detailed architecture.
- **`sqa-tai/`** — `sqa-tai` gem, a Ruby wrapper around the TA-Lib C library providing 136+ technical indicators (SMA, RSI, MACD, Bollinger Bands, etc.). The `sqa` gem delegates all indicator math here via `SQAI` / `SQA::TAI` — **do not add indicators to `sqa` itself**, contribute to `sqa-tai` instead. Requires the TA-Lib C library installed (`brew install ta-lib`).
- **`sqa-cli/`** — `sqa-cli` gem, command-line interface built on top of `sqa` (depends on a pinned `sqa` version in its gemspec; for local dev its `Gemfile` overrides with `path: "../sqa"`). Exposes analysis/backtesting/portfolio commands.
- **`sqa-advisor/`** — `sqa-advisor` gem, an agentic CLI that wraps Claude (via RubyLLM) with 14 financial tools built on `sqa` + Yahoo Finance data, producing structured Markdown investment reports. Its `Gemfile` also overrides `sqa` with `path: "../sqa"` for local dev. See `sqa-advisor/CLAUDE.md` for its agent/tool architecture (note: that file's "Local gem dependency" section still says `path: "../sqa/main"` — outdated, fix when next touching that doc).
- **`sqa-rails/`** — Rails 8 demo web app for SQA (uses ClaudeOnRails; see `sqa-rails/.claude-on-rails/context.md`). Renamed from `sqa_demo-rails`; the GitHub remote is still `madbomber/sqa_demo-rails`. Not a packaged gem — no `build`/`install`/`release` tasks.
- **`sqa-sinatra/`** — Sinatra demo web app for SQA with ApexCharts.js-based interactive charts; requires TA-Lib and Redis (for KBS strategy). Renamed from `sqa_demo-sinatra`; the GitHub remote is still `madbomber/sqa_demo-sinatra`. Packaged as the `sqa_demo-sinatra` gem (underscore differs from the directory name).

There is no `sqa-tai_backup/` anymore — it was a stale, stuck-mid-rebase checkout and has been removed from the workspace.

## Dependency order

```
sqa-tai  →  sqa  →  sqa-cli
                 →  sqa-advisor
                 →  sqa-rails
                 →  sqa-sinatra
```

When making a change that spans repos (e.g. adding an indicator), start in `sqa-tai`, then update `sqa`'s dependency, then downstream consumers as needed. Each downstream gemspec pins a `sqa` version — check before assuming a core change is immediately visible to `sqa-cli`/`sqa-advisor`.

## Common Commands

Prefer `asgard` (see Workspace Overview above) — `asgard quality` runs the full gate (Tests+Coverage, RuboCop, Flog, Flay, Reek) identically in every gem repo. Underlying raw commands, for reference — all 5 gem repos (`sqa`, `sqa-tai`, `sqa-cli`, `sqa-advisor`, `sqa-sinatra`) follow the same pattern:

```bash
bundle install
bundle exec rake test                          # run all tests (Minitest), with SimpleCov coverage
bundle exec rubocop                             # style/complexity gate, shared config (see below)
bundle exec rake flog_check                     # complexity gate (warn >=20, fail >=50)
bundle exec rake flay_check                      # duplication gate (mass >= 50)
bundle exec rake reek_check                      # code-smell gate (baseline-aware; see below)
ruby -Ilib:test test/path/to_test.rb            # run a single test file
bundle exec rake build                          # build the gem into pkg/
bundle exec rake install                        # install locally
```

Reek runs in two equivalent places, both baseline-aware (fail only on new/worsened files vs `.quality/reek_baseline.txt`): as a **Rake task** — `bundle exec rake reek_check`, included in each gem's `rake quality` gate alongside flog/flay — and in the **loki** (`repo_dev.loki`'s `reek_scan`/`reek_gate`), reachable via `asgard reek` / `asgard quality`. The rake task requires `reek` in the bundle (now an explicit dev dependency in every gemspec); the loki path loads reek from asgard's own environment. Ratchet the floor down after a genuine improvement with `asgard reek_baseline`.

`sqa-rails` is a standard Rails 8 app (`bin/rails test`, `bin/rails server` / `bin/dev`, `bin/rubocop`) — use `asgard test` / `asgard server` / `asgard rubocop` there rather than the gem-lifecycle commands above. It's not part of the shared RuboCop/Reek setup below (it uses `rubocop-rails-omakase` instead).

### RuboCop — shared config

All 5 gem repos share one `.rubocop.yml`, generated from `sqa_project/.rubocop.yml.common` — **edit the `.common` file, then run `asgard sync_rubocop` to propagate it**, don't edit a repo's `.rubocop.yml` directly (it'll just get overwritten next sync). The shared config is adapted from `robot_lab_project`'s real, working `.rubocop.yml` (Metrics thresholds, Style relaxations), not a generic default — see its own comments for why each cop is tuned the way it is. Notable repo-specific carve-outs already in there, so you don't have to rediscover them:
- `Style/OpenStructUse` and `Style/GlobalVars` disabled globally — both are deliberate architecture in this codebase (OpenStruct vectors, `$DEBUG_ME`/`$data`), not accidents.
- `Naming/VariableNumber` excludes `lib/api/alpha_vantage_api.rb`, `lib/sqa/tai/overlap_studies.rb`, `lib/sqa/tai/momentum_indicators.rb` — these mirror Alpha Vantage/TA-Lib's own literal API names (`t3`, `timeperiod1/2/3`) verbatim; renaming would break the API calls, not just the style.
- `Metrics/ModuleLength` excludes `lib/sqa/tai/pattern_recognition.rb`, `lib/sqa/tai/momentum_indicators.rb`, and the two largest `sqa-sinatra` helper modules — many independent thin indicator/route-handler wrappers, long by breadth not complexity.
- A few individual methods (data-extraction tools in `sqa-advisor`, already on the `Rakefile`'s `FLOG_EXCEPTIONS` list) carry inline `# rubocop:disable Metrics/AbcSize` for the same "field count, not logic" reason.

### Reek — grandfathered baseline, not a clean slate

Reek found ~450 pre-existing smells across the 5 repos when first enabled (2026-07-01) — nowhere near zero, and that's fine: the gate (`repo_dev.loki`'s `reek_gate`) only fails on **new or worsened** files per `.quality/reek_baseline.txt` in each repo, matching `robot_lab_project`'s own actual practice (most of its repos aren't clean either). Don't try to drive every repo to zero smells in one pass; fix smells opportunistically, and run `asgard reek_baseline` to ratchet the floor down after a genuine improvement. The shared `.reek.yml` lives at the workspace root (resolved by walking up from cwd, same mechanism as `robot_lab_project`'s).

### flog/flay/coverage gotcha (Ruby 4.x)

`flog`/`flay` depend on `prism`'s `ruby_parser` translation shim. Older `prism` (<1.9.0) needs the separate `ruby_parser` gem and will silently `exit(1)` on `require "flog"` if it's missing from the bundle — no error message, just a dead process. Newer `prism` (≥1.9.0) fixed that shim to use `sexp_processor` instead, which is already a transitive dependency. `flog` also needs `racc`, which isn't guaranteed to be pulled in transitively by a gem's other dependencies. Each gemspec here pins `flog`, `flay`, `racc`, `reek`, and `simplecov` as explicit dev dependencies for exactly this reason — if `flog_check`/`quality` ever silently produces zero output and a bare exit code 1 again after a dependency bump, check the resolved `prism` version first (`bundle exec ruby -e 'require "prism"; puts Prism::VERSION'` — want ≥1.9.0).

## Cross-cutting notes

- All gems require Ruby >= 3.1/3.2 — check each gemspec. The active dev Ruby in this workspace is 4.0.x (rbenv) — `sqa-rails` currently pins `.ruby-version` to `4.0.0-preview2`, which isn't installed (only 4.0.4/4.0.5 are); `asgard`/`bundle`/etc. will fail there with an rbenv "version not installed" error until that's reconciled.
- `sqa`'s `SQA::DataFrame` wraps `polars-df` (Rust-backed); prefer vectorized/column operations over row iteration everywhere downstream too. `polars-df` ships precompiled native gems for common platforms — if a `bundle install`/`update` ever falls back to compiling it from source (Cargo output, takes several minutes), check whether a matching precompiled version is already installed locally (`gem list polars-df`) and `bundle update polars-df sqa` to reuse it instead of waiting out the Rust build.
- TA-Lib arrays must be **oldest-first** (ascending chronological) — this convention from `sqa`/`sqa-tai` propagates to any code in `sqa-cli`/`sqa-advisor`/the demo apps that touches price arrays.
- API keys (Alpha Vantage, Anthropic/OpenAI) are read from environment variables only across all repos — never hardcoded.
- Gems in the shared rbenv gemset (`~/.rbenv/versions/4.0.5/...`) can get modified/removed by something outside any single repo's `bundle install` while you're mid-session (observed 2026-07-01: `bundle exec rubocop`/`rake` intermittently died with a bare exit code and zero output, or `Bundler::GemNotFound`/"gemspec ... missing or broken" for an unrelated gem, then worked again moments later with no code change). If a command that worked a minute ago suddenly fails with a missing/corrupt gem error, re-run `bundle install` (or `gem pristine <name>`) and retry before assuming it's a real regression — it likely isn't.

### Other Ruby 3.5/4.x compatibility fixes already applied in `sqa`

These were real bugs surfaced by running the test suite under Ruby 4.0, not tooling artifacts — if similar errors reappear (e.g. after a `polars-df` or `minitest` upgrade), check whether they've regressed:
- `ostruct` is a "bundled gem" (not default) in Ruby 3.5+; `sqa.gemspec` declares it explicitly since `lib/sqa/{backtest,multi_timeframe,stream}.rb` and `SQA::Strategy` vectors rely on `OpenStruct`.
- `Polars::Series#apply` was renamed to `#map_elements`, and `Polars::DataFrame#with_column` (singular) was removed in favor of `#with_columns` (plural) — fixed in `lib/sqa/data_frame.rb`, `lib/sqa/data_frame/alpha_vantage.rb`, `lib/sqa/stock.rb`.
- `Polars::DataFrame#sort`'s `reverse:` kwarg was renamed to `descending:` — same files.
- `Minitest::Mock` was split out of `minitest` into the separate `minitest-mock` gem starting at `minitest` 5.27 — `sqa.gemspec` now declares `minitest-mock` explicitly and `test/test_helper.rb` requires `minitest/mock`.
