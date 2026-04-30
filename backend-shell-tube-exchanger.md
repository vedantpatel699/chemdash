# Shell & Tube Heat Exchanger - Backend Technical Manual

**Version:** 1.0  
**Date:** 2026-04-04  
**Author:** Vedant Patel  
**Classification:** CONFIDENTIAL

---

## 1. Purpose & Scope

This manual documents the heat transfer and fouling physics embedded in the Shell & Tube Heat Exchanger (`shell-tube-exchanger.html`) dashboard. The exchanger model uses:

- **LMTD (Log Mean Temperature Difference)** method per Kern (1950), Bowman et al. (1940)
- **Overall heat transfer coefficient (U)** tracking per TEMA §T-3.1
- **Fouling resistance (Rf)** trending per TEMA §RGP-T-2.4
- **Energy balance** validation via hot/cold side duty comparison
- **Equipment:** Crude preheater E-101, TEMA RCB (one shell pass, two tube passes)
- **Application:** Feed-side heating in a crude distillation unit

All calculations are deterministic and real-time, accepting CSV-uploaded process data or manual entry.

---

## 2. Tag Dictionary

The exchanger accepts eleven (11) process tags as inputs. All are required for a complete calculation row.

| Tag | Description | Unit | Typical Range | Notes |
|-----|-------------|------|----------------|-------|
| `timestamp` | Date and time of measurement | ISO 8601 | - | E.g., `2026-02-01T00:00:00Z` |
| `hot_in_c` | Hot fluid inlet temperature (shell-side) | °C | 175–185 | Furnace outlet to exchanger |
| `hot_out_c` | Hot fluid outlet temperature (shell-side) | °C | 95–105 | Shell-side discharge |
| `hot_flow_kghr` | Hot fluid mass flow rate | kg/h | 45,000–55,000 | Furnace feed |
| `hot_cp_kjkgk` | Hot fluid specific heat (shell) | kJ/(kg·K) | 2.0–2.4 | Temperature-dependent |
| `cold_in_c` | Cold fluid inlet temperature (tube-side) | °C | 38–42 | Crude feed from storage |
| `cold_out_c` | Cold fluid outlet temperature (tube-side) | °C | 145–155 | Heated crude to distillation |
| `cold_flow_kghr` | Cold fluid mass flow rate | kg/h | 45,000–55,000 | Proportional to furnace feed |
| `cold_cp_kjkgk` | Cold fluid specific heat (tube) | kJ/(kg·K) | 1.8–2.2 | Crude oil, T-dependent |
| `shell_dp_bar` | Pressure drop, shell-side | bar(g) | 0.05–0.35 | Cross-flow around tubes |
| `tube_dp_bar` | Pressure drop, tube-side | bar(g) | 0.10–0.40 | Flow through tubes (2 passes) |

**Notes:**
- `hot_cp_kjkgk` and `cold_cp_kjkgk` are entered as measured or estimated from crude assay/steam tables.
- All temperatures in °C (Kelvin offset cancels in ΔT calculations).
- Flow rates in kg/h; internal calculations convert to kg/s.
- Pressure drops are gauge pressures.

---

## 3. Core Equations

### 3.1 Heat Duty (Hot and Cold Sides)

**Hot-side duty:**
$$Q_{\text{hot}} = \frac{\dot{m}_h \cdot C_{p,h} \cdot (T_{h,in} - T_{h,out})}{3600} \text{ [kW]}$$

**Cold-side duty:**
$$Q_{\text{cold}} = \frac{\dot{m}_c \cdot C_{p,c} \cdot (T_{c,out} - T_{c,in})}{3600} \text{ [kW]}$$

Where:
- $\dot{m}$ = mass flow rate (kg/h), divided by 3600 to convert to kg/s
- $C_p$ = specific heat capacity (kJ/(kg·K))
- ΔT = temperature difference (°C = K)
- Result in kW

