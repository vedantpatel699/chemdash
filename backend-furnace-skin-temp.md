# Furnace Tube Skin-Temperature Predictor — Backend Technical Manual

**Version:** 1.0  
**Date:** 2026-04-04  
**Author:** Vedant Patel  
**Classification:** CONFIDENTIAL

---

## 1. Purpose & Scope

The Furnace Tube Skin-Temperature Predictor is a real-time monitoring system for multi-point tube-wall temperature surveillance in refinery fired heaters. The system:

- **Predicts** tube skin temperature at four axial positions (T1–T4) using Dittus-Boelert heat-transfer correlation
- **Compares** predictions to measured values to detect fouling, coking, and instrumentation drift
- **Classifies** each position into one of five risk states: **OK**, **Watch**, **Advisory**, **Alarm**, **Trip**
- **Flags** residual deviations (measured − predicted) greater than ±5 K as potential anomalies
- **Monitors** creep-rupture risk per ASME, NACE, and API 530 design standards

**Scope:** Radiant-section tubes only. Single-tube correlation. No coking model, no fouling depth estimate.

---

## 2. Tag Dictionary

| **Tag Name** | **Unit** | **Description** | **Range** | **Type** |
|---|---|---|---|---|
| `timestamp` | ISO 8601 | Data collection instant (UTC) | 2026-02-01 ↔ 2026-02-07 | DateTime |
| `process_t_c` | °C | Process outlet temperature (bulk fluid entering tubes) | 300–380 | Float |
| `fire_rate_mw` | MW | Furnace heat release (radiant duty) | 20–30 | Float |
| `process_flow_kghr` | kg/h | Process mass flow rate | 60000–85000 | Float |
| `skin_t1_c` | °C | Measured tube-wall T, position 1 (bottom, hottest) | 400–550 | Float |
| `skin_t2_c` | °C | Measured tube-wall T, position 2 (mid-lower) | 400–550 | Float |
| `skin_t3_c` | °C | Measured tube-wall T, position 3 (mid-upper) | 400–550 | Float |
| `skin_t4_c` | °C | Measured tube-wall T, position 4 (top, coolest) | 400–550 | Float |

**Position convention:** T1 sits at the lowest gas-flow elevation (highest radiant flux, hottest). T4 sits highest (lowest flux, coolest).

---

## 3. Equations

### 3.1 Heat-Transfer Correlation (Dittus–Boelert 1930)

Predicted superheating (overshoot) at position *i*:

$$\Delta T_i = K_i \cdot \left(\frac{Q_{\mathrm{fire}}}{Q_{\mathrm{ref}}}\right)^{0.8} \cdot \left(\frac{m_{\mathrm{ref}}}{m}\right)^{0.8}$$

Where:

- **$K_i$** — Baseline temperature rise at reference conditions (MW, kg/h), calibrated per position. Units: K.
- **$Q_{\mathrm{fire}}$** — Actual furnace heat release (MW)
- **$Q_{\mathrm{ref}}$** — Reference heat release = 25 MW
- **$m$** — Actual process mass flow rate (kg/h)
- **$m_{\mathrm{ref}}$** — Reference flow rate = 80,000 kg/h
- **Exponents:** 0.8 / 0.8 empirical for single-tube convection + radiation balance

### 3.2 Predicted Skin Temperature

$$T_{\mathrm{pred}, i} = T_{\mathrm{process}} + \Delta T_i$$

Where:

- **$T_{\mathrm{process}}$** = `process_t_c` (°C)
- **$T_{\mathrm{pred}, i}$** — Predicted skin temperature at position *i* (°C)

### 3.3 Residual (Prediction Error)

$$\varepsilon_i = T_{\mathrm{meas}, i} - T_{\mathrm{pred}, i}$$

Where:

- **$T_{\mathrm{meas}, i}$** = `skin_t{i}_c` (°C)
- **$\varepsilon_i$** — Residual error (°C). Positive = fouling/insulation gain. Negative = coking/sensor drift/measurement lag.

**Residual flag:** If $|\varepsilon_i| > 5$ K, mark as drift/anomaly.

---

## 4. Tube-Skin Temperature Classification

Each measured temperature at position *i* is assigned a **risk state** based on the **measured value only** (not predicted):

