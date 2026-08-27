---
name: simetrik-cli
description: >
  Operate the Simetrik CLI (`simetrik`) end-to-end. BUILD reconciliation
  resources — native sources, segments, calculated columns, unions, joins,
  groups, legacy and advanced reconciliations, sweeps, compensations,
  consolidations, opening records, data exports — from a JSON definition
  file or a Markdown implementation plan. OPERATE them once built: manual
  reconciliation (`recon manual`, `adv-recon manual`), manual compensation
  (`adv-recon compensate`, `adv-recon compensation-records`),
  unreconciliation (`recon unreconcile`, `adv-recon unreconcile`),
  decompensation (`adv-recon decompensate`), run and `--reprocess`, async
  job tracking, schedules, alarms, approvals, snapshots. Also covers the
  Accounting Translator (`accounting*` groups), the Operation Center
  (`dataset`, `dashboard`), row-level reads with `data query`, reading the
  account's addon catalog (`account addons`), and the pre-query reads that
  cover 100% of a resource — `data profile` and `data aggregate` — used to
  validate results and state volume, coverage or absence without inferring
  from a page. Triggers:
  "simetrik", "create reconciliation", "crear fuente / unión / barrida",
  "conciliación manual", "manual reconciliation", "compensar registros",
  "compensación manual", "desconciliar", "unreconcile", "descompensar",
  "decompensate", "correr / reprocesar una conciliación", "task status",
  "programar", "data export", "modelo contable", "asientos contables",
  "dataset", "dashboard", "operation center", "data query", "account addons",
  "data profile", "data aggregate", "cuántos registros", "how many rows",
  "validar resultados", "validate results", "revisar hallazgo", or any
  multi-phase reconciliation setup.
---

# Simetrik CLI Operator

You orchestrate `simetrik` CLI commands to **build** reconciliation resources from a specification (JSON definition or implementation plan) and to **operate** them once built. This skill covers **ordering**, **critical execution rules**, and **non-obvious gotchas** that `--help` does NOT cover.

Run `--help` first for raw command syntax. The `references/` files document only cross-command knowledge: wire shapes, async semantics, dependency ordering, and API validation rules.

> **Latest CLI release required.** Upgrade with `simetrik update -y`. For fresh installs: `curl -fsSL https://cli.simetrik.com/install.sh | sh` (macOS/Linux) or `irm https://cli.simetrik.com/install.ps1 | iex` (Windows). Verify `simetrik --version` before work.

## Critical Execution Rules

1. **Stop on error.** Non-zero exit / API 4xx/5xx / validation failure → stop, show full error, propose fix, wait for user.
2. **No workarounds on API errors.** Exception: `--wait` returning 500 on async commands — poll `resource describe` / `task status` for real state.
3. **`--wait` does NOT always wait.** On `recon run`, `recon confirm`, `adv-recon confirm` and `adv-recon execute` it returns instantly with exit 0 without waiting. Check the truth table in **`references/operations.md`** before trusting any `--wait`, and verify completion before dependent phases.
4. **Client-side polling.** Any per-process `FAILED`/`ERROR`/`ERRORED`/`CANCELLED` aborts the wait. A failed `--wait` is a real failure — but a **timeout is not**: the job keeps running, poll `task status <EVENT_ID>` instead of re-dispatching.
5. **Use `--format json` on every create command.**
6. **Pass `-y/--yes` on every destructive command when running non-interactively** — without a TTY the confirmation prompt reads EOF and cancels with exit 2.
7. **Discover syntax via `simetrik <cmd> --help`.**
8. **A `list` result is a page, not the whole set — never read it as proof something doesn't exist.** `resource list` returns at most 50 rows (`--limit` default; max 200). A partial page now says so: `warnings` carries `Partial results: showing N of M …`. Two ways to get past it, in order of preference: **`resource list --search "<name>"`** when you know what you're looking for (filters server-side, reaches any page), or **`resource list --all`** to auto-page and return the complete set in one envelope (not combinable with `--offset`). `--all` can still come back short if the server stops returning rows early — check `warnings` for `Incomplete:` before treating the result as complete. Never conclude a resource is absent from a bare `resource list`.

## Prerequisites

```bash
simetrik --version       # confirm latest
simetrik config show     # confirm the profile in use
simetrik login           # authenticate if needed
```

Working against a specific profile? Carry it through every command in the run, prerequisites included — `config show` and `login` without it inspect and authenticate the *active* profile, not yours. Exporting `SIMETRIK_PROFILE` once covers the whole sequence:

```bash
export SIMETRIK_PROFILE=mx
simetrik config show     # now reports mx
simetrik login           # now authenticates mx
```

No profile in use → stop, ask user to log in. The profile in use is the one named by `--profile` or `SIMETRIK_PROFILE` when either is set — an unset `active_profile` is not a blocker then. Wrong version → run `simetrik update -y`.

**Working across workspaces.** One profile per workspace, selected per command — never by switching the active profile, which is shared state every other running command reads.

```bash
simetrik login --profile mx --workspace 42    # one session per workspace
simetrik --profile mx table list              # target it per command
SIMETRIK_PROFILE=mx simetrik table list       # same, for a whole shell
simetrik config remove-profile mx -y          # delete when done
```

Commands targeting different profiles are safe to run at the same time. An unknown profile fails with `PROFILE_NOT_FOUND` (exit 2) before any request is sent.

**Session tokens rotate mid-run.** For long sequences (>20 calls / >2 min), expect `AUTH_REQUIRED`. Recovery: `simetrik login` (interactive), resume from last persisted ID. Re-login at the start of each phase exceeding two minutes.

## Workspace

`workspace info` and `workspace settings` are read-only: the former is a resource-count overview, the latter surfaces the workspace's whitelisted feature settings (keyed by name, not by the feature's singular label — `segmentations` is plural) and its three edit restrictions. Neither subcommand writes; settings change through the platform's own configuration surfaces. Full contract, the exact whitelist names, and the restriction booleans in **`references/workspace.md`**.

## Read the user's intention FIRST

| User intent | Action |
|---|---|
| **Single command** ("rename this column", "create one segment") | Run only that command. Don't volunteer surrounding resources. |
| **Partial change** ("add a union segment", "add three columns") | Build only the dependency closure. Reuse existing IDs. |
| **Full plan / replication** | Follow phase guide below, ordered by the plan's dependency graph. |

## Resource dependency tree

