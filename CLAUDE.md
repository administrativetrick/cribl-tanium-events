# CLAUDE.md

This file provides guidance to Claude Code (and other AI assistants) when working with this repository.

## Repository Overview

This repository is a **Cribl Pack** (`cc-Tanium`, currently version `0.6.6`) that normalizes and reduces Tanium Connect data as it flows through Cribl Stream / LogStream. It is *not* application source code — it contains declarative YAML configuration (pipelines, routes, lookups) and sample data that Cribl imports and executes.

- Primary purpose: ingest Tanium Connect job output, route it by data type, rename/extract fields, and optionally reduce volume (~20% reduction on Tanium All Asset Reports).
- Distribution: installed into a Cribl Stream worker group as a Pack (zip/tar imported via the Cribl UI or API). There is no build step, no compiler, no test runner.
- License: Apache 2.0.

## Repository Layout

```
.
├── LICENSE
├── README.md                  # User-facing docs, release notes
├── package.json               # Pack manifest (name, version, author, tags)
├── default/
│   ├── samples.yml            # Index of sample log files (in data/samples/)
│   └── pipelines/
│       ├── route.yml          # Top-level Routes definition (filters -> pipelines)
│       ├── TaniumAsset/conf.yml
│       ├── TaniumDiscover/conf.yml
│       ├── TaniumSoftwareUsage/conf.yml
│       └── TaniumSplunkApp/conf.yml
└── data/
    ├── lookups/
    │   ├── application_approval_status.csv   # Enrichment lookup (MD5/name -> approved)
    │   ├── application_approval_status.yml   # Lookup metadata
    │   ├── locations.csv
    │   └── locations.yml
    └── samples/
        ├── gx6NAu.json        # TaniumAsset sample (per samples.yml)
        ├── 8ssZ1E.json        # TaniumDiscover sample
        └── qpvUkx.json        # TaniumSoftwareUsage sample
```

### Key conventions

- **Pipeline directory name == pipeline ID** referenced from `default/pipelines/route.yml` (`pipeline: TaniumAsset` matches `default/pipelines/TaniumAsset/conf.yml`).
- **All routes filter on `index=='tanium'`** — upstream Sources must set `index=tanium` (or the filters must be changed) for anything to match.
- **Sample files** live in `data/samples/<randomId>.json` and are registered in `default/samples.yml`. The random ID is the filename (without extension) and serves as the map key. `isPackOnly: true` keeps them scoped to this pack.
- **Lookup files** are CSV under `data/lookups/`, each with a companion `.yml` containing `size` and `rows` metadata.

## Pipelines

Each pipeline's `conf.yml` is a Cribl pipeline definition with an ordered `functions:` array. Common function IDs used here: `comment`, `eval`, `serde`, `rename`, `lookup`, `auto_timestamp`. Disabled functions use `disabled: true`; enabled or default use `disabled: null` / `disabled: false`.

| Pipeline | Purpose | Route filter |
|---|---|---|
| `TaniumAsset` | Sets sourcetype `tanium:asset:allassets`, JSON-extracts `_raw`, renames `ci_item_*` to friendly names (`hostname`, `serial_number`, `os_platform`, ...). | `/^.*ci_item_serial_number*/.test(_raw)` |
| `TaniumDiscover` | Splits managed/unmanageable via two `eval` filters; reserializes JSON dropping null/empty fields. | `/^.*MacAddress.*"MacOrganization.*Profile/.test(_raw)` |
| `TaniumSoftwareUsage` | Sets sourcetype `tanium:asset:softwareusage`; renames `ci_siu_*` to friendly names. | `/^.*ci_siu.*/.test(_raw)` |
| `TaniumSplunkApp` | Alternative pipeline preserving Tanium Splunk App field names; includes optional `auto_timestamp` (disabled by default, for syslog senders) and a `lookup` against `application_approval_status.csv` by `MD5_Hash`. | `index=='tanium'` (whole `tanium` index) |

### Route group

`route.yml` groups the three space-saving routes under `gLshfU` (name: `Space Saving`). The `Tanium-SplunkApp` route is outside that group. A comment warns that enabling the space-saving group breaks the Tanium Splunk App unless Splunk aliases are created — treat the space-saving group and the SplunkApp route as **mutually exclusive**.

## Making Changes

There is no build / test / lint tooling here. Editing is purely YAML/CSV.

### Adding or modifying a pipeline
1. Edit the `default/pipelines/<Name>/conf.yml` file. Preserve the `output: default`, `groups: {}`, `asyncFuncTimeout`, `functions:`, `description`, `streamtags` top-level keys.
2. When adding routes, add a new entry to `default/pipelines/route.yml` with a unique `id` (short random alphanumeric, matching existing style like `ZNdnXv`), a descriptive `filter`, and `pipeline: <matching-directory-name>`.
3. The `default` catch-all route must remain last.

### Adding a lookup
1. Put the CSV under `data/lookups/<name>.csv`.
2. Create `data/lookups/<name>.yml` with `size: <bytes>` and `rows: <data-row-count>` (excluding header).
3. Reference via a `lookup` function with `file: <name>.csv`.

### Adding sample data
1. Save the file at `data/samples/<randomId>.json` (follow the existing naming style; the ID is arbitrary but unique).
2. Register it in `default/samples.yml` with `sampleName`, `size`, `numEvents`, `created`, and `isPackOnly: true`.

### Version bump
- Update `version` in `package.json` (and the `name` suffix `cc-Tanium_<version>` if retaining the versioned name convention).
- Add a dated entry to the `## Release Notes` section of `README.md`.

## Git / Workflow Notes

- Primary branch: `main`. Feature work happens on topic branches.
- The repo has no CI, no hooks, no pre-commit config. Commits only need to touch the YAML/CSV/JSON/MD files.
- Do not introduce Node modules, build tooling, or lockfiles — `package.json` here is a Cribl Pack manifest, not an npm project (it has no `dependencies`, `scripts`, etc.).

## Gotchas for AI Assistants

- **Do not invent field renames.** Field names like `ci_item_serial_number`, `ci_siu_product_usage_name`, `MD5_Hash` come from Tanium's schema — check existing `rename` blocks before suggesting new ones.
- **Route filters are order-sensitive** within Cribl Stream's route evaluation. Placement in `route.yml` matters; preserve the existing order unless the change is explicitly about reordering.
- **Regex filters use JavaScript syntax.** `/^.*ci_siu.*/.test(_raw)` is JS, not PCRE-only.
- **`disabled: null` vs `disabled: false`.** Both behave as enabled in Cribl; match the surrounding style rather than normalizing globally (avoids noisy diffs).
- **`TaniumSplunkApp` route has no `groupId`** by design — don't add it to the Space Saving group.
- **`TaniumSoftwareUsage` rename entries with leading spaces** (e.g. `" ci_item_operating_system"`) are intentional — Tanium emits those field names with leading whitespace. Do not "fix" them unless the source data format has demonstrably changed.
- **Leading-space columns in `application_approval_status.csv`** (`" MD5_Hash"`, `" approved"`) are similarly intentional and must match the lookup function's configured field names.