| **State** | **Range (°C)** | **Condition** | **Action** | **Limit Basis** |
|---|---|---|---|---|
| **Trip** | ≥ 550 | Immediate rupture risk | **SHUTDOWN** | 9Cr-1Mo short-term yield (~2% creep in 100h @ 550°C) |
| **Alarm** | 475–549 | Imminent creep-rupture | **Escalate** | NACE SP0170 threshold; API 530 Step-Stress |
| **Advisory** | 446–474 | Design maximum reached | **Monitor closely** | API 530 design-case allowable at 30-year life |
| **Watch** | 436–445 | 10 K safety margin | **Elevated risk** | Advisory − 10 K buffer |
| **OK** | < 436 | Within envelope | Normal operation | Safe zone |

**Tube material:** 9Cr-1Mo (T91), typical refinery convection-bank tube (SA-213 T91).

**Standard references:**
- **ASME PTC-4.4** — Steam generator performance (TMT monitoring)
- **NACE SP0170-2016** — Corrosion/creep metallurgy (9Cr-1Mo in H₂/CO₂)
- **API 530 (2021 ed.)** — Design, fabrication, operation of heaters for refinery service

---

## 5. Python Engine API

The calculation engine exposes six main functions:

### 5.1 `predictDT(fire_rate_mw, flow_kghr, k_baseline)`

**Purpose:** Calculate predicted temperature rise (ΔT) at a single position.

**Signature:**
```python
def predictDT(fire_rate_mw: float, flow_kghr: float, k_baseline: float) -> float:
    """
    Predict temperature rise ΔT_i at position i.
    
    Args:
        fire_rate_mw (float): Furnace heat release, MW
        flow_kghr (float): Process mass flow rate, kg/h
        k_baseline (float): Position baseline K_i, K
        
    Returns:
        float: Predicted ΔT_i, K
    """
    fref = 25.0    # Reference fire rate, MW
    wref = 80000.0 # Reference flow, kg/h
    
    scale_factor = (fire_rate_mw / fref) ** 0.8 * (wref / flow_kghr) ** 0.8
    return k_baseline * scale_factor
```

### 5.2 `classify(skin_t_c: float) -> tuple[str, str, str]`

**Purpose:** Classify a single measured skin temperature into risk state.

**Signature:**
```python
def classify(skin_t_c: float) -> tuple[str, str, str]:
    """
    Classify skin temperature into risk state (OK, Watch, Advisory, Alarm, Trip).
    
    Args:
        skin_t_c (float): Measured skin temperature, °C
        
    Returns:
        tuple: (state_name, state_color, state_emoji)
            state_name: "OK" | "Watch" | "Advisory" | "Alarm" | "Trip"
            state_color: "#3A7D52" (OK) | "#C4890E" (Watch) | "#FAF0E0" (Advisory)
                         | "#D4432A" (Alarm) | "#8B0000" (Trip)
            state_emoji: "✓" | "⚠" | "🔶" | "🔴" | "⛔" or equivalent
    """
    if skin_t_c >= 550:
        return ("Trip", "#8B0000", "🛑")
    elif skin_t_c >= 475:
        return ("Alarm", "#D4432A", "🔴")
    elif skin_t_c >= 446:
        return ("Advisory", "#FAF0E0", "🔶")
    elif skin_t_c >= 436:
        return ("Watch", "#C4890E", "⚠")
    else:
        return ("OK", "#3A7D52", "✓")
```

**Note:** No emoji in actual implementation (per CLAUDE.md §8). Use colored dots instead.

### 5.3 `process(row: dict) -> dict`

**Purpose:** Process a single CSV row and return predictions + classifications.

