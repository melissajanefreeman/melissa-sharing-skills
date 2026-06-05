# Front Range Hazard Report — Nederland & surrounds

A single, self-contained HTML briefing that maps where wildfire, flood, and drought
exposure concentrates around **Nederland**, then widens to **Boulder County** and the
**surrounding Front Range counties** (Larimer, Gilpin, Jefferson, Weld, Broomfield, Adams).

## What it is

- **`index.html`** — open it in any modern browser. No build step, no server required.
- The maps are **interactive** and stream their hazard layers **live from government
  services**, so they render with the exact layers already applied:
  - **Wildfire** — USFS *Wildfire Risk to Communities* (Risk to Potential Structures /
    Wildfire Hazard Potential) raster ImageServers.
  - **Flood** — FEMA *National Flood Hazard Layer (NFHL)*.
  - **Cross-hazard (optional toggle)** — FEMA *National Risk Index* census tracts.
  - **Base imagery** — Esri *World Imagery* (satellite).
- Labelled neighborhood pins and historic fire perimeters (Cold Springs 2016, Marshall
  2021) are a **reasoned overlay** on top of the live data, for orientation.

## How to read it

The hazard **shading** on every map is real government data. The **ranking and the pins**
are a data-anchored *synthesis* — my best read of where risk concentrates given hazard
intensity, egress, fuels, and disaster history — not an engineered parcel-level model.
Neighborhood outlines and fire perimeters are **hand-placed approximations**. Each ranked
area carries an honest confidence bar. See §07 ("How this ranking was derived") and §08
("Government & primary sources") in the report itself.

> Not insurance, lending, or safety advice. Orient with this, then verify with the
> official tools (CO-WRAP, Boulder County floodplain map, FEMA NRI) before any decision.

## Notes

- Because the layers are live third-party services, an overlay may occasionally fail to
  draw if a government server is momentarily down — the base map and annotations still
  work, and every endpoint is listed in the report's Sources section so you can open it
  directly.
- To host it (e.g. GitHub Pages), just serve `index.html` over HTTPS; the map services
  require a secure context.