When creating multiple resources, build bottom-up — each resource must exist before anything that references it. If a dependency already exists in the workspace, resolve its ID and reuse — don't recreate.

Before creating any resource, verify its dependencies exist and are in the expected state via `describe` / `column list`. Don't assume — a segment might be missing, a union might not be confirmed, or a column might still be `pending`.

```text
Native Sources
   ↓ segments and columns depend on sources
Segments on sources
Calculated Columns on sources
   ↓ unions reference source segments
Unions → confirm → columns on union → segments on union
   ↓ recons reference union/source segments
Advanced Reconciliations (see references/advanced-recon.md)
Legacy Reconciliations (see references/legacy-recon.md)

Independent (position by what they reference):
├── Consolidations → depend on segments from any resource
├── Opening Records → depend on a native/source_union segment
├── Resource Joins → depend on two segments + their columns
└── Data Exports → depend on any materialised resource
```

**Not every segment is a legal input for every builder.** A join only accepts reconciliations; opening records only accept a `native`/`source_union` segment; a consolidation only reads `source_group` and `cumulative_balance` origins. Some of those are hard gates that return a typed error — but others exist only in the web UI's picker and **never fire on the CLI path**, notably **union inputs** (meant to be `native`/`snapshot`/`opening_records` only). There you get no create-time type error at all; the build can still fail later at confirm or execution. Check `resource describe <ID>` → `resource_type` before wiring anything into anything. Full matrix, error codes, and which rules you have to enforce yourself in **`references/resource-dependencies.md`**.

Capture every ID from every create call. Recover lost IDs with `resource list/describe`, `column list`, `<feature> describe` — when recovering by name, reach for `resource list --search "<name>"` rather than scanning a bare `resource list`, which stops at 50 rows (rule 8).

---

## Native Sources

```bash
simetrik source create "<NAME>" \
  --description "<desc>" \
  --columns '[{"name":"COL","label":"COL"}, ...]' \
  --format json
```

- **The column schema decides the table** — passing `--columns` builds the physical table on create, omitting them defers it to the first uploaded file, which defines the schema. There is no flag: a table with no schema to build it from, and columns no table backs, are both unreachable.
- **Deferring only works while the created resource carries a single column.** The core infers the schema from the first file when it counts exactly one (`skt_id`); anything more reads as already configured, so it skips the inference and every later read fails with Snowflake `002003 (42S02) … does not exist or not authorized`. `column list` straight after a create without `--columns` is what tells the two apart — pass `--columns` if it returns more than `skt_id`.
- **Verify `table_created: true` in the create response before uploading anything.** It is the only place the truth appears. On `false` the table does not exist, yet `resource describe` still reports `current_status: 1` (READY) and `file upload` still returns `uploaded: 1, failed: 0, exitCode: 0` — signing, the S3 PUT and the registration never touch Snowflake, so the failure only surfaces later in the async worker.
- **A source with no physical table cannot be repaired.** `source update` only renames, and `column raw --create-table` does not build it either. Archive it with `delete resource` and recreate it with `--columns`.
- Two fields per column: `name`, `label`. Any `type`/`format`/`data_format` key is ignored — the CLI strips it. Set the type after create with `column cast`.
- **`name` must be `[A-Z0-9_]` only.** Normalize: uppercase, collapse `[^A-Z0-9]+` to `_`, strip leading/trailing `_`.
- **Transliterate accents in resource names defensively** — the API's name regex is stricter than its label regex. Rename later with `source update --name` / `union update --name` / `adv-recon update --name`.
- **Set column types after create** — creates are type-free, so columns materialise as `string` until cast. Casting is the standard way to set a type: check `column list` → `data_format`, then cast. For more than one column use `column cast-bulk` — a single all-or-nothing batch, far cheaper than one call per column (see *Calculated Columns*).
- Exclude system columns: `SKT_ID`, `FILENAME`, `EMPTY`, `Y`.
- Load data with `simetrik file upload <files...> -s <SOURCE_ID> --format json`. Full surface in **`references/file-uploads.md`**.

## Segments

A segment is a saved filtered view of a resource — a **Grupo conciliable**. Full lifecycle (`create` / `describe` / `update` / `delete`), ID discovery, and the delete-and-dependencies rules are in **`references/segments.md`**.

⚠️ **The `default_segment` every resource ships is a SYSTEM group and cannot be edited — never treat it as the resource's Grupo conciliable.** Anything pointing at it is stuck with "all rows" for good, and a resource's group turns immutable once something downstream consumes it. Create your own with `segment create`; when no business criterion is known yet, make your own catch-all (`SKT_ID IS NOT NULL`) so the filter stays yours to narrow.

**Always use structured JSON (`filter_sets` array)** for `--filters`. The expression form has parser bugs.

- `filter_sets[]` → implicit AND between sets.
- `rules[]` inside one set → joined by that set's `condition` (`AND`|`OR`).
- One AND-clause = one filter_set. OR-group = one filter_set with `"condition":"OR"`.

⚠️ **The tree is FLAT — exactly one level of filter_sets. NEVER nest `filter_sets` (or `segment_filter_sets`) inside a filter_set.** A nested wrapper is persisted as an extra intermediate set with no rules, and the web UI cannot read that shape: the Grupo conciliable stops rendering the columns of its filter, and selecting it fails server-side (500, `KeyError: 'rules'`). Nothing looks wrong from the CLI — the create succeeds and the filter still applies through the API — so the only way to catch it is the verification below.

Gotchas that bite:
- **`segment update --filters` replaces ALL filter_sets (total replacement).** Empty list → `EMPTY_FILTERS` (exit 2); response without `filters_replaced: true` → `FILTERS_NOT_APPLIED` (exit 1), filters NOT changed.
- **`describe` does NOT round-trip into `--filters`.** It returns the STORED form — values quoted (`'SALE'`), operators spelled out (`is not null`) — not the input form (`SALE`, `is_nnull`). Feeding its output straight back into `update --filters` silently changes what the filter matches. Translate first (table in `references/filter-operators.md`).
- **Verify every write with `segment describe --refresh`:** every set's `filter_sets` must come back empty (`[]` is the healthy shape; a populated one is the broken tree), and each rule must sit on the column you meant. `describe` responses are cached, so without `--refresh` you can read a stale tree and get a false OK.
- **Deletion lives in the `delete` group, not `segment`:** `simetrik delete segment <id>` (destructive — needs `--yes` in JSON mode).