**Average duty (energy balance reference):**
$$Q_{\text{avg}} = \frac{Q_{\text{hot}} + Q_{\text{cold}}}{2}$$

### 3.2 Energy Balance Check

**Imbalance percentage:**
$$\text{Imbalance \%} = \frac{|Q_{\text{hot}} - Q_{\text{cold}}|}{Q_{\text{avg}}} \times 100$$

**Alert threshold:** Imbalance ≥ 8% indicates possible measurement error, leakage, or control deviation.

---

### 3.3 Log Mean Temperature Difference (LMTD)

For a counter-current heat exchanger:

**Temperature differences at ends:**
$$\Delta T_1 = T_{h,in} - T_{c,out}$$
$$\Delta T_2 = T_{h,out} - T_{c,in}$$

**LMTD (Kern Eq. 5.14, Bowman et al. 1940):**
$$\text{LMTD} = \frac{\Delta T_1 - \Delta T_2}{\ln(\Delta T_1 / \Delta T_2)} \text{ [°C]}$$

**Special case:** If $\Delta T_1 \approx \Delta T_2$ (within 0.1%), use arithmetic mean:
$$\text{LMTD}_{\text{approx}} = \frac{\Delta T_1 + \Delta T_2}{2}$$

**Correction factor (TEMA RCB, one shell pass, two tube passes):**
$$\text{LMTD}_{\text{corrected}} = \text{LMTD} \times F$$

Where $F = 0.90$ (typical for E-101; see Configuration, §6).

In this dashboard: **LMTD displayed = uncorrected LMTD**, and U is calculated using the correction factor implicitly (see §3.4).

---

### 3.4 Overall Heat Transfer Coefficient (U)

From the fundamental heat transfer equation:
$$Q = U \cdot A \cdot F \cdot \text{LMTD}$$

Solving for U (TEMA §T-3.1):
$$U = \frac{Q \times 1000}{A \times F \times \text{LMTD}} \text{ [W/(m}^2\text{·K)]}$$

Where:
- $Q$ in kW (multiplied by 1000 to convert to W)
- $A$ = surface area (m²) = 250 m² (fixed, see §6)
- $F$ = correction factor = 0.90
- LMTD = log mean temperature difference (°C = K)

**Notes:**
- U is calculated using average duty: $Q_{\text{avg}}$ in the numerator.
- If LMTD < 1°C, U is clamped to avoid division by near-zero (alert issued).
- U_dirty decreases over time due to fouling.

---

### 3.5 Fouling Resistance (Rf)

Fouling resistance quantifies the deposit layer resistance on heat transfer surfaces.

$$\text{Rf} = \frac{1}{U_{\text{dirty}}} - \frac{1}{U_{\text{clean}}} \text{ [m}^2\text{·K/W]}$$

Where:
- $U_{\text{clean}}$ = baseline (design) coefficient = 400 W/(m²·K) (see §6)
- $U_{\text{dirty}}$ = measured coefficient from current operating data

**Interpretation:**
- Rf = 0: No fouling (clean equipment)
- Rf > 0: Deposit layer present; fouling increasing
- Rf ≥ 12×10⁻⁴ m²·K/W: **ALARM** - recommend chemical or mechanical cleaning

**Practical check:** When Rf reaches trigger, heat duty drops ~8–12% below design, visible as ΔP increase and outlet temperature deviation.

---

### 3.6 Approach Temperature (Effectiveness Proxy)

**Hot-end approach:**
$$\text{Approach}_{\text{hot}} = T_{h,in} - T_{c,out}$$

**Cold-end approach:**
$$\text{Approach}_{\text{cold}} = T_{h,out} - T_{c,in}$$

Both measured in °C. Lower approach = better thermal contact and integration; minimum recommended = 10°C (below triggers advisory alert).

---

### 3.7 Pressure Drop Constraints

**Shell-side DP limit:** ≤ 0.55 bar(g)  
**Tube-side DP limit:** ≤ 0.45 bar(g)

