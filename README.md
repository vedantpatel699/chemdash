# ChemDash — Chemical Engineering Dashboards

A set of proof-of-concept monitoring dashboards for industrial process equipment. Built as self-contained, single-file HTML pages — no server, no database, no build step. Open in any modern browser.

## Dashboards

| Equipment | File | Key Metrics |
|-----------|------|-------------|
| Welcome / Home | `welcome-home.html` | Equipment grid, time-based greeting |
| Air Blower | `air-blower.html` | ASME PTC 10 efficiency, ISO 10816 vibration, bearing health |
| Fired Heater Efficiency | `fired-heater.html` | Direct & indirect efficiency, loss breakdown, excess air |
| Shell & Tube Exchanger | `shell-tube-exchanger.html` | LMTD effectiveness, fouling tracking, delta-P monitoring |
| Membrane Analyzer | `membrane-analyzer.html` | H2 permeance tracking, recovery & purity, selectivity |
| Furnace Skin Temp | `furnace-skin-temp.html` | Tube skin prediction, residual drift, TMT classification |

## Live Site

Published via GitHub Pages at: `https://vedantpatel699.github.io/chemdash/`

## Design System

- **Typography**: IBM Plex Sans (UI) + IBM Plex Mono (data values)
- **Style**: HPHMI / ISA-101 gray-default philosophy — color reserved for abnormal states
- **Layout**: Sidebar (180px) + topbar (56px) + scrollable content area
- **Cards**: Outlined style with 16px radius, dual-layer drop shadow
- **Status**: Green (normal), Amber (advisory), Red (critical) — always paired with glyph + text
- **Charts**: Chart.js 4.4.1 via CDN, Papa Parse 5.4.1 for CSV upload

## Features (per model dashboard)

- Full calculation engine (mirrors engine.py)
- Configurable settings and alert limits in sidebar
- CSV upload for custom data
- Export processed results as CSV
- In-page manual viewer panel
- 28-row demo dataset auto-loaded
- 3 tabs: Dashboard, Data & Log, About

## Usage

Open `index.html` (redirects to the home page) or any individual dashboard file directly in a browser. No install required.

## License

Proprietary — not for redistribution.