Full operator vocabulary and encoding rules in **`references/filter-operators.md`**; full segment surface in **`references/segments.md`**.

## Calculated Columns

Five kinds: `transformation-formula`, `cast`, `uniqueness`, `vlookup`, `dropdown` (plus `comment`).

Five non-negotiable rules:
1. **One writer at a time — per resource AND across its lineage.** Two transformations on the same resource in parallel race and roll back silently. Across resources nothing locks at all, so you serialise it yourself: **two operations may only overlap when their complete dependency closures are disjoint** — no shared ancestor *and* no shared descendant. That covers both directions (mutating a parent while a consumer runs, and running a consumer while a parent is mid-write) and the sideways case (two sources feeding the same union share no ancestor, yet collide on it). Any overlap leaves downstream rows stuck `pending` with **both commands exiting 0**. Preflight with `graph dependencies <RESOURCE_ID>` + `resource describe` on every neighbour — that is a snapshot, not a lock, so back it with an agreed maintenance window and the lineage's schedules paused; full rule in **`references/calculated-columns.md`**.
2. **`--wait` on `transformation-formula` is a no-op.** Poll `column list --resource <ID> --refresh` until `status=ready`. Column disappears (`GONE`) → formula failed.
3. **Cast from the real data, never from an assumption.** `column cast` must match how the values are actually stored — inspect with `data query` first, then cast. Pass the true layout with `--parse-as '<pattern>'` (repeatable; the order is the parse priority, for `date`/`datetime`/`time`) or, for numbers, `--decimal-separator` / `--thousands-separator`. A mismatched cast parses silently wrong — worse, it returns `ok:true` and writes `null` in every row, so verify with `data query` instead of trusting the envelope. **The pattern is written, not picked from a list**: there is no catalogue to match against, and a literal character goes in single quotes — `--parse-as "yyyy-MM-dd'T'HH:mm:ss.S"` for an ISO timestamp. `--show-as` sets how the value is *displayed* and is independent of how it is read. **Cast before loading data** — the notations apply on ingest, so casting an already-loaded table does not re-parse what is stored. Pattern catalogue and antipatterns in **`references/calculated-columns.md`**.
4. **Casting several columns? Use `column cast-bulk`, not a loop.** One batch per resource instead of one call per column:

   ```bash
   simetrik column cast-bulk --resource-id <RESOURCE_ID> --spec casts.json
   # casts.json → {"casts": [
   #   {"column_id": 101, "data_format": "numeric", "decimal_separator": ",", "three_digit_separator": "."},
   #   {"column_id": 102, "data_format": "date", "source_formats": ["dd/MM/yyyy"], "display_format": "yyyy-MM-dd"}
   # ]}
   ```

   - **All-or-nothing.** One invalid cast rejects the whole batch with a 422 listing *every* problem, and nothing is applied — so a partial failure never leaves half the columns converted.
   - **Every `column_id` must belong to `--resource-id`**, and none may repeat.
   - **Each entry accepts only these keys** — anything else is rejected locally before the request: `column_id`, `data_format`, `source_formats`, `display_format`, `decimal_separator`, `three_digit_separator`, `decimal_scale`, `format_description_id`, `custom_format_id`.
   - Don't split a large batch into smaller ones "to be safe": the cost is the table scan, not the column count, so smaller batches do strictly more work and give up the all-or-nothing guarantee.
   - Rule 3 still applies to each entry — verify with `data query` before casting.
5. **Confirm a transformation column with `data query` — `ok:true` and `status:ready` do not mean it holds values.** A column whose inferred `data_format` is not `string` used to materialise with every row `null`, and the documented workaround was to re-apply the type as a separate `column cast`. **That is fixed**: the create now populates the column on its own, so the extra cast is no longer required — but it stays harmless, and it is still the repair for a column created before the fix. The create response also carries a `warnings` entry (`Column N was created but its values were NOT populated…`) when the server catches a failed write; read it, but do not treat an empty `warnings` as proof of success. Details in **`references/calculated-columns.md`**.

Recover a transformation's formula with `column describe <COLUMN_ID>` → `data.transformation_query`. Full syntax in **`references/calculated-columns.md`**.

## Unions

Full sequence in **`references/unions.md`**. High-level:

```text
create → add-source-cells-bulk → rename → confirm --wait → poll until ready
→ calculated columns → output segments
```

Critical gotchas:
- **Rename BEFORE confirm.** Confirm freezes Snowflake column names. Post-confirm rename breaks formulas and VLookups.
- **Inactive cells use `"origin_column": null` explicitly** (not omitted). Omission prunes the column at confirm.
- **`union_columns` in create response is in REVERSE order.** Trust `position` field.

### Operating a live union

Every command in this section needs an agent-api that serves the matching source-union
endpoint. Against an older one they all answer 404 — check before promising them.

Editing a confirmed union does NOT require putting it back in draft — there is no such
operation. `add-segment` and `update-cells` land as pending:

```text
add-segment / update-cells → discard   (revert)
                           → publish?  ⚠️ no CLI path today — see below

remove-segment             → immediate. discard does NOT undo it
```

`union confirm` is the same verb for both: on a draft union it is the first publish, on a live one
it publishes the pending changes. Check `resource list --search` → `status_edition` before and
after, or read `has_pending_changes` from `union describe`.

⚠️ **Publishing needs an agent-api that carries the publish branch.** Against an older one
`union confirm` refuses a confirmed union with `NOT_DRAFT: Source union is already confirmed.`,
`union run` does not publish either, and the only CLI exit is `discard` — so an edit staged with
`add-segment` or `update-cells` can only be committed from the webapp. Verify on the target
environment before staging edits you may not be able to finish.

| Command | What it does |
|---|---|
| `union confirm <id>` | Publish. On a live union it alters the table for new columns, dispatches the new segments and clears `ongoing_changes` |
| `union run <id> [--wait]` | Reprocess on demand. **Only on an empty union** — a union holding data answers `UNION_NOT_EMPTY`, because the worker refuses to reprocess it. Against an agent-api without that guard the call is accepted and the job fails instead, stranding the resource in `PROCESSING`. Also refuses when origin resources are emptying or all empty; `--force` overrides that gate, not the empty one |
| `union add-segment <id> --segment <sid>` | Attach a source |
| `union remove-segment <id> <union_segment_id> -y` | **Destructive.** Detach a source — purges its rows, immediately |
| `union set-triggers <id> --segments <csv>` | Multi-trigger. `--timezone` is required past one trigger |
| `union set-trigger <id> --segment <sid> --type union\|reconciliation [--on-inconsistency manual\|empty]` | Single trigger. `union` = run only with that source; `reconciliation` = run with any |
| `union update-cells <id> --segment\|--column <id> --cells <json>` | Remap origin columns of a segment already mapped on the union |
| `union discard <id> -y` | **Destructive.** Revert every pending change |

