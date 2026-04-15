# CLAUDE.md

This file provides guidance to Claude Code (and other AI assistants) when working with this repository.

## Repository Overview

This repository is a **Cribl Pack** (`cc-Tanium`, currently version `0.7.0`) that normalizes, enriches, and reduces Tanium Connect data flowing through Cribl Stream / LogStream. It is *not* application source code — it contains declarative YAML configuration (pipelines, routes, lookups) and sample data that Cribl imports and executes.

- Primary purpose: ingest Tanium Connect job output, route it by data type (Asset, Discover, Software Usage, Threat Response, Comply, Patch, Client Status, Deploy), normalize fields (Splunk CIM / ECS friendly names), enrich with site/approval lookups, and reduce volume.
- Distribution: installed into a Cribl Stream worker group as a Pack (zip/tar imported via the Cribl UI or API). There is no build step, no compiler, no test runner.
- License: Apache 2.0.

## Repository Layout

```
.
├── LICENSE
├── README.md                  # User-facing docs, release notes, pipeline table
├── CLAUDE.md                  # This file (AI assistant guidance)
├── package.json               # Pack manifest (name, version, author, tags)
├── default/
│   ├── samples.yml            # Index of sample log files (in data/samples/)
│   └── pipelines/
│       ├── route.yml          # Routes definition (filters -> pipelines, grouped)
│       ├── TaniumAsset/conf.yml
│       ├── TaniumDiscover/conf.yml
│       ├── TaniumSoftwareUsage/conf.yml
│       ├── TaniumSplunkApp/conf.yml
│       ├── TaniumThreatResponse/conf.yml
│       ├── TaniumComply/conf.yml
│       ├── TaniumPatch/conf.yml
│       ├── TaniumClientStatus/conf.yml
│       └── TaniumDeploy/conf.yml
└── data/
    ├── lookups/
    │   ├── application_approval_status.csv   # md5/name -> approved (seeded)
    │   ├── application_approval_status.yml
    │   ├── locations.csv                     # CIDR subnet -> site/region/country/tz
    │   └── locations.yml
    └── samples/
        ├── gx6NAu.json   # TaniumAsset
        ├── 8ssZ1E.json   # TaniumDiscover
        ├── qpvUkx.json   # TaniumSoftwareUsage
        ├── tR3sP1.json   # TaniumThreatResponse
        ├── c0MpLy.json   # TaniumComply
        ├── pAt2Ch.json   # TaniumPatch
        ├── cL1SrV.json   # TaniumClientStatus
        └── dEp10y.json   # TaniumDeploy
```

### Key conventions

- **Pipeline directory name == pipeline ID** referenced from `default/pipelines/route.yml` (`pipeline: TaniumAsset` matches `default/pipelines/TaniumAsset/conf.yml`).
- **Routes are grouped** in `route.yml` under three groups: `gLshfU` (Space Saving — replaces SplunkApp), `sEcGrP` (Security — Threat Response, Comply), `oPsGrP` (Operations — Patch, Client Status, Deploy). The `Tanium-SplunkApp` and `default` routes sit outside any group.
- **Sample files** live in `data/samples/<id>.json` and are registered in `default/samples.yml`. The id is the filename without extension and serves as the map key. `isPackOnly: true` keeps them scoped to this pack.
- **Lookup files** are CSV under `data/lookups/`, each with a companion `.yml` containing `size` (bytes) and `rows` (data rows excluding header) metadata. Update both when changing CSV contents.

## Pipelines

Each pipeline's `conf.yml` is a Cribl pipeline definition with an ordered `functions:` array. Common function IDs used here: `comment`, `eval`, `serde`, `rename`, `lookup`, `auto_timestamp`, `regex_extract`, `mask`, `drop`. Disabled functions use `disabled: true`; enabled or default use `disabled: null` / `disabled: false`.

### Original (0.6.x) pipelines

| Pipeline | Purpose | Route filter |
|---|---|---|
| `TaniumAsset` | Sets sourcetype `tanium:asset:allassets`, JSON-extracts `_raw`, renames `ci_item_*` to friendly names. **Now also** enriches with `site`/`region`/`country`/`timezone` via CIDR lookup against `locations.csv` (when `ip_address` is present) and exposes a disabled-by-default username `mask` function. | `/^.*ci_item_serial_number*/.test(_raw)` |
| `TaniumDiscover` | Splits managed/unmanageable via two `eval` filters; reserializes JSON dropping null/empty fields. | `/^.*MacAddress.*"MacOrganization.*Profile/.test(_raw)` |
| `TaniumSoftwareUsage` | Sets sourcetype `tanium:asset:softwareusage`; renames `ci_siu_*` to friendly names. | `/^.*ci_siu.*/.test(_raw)` |
| `TaniumSplunkApp` | Alternative pipeline preserving Tanium Splunk App field names; includes optional `auto_timestamp` (disabled by default, for syslog senders) and a `lookup` against `application_approval_status.csv` by `MD5_Hash`. | `index=='tanium'` (catchall) |

### New (0.7.0) pipelines

