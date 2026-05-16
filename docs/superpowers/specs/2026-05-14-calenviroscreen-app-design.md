# CalEnviroScreen Geo-Agent App — Design

**Date:** 2026-05-14
**Status:** Draft for review
**Working directory:** `/home/cboettig/Documents/github/boettiger-lab/calenviroscreen`
**Reference app:** `../tpl-ca` (`boettiger-lab/tpl-ca`)

## Goal

A focused geo-agent app for California environmental-justice analysis, centered on
CalEnviroScreen 5.0. Two primary audiences:

1. **EJ advocates and community groups** — explore pollution burden, find disadvantaged
   communities, support funding and grant applications.
2. **Policymakers and agency staff** — explore CES scores by district/region, target
   investments, report on equity outcomes.

The app reuses the geo-agent CDN pattern (no JavaScript to write) and the tpl-ca
deployment shape (nginx + initContainer cloning the repo, server-side LLM proxy).

## Architecture

Four content files at repo root drive everything; the geo-agent CDN renders the map
and chat UI.

```
calenviroscreen/
├── index.html              # Loads geo-agent CDN bundle; page title + headings only
├── layers-input.json       # Catalog, map view, welcome message, 16 layer configs
├── system-prompt.md        # Tight, CES-specific domain guidance for the LLM
└── k8s/
    ├── configmap.yaml      # client config.template.json + nginx.conf.template
    ├── deployment.yaml     # nginx pod + git-clone initContainer
    ├── service.yaml        # ClusterIP on port 80
    └── ingress.yaml        # HAProxy ingress at calenviroscreen.nrp-nautilus.io
```

Identical operationally to tpl-ca: same nginx-alpine image, same `open-llm-proxy-secrets`
Secret in the `biodiversity` namespace, same MCP server, same node-affinity rule that
avoids GPU nodes.

## Configuration

### `layers-input.json` — top-level

```json
{
  "catalog":     "https://s3-west.nrp-nautilus.io/public-data/stac/catalog.json",
  "titiler_url": "https://titiler.nrp-nautilus.io",
  "mcp_url":     "https://duckdb-mcp.nrp-nautilus.io/mcp",
  "auto_approve": true,
  "links": { "github": "https://github.com/boettiger-lab/calenviroscreen" },
  "view":  { "center": [-119.5, 37.5], "zoom": 6 },
  "welcome": { "message": "...", "examples": [...] },
  "collections": [...]
}
```

### Layer set — 16 layers, 4 groups

Defaults: only `calenviroscreen-5-0` and `census-2024-county` visible at load; all
others off. Groups `Environmental Justice` and `Political Boundaries` are
default-expanded; `Conservation & Equity Investments` and `Environmental Co-benefits`
are collapsed.

**Environmental Justice (expanded)**

| `collection_id`         | Display name                                  | Notes                                   |
|--------------------------|-----------------------------------------------|-----------------------------------------|
| `calenviroscreen-5-0`    | CalEnviroScreen 5.0                           | Composite `CIscore_Pctl` choropleth (yellow→red). The headline layer. |
| `ca-dac-eda-2023`        | CA Disadvantaged Communities (DWR 2023, tract) | Official DAC designation under CA Water Code §79505.5(a). Uses `dac-tract-pmtiles` asset; key field is `DAC23` (Y/N). |
| `svi-2022`               | CDC Social Vulnerability Index                | Tract and county sub-assets.            |
| `mappinginequality`      | HOLC Redlining Grades (1930s)                 | Historical context; HOLC cities only. Note id has no hyphens. Key field is `grade` (A/B/C/D). |

**Political Boundaries (expanded)**

| `collection_id`     | Display name              |
|----------------------|---------------------------|
| `census-2024-county` | CA Counties               |
| `census-2024-cd`     | Congressional Districts   |
| `census-2025-sldu`   | Senate Districts          |
| `census-2025-sldl`   | Assembly Districts        |

**Conservation & Equity Investments (collapsed)**

| `collection_id`                   | Display name                  |
|------------------------------------|-------------------------------|
| `conservation-almanac-2024-sites`  | TPL Conservation Almanac      |
| `landvote`                         | Conservation Ballot Measures  |
| `wcb-approved-projects`            | WCB Approved Projects         |
| `cpad-units-2025b`                 | CPAD Protected Areas          |

**Environmental Co-benefits (collapsed)**

| `collection_id`                  | Display name                 |
|-----------------------------------|------------------------------|
| `irrecoverable-carbon`            | Irrecoverable Carbon (2024)  |
| `calfire-2024-firep`              | Wildfire Perimeters          |
| `wetlands-nwi`                    | NWI Wetlands                 |
| `usfws-critical-habitat-final`    | USFWS Critical Habitat       |

#### Layer styling notes

- CES, SVI, and HOLC layers are choropleths styled on a single percentile/grade
  column with a yellow→red ramp; per the geo-agent gotcha, polygons use
  `fill-color` + `outline_style`, never `layer_type: "line"`.
- All layer style/tooltip configs are copied from tpl-ca where the layer
  already exists in tpl-ca; only the four `Political Boundaries` styles and
  the CES layer get small tweaks to make CES the visual focus.
- Each `collection_id` is **verified against the live STAC collection `id` field**
  before the config is written (mismatches silently break the layer — the #1
  geo-agent gotcha).

#### STAC verification (already done during planning)

Both net-new collections were inspected and confirmed:

- `ca-dac-eda-2023` — at `public-ca-dac/stac-collection.json`. Multiple pmtiles assets across DAC/EDA and Tract/Place/BlockGroup; the app uses `dac-tract-pmtiles`. Schema column for DAC flag: `DAC23` (values `Y`, `N`, `Data Not Available`).
- `mappinginequality` — at `public-mappinginequality/stac-collection.json`. Single `mappinginequality-pmtiles` asset. **Id contains no hyphens** (`mappinginequality`, not `mapping-inequality`).