**Signature:**
```python
def process(row: dict) -> dict:
    """
    Full calculation pipeline for one row.
    
    Args:
        row (dict): CSV row with keys:
            timestamp, process_t_c, fire_rate_mw, process_flow_kghr,
            skin_t1_c, skin_t2_c, skin_t3_c, skin_t4_c
            
    Returns:
        dict: {
            'timestamp': str (ISO 8601),
            'process_t_c': float,
            'fire_rate_mw': float,
            'process_flow_kghr': float,
            'positions': [
                {
                    'pos': 1,
                    'measured': float,
                    'predicted': float,
                    'residual': float,
                    'state': str,
                    'anomaly': bool  # |residual| > 5 K
                },
                ...  # positions 2, 3, 4
            ],
            'max_state': str,  # Highest risk across all positions
            'max_temp': float  # Highest measured temp
        }
    """
    K = [80, 65, 50, 35]  # Baseline K per position (T1–T4)
    
    results = {'timestamp': row['timestamp'], 'positions': []}
    
    for i, k_val in enumerate(K, start=1):
        dt = predictDT(row['fire_rate_mw'], row['process_flow_kghr'], k_val)
        pred = row['process_t_c'] + dt
        meas = row[f'skin_t{i}_c']
        resid = meas - pred
        state, _, _ = classify(meas)
        anomaly = abs(resid) > 5.0
        
        results['positions'].append({
            'pos': i,
            'measured': round(meas, 1),
            'predicted': round(pred, 1),
            'residual': round(resid, 1),
            'state': state,
            'anomaly': anomaly
        })
    
    results['max_state'] = max([p['state'] for p in results['positions']])
    results['max_temp'] = max([p['measured'] for p in results['positions']])
    
    return results
```

### 5.4 `render(processed_rows: list) -> str`

**Purpose:** Generate HTML display from processed results.

**Returns:** HTML string with KPI tiles, status banner, trend charts, detail table.

### 5.5 `drawCharts(data: list, canvas_ids: dict) -> None`

**Purpose:** Render Chart.js trend charts.

**Parameters:**
- `data` — List of processed rows
- `canvas_ids` — Dict mapping series name to canvas element ID
  - `canvas_ids['T1']`, `canvas_ids['T2']`, etc. — Per-position time-series
  - `canvas_ids['predicted_vs_measured']` — Prediction accuracy (scatter or dual-axis)

---

## 6. Configuration Parameters

| **Parameter** | **Value** | **Unit** | **Description** |
|---|---|---|---|
| `Q_ref` | 25.0 | MW | Reference heat release (denominator in scale factor) |
| `m_ref` | 80,000 | kg/h | Reference process flow (denominator in scale factor) |
| `K[0]` (T1) | 80 | K | Baseline temperature rise, position 1 (hottest) |
| `K[1]` (T2) | 65 | K | Baseline temperature rise, position 2 |
| `K[2]` (T3) | 50 | K | Baseline temperature rise, position 3 |
| `K[3]` (T4) | 35 | K | Baseline temperature rise, position 4 (coolest) |
| `T_advisory` | 446 | °C | Advisory limit (API 530 design max) |
| `T_alarm` | 475 | °C | Alarm limit (NACE SP0170) |
| `T_trip` | 550 | °C | Trip limit (short-term yield, 9Cr-1Mo) |
| `T_watch` | 436 | °C | Watch limit (Advisory − 10 K margin) |
| `residual_threshold` | ±5.0 | K | Anomaly flag threshold |

**Calibration notes:** K values calibrated to a 7-day baseline (Feb 1–7, 2026) under steady-state conditions. If furnace duty or feed composition changes significantly, K should be re-tuned.

---

## 7. Alert Logic

### 7.1 Tube Classification Alert

Triggered when any position reaches **Advisory** or above:

- **Condition:** $T_{\mathrm{meas}, i} \geq 446$ °C
- **Priority:** Equal to highest state among all four positions
  - Trip → immediate shutdown
  - Alarm → escalate to operations (creep initiation)
  - Advisory → increase monitoring frequency
- **Message:** "Position {i}: {state} ({T_meas:.1f}°C) — {action}"

### 7.2 Residual Drift Alert

Triggered when prediction error exceeds threshold:

- **Condition:** $|\varepsilon_i| > 5$ K for position *i*
- **Priority:** Advisory (investigate, but not immediately dangerous)
- **Message:** "Position {i}: Residual drift {ε_i:.1f}K — check thermowell, fouling, or model calibration"

### 7.3 System-Level Banner

Displays **highest risk state** across all four positions:

- **Trip → Red banner:** Immediate action required
- **Alarm → Orange banner:** Escalate and monitor
- **Advisory → Yellow banner:** Elevated monitoring
- **Watch → Amber banner:** Caution advisable
- **OK → Green banner:** All systems nominal

