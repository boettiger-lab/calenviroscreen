# CalEnviroScreen Geo-Agent App Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build and deploy a static geo-agent app focused on CalEnviroScreen 5.0 for California environmental-justice analysis, served at `calenviroscreen.nrp-nautilus.io` from the `boettiger-lab/calenviroscreen` GitHub repo.

**Architecture:** Four content files (`index.html`, `layers-input.json`, `system-prompt.md`, `k8s/*.yaml`) at the repo root, plus a short README. No custom JavaScript — the geo-agent CDN bundle does the rendering. Kubernetes pod is nginx-alpine with an `initContainer` that clones the repo at pod start. The same shape as the working `tpl-ca` deployment, with only the four content files differing.

**Tech Stack:** Static HTML, JSON, Markdown; Kubernetes (HAProxy ingress in the `biodiversity` namespace on NRP Nautilus); `gh` CLI; `kubectl`.

**Reference design:** [docs/superpowers/specs/2026-05-14-calenviroscreen-app-design.md](../specs/2026-05-14-calenviroscreen-app-design.md)

**Reference app to copy patterns from:** `/home/cboettig/Documents/github/boettiger-lab/tpl-ca/`

---

## File Structure

All paths are relative to the working directory `/home/cboettig/Documents/github/boettiger-lab/calenviroscreen/`.

```
calenviroscreen/
├── .gitignore                                          # OS junk only
├── README.md                                           # One-paragraph description + URLs
├── index.html                                          # Page chrome; loads geo-agent CDN bundle
├── layers-input.json                                   # Catalog, view, welcome, 16 layer configs
├── system-prompt.md                                    # Tight CES-specific LLM guidance
├── k8s/
│   ├── configmap.yaml                                  # client config.template.json + nginx.conf.template
│   ├── deployment.yaml                                 # nginx + initContainer git-clone
│   ├── service.yaml                                    # ClusterIP :80
│   └── ingress.yaml                                    # haproxy → calenviroscreen.nrp-nautilus.io
└── docs/superpowers/                                   # (already exists)
    ├── specs/2026-05-14-calenviroscreen-app-design.md  # (already written)
    └── plans/2026-05-14-calenviroscreen-app.md         # (this file)
```

---

## Task 1: Initialize repo and commit existing spec/plan

**Files:**
- Create: `.gitignore`
- Create: `README.md`

- [ ] **Step 1: Verify working directory and existing spec are in place**

Run:
```bash
cd /home/cboettig/Documents/github/boettiger-lab/calenviroscreen
ls docs/superpowers/specs/2026-05-14-calenviroscreen-app-design.md
ls docs/superpowers/plans/2026-05-14-calenviroscreen-app.md
```
Expected: both files listed without error.

- [ ] **Step 2: Confirm no repo already exists upstream**

Run:
```bash
gh repo view boettiger-lab/calenviroscreen 2>&1 | head -3
```
Expected: `GraphQL: Could not resolve to a Repository` (repo does not yet exist). If it *does* exist, STOP and check with the user before proceeding — the plan's GitHub-creation step needs to be skipped, and the existing repo's contents need to be considered.

- [ ] **Step 3: Write `.gitignore`**

Write `/home/cboettig/Documents/github/boettiger-lab/calenviroscreen/.gitignore` with this content:

```
.DS_Store
*.swp
*.swo
*~
.vscode/
.idea/
```

- [ ] **Step 4: Write `README.md`**

Write `/home/cboettig/Documents/github/boettiger-lab/calenviroscreen/README.md` with this content:

```markdown
# CalEnviroScreen Geo-Agent

A focused geo-agent app for California environmental-justice analysis, centered on
[CalEnviroScreen 5.0 (Draft, January 2026)](https://oehha.ca.gov/calenviroscreen).

**Live app:** https://calenviroscreen.nrp-nautilus.io

The app combines a MapLibre map with an LLM chat assistant. The assistant can query
the underlying GeoParquet datasets via a DuckDB MCP server, so users can ask questions
in plain English about pollution burden, demographics, district-level CES summaries,
and the overlap between burdened communities and conservation investments.

Built on the [boettiger-lab/geo-agent](https://github.com/boettiger-lab/geo-agent)
framework. Configuration lives in three files: `layers-input.json`,
`system-prompt.md`, and `index.html`.
```

- [ ] **Step 5: Initialize git, stage, and commit**

Run:
```bash
git init -b main
git add .gitignore README.md docs/
git commit -m "init: design spec, implementation plan, README, gitignore"
```
Expected: one commit on `main` containing the spec, the plan, README, and .gitignore.

- [ ] **Step 6: Verify**

Run: `git log --oneline && git status`
Expected: one commit listed; working tree clean.

---

## Task 2: Verify STAC collection ids match the live catalog