The implementation plan re-verifies all 16 collection ids against the live catalog before the JSON is committed (geo-agent's #1 gotcha).

### `system-prompt.md` — content brief

Tight (~15-20 lines). Only includes guidance the agent will plausibly get wrong
without it. Per the geo-agent skill, **does not** repeat STAC metadata
(column lists, S3 paths, collection titles) — those are auto-injected.

Required content:

1. One-sentence role: California EJ geospatial analyst supporting advocates and
   policymakers.
2. California scope: filter to CA from the start; never return national/multi-state
   results as a stepping stone.
3. No geolocation: if user says "my county" or "near me," ask which county/district
   before querying.
4. DAC ≠ high CES: the official SB 535 / AB 1550 designation lives in the
   `ca-dac-eda-2023` layer. Don't conflate "high CES percentile" with "designated DAC."
5. CES 5.0 is **draft** (released January 2026 by OEHHA). Note draft status when
   citing scores.
6. Percentiles are within California — a 90th-percentile tract is burdened
   relative to other CA tracts, not nationally.
7. Indicator-specific questions ("asthma in Fresno") should query the indicator
   column (`Asthma_Pctl`), not just the composite.
8. Redlining caveat: `mapping-inequality` only covers historical HOLC-mapped cities
   (LA, SF, Oakland, San Diego, Sacramento, Stockton, Fresno). Absence from the layer
   does not mean absence of historical discrimination.
9. Ambiguous "most burdened" — could mean highest composite, highest single
   indicator, most population in high-percentile tracts, etc. Ask in one sentence
   before running long queries.

### `welcome.examples` — six prompts

1. *"Which census tracts in Alameda County are in the top 10% of CES burden, and which indicators drive the score?"*
2. *"Where do CalEnviroScreen high-burden tracts overlap historical redlining grades?"*
3. *"Which Assembly districts have the largest population living in SB 535 designated disadvantaged communities?"*
4. *"How does PM2.5 in Fresno County compare to Los Angeles County?"*
5. *"Show me Conservation Almanac sites within or adjacent to high-CES tracts — where has conservation investment reached burdened communities?"*
6. *"In Senate District 18, what share of residents live in tracts with both high pollution burden and high asthma rates?"*

### `index.html`

Copy of tpl-ca's `index.html` with three string substitutions:

- `<title>TPL California — Protected Lands Explorer</title>` → `<title>CalEnviroScreen — California Environmental Justice Explorer</title>`
- `<h3>TPL CA Data Assistant</h3>` → `<h3>CalEnviroScreen Assistant</h3>`
- input placeholder `"Ask about California protected lands…"` → `"Ask about pollution burden and environmental justice…"`

All CDN script/style URLs (geo-agent@v3.6.1, MapLibre, PMTiles, etc.) stay
identical to tpl-ca.

## Deployment

### Kubernetes manifests

Copy tpl-ca's four `k8s/*.yaml` files with a global rename `tpl-ca` → `calenviroscreen`,
plus:

- **Ingress host:** `calenviroscreen.nrp-nautilus.io`.
- **Init container `git clone`:** `https://github.com/boettiger-lab/calenviroscreen.git`.
- **Secret reference:** `open-llm-proxy-secrets` (unchanged — shared across apps).
- **MCP URL:** `https://duckdb-mcp.nrp-nautilus.io/mcp` (unchanged).
- **Node affinity:** existing rule that excludes GPU nodes (unchanged).
- **Resource requests:** existing 100m CPU / 128Mi memory (unchanged — same workload).

### Repository creation

The init container clones the GitHub repo at pod start, so the repo **must exist
on GitHub before `kubectl apply`**. The implementation plan creates the repo
explicitly with:

```bash
gh repo create boettiger-lab/calenviroscreen \
  --public \
  --description "CalEnviroScreen 5.0 geo-agent app for California environmental-justice analysis" \
  --source . --remote origin --push
```

Sequence: write files → `git init` + initial commit locally → `gh repo create`
(which pushes) → `kubectl apply -f k8s/`.

### Rollout verification

After apply:

1. `kubectl rollout status deployment/calenviroscreen -n biodiversity`
2. `kubectl get ingress calenviroscreen-ingress -n biodiversity` — confirm address.
3. Hit `https://calenviroscreen.nrp-nautilus.io/health` — expect `healthy`.
4. Load `https://calenviroscreen.nrp-nautilus.io/` in browser — confirm map renders,
   CES layer is visible, chat sends a test query and returns a response.

## Out of scope

- Per-indicator styled layers (PM2.5, asthma, etc.) — agent does this via SQL on
  the parquet asset.
- `docs.html` separate documentation page — tpl-ca has one but it's optional.
  Defer until a clear need arises.
- Auth / private deployment — public app, no gating.
- Custom JS — geo-agent CDN handles all rendering.
- Adding new STAC collections — all 16 layers already exist in the public-data
  catalog. No data work required.

## Risks and unknowns

1. **CES 5.0 is draft and may be republished** with different values during
   OEHHA finalization. System prompt instructs the agent to disclose draft status.
2. **GitHub repo name `calenviroscreen` may collide** with a pre-existing repo in
   the `boettiger-lab` org. Mitigation: check with `gh repo view boettiger-lab/calenviroscreen` before creating.
3. **Schema drift on the two net-new collections.** If OEHHA / DWR republish with
   renamed columns (e.g. `DAC23` → `DAC25`), the styled `match` expressions break
   silently. Tooltip fields would also need updating. Low likelihood within the
   current vintage, but a known-class failure mode.