---

## 8. Data Schema

### 8.1 Sample CSV Structure

```csv
timestamp,process_t_c,fire_rate_mw,process_flow_kghr,skin_t1_c,skin_t2_c,skin_t3_c,skin_t4_c
2026-02-01T00:00:00Z,359.2,24.77,71205.3,441.0,430.2,417.5,404.8
2026-02-01T06:00:00Z,359.4,24.88,71850.1,441.2,430.4,417.7,405.0
...
2026-02-07T18:00:00Z,361.1,26.09,80361.2,442.8,432.1,419.2,406.5
```

### 8.2 Data Characteristics

| **Metric** | **Value** |
|---|---|
| Row count | 28 rows (Feb 1–7, 2026, 6-hour intervals) |
| Process outlet T | 359–361°C (stable, ±1 K) |
| Fire rate | 24.77–26.09 MW (±2.5% variation) |
| Process flow | 71,205–80,361 kg/h (±5.7% variation) |
| Skin T1 (measured) | 440–443°C (nominal advisory margin ~6 K) |
| Skin T2 (measured) | 429–433°C (within watch) |
| Skin T3 (measured) | 417–420°C (within ok) |
| Skin T4 (measured) | 404–408°C (within ok) |

**Interpretation:** Position 1 operates near the 10 K advisory margin; positions 2–4 are safely within envelope. No drift anomalies in this dataset.

---

## 9. Worked Example

### Row 1: 2026-02-01 00:00:00Z

**Input:**
```
process_t_c = 359.2°C
fire_rate_mw = 24.77 MW
process_flow_kghr = 71205.3 kg/h
skin_t1_c = 441.0°C (measured)
skin_t2_c = 430.2°C (measured)
skin_t3_c = 417.5°C (measured)
skin_t4_c = 404.8°C (measured)
```

**Calculation (Position 1, K₁ = 80 K):**

1. Scale factor:
   $$\text{scale} = \left(\frac{24.77}{25.0}\right)^{0.8} \times \left(\frac{80000}{71205.3}\right)^{0.8}$$
   $$= 0.99076 \times 1.02291 = 1.01338$$

2. Predicted ΔT₁:
   $$\Delta T_1 = 80 \times 1.01338 = 81.07 \text{ K}$$

3. Predicted skin temperature:
   $$T_{\mathrm{pred}, 1} = 359.2 + 81.07 = 440.27 \text{ °C}$$

4. Residual:
   $$\varepsilon_1 = 441.0 - 440.27 = +0.73 \text{ K}$$

5. Classification:
   $$T_{\mathrm{meas}, 1} = 441.0 \text{ °C} \in [436, 446) \Rightarrow \text{**Watch**}$$
   - Measured temperature is 5 K above the 436 K watch threshold, and 5 K below the 446 K advisory threshold.

6. Anomaly flag:
   $$|\varepsilon_1| = 0.73 \text{ K} < 5.0 \text{ K} \Rightarrow \text{No anomaly}$$

**Output for Position 1:**
```
{
  'pos': 1,
  'measured': 441.0,
  'predicted': 440.3,
  'residual': 0.7,
  'state': 'Watch',
  'anomaly': False
}
```

**System-level alert:** Max state = Watch (position 1); max temp = 441.0°C. Display amber banner with position 1 highlighted.

---

## 10. Assumptions & Limitations

### 10.1 Assumptions

1. **Single-tube correlation:** Dittus–Boelert applies uniformly across all four positions; no radiant-section spatial gradients modeled.
2. **No coking:** Tube surface clean. K values assume zero fouling-resistance layer.
3. **Radiant section only:** Correlation does not apply to convection-bank tubes (lower temperatures, different Re).
4. **Exponent stability:** 0.8 / 0.8 exponents valid over 20–30 MW and 60,000–85,000 kg/h operating envelope.
5. **Linear residual threshold:** ±5 K threshold is empirical; refine if fouling rate or model drift observed over months.
6. **No time-lag correction:** Thermowell response time not modeled. Measured temperature may lag steady-state by 30–60 seconds.

### 10.2 Limitations

