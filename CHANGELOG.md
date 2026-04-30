# Changelog

All notable changes to Ferriq are documented in this file.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

---

## [1.0.0] - 2026-04-29

First production release of Ferriq - five verified engineering dashboards
with full calculation engines, deployed to GitHub Pages.

### Added

- **Air Blower** dashboard - centrifugal blower performance monitoring with
  ASME PTC 10 polytropic efficiency, surge margin, and vibration tracking.
- **Fired Heater** dashboard - gas-fired heater efficiency via ASME PTC 4
  indirect method with stack-loss, radiation-loss, and excess-air analysis.
- **Shell-Tube Exchanger** dashboard - LMTD-method duty calculation, overall
  U, fouling resistance (Rf) tracking against commissioning baseline, and
  end-approach temperature monitoring per TEMA RCB / Kern.
- **Membrane Analyzer** dashboard - H₂-recovery membrane module monitoring
  with stage cut, recovery, actual separation factor, log-mean partial-pressure
  driving force, and normalized permeance index per Wijmans & Baker (1995).
- **Furnace Skin Temp** dashboard - tube-metal-temperature prediction via
  Dittus-Boelter correlation with measured-vs-predicted residual drift
  flagging and three-tier TMT limit classification (advisory / alarm / trip).
- **Welcome / Home** page - equipment grid with live status pills, KPI
  summaries, and navigation to all five dashboards.
- **PDF manual download** button on all five dashboards (print-to-PDF via
  styled new window).
- **Backend technical manuals** (`backend-*.md`) - comprehensive internal
  reference docs for each model (equations, tag dictionaries, data schemas,
  worked examples, engineering standards). Present in repo but not linked
  from the public site.
- Gold design system - IBM Plex Sans/Mono typography, warm brown/gold
  palette, HPHMI/ISA-101 gray-default philosophy, triple-encoded status.
- Chart.js 4.4.1 trend charts with dual-axis support, threshold lines,
  and Papa Parse 5.4.1 CSV upload with tag aliasing.
- Configurable settings/limits panel in each dashboard sidebar.
- 28-row synthetic demo datasets (7 days × 4 hr or 6 hr intervals) with
  planted fault signatures for each model.

### Removed

- Placeholder dashboards from initial sync (chemical-reactor, compressor-system,
  distillation-column, equipment-dashboard, heat-exchanger).

### Fixed

- Shell-tube exchanger: corrected approach temperature formula (hot-end and
  cold-end were swapped), fixed four limit defaults (Rf trigger, approach,
  shell ΔP, tube ΔP), corrected fouling alert severity to alarm, fixed
  imbalance threshold (5% → 8%), and LMTD edge case when ΔT₁ ≈ ΔT₂.
- Furnace skin temp: replaced fabricated demo data (wrong dates, scale, count)
  with original 28-row series.
- Air blower: extended truncated demo data from 15 to full 28 rows.

---

[1.0.0]: https://github.com/vedantpatel699/Ferriq/releases/tag/v1.0.0