Gotchas:
- **`run` without `--wait` is a false green.** The dispatch answers `ok: true` / exit 0 and the async job can fail afterwards. Pass `--wait` (it exits 1 on a failed task) or poll `task wait` on the returned `task_event_id`.
- **`union confirm --wait` waits on a draft union** and exits 1 on a failed task. Publishing a live union is synchronous — it reports no `task_event_id` and there is nothing to poll. See the `--wait` truth table in `references/operations.md`.
- **`update-cells` needs one cell per raw union column**, not a subset. The `union_cell_id` of each one comes from `union describe` → `union_columns[].cells[]`.
- **`update-cells --column` only reaches pending columns.** A published column returns `NOT_DRAFT` — remap by `--segment` instead.
- **`set-triggers` replaces the whole trigger list.** Pass every trigger you want to keep — it is not additive. Omitting `--segments` leaves the existing list untouched, and the response reports the stored triggers, not what you sent.
- **`union_segment_id` is not `segment_id`.** The trigger, remove and remap commands take the union-segment id from `union describe`.

## Advanced Reconciliation

`create → column-select → segmentation × 2 → sweep-create × N → columns + output segments → confirm → execute`

- **Transformation columns** on the adv-recon `resource_id`: reference inherited columns by their dot-prefixed `label` (e.g. `A.amount`), NOT the internal `name` — whenever the label has a writable form, the physical name is rejected with `PHYSICAL_COLUMN_NAME`. Output segment filters take the same label form. The one exception — a label with no writable form (a dot with a multi-character base, e.g. `banco_bb.amount`) is still addressed by name — is in **`references/calculated-columns.md`**.
- **Editing a CONFIRMED adv-recon** needs a *version with changes*: a confirmed adv-recon is blocked for updates (`sweep-update`/`sweep-delete` → `VALIDATION_FAILED · Confirmed reconciliation/Sweep cannot be updated/deleted`). `version-create` opens a **draft copy** (returns `new_reconciliation_id`); edit the **draft's** sweeps, optionally `execute --new-version` to preview, then `confirm` (merge → swap) or `version-discard`. Steps that address the ORIGINAL id (`version-create`, `confirm`, `version-discard`) vs the DRAFT id (`sweep-update`, `sweep-delete`, `execute`) are the easiest thing to mix up. `describe` reports `status_edition` / `has_pending_version` / `new_version_id`. Re-segmenting after confirm now goes through this cycle — no need to recreate the adv-recon. Full cycle in **`references/advanced-recon.md` §7.6**.

Full surface — create, segmentation, sweeps (1:1 / M:N / compensation), confirm/execute polling — in **`references/advanced-recon.md`**.

### Manual reconciliation & manual compensation (adv-recon)

Three destructive/read subcommands that operate on **already-executed** rows, not on configuration. **There is no per-record undo** — always confirm the group list with `data query` before firing.

```bash
# Cross N side-A records against M side-B records
simetrik adv-recon manual <ADV_RECON_ID> \
  --groups '[{"selected_skt_ids_a":["a1","a2"],"selected_skt_ids_b":["b1"]}]' -y

# Compensate records WITHIN one source side (bucket A vs bucket B of the same side).
# First run: pass the settings flags to configure the side in the same command.
simetrik adv-recon compensate <ID> --side A \
  --criteria-column-a <COL> --criteria-column-b <COL> \
  --compensation-groups '[{"prefix_side":"A","name":"PAYMENT","criteria_column_id":789,"group_values":["PAYMENT"]},{"prefix_side":"B","name":"REFUND","criteria_column_id":789,"group_values":["REFUND"]}]' \
  --groups '[{"selected_skt_ids_a":["a1"],"selected_skt_ids_b":["a2"]}]' -y
simetrik adv-recon compensation-records <ID> --side A --bucket A      # find candidate skt_ids
# Once configured, omit the settings flags to reuse the stored settings:
simetrik adv-recon compensate <ID> --side A \
  --groups '[{"selected_skt_ids_a":["a1"],"selected_skt_ids_b":["a2"]}]' -y
```

- **`--groups` is sent as-is** as `groups_to_reconcile`. Only keys `selected_skt_ids_a` / `selected_skt_ids_b` are accepted, both non-empty arrays of string `skt_id`s. `--groups` and `--groups-file` **combine** (flag first, then file) — use the file when thousands of groups won't fit argv.
- **In `compensate`, the `a`/`b` suffixes name the two configured buckets of the SAME side (`--side A|B`), not the two sides of the reconciliation.** `--side` and `--bucket` are case-sensitive uppercase.
- **Compensation requires the side configured.** The settings flags of `compensate` (`--criteria-column-a/b`, `--tolerance`, `--data-validation`, `--operation`, `--compensation-groups`/`-file`) are **all-or-nothing** (a subset exits 2 naming what is missing) and **overwrite the whole configuration of the side** (exactly 2 buckets, one per `prefix_side`, both sharing the same `criteria_column_id`, ≤10 `group_values` each) — the confirmation prompt warns about the overwrite. Settings + execution are one atomic request: a rejected run also reverts the settings. Without settings flags, an unconfigured side returns `COMPENSATION_NOT_CONFIGURED` / `COMPENSATION_MISCONFIGURED` and the error names the settings flags that fix it.
- **`compensate --compensation-groups` ≠ `sweep-create --compensation-groups`.** The bucket shape here has no `source_side` and no `due_column_id`. `adv-recon compensation-create` is **deprecated** — it only patches the criteria of an already-configured side.
- **Optional `--criteria-*` on `manual` overwrite the configured comparison criteria** for that call: `--tolerance` (default 0), `--data-validation` (block when exceeded), `--operation absolute_difference|sum|subtraction`. All four require both `--criteria-column-a` and `--criteria-column-b`. Missing configured criteria surfaces as `CRITERIA_NOT_CONFIGURED`.
- **Async:** dispatch returns `data.task_event_id` (**not** `event_id` like `run`/`confirm`). Pass `-w/--wait` (`-t`, default 120 s) to poll; a failed task exits `1`.
- `compensation-records` is a paginated read (`--page`, `--page-size` clamped to 200, default `-f table`) and the only one of the family that is not destructive; its `skt_id`s feed `compensate --groups` directly.
- Destructive commands need `-y/--yes` in JSON output mode, else `CONFIRMATION_REQUIRED` (exit 2). Local validation errors (`MISSING_OPTION`, `INVALID_OPTION`, `INVALID_JSON`) exit 2 before any API call.
- **Reversal:** compensated (`OFFSET`) records are reverted per segment with `adv-recon decompensate` (**`references/decompensation.md`**).

