# Review — Changes Since `14550e1`

*Reviewer: change-reviewer sub-agent. Scope: `git diff` on the one modified file plus a full
read of the three untracked files. Every factual claim in the reviewed text was re-verified
against the repository and by running the backend test suite.*

## What changed

| File | Status | Change |
|---|---|---|
| `planning/PLAN.md` | Modified | **+558 lines, 0 deletions** — pure append of two new sections, §13 and §14 |
| `.claude/agents/change-reviewer.md` | New | 6-line sub-agent definition (this reviewer) |
| `.claude/agents/codex-reviewer.md` | New | 9-line sub-agent that shells out to `codex exec` |
| `.claude/commands/doc-review.md` | New | 1-line slash command for doc review |

No source code changed. Nothing was deleted or rewritten — §13/§14 sit strictly after §12,
so the existing spec is untouched.

---

## 1. `planning/PLAN.md` — the review sections

### 1.1 Accuracy: verified, and it holds up

The value of §14 rests entirely on whether its code-grounded claims are true. I checked the
falsifiable ones. **All of them are correct**, several to the digit:

| Claim | Verified |
|---|---|
| §14 #66 — `stream.py` defines `router` at module scope and the factory registers onto it | True. `router = APIRouter(prefix="/api/stream")` at line 18; `@router.get("/prices")` inside `create_stream_router`. Repeat calls stack duplicate routes on one shared object. |
| §14 #67 — no heartbeat; `_generate_events` yields only on `version` change | True. The only unconditional yield is the opening `retry: 1000`. |
| §14 #68 — `PriceCache.update()` locks per-ticker; `version` is read without the lock | True. The `version` property returns `self._version` with no `with self._lock`. |
| §14 #69 — remove/re-add re-seeds from `SEED_PRICES` | True. `simulator.py:151` — `SEED_PRICES.get(ticker, random.uniform(50.0, 300.0))`. |
| §14 #70 — per-tick sigma table | Arithmetic checks out. dt = 0.5/(252·6.5·3600) = 8.48e-8; NVDA 800·0.40·√dt = $0.093, AAPL 190·0.22·√dt = $0.012. Seed prices and sigmas match `seed_prices.py`. |
| §14 #79 — skill directory is `cerebras`, frontmatter `name:` is `cerebras-inference` | True, exactly as described. |
| §14 #81 — only `claude.yml` + `claude-code-review.yml` in `.github/workflows/` | True. No test workflow. |
| §14 #82 — 91% coverage, 73 tests, `stream.py` 36 stmts / 24 miss / 33% | **Exact.** Re-ran `uv run pytest --cov`: `TOTAL 349 31 91%`, `stream.py 36 24 33%`, 73 passed in 4.07s. |
| §14 S8 — `factory.py` imports `MassiveDataSource` unconditionally | True, top-level import. |
| §14.2 #63–65 — `.gitignore` misfires | True. `git check-ignore -v frontend/lib/api.ts` → `.gitignore:17:lib/`. `node_modules/`, `out/`, `db/finally.db` all **not** ignored. |
| §14.1 — no `frontend/`, `test/`, `scripts/`, `db/`, `Dockerfile`, `.env`, `.env.example` | True. Root holds only `.claude .github backend planning CLAUDE.md LICENSE README.md .gitignore`. `README.md:30` really does open with `cp .env.example .env`. |
| §13 #36 — `httpx`, `litellm`, `python-dotenv` absent from deps | True. Dev extra is pytest, pytest-asyncio, pytest-cov, ruff. |

I found **no false claims** in either section. That is unusual for a document of this size and
worth saying plainly.

### 1.2 Defects in the new text

**D1 — [Blocker for the doc] The numbering is broken.** §14's preamble says *"Numbering
continues from §13."* §13's numbered items end at **37**; §14 begins at **61**. Items 38–60 do
not exist anywhere in the file. Either an intermediate review pass was dropped and its
references went with it, or the range was chosen arbitrarily. Consequence: a reader who meets a
cross-reference to "#45" has no way to know it was never written, and the gap invites a third
pass to collide. Fix: renumber §14 as 38–60, or state explicitly that 38–60 are reserved or
withdrawn. (The `S`-series is fine — §13 ends at S7, §14 starts at S8.)