| Pipeline | Purpose | Default index / sourcetype | Route filter |
|---|---|---|---|
| `TaniumThreatResponse` | EDR / Threat Response process, file, and network events. Maps to Splunk CIM Endpoint / Elastic ECS (`dest`, `user`, `process_name`, `process_hash_sha256`, `src_ip`, `dest_ip` ...). Auto-timestamps from `_raw.timestamp`. Optional username mask. | `tanium_security` / `tanium:threatresponse` | `/process_sha256\|process_md5\|process_command_line/` |
| `TaniumComply` | Vulnerability scan results. Renames common fields to CIM Vulnerabilities (`dest`, `cve`, `cvss`, `severity`, `signature`); falls back to a `regex_extract` for `CVE-YYYY-NNNN`. Lower-cases `severity`. | `tanium_security` / `tanium:comply:vulnerability` | `/CVE-\d{4}-\d+\|cvss_score\|vulnerability_severity/` |
| `TaniumPatch` | Patch installation/applicability. Maps to CIM Updates (`dest`, `signature`, `kb`, `status`, `vendor_product`). Includes a disabled-by-default `drop` of `Installed`+`NotApplicable` noise. | `tanium_ops` / `tanium:patch:status` | `/patch_bulletin\|kb_article\|patch_status/` |
| `TaniumClientStatus` | Tanium agent health. Renames `tanium_client_version`, `last_registration`, `operating_system`, then `eval`s `hours_since_registration` for downstream alerting on stale agents. | `tanium_ops` / `tanium:client:status` | `/tanium_client_version\|last_registration/` |
| `TaniumDeploy` | Software deployment status events. Renames to `dest`, `deployment`, `package`, `status`, `start_time`/`end_time`. Optional disabled-by-default `drop` of succeeded deployments older than 7 days. | `tanium_ops` / `tanium:deploy:status` | `/deployment_name\|deployment_status\|package_version/` |

### Route groups

`route.yml` defines three groups:

- `gLshfU` **Space Saving** — `TaniumAsset`, `TaniumDiscover`, `TaniumSoftwareUsage`. Mutually exclusive with the `Tanium-SplunkApp` route (enabling both will double-process). A `comments:` warning in `route.yml` calls this out.
- `sEcGrP` **Security** — `TaniumThreatResponse`, `TaniumComply`. Default to `index=tanium_security`.
- `oPsGrP` **Operations** — `TaniumPatch`, `TaniumClientStatus`, `TaniumDeploy`. Default to `index=tanium_ops`.

The `default` catch-all route must remain last.

## Making Changes

There is no build / test / lint tooling here. Editing is purely YAML/CSV/JSON.

### Adding or modifying a pipeline
1. Edit the `default/pipelines/<Name>/conf.yml` file. Preserve the top-level keys: `output: default`, `groups: {}`, `asyncFuncTimeout: 1000`, `functions:`, `description`, `streamtags`.
2. When adding routes, add a new entry to `default/pipelines/route.yml` with a unique `id` (short alphanumeric matching existing style: `ZNdnXv`, `wPqR7z`, ...), a descriptive `filter`, `pipeline: <matching-directory-name>`, and an optional `groupId`.
3. The `default` catch-all route must remain last.

### Adding a lookup
1. Put the CSV under `data/lookups/<name>.csv`.
2. Create `data/lookups/<name>.yml` with `size: <bytes>` and `rows: <data-row-count>` (excluding header). Use `wc -c` to get the byte size.
3. Reference via a `lookup` function with `file: <name>.csv`.

### Adding sample data
1. Save the file at `data/samples/<id>.json` (id is arbitrary but unique; existing ids are short alphanumeric like `tR3sP1`, `c0MpLy`).
2. Sample format: a JSON array of Cribl event objects. At minimum each event needs `_raw` (string). Optionally include `sourcetype`, `index`, `_time`, etc.
3. Register it in `default/samples.yml` with `sampleName`, `size`, `numEvents`, `created` (ms epoch), and `isPackOnly: true`.

### Version bump
- Update `version` in `package.json` (and the `name` suffix `cc-Tanium_<version>` if retaining the versioned name convention).
- Add a dated entry to the `## Release Notes` section of `README.md`.
- Update the version number at the top of this file (`CLAUDE.md`).

## Git / Workflow Notes

- Primary branch: `main`. Feature work happens on topic branches.
- The repo has no CI, no hooks, no pre-commit config.
- `package.json` is a **Cribl Pack manifest**, not an npm project — do not introduce `dependencies`, `scripts`, lockfiles, or Node modules.

## Gotchas for AI Assistants

- **Do not invent field renames.** Tanium field names like `ci_item_serial_number`, `ci_siu_product_usage_name`, `MD5_Hash`, `process_sha256`, `cve_id`, `patch_bulletin` come from Tanium's schema or established CIM/ECS conventions — check existing `rename` blocks before suggesting new ones.
- **Route filters are order-sensitive** within Cribl Stream's route evaluation. Specific routes must precede the `Tanium-SplunkApp` catch-all, and `default` must remain last.
- **Regex filters use JavaScript syntax** (e.g. `/^.*ci_siu.*/.test(_raw)`).
- **`disabled: null` vs `disabled: false`.** Both behave as enabled in Cribl; match the surrounding style rather than normalizing globally (avoids noisy diffs).
- **`TaniumSplunkApp` route has no `groupId`** by design — don't add it to the Space Saving group, and don't enable it simultaneously with the Space Saving routes.
- **`TaniumSoftwareUsage` rename entries with leading spaces** (e.g. `" ci_item_operating_system"`) are intentional — Tanium emits those field names with leading whitespace. Do not "fix" them unless the source data format has demonstrably changed.
- **Leading-space columns in `application_approval_status.csv`** (`" MD5_Hash"`, `" approved"`) are similarly intentional and must match the lookup function's configured field names.
- **Security/Ops pipelines default to dedicated indexes** (`tanium_security`, `tanium_ops`) rather than the shared `tanium` index. If a deployment expects everything in `tanium`, change the `eval` at the top of the affected pipeline.
- **Mask functions are disabled by default.** They MD5-hash username fields. Only enable when sending to a lower-trust destination — they are not meant to be on globally.
- **Lookup metadata must match the CSV.** When you edit `*.csv`, also update `size` (bytes) and `rows` (data rows excluding header) in the companion `.yml`.