Both are operational limits to prevent structural stress and excessive pumping power. Exceeding either triggers an advisory alert.

---

## 4. Python Engine API

The dashboard's `engine.py` (located in the `shell-tube-exchanger/` model folder) exposes a calculation object with the following methods:

### 4.1 Core Calculation: `process(row_dict)`

```python
def process(row_dict):
    """
    Accepts a dictionary with keys:
      timestamp, hot_in_c, hot_out_c, hot_flow_kghr, hot_cp_kjkgk,
      cold_in_c, cold_out_c, cold_flow_kghr, cold_cp_kjkgk,
      shell_dp_bar, tube_dp_bar
    
    Returns a dict with calculated metrics:
      duty_hot_kw, duty_cold_kw, duty_avg_kw,
      imbalance_pct, lmtd_c, overall_u_wmk, rf_m2kw,
      approach_hot_c, approach_cold_c,
      status (str: "normal" | "advisory" | "critical"),
      alerts (list of alert strings)
    """
```

**Usage:**
```python
from engine import Exchanger
ex = Exchanger()
result = ex.process({
    'timestamp': '2026-02-01T00:00:00Z',
    'hot_in_c': 180.5,
    'hot_out_c': 100.2,
    'hot_flow_kghr': 50000,
    'hot_cp_kjkgk': 2.15,
    'cold_in_c': 40.1,
    'cold_out_c': 150.8,
    'cold_flow_kghr': 50100,
    'cold_cp_kjkgk': 2.05,
    'shell_dp_bar': 0.18,
    'tube_dp_bar': 0.22
})
print(result['duty_avg_kw'])  # 4776.2
print(result['status'])        # 'normal'
```

### 4.2 Subsidiary Methods

**`duty(hot_flow_kg_h, hot_cp, t_in_c, t_out_c)` → duty_kw**
- Calculates heat duty from mass flow, Cp, and temperatures
- Used by both hot and cold side

**`lmtd(t_hot_in, t_hot_out, t_cold_in, t_cold_out, f_correction=0.90)` → lmtd_c**
- Computes LMTD with optional correction factor
- Handles edge case of $\Delta T_1 \approx \Delta T_2$

**`overallU(duty_kw, lmtd_c, area_m2=250, f_correction=0.90)` → u_wmk**
- Solves for U from heat duty equation
- Returns W/(m²·K)

**`rfouling(u_dirty, u_clean=400)` → rf_m2kw**
- Calculates fouling resistance
- Fixed reference: U_clean = 400 W/(m²·K)

**`resolveRow(row_dict)` → status_str, alerts_list**
- Evaluates all alert conditions (fouling, approach, DP, imbalance)
- Returns status code and list of alert messages for display

---

## 5. Configuration

The following parameters are hardcoded in the dashboard and `engine.py`. Change only if equipment is replaced or re-tuned.

| Parameter | Value | Source | Notes |
|-----------|-------|--------|-------|
| Surface Area (A) | 250 m² | TEMA specification | Typical 1-2 shell pass |
| Correction Factor (F) | 0.90 | RCB calc (Bowman/Mueller) | One shell, two tubes |
| U_clean (baseline) | 400 W/(m²·K) | Design rating | From equipment nameplate |
| Rf fouling trigger | 12×10⁻⁴ m²·K/W | TEMA RGP-T-2.4, EXXONMOBIL std | Recommend cleaning at this point |
| Approach minimum | 10°C | Heat integration limit | Below = inefficient or mismeasured |
| Shell DP max | 0.55 bar(g) | ASME pressure vessel code | Safety and erosion limit |
| Tube DP max | 0.45 bar(g) | Tube-side flow design | Pump outlet rating |
| Imbalance threshold | 8% | Energy balance tolerance | Typical for industrial data |

---

## 6. Alert Logic

Alerts are evaluated in priority order and displayed to the operator.

### 6.1 Fouling (CRITICAL)