**D2 — Tag discipline is inconsistent.** §14 promises *"Same tags: [Blocker] / [Clarify] /
[Simplify]"* for every finding, but items **63, 64, 65 and 72 carry no tag on the item line** —
their severity lives only in the section heading (`### 14.2 … — [Blocker]`, `### 14.4 … —
[Blocker]`). Anyone grepping for `[Blocker]` to build a work list will miss four items,
including #72, which the document itself treats as one of its most important findings.

**D3 — Two "highest-leverage" claims compete.** §14.8 nominates the `.gitignore` as the single
change to make first. §13.10's build order puts the DB layer at step 1 and never mentions the
`.gitignore`. They are compatible in principle (fix ignore rules, *then* start step 1), but
neither section says so. One sentence in §13.10 — *"step 0: fix `.gitignore` (§14.2)"* — removes
the ambiguity for an agent that reads only one of the two.

**D4 — §13 and §14 are questions, not decisions.** This is the structural risk, not a nit.
There are now **~14 items tagged [Blocker]** across the two sections, each phrased as an open
question with a recommendation ("Which rule applies?", "Pick one"). Nobody has answered any of
them, and nothing distinguishes *recommended* from *decided*. Since CLAUDE.md tells agents
PLAN.md is the contract, the next builder agent meets a contract that asks it 14 questions. The
recommendations are good enough to adopt nearly wholesale — the missing step is a decision pass
that folds the accepted answers **into §§1–12** and leaves §13/§14 as an appendix of rationale.
Without that, every downstream agent re-litigates the same choices, and they will not agree.

**D5 — [Simplify] PLAN.md is `@`-imported in full, and just tripled in size.** `CLAUDE.md`
contains `@planning/PLAN.md`, so the entire file loads into every session's context. The file
went **21,852 → 61,152 bytes (456 → 1,014 lines)**, a ~180% increase — roughly 10k extra tokens
on every turn of every session, for content that is review commentary rather than
specification. Note the contrast: `MARKET_DATA_SUMMARY.md` is deliberately *not* imported
("Consult these docs only when required"). Recommend moving §13/§14 to
`planning/REVIEW_NOTES.md` with a one-line pointer in PLAN.md — or, better and per D4, resolving
them into §§1–12 and archiving the notes to `planning/archive/`.

**D6 — Minor factual imprecision.** §13 #36 describes the dev extra as "pytest, coverage and
ruff"; it also contains `pytest-asyncio`, which matters because `asyncio_mode = "auto"` depends
on it. Doesn't change the conclusion (`httpx` is genuinely missing), but the inventory is
incomplete.

**D7 — Two findings are stronger than stated.** Worth upgrading when they get actioned:

- #66 (shared module-level router) is not only a test-fixture hazard. `create_stream_router` is
  the one wiring pattern in the repo, and #73 correctly notes it will be copied into the
  portfolio, watchlist and chat routers. Fixing it now costs one line; fixing it after four
  routers copy it costs four.
- #67 (no heartbeat) and #68 (torn snapshot) both live in `stream.py` — the module measured at
  **33% coverage**, the worst in the codebase. That is not a coincidence, and §14 #82 is right
  that a flat 91% gate hides it.

### 1.3 What the sections get right

- Grounding claims in executed commands (`git check-ignore -v`, a REPL transcript showing
  `r1 is r2 == True`, a measured coverage table) rather than assertion. Every one reproduced.
- #72 (a blocking `litellm.completion` freezing the event loop, and therefore the price stream)
  is the single best finding in either section: invisible when the chat feature is built in
  isolation, glaring in production, and it correctly points at the `asyncio.to_thread` pattern
  `massive_client.py` already uses.
- #75's empty/loading/error-state table and #76's missing up-green/down-red palette are exactly
  the cross-agent seams that otherwise produce a UI that looks assembled by three people.
- §14.1's repo-vs-doc table is the right opening, because §9's "There is an OPENROUTER_API_KEY
  in the `.env` file" is actively false and would send an agent down a confusing failure path.

---

## 2. `.claude/agents/` and `.claude/commands/`

These are the first agent and command definitions in the repo. `.claude/settings.json` and
`.claude/skills/cerebras/` are already tracked, so committing these is consistent with existing
practice — they should be staged, not left untracked.