`collection_id` mismatches silently break layers (geo-agent's #1 gotcha). This task
verifies all 16 layers — with extra attention to the two net-new collections
(`ca-dac-eda-2023` and `mappinginequality`) where ids and asset keys were already
checked during planning.

**Pre-validated facts (do not re-derive — these were inspected during planning):**

| Layer        | Live `id`            | pmtiles asset key             | Notes |
|--------------|----------------------|-------------------------------|-------|
| CA DAC       | `ca-dac-eda-2023`    | `dac-tract-pmtiles`           | DAC field is `DAC23` (values `Y`/`N`/`Data Not Available`); MHI is `MHI23`; population `Pop23`; geoid `GEOID20`. |
| Mapping Ineq.| `mappinginequality`  | `mappinginequality-pmtiles`   | **NOTE: no hyphens in id.** HOLC grade column is `grade` (A/B/C/D); there's also a pre-baked `fill` hex color column. |

**Files:** None changed. This task verifies facts before deployment.

- [ ] **Step 1: Verify all collection ids match**

Run:
```bash
python3 << 'EOF'
import json, urllib.request
checks = [
    ("calenviroscreen-5-0",            "https://s3-west.nrp-nautilus.io/public-calenviroscreen/stac-collection.json"),
    ("ca-dac-eda-2023",                "https://s3-west.nrp-nautilus.io/public-ca-dac/stac-collection.json"),
    ("svi-2022",                       "https://s3-west.nrp-nautilus.io/public-social-vulnerability/2022/stac-collection.json"),
    ("mappinginequality",              "https://s3-west.nrp-nautilus.io/public-mappinginequality/stac-collection.json"),
    ("census-2024-county",             "https://s3-west.nrp-nautilus.io/public-census/census-2024/county/stac-collection.json"),
    ("census-2024-cd",                 "https://s3-west.nrp-nautilus.io/public-census/census-2024/cd/stac-collection.json"),
    ("census-2025-sldu",               "https://s3-west.nrp-nautilus.io/public-census/census-2025/sldu/stac-collection.json"),
    ("census-2025-sldl",               "https://s3-west.nrp-nautilus.io/public-census/census-2025/sldl/stac-collection.json"),
    ("conservation-almanac-2024-sites","https://s3-west.nrp-nautilus.io/public-tpl/conservation-almanac-2024-sites/stac-collection.json"),
    ("landvote",                       "https://s3-west.nrp-nautilus.io/public-tpl/landvote/stac-collection.json"),
    ("wcb-approved-projects",          "https://s3-west.nrp-nautilus.io/public-tpl/wcb-approved-projects/stac-collection.json"),
    ("cpad-units-2025b",               "https://s3-west.nrp-nautilus.io/public-cpad/cpad-units-stac-collection.json"),
    ("irrecoverable-carbon",           "https://s3-west.nrp-nautilus.io/public-carbon/stac-collection.json"),
    ("calfire-2024-firep",             "https://s3-west.nrp-nautilus.io/public-fire/calfire-2024/firep/stac-collection.json"),
    ("wetlands-nwi",                   "https://s3-west.nrp-nautilus.io/public-wetlands/nwi/stac-collection.json"),
    ("usfws-critical-habitat-final",   "https://s3-west.nrp-nautilus.io/public-usfws/critical-habitat/final/stac-collection.json"),
]
fail = 0
for expected_id, url in checks:
    try:
        d = json.loads(urllib.request.urlopen(url, timeout=10).read())
        live = d.get('id')
        ok = live == expected_id
        print(f"{'OK ' if ok else 'BAD'}  expected={expected_id!r:35} live={live!r}")
        if not ok: fail += 1
    except Exception as e:
        print(f"ERR {expected_id}: {e}")
        fail += 1
print()
print(f"{fail} failures — halt and reconcile before writing layers-input.json" if fail else "All 16 collection_ids match live STAC ids.")
EOF
```
Expected: every line prints `OK` and the final summary prints `All 16 collection_ids match live STAC ids.` If any line shows `BAD`, halt and reconcile — the recorded `live=...` value is the one to use, not the planned `expected=...`.

- [ ] **Step 2: Verify the two net-new collections have their pmtiles assets**

Run:
```bash
python3 << 'EOF'
import json, urllib.request
checks = [
    ("https://s3-west.nrp-nautilus.io/public-ca-dac/stac-collection.json",        "dac-tract-pmtiles"),
    ("https://s3-west.nrp-nautilus.io/public-mappinginequality/stac-collection.json", "mappinginequality-pmtiles"),
]
for url, key in checks:
    d = json.loads(urllib.request.urlopen(url, timeout=10).read())
    a = d.get('assets', {}).get(key)
    print(f"{'OK ' if a else 'BAD'} {key}: {a.get('type') if a else 'MISSING'}")
EOF
```
Expected:
```
OK  dac-tract-pmtiles: application/vnd.pmtiles
OK  mappinginequality-pmtiles: application/vnd.pmtiles
```
If either is missing, halt — the corresponding layer block in Task 5 must be deleted from the JSON before saving.

---

## Task 3: Write `index.html`

**Files:**
- Create: `index.html`

Copy of tpl-ca's `index.html` with three string substitutions only.

- [ ] **Step 1: Write `index.html`**

Write `/home/cboettig/Documents/github/boettiger-lab/calenviroscreen/index.html` with this exact content:

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CalEnviroScreen — California Environmental Justice Explorer</title>

    <!--
        Geo-Chat core modules loaded from CDN.
        Pin to a tag (@v1.0.0) for production stability,
        or use @main for staging/development.
    -->
    <script type="importmap">
    {
        "imports": {
            "@modelcontextprotocol/sdk/client/index.js": "https://esm.sh/@modelcontextprotocol/sdk@1.12.0/client/index",
            "@modelcontextprotocol/sdk/client/streamableHttp.js": "https://esm.sh/@modelcontextprotocol/sdk@1.12.0/client/streamableHttp"
        }
    }
    </script>

    <!-- MapLibre GL JS -->
    <script src="https://unpkg.com/maplibre-gl@5.22.0/dist/maplibre-gl.js"></script>
    <link href="https://unpkg.com/maplibre-gl@5.22.0/dist/maplibre-gl.css" rel="stylesheet" />

    <!-- PMTiles protocol -->
    <script src="https://unpkg.com/pmtiles@3.0.7/dist/pmtiles.js"></script>

    <!-- h3-js (for hex grid overlay) -->
    <script src="https://unpkg.com/h3-js@4.1.0/dist/h3-js.umd.js"></script>

    <!-- Markdown rendering -->
    <script src="https://cdn.jsdelivr.net/npm/marked/marked.min.js"></script>

    <!-- Code highlighting -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/styles/github-dark.min.css">
    <script src="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/highlight.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/languages/sql.min.js"></script>

    <!-- Core styles from CDN -->
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/gh/boettiger-lab/geo-agent@v3.6.1/app/style.css">
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/gh/boettiger-lab/geo-agent@v3.6.1/app/chat.css">
</head>

<body>
    <!-- Map -->
    <div id="map"></div>

    <!-- Layer controls — generated by MapManager.generateMenu() -->
    <div id="menu"></div>


    <!-- Chat interface -->
    <div id="chat-container">
        <div id="chat-header">
            <h3>CalEnviroScreen Assistant</h3>
            <button id="chat-toggle" title="Toggle chat">−</button>
        </div>
        <div id="chat-messages"></div>
        <div id="chat-input-container">
            <input type="text" id="chat-input" placeholder="Ask about pollution burden and environmental justice…" autocomplete="off">
            <button id="chat-send">Send</button>
        </div>
        <div id="chat-footer">
            <select id="model-selector" title="Select model"></select>
        </div>
    </div>

    <!-- Boot from CDN -->
    <script type="module" src="https://cdn.jsdelivr.net/gh/boettiger-lab/geo-agent@v3.6.1/app/main.js"></script>
</body>

</html>
```

- [ ] **Step 2: Verify HTML is well-formed**

Run:
```bash
python3 -c "from html.parser import HTMLParser; HTMLParser().feed(open('/home/cboettig/Documents/github/boettiger-lab/calenviroscreen/index.html').read()); print('OK')"
```
Expected: `OK` (no exception).

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add index.html (CDN bootstrap, CES-specific page chrome)"
```

---

## Task 4: Write `system-prompt.md`

**Files:**
- Create: `system-prompt.md`

Tight prompt (~15-20 lines) covering only non-obvious guidance. STAC metadata (column lists, S3 paths, collection titles) is auto-injected — do not duplicate.

- [ ] **Step 1: Write `system-prompt.md`**

Write `/home/cboettig/Documents/github/boettiger-lab/calenviroscreen/system-prompt.md` with this exact content:

```markdown
# CalEnviroScreen Environmental Justice Analyst

You are a geospatial data analyst helping California environmental-justice advocates and policymakers explore pollution burden, social vulnerability, and the overlap between burdened communities and conservation investments.

**California scope.** Always filter to California. Never return intermediate national or multi-state results as a stepping stone.

**No geolocation.** You do not know where the user is. If a user says "my county," "near me," or similar, ask which county or district before querying.

## Interpreting CalEnviroScreen 5.0

- CES Score = Pollution Burden Score × Population Characteristics Score (both 0–10, product ~0–100). The percentile `CIscore_Pctl` is the standard handle.
- CES 5.0 is **draft** (released January 2026 by OEHHA). When citing specific scores, note draft status — final values may shift.
- Percentiles are **within California**, not national. A 90th-percentile tract is burdened relative to other CA tracts.
- For indicator-specific questions (e.g., "asthma in Fresno"), query the indicator column (`Asthma_Pctl`, `AirPM25_Pctl`, etc.), not just the composite.

## DAC ≠ high CES

The official "Disadvantaged Community" designation under SB 535 and AB 1550 — used to allocate Greenhouse Gas Reduction Fund dollars — lives in the `ca-dac-eda-2023` layer. Don't conflate "high CES percentile" with "designated DAC." Use the dedicated layer when the user asks about official designations or funding eligibility.

## Redlining caveat

`mapping-inequality` covers historical HOLC-mapped cities only (LA, SF, Oakland, San Diego, Sacramento, Stockton, Fresno). Absence from the layer does not mean absence of historical discrimination.

## Ambiguous queries

"Most burdened" can mean highest composite, highest single indicator, most population in high-percentile tracts, or worst on a specific axis. Ask in one sentence before running long queries.
```

- [ ] **Step 2: Sanity-check the prompt is reasonable length**

Run:
```bash
wc -l /home/cboettig/Documents/github/boettiger-lab/calenviroscreen/system-prompt.md
```
Expected: 25–35 lines (target ~30 — tight but readable).

- [ ] **Step 3: Commit**

```bash
git add system-prompt.md
git commit -m "feat: add system prompt with CES-specific guidance"
```

---

## Task 5: Write `layers-input.json`

**Files:**
- Create: `layers-input.json`

This is the heart of the app. Encodes top-level config, view, welcome message,
and 14–16 layer configurations (depending on what Task 2 verified for the two
net-new collections).

The structure mirrors tpl-ca's. Layer styles for the 14 shared layers are copied
verbatim from `/home/cboettig/Documents/github/boettiger-lab/tpl-ca/layers-input.json`
to preserve behavior — only `group` names, ordering, and a few `visible` defaults
change. Two NEW layers (CA DAC, Mapping Inequality) use the exact ids/asset keys
verified in Task 2.

- [ ] **Step 1: Write `layers-input.json`**

Write `/home/cboettig/Documents/github/boettiger-lab/calenviroscreen/layers-input.json` with this exact content:

```json
{
    "catalog": "https://s3-west.nrp-nautilus.io/public-data/stac/catalog.json",
    "titiler_url": "https://titiler.nrp-nautilus.io",
    "mcp_url": "https://duckdb-mcp.nrp-nautilus.io/mcp",
    "auto_approve": true,
    "links": {
        "github": "https://github.com/boettiger-lab/calenviroscreen"
    },
    "view": {
        "center": [-119.5, 37.5],
        "zoom": 6
    },
    "welcome": {
        "message": "Hi! I can help you explore California environmental-justice data — pollution burden, social vulnerability, disadvantaged-community designations, and the overlap between burdened communities and conservation investment. Try asking:",
        "examples": [
            "Which census tracts in Alameda County are in the top 10% of CES burden, and which indicators drive the score?",
            "Where do CalEnviroScreen high-burden tracts overlap historical redlining grades?",
            "Which Assembly districts have the largest population living in SB 535 designated disadvantaged communities?",
            "How does PM2.5 in Fresno County compare to Los Angeles County?",
            "Show me Conservation Almanac sites within or adjacent to high-CES tracts — where has conservation investment reached burdened communities?",
            "In Senate District 18, what share of residents live in tracts with both high pollution burden and high asthma rates?"
        ]
    },
    "collections": [
        {
            "collection_id": "calenviroscreen-5-0",
            "preload": true,
            "collection_url": "https://s3-west.nrp-nautilus.io/public-calenviroscreen/stac-collection.json",
            "group": { "name": "Environmental Justice", "collapsed": false },
            "assets": [
                {
                    "id": "ces5-pmtiles",
                    "display_name": "CalEnviroScreen 5.0",
                    "group": "Environmental Justice",
                    "visible": true,
                    "default_style": {
                        "fill-color": [
                            "interpolate", ["linear"], ["get", "CIscore_Pctl"],
                            0,   "#1A9641",
                            25,  "#A6D96A",
                            50,  "#FFFFBF",
                            75,  "#FDAE61",
                            100, "#D7191C"
                        ],
                        "fill-opacity": 0.7
                    },
                    "outline_style": { "line-color": "#555555", "line-width": 0.5 },
                    "tooltip_fields": ["tract", "county", "ZIP", "CIscore", "CIscore_Pctl", "Pollution_Pctl", "PopChar_Pctl"]
                }
            ]
        },
        {
            "collection_id": "ca-dac-eda-2023",
            "collection_url": "https://s3-west.nrp-nautilus.io/public-ca-dac/stac-collection.json",
            "group": { "name": "Environmental Justice", "collapsed": false },
            "assets": [
                {
                    "id": "dac-tract-pmtiles",
                    "display_name": "CA Disadvantaged Communities (DWR 2023, tract)",
                    "group": "Environmental Justice",
                    "visible": false,
                    "default_style": {
                        "fill-color": [
                            "match", ["get", "DAC23"],
                            "Y", "#6A1B9A",
                            "N", "rgba(0,0,0,0)",
                            "rgba(150,150,150,0.2)"
                        ],
                        "fill-opacity": 0.5
                    },
                    "outline_style": { "line-color": "#4A148C", "line-width": 0.4 },
                    "tooltip_fields": ["GEOID20", "DAC23", "MHI23", "Pop23"]
                }
            ]
        },
        {
            "collection_id": "svi-2022",
            "collection_url": "https://s3-west.nrp-nautilus.io/public-social-vulnerability/2022/stac-collection.json",
            "group": { "name": "Environmental Justice", "collapsed": false },
            "assets": [
                {
                    "id": "tract-pmtiles",
                    "display_name": "CDC Social Vulnerability Index (tract)",
                    "group": "Environmental Justice",
                    "visible": false,
                    "default_style": {
                        "fill-color": [
                            "interpolate", ["linear"], ["get", "RPL_THEMES"],
                            0,    "#1A9641",
                            0.25, "#A6D96A",
                            0.5,  "#FFFFBF",
                            0.75, "#FDAE61",
                            1,    "#D7191C"
                        ],
                        "fill-opacity": 0.6
                    },
                    "outline_style": { "line-color": "#555555", "line-width": 0.3 },
                    "default_filter": ["==", ["get", "ST_ABBR"], "CA"],
                    "tooltip_fields": ["COUNTY", "FIPS", "RPL_THEMES", "ST_ABBR"]
                },
                {
                    "id": "county-pmtiles",
                    "display_name": "CDC Social Vulnerability Index (county)",
                    "group": "Environmental Justice",
                    "visible": false,
                    "default_style": {
                        "fill-color": [
                            "interpolate", ["linear"], ["get", "RPL_THEMES"],
                            0,    "#1A9641",
                            0.25, "#A6D96A",
                            0.5,  "#FFFFBF",
                            0.75, "#FDAE61",
                            1,    "#D7191C"
                        ],
                        "fill-opacity": 0.7
                    },
                    "outline_style": { "line-color": "#555555", "line-width": 0.5 },
                    "default_filter": ["==", ["get", "ST_ABBR"], "CA"],
                    "tooltip_fields": ["COUNTY", "FIPS", "RPL_THEMES", "ST_ABBR"]
                }
            ]
        },
        {
            "collection_id": "mappinginequality",
            "collection_url": "https://s3-west.nrp-nautilus.io/public-mappinginequality/stac-collection.json",
            "group": { "name": "Environmental Justice", "collapsed": false },
            "assets": [
                {
                    "id": "mappinginequality-pmtiles",
                    "display_name": "HOLC Redlining Grades (1930s)",
                    "group": "Environmental Justice",
                    "visible": false,
                    "default_style": {
                        "fill-color": [
                            "match", ["get", "grade"],
                            "A", "#1B5E20",
                            "B", "#1565C0",
                            "C", "#FBC02D",
                            "D", "#C62828",
                            "#9E9E9E"
                        ],
                        "fill-opacity": 0.6
                    },
                    "outline_style": { "line-color": "#212121", "line-width": 0.4 },
                    "tooltip_fields": ["city", "state", "grade", "label"]
                }
            ]
        },
        {
            "collection_id": "census-2024-county",
            "preload": true,
            "collection_url": "https://s3-west.nrp-nautilus.io/public-census/census-2024/county/stac-collection.json",
            "group": { "name": "Political Boundaries", "collapsed": false },
            "assets": [
                {
                    "id": "county-pmtiles",
                    "alias": "census-county-pmtiles",
                    "display_name": "California Counties",
                    "group": "Political Boundaries",
                    "visible": true,
                    "default_style": { "fill-color": "#00695C", "fill-opacity": 0.05 },
                    "outline_style": { "line-color": "#00695C", "line-width": 1.5 },
                    "default_filter": ["==", ["get", "STATEFP"], "06"],
                    "tooltip_fields": ["NAMELSAD", "NAME", "GEOID"]
                }
            ]
        },
        {
            "collection_id": "census-2024-cd",
            "preload": true,
            "collection_url": "https://s3-west.nrp-nautilus.io/public-census/census-2024/cd/stac-collection.json",
            "group": { "name": "Political Boundaries", "collapsed": false },
            "assets": [
                {
                    "id": "cd",
                    "display_name": "Congressional Districts",
                    "group": "Political Boundaries",
                    "visible": false,
                    "default_style": { "fill-color": "#1565C0", "fill-opacity": 0.05 },
                    "outline_style": { "line-color": "#1565C0", "line-width": 1.5 },
                    "default_filter": ["==", ["get", "STATEFP"], "06"],
                    "tooltip_fields": ["NAMELSAD", "GEOID", "STATEFP"]
                }
            ]
        },
        {
            "collection_id": "census-2025-sldu",
            "preload": true,
            "collection_url": "https://s3-west.nrp-nautilus.io/public-census/census-2025/sldu/stac-collection.json",
            "group": { "name": "Political Boundaries", "collapsed": false },
            "assets": [
                {
                    "id": "sldu-pmtiles",
                    "display_name": "CA Senate Districts",
                    "group": "Political Boundaries",
                    "visible": false,
                    "default_style": { "fill-color": "#BF360C", "fill-opacity": 0.05 },
                    "outline_style": { "line-color": "#BF360C", "line-width": 1.5 },
                    "default_filter": ["==", ["get", "STATEFP"], "06"],
                    "tooltip_fields": ["NAMELSAD", "GEOID", "SLDUST"]
                }
            ]
        },
        {
            "collection_id": "census-2025-sldl",
            "preload": true,
            "collection_url": "https://s3-west.nrp-nautilus.io/public-census/census-2025/sldl/stac-collection.json",
            "group": { "name": "Political Boundaries", "collapsed": false },
            "assets": [
                {
                    "id": "sldl-pmtiles",
                    "display_name": "CA Assembly Districts",
                    "group": "Political Boundaries",
                    "visible": false,
                    "default_style": { "fill-color": "#6A1B9A", "fill-opacity": 0.05 },
                    "outline_style": { "line-color": "#6A1B9A", "line-width": 1.5 },
                    "default_filter": ["==", ["get", "STATEFP"], "06"],
                    "tooltip_fields": ["NAMELSAD", "GEOID", "SLDLST"]
                }
            ]
        },
        {
            "collection_id": "conservation-almanac-2024-sites",
            "preload": true,
            "collection_url": "https://s3-west.nrp-nautilus.io/public-tpl/conservation-almanac-2024-sites/stac-collection.json",
            "group": { "name": "Conservation & Equity Investments", "collapsed": true },
            "assets": [
                {
                    "id": "conservation-almanac-2024-sites-pmtiles",
                    "display_name": "TPL Conservation Almanac",
                    "group": "Conservation & Equity Investments",
                    "visible": false,
                    "default_style": {
                        "fill-color": [
                            "match", ["get", "purchase_type"],
                            "FSP", "#2E7D32",
                            "ESM", "#1565C0",
                            "FNE", "#6A1B9A",
                            "OTH", "#E65100",
                            "#9E9E9E"
                        ],
                        "fill-opacity": 0.6
                    },
                    "tooltip_fields": ["site", "owner", "owner_type", "manager", "access_type", "purchase_type", "acres", "year", "county"]
                }
            ]
        },
        {
            "collection_id": "landvote",
            "preload": true,
            "collection_url": "https://s3-west.nrp-nautilus.io/public-tpl/landvote/stac-collection.json",
            "group": { "name": "Conservation & Equity Investments", "collapsed": true },
            "assets": [
                {
                    "id": "landvote-pmtiles",
                    "display_name": "Conservation Ballot Measures",
                    "group": "Conservation & Equity Investments",
                    "visible": false,
                    "default_filter": ["all",
                        ["==", ["get", "state"], "CA"],
                        ["!=", ["get", "jurisdiction"], "State"],
                        [">=", ["get", "year"], 2010]
                    ],
                    "default_style": {
                        "fill-color": [
                            "match", ["get", "status"],
                            "Pass",  "#1B5E20",
                            "Pass*", "#2E7D32",
                            "Fail",  "#C62828",
                            "#9E9E9E"
                        ],
                        "fill-opacity": 0.5
                    },
                    "outline_style": { "line-color": "#424242", "line-width": 0.8 },
                    "tooltip_fields": ["municipal", "county", "year", "status", "finance_mechanism", "conservation_funds_approved", "purpose"]
                }
            ]
        },
        {
            "collection_id": "wcb-approved-projects",
            "collection_url": "https://s3-west.nrp-nautilus.io/public-tpl/wcb-approved-projects/stac-collection.json",
            "group": { "name": "Conservation & Equity Investments", "collapsed": true },
            "assets": [
                {
                    "id": "wcb-approved-projects-pmtiles",
                    "display_name": "WCB Approved Projects",
                    "group": "Conservation & Equity Investments",
                    "visible": false,
                    "default_style": { "fill-color": "#00695C", "fill-opacity": 0.5 },
                    "outline_style": { "line-color": "#004D40", "line-width": 0.6 },
                    "tooltip_fields": ["ProjName", "PrimGrante", "County", "Acres", "WCBFunding", "Type", "Program", "dtmBoardAp"]
                }
            ]
        },
        {
            "collection_id": "cpad-units-2025b",
            "collection_url": "https://s3-west.nrp-nautilus.io/public-cpad/cpad-units-stac-collection.json",
            "group": { "name": "Conservation & Equity Investments", "collapsed": true },
            "assets": [
                {
                    "id": "units-pmtiles",
                    "display_name": "CPAD Protected Areas",
                    "group": "Conservation & Equity Investments",
                    "visible": false,
                    "default_style": { "fill-color": "#388E3C", "fill-opacity": 0.5 },
                    "tooltip_fields": ["UNIT_NAME", "AGNCY_NAME", "ACCESS_TYP", "ACRES"]
                }
            ]
        },
        {
            "collection_id": "irrecoverable-carbon",
            "collection_url": "https://s3-west.nrp-nautilus.io/public-carbon/stac-collection.json",
            "group": { "name": "Environmental Co-benefits", "collapsed": true },
            "assets": [
                {
                    "id": "v2-irrecoverable-2024-cog",
                    "display_name": "Irrecoverable Carbon (2024)",
                    "group": "Environmental Co-benefits",
                    "visible": false,
                    "colormap": "reds",
                    "rescale": "0,150"
                }
            ]
        },
        {
            "collection_id": "calfire-2024-firep",
            "collection_url": "https://s3-west.nrp-nautilus.io/public-fire/calfire-2024/firep/stac-collection.json",
            "group": { "name": "Environmental Co-benefits", "collapsed": true },
            "assets": [
                {
                    "id": "firep-pmtiles",
                    "display_name": "Wildfire Perimeters",
                    "group": "Environmental Co-benefits",
                    "visible": false,
                    "default_filter": [">=", ["get", "YEAR_"], 2000],
                    "default_style": {
                        "fill-color": [
                            "interpolate", ["linear"], ["get", "YEAR_"],
                            2000, "#FFCCBC",
                            2015, "#FF7043",
                            2024, "#B71C1C"
                        ],
                        "fill-opacity": 0.5
                    },
                    "outline_style": { "line-color": "#B71C1C", "line-width": 0.5 },
                    "tooltip_fields": ["FIRE_NAME", "YEAR_", "GIS_ACRES", "AGENCY", "CAUSE"]
                }
            ]
        },
        {
            "collection_id": "wetlands-nwi",
            "collection_url": "https://s3-west.nrp-nautilus.io/public-wetlands/nwi/stac-collection.json",
            "group": { "name": "Environmental Co-benefits", "collapsed": true },
            "assets": [
                {
                    "id": "nwi-pmtiles",
                    "display_name": "NWI Wetlands",
                    "group": "Environmental Co-benefits",
                    "visible": false,
                    "default_style": {
                        "fill-color": [
                            "match", ["get", "WETLAND_TYPE"],
                            "Freshwater Emergent Wetland",        "#7CCD7C",
                            "Freshwater Forested/Shrub Wetland",  "#008000",
                            "Freshwater Pond",                    "#2980B9",
                            "Lake",                               "#1F77B4",
                            "Lakes",                              "#1F77B4",
                            "Riverine",                           "#0099CC",
                            "Estuarine and Marine Wetland",       "#FF7F00",
                            "Estuarine and Marine Deepwater",     "#0F0F73",
                            "#9E9E9E"
                        ],
                        "fill-opacity": 0.6
                    },
                    "outline_style": { "line-color": "#333333", "line-width": 0.3, "line-opacity": 0.5 },
                    "tooltip_fields": ["WETLAND_TYPE", "ATTRIBUTE", "state_code"]
                }
            ]
        },
        {
            "collection_id": "usfws-critical-habitat-final",
            "collection_url": "https://s3-west.nrp-nautilus.io/public-usfws/critical-habitat/final/stac-collection.json",
            "group": { "name": "Environmental Co-benefits", "collapsed": true },
            "assets": [
                {
                    "id": "final-pmtiles",
                    "display_name": "USFWS Critical Habitat",
                    "group": "Environmental Co-benefits",
                    "visible": false,
                    "default_style": {
                        "fill-color": [
                            "match", ["get", "listing_st"],
                            "Endangered",  "#B71C1C",
                            "Threatened",  "#EF6C00",
                            "Extinction",  "#424242",
                            "#9E9E9E"
                        ],
                        "fill-opacity": 0.5
                    },
                    "outline_style": { "line-color": "#7F0000", "line-width": 0.4 },
                    "tooltip_fields": ["comname", "sciname", "listing_st", "unitname", "status", "pubdate"]
                }
            ]
        }
    ]
}
```

**Note:** If Task 2 found that either `ca-dac-eda-2023` or `mappinginequality` is missing its pmtiles, delete the corresponding object literal from the JSON above before saving. Otherwise save the JSON exactly as written.

- [ ] **Step 2: Validate the JSON parses**

Run:
```bash
python3 -c "import json; d=json.load(open('/home/cboettig/Documents/github/boettiger-lab/calenviroscreen/layers-input.json')); print('OK,', len(d['collections']), 'collections')"
```
Expected: `OK, 16` (or 15/14 if a net-new layer was dropped in Task 2).

- [ ] **Step 3: Verify every `collection_id` exists in the live STAC catalog**

Run:
```bash
python3 << 'EOF'
import json, urllib.request
cfg = json.load(open('/home/cboettig/Documents/github/boettiger-lab/calenviroscreen/layers-input.json'))
fail = 0
for c in cfg['collections']:
    cid = c['collection_id']
    url = c['collection_url']
    try:
        d = json.loads(urllib.request.urlopen(url, timeout=10).read())
        live_id = d.get('id')
        ok = live_id == cid
        print(f"{'OK ' if ok else 'BAD'}  {cid} (live id: {live_id})")
        if not ok: fail += 1
    except Exception as e:
        print(f"ERR {cid}: {e}")
        fail += 1
print(f"{fail} failures" if fail else "All collection_ids match live STAC ids.")
EOF
```
Expected: every line `OK` and `All collection_ids match live STAC ids.` If any
mismatch, STOP and reconcile — geo-agent will silently fail to load mismatched
layers.

- [ ] **Step 4: Commit**

```bash
git add layers-input.json
git commit -m "feat: add layers-input.json (16 layers, EJ-focused, CES default-visible)"
```

---

## Task 6: Write `k8s/configmap.yaml`

**Files:**
- Create: `k8s/configmap.yaml`

Two ConfigMaps in one file: client-side `config.json` template (no secrets) and
server-side `nginx.conf` template (which gets `${PROXY_KEY}` substituted at pod
start). Same structure as tpl-ca, renamed.

- [ ] **Step 1: Create the `k8s/` directory**

Run:
```bash
mkdir -p /home/cboettig/Documents/github/boettiger-lab/calenviroscreen/k8s
```

- [ ] **Step 2: Write `k8s/configmap.yaml`**

Write `/home/cboettig/Documents/github/boettiger-lab/calenviroscreen/k8s/configmap.yaml` with this exact content:

```yaml
# LLM config template — NO SECRETS.
#
# The PROXY_KEY lives only in the nginx config (see `calenviroscreen-nginx`
# below), which injects the Authorization header server-side via
# `proxy_set_header`. The browser only sees a relative `/api/llm`
# endpoint and never holds the key.
apiVersion: v1
kind: ConfigMap
metadata:
  name: calenviroscreen-config
data:
  config.template.json: |
    {
        "mcp_server_url": "${MCP_SERVER_URL}",
        "llm_model": "glm-5",
        "llm_models": [
            { "value": "glm-5",        "label": "NRP GLM-5",          "endpoint": "/api/llm", "api_key": "unused" },
            { "value": "nemotron",     "label": "DSE Nemotron",       "endpoint": "/api/llm", "api_key": "unused" },
            { "value": "qwen3",        "label": "NRP Qwen3",          "endpoint": "/api/llm", "api_key": "unused" },
            { "value": "qwen3-small",  "label": "NRP Qwen3 Small",    "endpoint": "/api/llm", "api_key": "unused" },
            { "value": "gpt-oss",      "label": "NRP GPT-OSS 120B",   "endpoint": "/api/llm", "api_key": "unused" },
            { "value": "kimi",         "label": "NRP Kimi K2.5",      "endpoint": "/api/llm", "api_key": "unused" },
            { "value": "minimax-m2",   "label": "NRP MiniMax M2.5",   "endpoint": "/api/llm", "api_key": "unused" }
        ]
    }

---
# Nginx configuration — reverse-proxies /api/llm/ to open-llm-proxy and
# injects the Authorization header from ${PROXY_KEY} at container start.
apiVersion: v1
kind: ConfigMap
metadata:
  name: calenviroscreen-nginx
data:
  nginx.conf.template: |
    server {
        listen 80;
        server_name _;

        root /usr/share/nginx/html;
        index index.html;

        add_header Access-Control-Allow-Origin *;
        add_header Access-Control-Allow-Methods "GET, POST, OPTIONS";
        add_header Access-Control-Allow-Headers "DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Accept";

        location /api/llm/ {
            proxy_pass https://open-llm-proxy.nrp-nautilus.io/v1/;
            proxy_set_header Authorization "Bearer ${PROXY_KEY}";
            proxy_set_header Host open-llm-proxy.nrp-nautilus.io;
            proxy_ssl_server_name on;

            proxy_read_timeout 300s;
            proxy_send_timeout 300s;
            proxy_buffering off;
        }

        location / {
            try_files $uri $uri/ /index.html;
            add_header Cache-Control "no-store, no-cache, must-revalidate, proxy-revalidate, max-age=0";
            add_header Pragma "no-cache";
            add_header Expires "0";
        }

        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
    }
```

- [ ] **Step 3: Validate YAML parses (two-doc file)**

Run:
```bash
python3 -c "import yaml,sys; docs=list(yaml.safe_load_all(open('/home/cboettig/Documents/github/boettiger-lab/calenviroscreen/k8s/configmap.yaml'))); print(len(docs), 'docs'); [print(d['metadata']['name']) for d in docs]"
```
Expected: `2 docs` then `calenviroscreen-config` and `calenviroscreen-nginx`.

---

## Task 7: Write `k8s/deployment.yaml`

**Files:**
- Create: `k8s/deployment.yaml`

- [ ] **Step 1: Write `k8s/deployment.yaml`**

Write `/home/cboettig/Documents/github/boettiger-lab/calenviroscreen/k8s/deployment.yaml` with this exact content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: calenviroscreen
  labels:
    app: calenviroscreen
spec:
  replicas: 1
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: calenviroscreen
  template:
    metadata:
      labels:
        app: calenviroscreen
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: feature.node.kubernetes.io/pci-10de.present
                    operator: NotIn
                    values: ["true"]
      initContainers:
      - name: git-clone
        image: alpine/git:2.52.0
        command:
        - sh
        - -c
        - |
          git clone --depth 1 https://github.com/boettiger-lab/calenviroscreen.git /tmp/repo
          cp /tmp/repo/index.html /usr/share/nginx/html/
          cp /tmp/repo/layers-input.json /usr/share/nginx/html/
          cp /tmp/repo/system-prompt.md /usr/share/nginx/html/
        volumeMounts:
        - name: website-content
          mountPath: /usr/share/nginx/html
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        env:
        - name: TMPDIR
          value: "/tmp"
        - name: MCP_SERVER_URL
          value: "https://duckdb-mcp.nrp-nautilus.io/mcp"
        - name: PROXY_KEY
          valueFrom:
            secretKeyRef:
              name: open-llm-proxy-secrets
              key: proxy-key
              optional: true
        command:
        - sh
        - -c
        - |
          apk add --no-cache gettext
          # Client-side config: no secrets.
          envsubst '$${MCP_SERVER_URL}' < /config/config.template.json \
            > /usr/share/nginx/html/config.json
          # Server-side nginx config: substitutes ${PROXY_KEY} into the
          # /api/llm/ reverse proxy's Authorization header. Limited to
          # ${PROXY_KEY} so nginx's own $uri / $remote_addr variables
          # are preserved.
          envsubst '$${PROXY_KEY}' < /nginx-template/nginx.conf.template \
            > /etc/nginx/conf.d/default.conf
          nginx -g 'daemon off;'
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
        volumeMounts:
        - name: website-content
          mountPath: /usr/share/nginx/html
        - name: nginx-template
          mountPath: /nginx-template
          readOnly: true
        - name: nginx-conf-d
          mountPath: /etc/nginx/conf.d
        - name: config-template
          mountPath: /config
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
      volumes:
      - name: website-content
        emptyDir: {}
      - name: config-template
        configMap:
          name: calenviroscreen-config
      - name: nginx-template
        configMap:
          name: calenviroscreen-nginx
      - name: nginx-conf-d
        emptyDir: {}
```

Note the deliberate differences from tpl-ca's deployment:
- All `tpl-ca` strings replaced with `calenviroscreen`.
- The init-container `cp` list omits `docs.html` (calenviroscreen has no docs page).
- ConfigMap names (`calenviroscreen-config`, `calenviroscreen-nginx`) match Task 6.
- Same Secret reference `open-llm-proxy-secrets` (shared across apps — do NOT rename).

- [ ] **Step 2: Validate YAML**

Run:
```bash
python3 -c "import yaml; d=yaml.safe_load(open('/home/cboettig/Documents/github/boettiger-lab/calenviroscreen/k8s/deployment.yaml')); print(d['kind'], d['metadata']['name'])"
```
Expected: `Deployment calenviroscreen`.

---

## Task 8: Write `k8s/service.yaml`

**Files:**
- Create: `k8s/service.yaml`

- [ ] **Step 1: Write `k8s/service.yaml`**

Write `/home/cboettig/Documents/github/boettiger-lab/calenviroscreen/k8s/service.yaml` with this exact content:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: calenviroscreen
  labels:
    app: calenviroscreen
spec:
  type: ClusterIP
  selector:
    app: calenviroscreen
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      name: http
```

- [ ] **Step 2: Validate**

Run:
```bash
python3 -c "import yaml; d=yaml.safe_load(open('/home/cboettig/Documents/github/boettiger-lab/calenviroscreen/k8s/service.yaml')); print(d['kind'], d['metadata']['name'])"
```
Expected: `Service calenviroscreen`.

---

## Task 9: Write `k8s/ingress.yaml`

**Files:**
- Create: `k8s/ingress.yaml`

- [ ] **Step 1: Write `k8s/ingress.yaml`**

Write `/home/cboettig/Documents/github/boettiger-lab/calenviroscreen/k8s/ingress.yaml` with this exact content:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: calenviroscreen-ingress
  annotations:
    haproxy-ingress.github.io/cors-enable: "true"
    haproxy-ingress.github.io/cors-allow-origin: "*"
    haproxy-ingress.github.io/cors-allow-methods: "GET, POST, OPTIONS"
    haproxy-ingress.github.io/cors-allow-headers: "DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization,mcp-session-id"
    haproxy-ingress.github.io/cors-allow-credentials: "true"
    haproxy-ingress.github.io/cors-max-age: "86400"
    haproxy-ingress.github.io/timeout-server: "600s"
    haproxy-ingress.github.io/timeout-tunnel: "3600s"
spec:
  ingressClassName: haproxy
  tls:
    - hosts:
        - calenviroscreen.nrp-nautilus.io
  rules:
    - host: calenviroscreen.nrp-nautilus.io
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: calenviroscreen
                port:
                  number: 80
```

- [ ] **Step 2: Validate**

Run:
```bash
python3 -c "import yaml; d=yaml.safe_load(open('/home/cboettig/Documents/github/boettiger-lab/calenviroscreen/k8s/ingress.yaml')); print(d['kind'], d['metadata']['name'], 'host:', d['spec']['rules'][0]['host'])"
```
Expected: `Ingress calenviroscreen-ingress host: calenviroscreen.nrp-nautilus.io`.

- [ ] **Step 3: Commit all k8s manifests**

```bash
git add k8s/
git commit -m "feat: add k8s manifests (deployment, service, ingress, configmap)"
```

---

## Task 10: Create GitHub repo and push

**Files:** None changed locally. This task publishes the repo so the init container can clone it.

- [ ] **Step 1: Re-verify the repo does not exist**

Run:
```bash
gh repo view boettiger-lab/calenviroscreen 2>&1 | head -3
```
Expected: `GraphQL: Could not resolve to a Repository`. If the repo exists now (it didn't in Task 1), STOP and reconcile — someone else may be working on the same thing, or a prior aborted attempt left state behind.

- [ ] **Step 2: Confirm local repo state is clean**

Run:
```bash
git status && git log --oneline
```
Expected: clean working tree, ~4 commits (init, index.html, system-prompt.md, layers-input.json, k8s — depending on how you batched).

- [ ] **Step 3: Create the GitHub repo and push**

Run:
```bash
cd /home/cboettig/Documents/github/boettiger-lab/calenviroscreen
gh repo create boettiger-lab/calenviroscreen \
  --public \
  --description "CalEnviroScreen 5.0 geo-agent app for California environmental-justice analysis" \
  --source . \
  --remote origin \
  --push
```
Expected: `gh` reports the repo was created and that `main` was pushed.

- [ ] **Step 4: Confirm remote contents are correct**

Run:
```bash
gh repo view boettiger-lab/calenviroscreen --json url,visibility,defaultBranchRef --jq .
git ls-remote origin main
```
Expected: visibility is `PUBLIC`, defaultBranchRef name is `main`, and `ls-remote` shows a SHA matching `git rev-parse main`.

---

## Task 11: Apply k8s manifests and verify deployment

Before this task, ensure you're loading the `superpowers:nrp-k8s` skill and confirm you're targeting the `biodiversity` namespace.

- [ ] **Step 1: Confirm kube context and namespace**

Run:
```bash
kubectl config current-context
kubectl get ns biodiversity
```
Expected: a context that targets the NRP Nautilus cluster, and `biodiversity` namespace exists.

- [ ] **Step 2: Confirm the shared LLM-proxy Secret exists**

Run:
```bash
kubectl -n biodiversity get secret open-llm-proxy-secrets
```
Expected: the secret is listed (it is shared with tpl-ca and other apps). If it doesn't exist, the app will still deploy but the LLM proxy won't work — fix that first.

- [ ] **Step 3: Dry-run apply to catch syntax / schema errors**

Run:
```bash
kubectl -n biodiversity apply -f /home/cboettig/Documents/github/boettiger-lab/calenviroscreen/k8s/ --dry-run=server
```
Expected: each of the five resources (2 ConfigMaps, 1 Deployment, 1 Service, 1 Ingress) reports `(server dry run)`. Any errors here must be fixed before the real apply.

- [ ] **Step 4: Apply**

Run:
```bash
kubectl -n biodiversity apply -f /home/cboettig/Documents/github/boettiger-lab/calenviroscreen/k8s/
```
Expected: all five resources created (or unchanged on re-apply).

- [ ] **Step 5: Wait for rollout**

Run:
```bash
kubectl -n biodiversity rollout status deployment/calenviroscreen --timeout=180s
```
Expected: `deployment "calenviroscreen" successfully rolled out`.

- [ ] **Step 6: Inspect pod for init-container or runtime errors**

Run:
```bash
kubectl -n biodiversity get pods -l app=calenviroscreen -o wide
kubectl -n biodiversity logs deployment/calenviroscreen --all-containers --tail=50
```
Expected: pod is Running with 1/1 ready; init-container's `git clone` reported success; nginx log shows no errors and serves `/health 200`.

- [ ] **Step 7: Verify ingress hostname is provisioned**

Run:
```bash
kubectl -n biodiversity get ingress calenviroscreen-ingress
```
Expected: `HOSTS` column shows `calenviroscreen.nrp-nautilus.io`, `ADDRESS` is set (it can take a couple of minutes the first time).

- [ ] **Step 8: Curl the deployed health endpoint and root**

Run:
```bash
curl -sS https://calenviroscreen.nrp-nautilus.io/health
curl -sS -o /tmp/ces-index.html -w '%{http_code} %{size_download}\n' https://calenviroscreen.nrp-nautilus.io/
grep -c "CalEnviroScreen Assistant" /tmp/ces-index.html
```
Expected: first command prints `healthy`; second prints `200` and a nonzero byte count; third prints `1` (matching the chat-panel heading).

- [ ] **Step 9: Browser verification**

Open `https://calenviroscreen.nrp-nautilus.io/` in a browser. Verify:
1. Map renders, centered on California, zoom ~6.
2. CalEnviroScreen layer is visible by default (yellow→red choropleth).
3. Layer panel groups are in order: Environmental Justice (expanded), Political Boundaries (expanded), Conservation & Equity Investments (collapsed), Environmental Co-benefits (collapsed).
4. Welcome message lists the six prompts from `layers-input.json`.
5. Clicking the first welcome prompt ("Which census tracts in Alameda County are in the top 10% of CES burden…") triggers a chat response with reasonable content (this is the LLM/MCP smoke test).

If any verification fails, do not declare the task complete. Reload `kubectl logs deployment/calenviroscreen --tail=200` and the browser dev-tools console; root-cause before re-attempting.

- [ ] **Step 10: Final commit and push**

If any in-flight fixes were made during deployment (e.g., a fixed `collection_id` in `layers-input.json`):

```bash
git status
git add -A
git commit -m "fix: <specific fix discovered during deployment>"
git push origin main
# Force the pod to re-clone the fresh main:
kubectl -n biodiversity rollout restart deployment/calenviroscreen
kubectl -n biodiversity rollout status deployment/calenviroscreen --timeout=180s
```

Otherwise just confirm:
```bash
git status
```
Expected: working tree clean, branch up-to-date with origin/main.

---

## Done

The app is live at `https://calenviroscreen.nrp-nautilus.io` and the source is at
`https://github.com/boettiger-lab/calenviroscreen`. Total deployment artifacts:
nine files in the repo plus five K8s resources in `biodiversity` namespace.