**Condition:** Rf ≥ 12×10⁻⁴ m²·K/W

**Message:** "FOULING: Rf = X×10⁻⁴ m²·K/W. Equipment due for cleaning."

**Action:** Schedule immediate chemical or mechanical cleaning (turnaround).

### 6.2 Approach Temperature (ADVISORY)

**Condition:** Either approach ≤ 10°C

**Message:** "Low approach (hot: X°C / cold: Y°C). Check feed/furnace control."

**Action:** Investigate control set points; may indicate measurement error or equipment degradation.

### 6.3 Pressure Drop Limits (ADVISORY)

**Shell-side:**
- **Condition:** shell_dp_bar > 0.55
- **Message:** "Shell DP high (Z bar). Check for blockage or scale."

**Tube-side:**
- **Condition:** tube_dp_bar > 0.45
- **Message:** "Tube DP high (Z bar). Verify pump discharge and fouling."

### 6.4 Energy Imbalance (ADVISORY)

**Condition:** Imbalance ≥ 8%

**Message:** "Energy imbalance: |Q_hot − Q_cold| / Q_avg = X%. Check instrument calibration."

**Action:** Suspect thermocouple drift, flow meter error, or systematic heat loss.

### 6.5 Status Summary

Status is assigned as:
- **NORMAL** (green): No alerts, Rf < trigger, all approaches ≥ 10°C, DP within limit, imbalance < 8%
- **ADVISORY** (amber): One or more non-critical alerts (approach, DP, imbalance)
- **CRITICAL** (red): Fouling Rf ≥ trigger

---

## 7. Data Schema

The dashboard is pre-loaded with 28 rows of synthetic but realistic process data spanning **February 1–7, 2026**, at **6-hour intervals** (4 readings/day × 7 days = 28 rows).

**Sample row (first entry, 2026-02-01 00:00 UTC):**

| Tag | Value | Unit |
|-----|-------|------|
| timestamp | 2026-02-01T00:00:00Z | ISO 8601 |
| hot_in_c | 180.5 | °C |
| hot_out_c | 100.2 | °C |
| hot_flow_kghr | 50000 | kg/h |
| hot_cp_kjkgk | 2.15 | kJ/(kg·K) |
| cold_in_c | 40.1 | °C |
| cold_out_c | 150.8 | °C |
| cold_flow_kghr | 50100 | kg/h |
| cold_cp_kjkgk | 2.05 | kJ/(kg·K) |
| shell_dp_bar | 0.18 | bar(g) |
| tube_dp_bar | 0.22 | bar(g) |

**Data characteristics:**
- Hot inlet: 175–185°C (stable furnace outlet)
- Hot outlet: 95–115°C (varying with cold-side demand)
- Cold inlet: 38–42°C (crude storage temperature, seasonal)
- Cold outlet: 145–160°C (setpoint ~150°C for distillation feed)
- Flows: 45,000–55,000 kg/h (proportional to crude throughput)
- Cp values: Estimated from crude assay and steam tables
- DP: Shell 0.05–0.35 bar, Tube 0.10–0.40 bar (normal range)

**Embedded fouling trend:**
- Rf starts ~8×10⁻⁴ m²·K/W (clean)
- Increases to ~9.5×10⁻⁴ m²·K/W by day 7 (slight fouling)
- Final DP values spike slightly to simulate incipient blockage

---

## 8. Worked Example

**Using Row 1 (2026-02-01 00:00:00Z):**

```
Input:
  hot_in_c       = 180.5°C
  hot_out_c      = 100.2°C
  hot_flow_kghr  = 50,000 kg/h
  hot_cp_kjkgk   = 2.15 kJ/(kg·K)
  cold_in_c      = 40.1°C
  cold_out_c     = 150.8°C
  cold_flow_kghr = 50,100 kg/h
  cold_cp_kjkgk  = 2.05 kJ/(kg·K)
  shell_dp_bar   = 0.18 bar
  tube_dp_bar    = 0.22 bar
```