**A1 — [Blocker] Both reviewer agents write to the same file, `planning/REVIEW.md`.**
`change-reviewer` reviews *uncommitted changes*; `codex-reviewer` reviews *PLAN.md*. Running one
after the other silently destroys the first result — no merge, no warning — and the file is not
gitignored, so whichever survives gets committed as if it were the whole story. Recommend
distinct paths (`planning/reviews/changes-<date>.md`, `planning/reviews/plan-codex.md`) or
appending under a dated heading.

**A2 — [Clarify] Should `planning/REVIEW.md` be committed at all?** It is a point-in-time
artifact that goes stale the moment the next commit lands, and nothing in `.gitignore` covers
it. Decide: transient (gitignore it) or durable (then it needs a date and commit stamp, which
neither agent instructs). Note that §13/§14 already demonstrate the durable pattern working well
*inside* PLAN.md — which is arguably where this content belongs.

**A3 — [Clarify] `change-reviewer` has no `tools:` restriction**, so it inherits everything
including Bash and Write. It needs Write for one file and Bash for `git diff`; full access means
a review agent can also edit source, commit, or push. Consider narrowing to
`tools: Bash, Read, Grep, Glob, Write`. Same for `codex-reviewer`, which needs only Bash.

**A4 — `codex-reviewer` is fragile in three ways.** The `codex` binary is present on this
machine (`~/AppData/Local/Programs/OpenAI/Codex/bin/codex`), so it works here, but:

1. No check that `codex` exists. On any other clone the agent's single mandatory command fails
   and, because the prompt says "Do not review yourself", it has no fallback and returns nothing.
2. `codex exec` runs with default sandbox and approval settings, which may prompt or block in a
   non-interactive sub-agent. Pin the flags you want.
3. It hard-codes both the input (`planning/PLAN.md`) and the output path, so it cannot review any
   other document — despite `doc-review.md` being parameterized. Consider accepting an argument.

**A5 — Typos in text the model reads as instructions.** `change-reviewer.md` description: "all
**chages** made since the last commit". `codex-reviewer.md`: "write your **feed back**".
Descriptions drive agent selection, so keep them clean.

**A6 — `doc-review.md` has no frontmatter.** Command definitions normally carry `description`
and `argument-hint`. Without them the command shows no help and `$ARGUMENTS` is undiscoverable —
add `argument-hint: <filename>` at minimum. Worth noting this command produced §13/§14 ("add …
to a new section at the end, along with any opportunities to simplify") and matches the output
exactly: the command works; it is the *destination* (D5) that needs thought.

**A7 — Line endings and trailing newlines.** All three files are CRLF, and the two agent files
have **no trailing newline**. The repo has no `.gitattributes` and `core.autocrlf` is on
(`git diff` warned "LF will be replaced by CRLF"), so these will churn for collaborators on
macOS and Linux — relevant given §11 promises cross-platform scripts. A three-line
`.gitattributes` (`* text=auto`, `*.sh text eol=lf`, `*.ps1 text eol=crlf`) settles it before
more files land.

---

## 3. Recommended order of action

1. **Fix `.gitignore`** (§14.2). Agreed with §14.8 — it is the only finding whose failure mode is
   invisible, and every hour of frontend work before the fix is work that can silently vanish.
2. **Answer the ~14 [Blocker] items and fold the decisions into §§1–12** (D4). Until this
   happens, PLAN.md is not a contract.
3. **Renumber §14 and tag items 63, 64, 65, 72** (D1, D2) — five minutes, prevents reference rot.
4. **Move §13/§14 out of the `@`-imported PLAN.md** once resolved (D5).
5. **Fix `stream.py`'s module-level router now** (#66) — before four more routers copy the shape.
6. **Give the two reviewer agents distinct output paths** (A1) and decide REVIEW.md's fate (A2).
7. **Stage `.claude/agents/` and `.claude/commands/`** — consistent with the already-tracked
   `.claude/settings.json` and `.claude/skills/`.

## Verdict

The PLAN.md addition is high-quality, factually reliable work — I could not find a single
incorrect claim in 558 lines, and #72 alone justifies the pass. Its weaknesses are structural
rather than analytical: broken numbering, four untagged findings, and above all fourteen
blockers left *asked* rather than *answered* inside a document agents are told to treat as the
contract, while tripling the size of a file loaded into every session. The `.claude/` additions
are useful and correctly shaped, with one real bug — both reviewers overwrite the same output
file — and a scattering of hygiene issues.
