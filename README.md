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