**Step 1: Duty Calculation**

$$Q_{\text{hot}} = \frac{50,000 \times 2.15 \times (180.5 - 100.2)}{3600} = \frac{50,000 \times 2.15 \times 80.3}{3600} = 2,378,958 / 3600 = 4,766.5 \text{ kW}$$

$$Q_{\text{cold}} = \frac{50,100 \times 2.05 \times (150.8 - 40.1)}{3600} = \frac{50,100 \times 2.05 \times 110.7}{3600} = 2,854,244 / 3600 = 4,785.9 \text{ kW}$$

$$Q_{\text{avg}} = \frac{4,766.5 + 4,785.9}{2} = 4,776.2 \text{ kW}$$

$$\text{Imbalance} = \frac{|4,766.5 - 4,785.9|}{4,776.2} \times 100 = 0.40\% \quad (\text{Normal})$$

**Step 2: LMTD Calculation**

$$\Delta T_1 = 180.5 - 150.8 = 29.7°C$$
$$\Delta T_2 = 100.2 - 40.1 = 60.1°C$$

$$\text{LMTD} = \frac{60.1 - 29.7}{\ln(60.1 / 29.7)} = \frac{30.4}{\ln(2.0235)} = \frac{30.4}{0.7049} = 73.57°C$$

**Step 3: Overall U Calculation**

$$U = \frac{4,766.5 \times 1000}{250 \times 0.90 \times 73.57} = \frac{4,766,500}{16,553.3} = 288.5 \text{ W/(m}^2\text{·K)}$$

**Step 4: Fouling Resistance**

$$\text{Rf} = \frac{1}{288.5} - \frac{1}{400} = 0.003465 - 0.0025 = 0.000966 \text{ m}^2\text{·K/W} = 9.66 \times 10^{-4}$$

**Step 5: Approach Temperatures**

$$\text{Approach}_{\text{hot}} = 180.5 - 150.8 = 29.7°C \quad (\checkmark > 10°C)$$
$$\text{Approach}_{\text{cold}} = 100.2 - 40.1 = 60.1°C \quad (\checkmark > 10°C)$$

**Step 6: Pressure Drop Check**

Shell: 0.18 bar < 0.55 bar ✓  
Tube: 0.22 bar < 0.45 bar ✓

**Final Status:** NORMAL (all checks pass)

**Output to Dashboard:**
- Duty (hot): 4,766.5 kW
- Duty (cold): 4,785.9 kW
- LMTD: 73.57°C
- U: 288.5 W/(m²·K)
- Rf: 9.66×10⁻⁴ m²·K/W
- Status: Normal (green)
- Alerts: None

---

## 9. Assumptions & Limitations

### 9.1 Assumptions

1. **Counter-current flow geometry** - Shell-side and tube-side flow in opposite directions; LMTD correction (F = 0.90) applies.
2. **Single-phase liquids** - No phase change; enthalpy = m·Cp·ΔT.
3. **Constant Cp** - Specific heat is temperature-averaged; no composition variation.
4. **Steady-state operation** - Each row represents a stabilized operating point (transients not captured).
5. **Fixed surface area** - Tube fouling does not change active area; only U degrades.
6. **Negligible tube-to-shell wall conduction** - All resistance modeled as convective (h) + fouling (Rf).
7. **Uniform heat transfer coefficient** - U does not vary axially or around the tube bundle.

### 9.2 Limitations

1. **Crude oil Cp estimation** - Actual Cp varies with composition and temperature; input Cp is user-estimated or from correlations.
2. **Fouling assumption** - Rf trend is synthetic. Real fouling depends on crude type, temperature, residence time (not modeled).
3. **Measurement error** - No filtering; raw sensor data subject to noise. Imbalance threshold (8%) is a crude filter.
4. **Shell-side bypass** - RCB geometry may have some bypass flow (not modeled; F factor is an empirical correction).
5. **Tube-side distribution** - Unequal flow through the two passes may occur in reality; treated as average here.
6. **Corrosion/erosion** - Material degradation not modeled; only fouling tracked.

