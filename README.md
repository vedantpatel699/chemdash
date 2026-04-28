# ChemDash — Chemical Engineering Dashboards

A set of proof-of-concept monitoring dashboards for industrial process equipment. Built as self-contained, single-file HTML pages — no server, no database, no build step. Open in any modern browser.

## Dashboards

| Equipment | File | Key Metrics |
|-----------|------|-------------|
| Welcome / Home | `welcome-home.html` | Fleet overview, status summary, navigation |
| RO Membrane Analyzer | `equipment-dashboard.html` | Flux, TMP, recovery, permeate quality |
| Distillation Column | `distillation-column.html` | Top/bottom temp, reflux ratio, column ΔP |
| Heat Exchanger | `heat-exchanger.html` | Overall U, fouling factor, approach ΔT |
| Chemical Reactor | `chemical-reactor.html` | Conversion, reactor temp, residence time |
| Compressor System | `compressor-system.html` | Shaft speed, vibration RMS, polytropic η |

## Live Site

Published via GitHub Pages at: `https://<username>.github.io/chemdash-dashboards/`

## Design System

- **Typography**: IBM Plex Sans (UI) + IBM Plex Mono (data values)
- **Style**: HPHMI / ISA-101 gray-default philosophy — color reserved for abnormal states
- **Layout**: Sidebar (64px) + topbar (56px) + scrollable content area
- **Cards**: Outlined style with 16px radius, dual-layer drop shadow
- **Status**: Green (normal), Amber (advisory), Red (critical) — always paired with glyph + text

## Usage

Open `index.html` (redirects to the home page) or any individual dashboard file directly in a browser. No install required.

## License

Proprietary — not for redistribution.