## Legacy / Standard Reconciliations

`recon create → column-select → cross-create (per sweep) → confirm --wait`

Distinct from `adv-recon`. Column categories: inherited (auto from segments), `column chained` (user-created), `chained_*` system columns — three distinct things. Details in **`references/legacy-recon.md`**.

- **Editing a CONFIRMED recon** versions *in place* — there is **no cloned resource** (that is the advanced flow). A `ruleset create` on a confirmed recon lands as **pending** and changes nothing on its own — the recon keeps running the previous rulesets until `recon confirm`. `resource describe --format json` → `details.has_pending_changes` / `details.pending_ruleset_ids` shows what is staged; `recon run --preview` runs the draft table; `recon confirm` promotes it; `recon version-discard` drops it. **`ruleset create` returning `201` on a confirmed recon does not mean the results moved** — this is the single most common surprise. Full cycle in **`references/legacy-recon.md`**.

### Manual reconciliation (legacy)

```bash
simetrik recon manual <RECON_ID> --mode one_to_one \
  --groups '[{"id_a":"a1","id_b":"b1"}]' -y
simetrik recon manual <RECON_ID> --mode batch \
  --groups '[{"id_a":["a1","a2"],"id_b":["b1"]},{"segment_id_a":253,"id_b":["b9"]}]' -y
```

Same async/confirmation/exit-code semantics as `adv-recon manual` (`task_event_id`, `-w/--wait`, `-y` required in JSON mode, no per-record undo), with these legacy-only differences:

- **`--mode` is required**: `one_to_one` (groups of scalar `id_a`/`id_b`) or `batch` (each side is an id **list** `id_a`/`id_b` **or** a whole `segment_id_a`/`segment_id_b` — never both on the same side).
- The array goes on the wire as **`skt_id_groups`** (adv-recon uses `groups_to_reconcile`), and criteria uses `is_activate` instead of `data_validation`.
- **`--operation` values are `abs|sum|sub`** here (adv-recon uses `absolute_difference|sum|subtraction`).
- **`--groups-file` also accepts CSV** (`id_a,id_b` per line, header tolerated) as a `one_to_one` convenience; adv-recon accepts JSON only.
- `-c/--comment` max 300 chars (250 on adv-recon) — enforced by the API, not the CLI. `--tag` must be enabled for manual reconciliation.

## Operating a built reconciliation (day-2)

Everything above builds. This is what you do afterwards: run it, pair records by hand, track the job, schedule it, watch it.

| Goal | Command | Reference |
|---|---|---|
| Run / re-run | `recon run`, `adv-recon execute` (`--reprocess` to backfill) | **`references/operations.md`** |
| Edit a **confirmed** recon (version with changes) | adv: `adv-recon version-create` → edit draft → `confirm` / `version-discard`. legacy: `ruleset create` (lands pending) → `recon confirm` / `recon version-discard` | **`references/advanced-recon.md` §7.6**, **`references/legacy-recon.md`** |
| Operate a live union | `union run`, `add-segment`, `remove-segment`, `set-trigger(s)`, `update-cells`, `discard` | **`references/unions.md` §4h** |
| Pair records manually | `recon manual`, `adv-recon manual` | **`references/manual-reconciliation.md`** |
| Compensate records manually | `adv-recon compensate`, `adv-recon compensation-records` | **`references/advanced-recon.md`** |
| Unreconcile a segment | `recon unreconcile`, `adv-recon unreconcile` | **`references/unreconciliation.md`** |
| Decompensate a segment | `adv-recon decompensate` | **`references/decompensation.md`** |
| Bulk-write comment/dropdown cells | `data update-cells` (multi-column, synchronous, no `--wait`) | **`references/data-cells.md`** |
| Edit a source group's grain | `group add-column`, `add-value`, `remove-column`, `remove-value` (synchronous, no `--wait`) | **`references/source-groups.md`** |
| Track an async job | `task status <event_id>`, `task wait <event_id>` | **`references/operations.md`** |
| Recurring runs | `schedule create-*` / `update-*` / `list-*` / `delete` | **`references/scheduling.md`** |
| Events, alarms, approvals, snapshots | `notification listen`, `alarm`, `approval`, `snapshot` | **`references/monitoring.md`** |

Three rules that override intuition:

1. **`--wait` is a no-op on `recon run`, `recon confirm`, `adv-recon confirm` and `adv-recon execute`** — they read `data.event_id` while the BFF returns `task_event_id`, so the wait block is skipped and the command exits 0 immediately. Capture `data.task_event_id` and `task wait` on it, or poll `resource describe` (`current_status` 1=READY, 4=PROCESSING). Full truth table in `operations.md`.
2. **Manual reconciliation has no per-record undo.** The only reversal is unreconciling the whole segment — from the CLI with `recon unreconcile` / `adv-recon unreconcile` (`references/unreconciliation.md`); compensated (`OFFSET`) records are reverted per segment with `adv-recon decompensate` (`references/decompensation.md`). Both need `-y` in any non-interactive run, and `--criteria-*` **overwrites** the stored comparison criteria.
3. **A `--wait` timeout is not a failure.** The job is still running; poll `task status`. Re-dispatching starts a second job — or, in `recon manual`, creates contradictory pairings, since the simple flow does not check for duplicate or non-existent ids.

`recon manual` and `adv-recon manual` are **two different contracts**, not two flavours of one command (different group keys, mode flag, criteria rules, comment caps and server-side guards). Read the side-by-side table in `manual-reconciliation.md` before composing any payload.

## Annotating what you build

`--description` is the only metadata field the CLI writes on a resource. There are no resource tags — the `--tag` on `recon manual` / `adv-recon manual` tags the *manual operation*, not the resource, and won't show up as a label on the resource.

Available on `source`, `union`, `recon`, `adv-recon`, `join`, `group`, `consolidation`, `opening-records`, `dashboard`, `dataset`, `alarm`, `cumulative-balance`, and on individual columns via `column update --description` (max 250 chars). Pass `""` to clear one.