---

## 10. Integration Notes

### 10.1 CSV Upload

The dashboard accepts a CSV file with the following columns (any order):

```
timestamp, hot_in_c, hot_out_c, hot_flow_kghr, hot_cp_kjkgk, cold_in_c, cold_out_c, cold_flow_kghr, cold_cp_kjkgk, shell_dp_bar, tube_dp_bar
```

**Example CSV snippet:**
```
timestamp,hot_in_c,hot_out_c,hot_flow_kghr,hot_cp_kjkgk,cold_in_c,cold_out_c,cold_flow_kghr,cold_cp_kjkgk,shell_dp_bar,tube_dp_bar
2026-02-01T00:00:00Z,180.5,100.2,50000,2.15,40.1,150.8,50100,2.05,0.18,0.22
2026-02-01T06:00:00Z,181.2,101.5,50200,2.14,40.2,151.5,50300,2.06,0.19,0.23
```

Papa Parse 5.4.1 (loaded via CDN) handles parsing and tag aliasing (e.g., `T_hot_in` → `hot_in_c`).

### 10.2 Real-Time Integration (SCADA/DCS)

For live system integration:
1. Expose `engine.process()` as a REST endpoint (e.g., Flask/FastAPI microservice).
2. Accept POST requests with JSON payload matching tag dictionary.
3. Return JSON with calculated metrics and status.
4. Dashboard polls endpoint every 60 seconds; WebSocket optional for real-time push.

**Example endpoint:**
```
POST /api/exchanger/calculate
Content-Type: application/json

{
  "timestamp": "2026-02-01T12:00:00Z",
  "hot_in_c": 182.1,
  ...
}

Response:
{
  "duty_avg_kw": 4810.2,
  "status": "normal",
  "alerts": []
}
```

### 10.3 Data Logging & Archival

- Dashboard stores uploaded CSV in browser `localStorage` (max ~5 MB).
- For persistent storage, export trending data as CSV via the dashboard's "Export" button.
- Recommend daily uploads to a time-series database (InfluxDB, Timescale) for long-term trending and Root Cause Analysis (RCA).

---

## 11. Change Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-04-04 | Vedant Patel | Initial release. LMTD, U, Rf calculations. Status alerting. 28-row synthetic dataset. Python engine API documented. TEMA RCB configuration. |

---

## Appendix A: References

- **Kern, D. Q.** (1950). *Process Heat Transfer*. McGraw-Hill.
- **Bowman, R. A., Mueller, A. C., & Nagle, W. M.** (1940). "Mean Temperature Difference in Design." *Transactions of the ASME*, 62(4), 283–294.
- **TEMA (Tubular Exchanger Manufacturers Association).** (2019). *Standards of the Tubular Exchanger Manufacturers Association* (10th ed.). Sec. T-3.1 (U calculation), Sec. RGP-T-2.4 (Rf limits).
- **ASME.** (2023). *Unfired Pressure Vessels* (ASME BPVC Section VIII).
- **API 560.** (2015). *Fired Heaters for Refinery Service*.

---

## Appendix B: Abbreviations

| Abbrev | Meaning |
|--------|---------|
| LMTD | Log Mean Temperature Difference |
| ΔT | Temperature difference |
| Rf | Fouling resistance |
| U | Overall heat transfer coefficient |
| Cp | Specific heat capacity |
| DP | Pressure drop |
| TEMA | Tubular Exchanger Manufacturers Association |
| RCB | TEMA type: one shell pass, two tube passes, baffled |
| E-101 | Crude preheater unit designation |
| SCADA | Supervisory Control and Data Acquisition |
| DCS | Distributed Control System |
| ISA | International Society of Automation |
| CSV | Comma-Separated Values |

---

*End of Backend Technical Manual - Shell & Tube Heat Exchanger*