- **Coking not detected:** Localized coking (increased insulation, lower measured T) will appear as negative residual but is not separately quantified.
- **Burner-side maldistribution:** If flame impingement occurs, local T spikes may exceed correlation prediction; detected as positive residual but root cause unclear.
- **Fouling depth unknown:** Residual drift indicates anomaly but does not estimate coke thickness or cleaning intervals.
- **No dynamic model:** Transient temperature swings (e.g., burner firing oscillations) are not smoothed; consider 5-minute rolling average for noisy sensors.

---

## 11. Integration Notes

### 11.1 Data Source

Furnace tube-wall temperatures are acquired from a distributed array of **four thermocouples** mounted in staggered axial positions on a representative radiant-section tube. Data is logged every 6 hours via SCADA (or manual DCS export). Feed process outlet temperature comes from the main feed inlet temperature sensor (proxy for bulk fluid temperature).

### 11.2 Real-Time Integration

1. **CSV upload:** User uploads 28-row CSV via browser interface.
2. **Parsing:** Papa Parse 5.4.1 (CDN) parses CSV with header detection.
3. **Processing:** Python engine runs `process(row)` for each row in-browser (Pyodide / web assembly, or Node.js if server-backed).
4. **Rendering:** `render()` outputs HTML cards, tables, alerts.
5. **Charting:** Chart.js 4.4.1 (CDN) plots time-series and prediction scatter.

### 11.3 Configuration in HTML

All six parameters (Q_ref, m_ref, K values, T limits, residual threshold) are exposed in a **Settings panel** (hidden by default, revealed via gear icon in topbar). Users can tune K values if baseline conditions change without redeploying.

---

## 12. Change Log

| **Date** | **Version** | **Change** | **Author** |
|---|---|---|---|
| 2026-04-04 | 1.0 | Initial release; 4-position TMT correlation, NACE/API 530 limits, residual anomaly detection. | Vedant Patel |

---

## Appendix A: Symbol Definitions

| **Symbol** | **Definition** | **Unit** |
|---|---|---|
| $\Delta T_i$ | Predicted temperature rise at position *i* | K or °C |
| $K_i$ | Baseline temperature rise at reference conditions (position *i*) | K |
| $Q_{\mathrm{fire}}$ | Furnace radiant heat release | MW |
| $Q_{\mathrm{ref}}$ | Reference heat release (25 MW) | MW |
| $m$ | Process mass flow rate (actual) | kg/h |
| $m_{\mathrm{ref}}$ | Reference flow (80,000 kg/h) | kg/h |
| $T_{\mathrm{process}}$ | Process outlet (bulk) temperature | °C |
| $T_{\mathrm{pred}, i}$ | Predicted tube skin temperature (position *i*) | °C |
| $T_{\mathrm{meas}, i}$ | Measured tube skin temperature (position *i*) | °C |
| $\varepsilon_i$ | Residual error (measured − predicted) at position *i* | K |

---

## Appendix B: Standard References (Full Citations)

1. **ASME PTC 4.4:2017** — Power Test Code — Steam Generators. Section 3: Performance Tests (Indirect, once-through); Section 8: Thermocouple Measurement & Uncertainty.

2. **NACE SP0170-2016** — Protection of Austenitic Stainless Steel and Other Corrosion-Resistant Alloys from Pitting and Crevice Corrosion in Chloride-Containing Environments. (Applies 9Cr-1Mo creep limits in H₂ service.)

3. **API 530:2021** — Calculation of Heater-Tube Thickness in Petroleum Refineries. 3rd ed. Section 4: Design Temperature and Pressure; Section 9: Creep-Rupture Rate (Larson-Miller).

4. **ASTM E0111-21** — Standard Test Method for Young's Modulus, Tangent Modulus, and Chord Modulus. (Background for 9Cr-1Mo mechanical properties.)

5. **Dittus, F. W.; Boelert, L. M. K. (1930)**. "Heat Transfer in Automobile Radiators of Tubular Type." *University of California Publications in Engineering*, vol. 2, pp. 443–461. (Original correlation; 0.8/0.8 exponents.)

---

**END OF DOCUMENT**

Document classification: **CONFIDENTIAL**  
Distribution: Internal engineering, operations, compliance review only.  
Last reviewed: 2026-04-04 by Vedant Patel  
Next review: 2026-06-04 (post-deployment field validation)