Don't confuse it with the three unrelated "comment" surfaces: `column comment` creates a column where each **row** gets its own text, `recon manual --comment` justifies one manual **match**, and `approval respond --comment` justifies an **approval**. None of them annotates a resource.

To **fill** the row-level texts of a comment (or dropdown) column in bulk — one value across a filter/segment, or per-cell values from a file — use `data update-cells` (**`references/data-cells.md`**). It writes comment/dropdown columns only; pending-management columns are out of its scope by design — referenced by name they fail locally with `INVALID_OPTION` (no write sent), and by raw `column_id` the server returns `COLUMN_NOT_EDITABLE`.

## Source Groups

Aggregated view of a segment (`group` family). The **grain** = dimensions (group-by columns) + aggregations (`(column, function)` values; **one aggregation per column, max**). Read it with `resource describe <resource_id>`; edit it additively — accumulative groups only — with `group add-column` / `add-value` / `remove-column` / `remove-value` (`--columns` is CSV ids, NOT JSON; `remove-value` takes the **parent column id**, not the aggregation row id; both `remove-*` are destructive and need `-y` non-interactively). 11 valid functions, `COUNT_DISTINCT` is not one of them. Full surface, id duality and the function/type matrix in **`references/source-groups.md`**.

## Consolidations

Grid resource (segment rows × custom columns). `consolidation create` provisions placeholder `COLUMN_NAME_5` — rename/retype via `update-column`, do NOT add + delete it. Full surface in **`references/consolidations.md`**.

## Opening Records

Attaches to `native`/`source_union` segment (NOT recon segments). Needs numeric criteria column. `--columns` required on create despite `--help`. Details in **`references/opening-records.md`**.

## Folders

Almost every build command takes `--folder <id>`. Discover the ID with `folder list` — folders are NOT resources, so they never show up in `resource list` and filtering that endpoint by a folder type returns an empty list rather than an error.

`folder create` always makes type `1` (source); the other five types exist because the web UI creates them, so a listing mixes both — read the type column. `--parent` matches direct children only. `folder delete` refuses non-empty folders; move resources out with `resource move-to-folder` first. Details in **`references/folders.md`**.

## Account

`account addons` reads the account's addon catalog, grouped by application because addon ids are
local to each application. The account is resolved server-side from the session token — there is
no id flag.

The list is a catalog, not an entitlement: a listed addon is not one this CLI (or any CLI) can
switch on — the only writer is the Django Admin. Details, including the three application states
and the unrecognized-ids row, in **`references/account.md`**.

## Resource Joins (Combinaciones)

Clause schema differs per endpoint: `join create` uses `column_a_id`/`column_b_id`; `configuration add/update` use `column_a`/`column_b`. CLI normalises both.

To add a relation to a combination that already has segments, submit the connected segment (the anchor) on either side: each side is matched by segment id — reused when already connected, created otherwise — and `--position-a/-b` are advisory. Clause columns must belong to the resource behind their own side's segment. No CLI command lists a combination's segments, configurations or clause ids; keep the ids from your `create`/`configuration add` responses. New schedules must be created via web UI. Details in **`references/resource-joins.md`**.

## Data Exports

Ship resource data to cloud storage (S3/GCS), customer API, or Snowflake data-share. Independent of recon pipeline.

**Connections are support-provisioned** — there is no CLI verb to create one, and the CLI requires at least one connection in the workspace before any export command works. Resolve a `connection.id` with `export list-connections` first; if it's empty, give the user that feedback — they need to request a connection from Simetrik support before the export can be built.

Key gotchas: `name` is `[a-zA-Z0-9 ]` only; the action is created *with* the export (the type is inferred from the connection); inspect with `export get <id>`; there is no `export delete` (disable instead); `custom_folder_path` / `partition_by` / `segment_id` are web-UI only; one export per `(workspace, resource_type, resource_id, topic)`. Each is spelled out in the reference.

When building an export interactively, walk the user through the **build checklist** in the reference and don't apply silent defaults without echoing them back.

Full surface — topic/event matrix, build checklist, column-selection, filters, file-format options, partition rules, API payloads, troubleshooting — in **`references/data-export.md`**.

## Accounting Models (Accounting Translator)

A separate surface from the reconciliation pipeline above: turn a reconciliation's data into ERP accounting entries. It is exposed as **four command groups** — start at `accounting schema` (the catalog) and consult it before composing any spec.

| Group | Owns | Reference |
|---|---|---|
| `accounting` | `schema` catalog + shared rules (flags, enums, envelopes) | **`references/accounting.md`** (hub) |
| `accounting-automation` | model lifecycle: `create`, `update-*`, `activate`/`delete`, `send-*`, `list`/`describe`, `currencies`, `entries` | **`references/accounting-automation.md`** |
| `accounting-integrations` | `list-connections` + `payload {list,schema,create,update,delete,link,erp-names}` | **`references/accounting-integrations.md`** |
| `accounting-entry` | `entry` detail + entry mutations (`discard-/resend-/cancel-/approve-cancel-/reject-cancel-entries`) | **`references/accounting-entry.md`** |

```bash
simetrik accounting schema                                          # self-describing catalog — consult FIRST, never guess
simetrik accounting schema --section create --example > spec.json
simetrik accounting-automation create --spec spec.json --dry-run    # validate without persisting
simetrik accounting-automation create --spec spec.json --wait --id-only
simetrik accounting-integrations payload --help                     # accounting payload subcommands
```

Key gotchas:
- **`accounting schema` is the source of truth.** Pull required fields, enums, domain rules and error codes from `accounting schema --section <s>` before composing any spec — never guess a field or enum.
- **`reconciliation_id` ≠ `resource_id`.** The spec needs the reconciliation's `id` (`recon describe` → `data.id`), NOT the `recon list` id (which is the `resource_id`). This is the #1 create failure.
- **Lifecycle gates everything:** `PRELIMINARY → DRAFT → (send-to-production) → PRODUCTION`; `is_active` is orthogonal. `update-advanced` is DRAFT-only; PRODUCTION blocks renames/FIXED edits. Deactivate before delete.
- **`create`/`update-advanced` are async** — pass `--wait` (`--timeout`, default 600s). Use `--dry-run` on create; all domain errors accumulate, fix them in one pass.
- **Group quirk:** `entries` (list-many) is in `accounting-automation`; `entry` (one) + mutations are in `accounting-entry`. Don't "fix" it.
- **Payloads** (`accounting-integrations payload …`) are a scaffold → adjust `origin` → create flow; `payload link --check` validates field/section compatibility before committing.

Start in **`references/accounting.md`** (the hub: 4-group index, schema catalog, enums, envelopes, flags, when-in-doubt); each group has its own reference linked above.

## Operation Center (Datasets & Dashboards)

The analytics layer on top of the reconciliation pipeline: turn resource data into **datasets** (saved, materialised SQL queries) and chart them on **dashboards**. Two independent command groups.

| Group | Owns | Reference |
|---|---|---|
| `dataset` | saved SQL over graph resources: create / version / materialize / refresh / schedule / preview / lineage | **`references/dataset.md`** |
| `dashboard` | dashboard → pages → visuals (charts bound to datasets) + access control + **contexts** (free-text docs on a dashboard/page/chart via `context-set/get/list/clear/export/import`) | **`references/dashboard.md`** |

```bash
simetrik dataset list-resources                         # graph resources you can query
simetrik dataset create "<name>" --query "<sql>" --id-only
simetrik dataset materialize <id> --wait                # async (202) → polls status
simetrik dashboard chart-types-list                     # valid visual --type values
simetrik dashboard visual-create <dash> <page> --type bar --title "<t>" --dataset <datasetId>
```

Key gotchas:
- **Column scope is the user's call, not yours.** For a dataset over a resource, ask *"100% of the columns or a subset?"* — default to all columns via `dataset create-from-resource <resourceId> --name <n>` (backend `SELECT *`; one complete dataset feeds N visuals + global filters). Never trim columns to "optimize" the query on your own (see `references/dataset.md`).
- **Materialization is async.** `materialize` enqueues the run; pass `--wait` (polls `dataset status` ~2 s to `completed`/`failed`). `refresh --ids` is capped at 50.
- **`preview` uses `--version-id`** (not `--version`, to dodge the global `-V`) and it is required. `describe --columns` needs the dataset materialised first.
- **Visuals bind a dataset UUID + a chart type from `chart-types-list`.** `visual-update` forwards a `--type` change but the backend may not re-type an existing chart (delete + recreate if needed). `--dataset` takes the id, not a name.
- **Dashboard access** (`access-set` / `access-make-private|public`) needs the `oc:manage_access` permission (else 403); `clone` hard-codes the copy name.

Full surface in **`references/dataset.md`** and **`references/dashboard.md`**.

## Reading Resource Data — Prepare the Scenario First

**`data query` is the illustration tool, not the exploration tool.** It returns a **page** — 50 rows unfiltered, ≤ 200 filtered, offset ceiling 10 000 — ordered by `SKT_CREATED_AT DESC`, so it is the most recent slice, never a representative sample. Explore with aggregates; illustrate with rows.

### The rule

> **No claim of absence, completeness or coverage without `data profile` or `data aggregate` in the same run. And the finding must cite the field that backs it.**

Not acceptable: *"the union contains only one of the two settlement sources"*.
Acceptable: *"the union contains only LIQUIDACION_A — `lineage.inputs_without_rows: ["LIQUIDACION_B"]`, `matched_rows: 271 of 271`"*.

A rule that leaves no trace is a suggestion. **If you cannot cite the field, you cannot make the claim.**

### Order of operations

1. **`data profile <RESOURCE_ID>`** — always first. No options needed. One call returns row volume, time range, per-origin counts, nulls in matching keys, and the resource's declared inputs with the rows each contributes. This is the scenario.
2. **`data aggregate <RESOURCE_ID> --group-by <col>`** — only when the profile shows something that needs cutting by another dimension. `--metrics` defaults to `COUNT(*) as rows`, so a breakdown needs nothing else. Reads 100% of matching rows; **no offset ceiling applies**, because nothing is paged.
3. **`data query`** — last, to show concrete example rows of something already identified.

### Reading a profile

| Field | What it settles |
|---|---|
| `matched_rows` / `resource_rows` | The share your numbers cover. "271 of 271" and "271 of 4,000,000" support opposite conclusions — always read both. |
| `time_range` | Tells "no rows exist" apart from "no rows in this window". |
| `breakdowns[]` | Row counts per structural column. An **empty list** means the resource type has no such column — never "nothing to report". |
| `lineage.inputs_without_rows: []` | Checked: every declared input has rows. **No absence to report.** |
| `lineage.inputs_without_rows: ["X"]` | X is declared but contributes zero rows **in this window** — not "X is not part of the resource". |
| `lineage.available: false` + `inputs_without_rows: null` | The lineage could **not be determined**. This supports no absence claim at all. Never read a `null` as an empty list. |
| `truncated: true` (aggregate) | Groups exist that are not in `groups`. What you received is not the set that exists. |

### What never supports an absence claim

- A page of `data query`, at any `--page-size`. A bigger page of a `SKT_CREATED_AT DESC` slice is a bigger biased sample, not better evidence.
- An empty `breakdowns` list.
- `lineage.available: false`, or `inputs_without_rows: null`.
- A `warnings` entry on stderr means the read was capped — the rows on screen are a slice.

**Composition is structural, not statistical.** "Which sources compose this union" is answered by `lineage` (or `graph dependencies` / `union describe`), never by counting rows: a source configured in the union but with no rows in the period is indistinguishable, from the rows alone, from a source that is absent.

Full surface in **`references/pre-query.md`**.

### Row-level reads

`simetrik data query <resource_id>` reads the **actual rows** inside any resource (source, union, recon, consolidation, …) — distinct from `resource describe` (schema, not rows).

**Query by Grupo conciliable (segment) — the day-to-day pattern.** A segment is a saved filtered view of a resource = a Grupo conciliable. Discover its ID with `segment list <RESOURCE_ID> --format json` (add `--no-defaults` to hide the platform's own), then:

```bash
simetrik data query <RESOURCE_ID> --segment <SEGMENT_ID> --format json
```

- **A segment lifts the 50-row unfiltered cap** — without any `--filter`/`--filter-json`/`--segment` a query returns at most 50 rows. Passing `--segment` (or a filter) returns the whole group, paged (≤ 200/page, offset ≤ 10,000).
- **Stack** `--segment` with `--filter` to narrow within the group (AND-combined).
- **`-f` is `--filter`, NOT `--format`.** Use long `--format json`.

**Compare a resource against a local file (find discrepancies).** There is no built-in `compare` verb — build it on `data query`: pick the join key → query the matching Grupo conciliable to CSV/JSON (page through if needed) → normalize both sides → diff on the key → report three buckets: **only-in-resource**, **only-in-file**, **value-mismatch**.

Full command surface, hard limits, async model, the pagination loop, and the comparison recipe (with a `jq` diff) in **`references/data-query.md`**. Pre-query protocol in **`references/pre-query.md`**. Operator vocabulary in **`references/filter-operators.md`**.

## Error Handling

Full diagnostic tree in **`references/error-handling.md`**. Most common:

| Error | First action |
|---|---|
| `DUPLICATE_NAME` | `resource list --search "<name>"` |
| `NOT_FOUND` | `resource describe <id>` to confirm |
| `NOT_DRAFT` | Confirmed — never retry against the same one. adv-recon: `version-create` → edit the draft → `confirm`. legacy recon: `ruleset create` (lands pending) → `recon confirm`. union: edited live, no version (see error-handling.md) |
| `Confirmed reconciliation/Sweep … cannot be updated/deleted` | adv-recon is blocked for updates — `adv-recon version-create`, edit the draft, then `confirm`/`version-discard` |
| `401`/`AUTH_REQUIRED` | `simetrik login` |
| `PERMISSION_DENIED` | STOP — admin must grant scope |

**On ANY error:** stop, show full output, propose fix, wait for user.

## Reading Definition Files

MPA/definition JSON shapes and field-to-CLI mapping in **`references/json-schema-reference.md`**.

## Validation

After each phase: verify via `describe`/`column list`. Spot-check: `simetrik data query <ID> --page-size 5 --format json`.

## Reference index

- **`references/resource-dependencies.md`** — which resource types can feed which: the input-type matrix per builder, the hard gates vs. the product rules nothing enforces, and the preflight check
- **`references/workspace.md`** — `workspace info`/`settings`: the settings whitelist (exact stored names), the three restriction booleans, and the `warnings` array as the signal for a failed read
- **`references/folders.md`** — folder lifecycle: `list`/`create`/`update`/`delete`, ID discovery for `--folder`, the six folder types, direct-children `--parent`, empty-only deletion
- **`references/account.md`** — `account addons`: catalog-not-entitlement caveat, grouping by application, the three application states, unrecognized ids
- **`references/segments.md`** — segment (Grupo conciliable) lifecycle: `list`/`create`/`describe`/`update`/`delete`, ID discovery, filter replacement via `update --filters` (total replacement), deletion + dependency cautions
- **`references/data-query.md`** — `data query` command surface, hard limits (50-row unfiltered cap, 200/page, 10k offset), async model, querying by Grupo conciliable (segment), and the compare-against-local-file recipe
- **`references/filter-operators.md`** — segment filters, `data query` parser, operator vocabulary
- **`references/calculated-columns.md`** — formula syntax, `--language`, dependency ordering, cross-resource concurrency (the neighbour rule), all column types
- **`references/unions.md`** — two-phase creation, bulk-cell payload, rename-before-confirm, operating a live union
- **`references/advanced-recon.md`** — full lifecycle: column-select, segmentation, sweeps, compensation, confirm/execute; `manual`, `compensate`, `compensation-records`
- **`references/legacy-recon.md`** — standard recon flow, column categories, limitations, `recon manual` (one_to_one / batch)
- **`references/operations.md`** — day-2 hub: the `--wait` truth table (four commands that don't wait), run/re-run semantics, `task status`/`wait`, reading run state, what is irreversible
- **`references/manual-reconciliation.md`** — `recon manual` vs `adv-recon manual`: the two contracts side by side, finding the skt_ids, `--groups-file` bulk runs, criteria & tolerance, every rejection code
- **`references/unreconciliation.md`** — `recon unreconcile` / `adv-recon unreconcile`: segment-level undo, `--all`, the draft-segment (`segment create --draft`) workflow, preview guards, lock choice
- **`references/decompensation.md`** — `adv-recon decompensate`: reverting compensated (`OFFSET`) records per segment
- **`references/scheduling.md`** — `schedule` group: coverage matrix, the four spellings of "when", `create-join` gap
- **`references/monitoring.md`** — `task`, `notification listen`, `alarm` (exact `--params` schemas), `approval`, `snapshot`
- **`references/source-groups.md`** — source group grain: dimensions vs aggregations, the two kinds of id, additive editing (`add-column`/`add-value`/`remove-column`/`remove-value`), the 11 functions and their type matrix
- **`references/consolidations.md`** — placeholder column, command map, naming rules
- **`references/opening-records.md`** — constraints, criteria columns, delete quirks
- **`references/resource-joins.md`** — Combinaciones lifecycle, clause schema per endpoint
- **`references/data-export.md`** — end-to-end: topics, events, columns, filters, file formats, troubleshooting
- **`references/file-uploads.md`** — `file upload` orchestrator, manual flow, skip codes, exit codes
- **`references/accounting.md`** — Accounting Translator **hub** (`accounting` group): 4-group index, `schema` catalog, enums, response envelopes, flags & `-f` defaults, when-in-doubt
- **`references/accounting-automation.md`** — `accounting-automation`: model lifecycle, `create` spec & `fields` model, validation codes, edit/update, `send-*`, `entries`
- **`references/accounting-integrations.md`** — `accounting-integrations`: `list-connections` + accounting `payload` scaffold/adjust/create/link
- **`references/accounting-entry.md`** — `accounting-entry`: single-entry detail + entry mutations and their state machine
- **`references/dataset.md`** — Operation Center `dataset`: saved SQL, materialize/refresh (async), versions, schedules, preview, lineage
- **`references/dashboard.md`** — Operation Center `dashboard`: pages → visuals, chart types, `visual-historical`, access control, clone, contexts (`context-set/get/list/clear/export/import`)
- **`references/error-handling.md`** — full error reference, async errors, permission triage
- **`references/json-schema-reference.md`** — MPA/definition JSON shapes, field-to-CLI mapping

Internal surface — hidden from profiles that cannot run it, and denied by the
server regardless of what the CLI shows. Read these only when working on it:

- **`references/internal/marketplace-templates.md`** — `template marketplace`: catalog reads, publish/edit payload rules, `--regenerate` ordering
- **`references/internal/smart-rules.md`** — `smart-rules suggest`: agent-only, the two exclusive modes, segments accepted but inert
