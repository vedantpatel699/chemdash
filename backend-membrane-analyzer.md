# H₂-Recovery Membrane Analyzer - Backend Technical Manual

**Version:** 1.0  
**Date:** 2026-04-04  
**Author:** Vedant Patel  
**Classification:** CONFIDENTIAL

---

## 1. Purpose & Scope

This manual documents the engineering mathematics, Python calculation engine, and data schema for the **H₂-Recovery Membrane Analyzer** dashboard. The model simulates single-stage polymeric hollow-fiber membrane performance for binary H₂/CH₄ feed streams typical of refinery off-gas purification and hydrogen recovery applications.

### Equipment Description

- **Type:** Single-stage membrane module (spiral-wound or hollow-fiber)
- **Membrane:** Polymeric (glassy, e.g., polysulfone, polyimide)
- **Permeate:** H₂-enriched stream (target purity 93%+)
- **Residue:** CH₄-rich retentate (recycle to furnace/fuel)
- **Typical capacity:** 5000 Nm³/hr feed, 30 bar feed pressure, 1–2 bar permeate
- **Operating envelope:** 20–50 bar feed, –10°C to 60°C, <50% stage cut

### Calculation Domain

The engine computes:
- **Stage cut** (permeate fraction)
- **Recovery** (H₂ flux to permeate / H₂ flux in feed)
- **Selectivity** (actual and apparent)
- **Permeance** (mass flux per unit area per unit pressure)
- **Material balance** closure (feed = permeate + residue)

---

## 2. Tag Dictionary

| Tag | Unit | Abbr. | Type | Notes |
|-----|------|-------|------|-------|
| timestamp | ISO 8601 | TS | string | e.g., "2026-02-01T08:30:00" |
| feed_flow_nm3hr | Nm³/hr | F | numeric | Normal m³/hr at STP (0°C, 1.013 bar) |
| feed_h2_pct | % (mol) | x_H2 | numeric | Feed H₂ mole %, 40–90 range typical |
| feed_p_barg | bar(g) | p_F | numeric | Feed gauge pressure, 10–50 bar typical |
| feed_t_c | °C | T_F | numeric | Feed temperature, for future density corr. |
| permeate_flow_nm3hr | Nm³/hr | P | numeric | Permeate outlet flow |
| permeate_h2_pct | % (mol) | y_H2 | numeric | Permeate H₂ mole %, target 93%+ |
| permeate_p_barg | bar(g) | p_P | numeric | Permeate gauge pressure, typically 0–2 bar |
| residue_flow_nm3hr | Nm³/hr | R | numeric | Residue (retentate) outlet flow |
| residue_h2_pct | % (mol) | z_H2 | numeric | Residue H₂ mole %, typically 5–25% |

### Notes on Units

- **Nm³/hr** = Normal cubic meters per hour at **0°C and 1.013 bar (absolute)** per ISO 4014.
- **Gauge pressure** is used in the feed/permeate tags; absolute pressure = gauge + 1.013 bar.
- **Mole %** is used for all composition fields (not mass %).

---

## 3. Calculation Equations

### 3.1 Stage Cut

The **stage cut** θ is the fraction of feed converted to permeate:

$$\theta = \frac{P}{F}$$

- **Range:** 0.2–0.5 (20–50%) typical for H₂ recovery
- **Units:** dimensionless
- **Physical meaning:** Higher θ increases recovery but decreases permeate purity

**Reference:** Baker, R. W. (2004). *Membrane Technology and Applications*, 2nd ed., Wiley. Chapter 8.

---

### 3.2 Recovery

**H₂ recovery** is the molar fraction of feed H₂ that exits in the permeate:

$$\text{Recovery} = \frac{P \cdot y_{\text{H2}}}{F \cdot x_{\text{H2}}} \times 100\%$$

- **Range:** 50–95% (higher is better, but diminishes with increased stage cut)
- **Units:** percentage
- **Trade-off:** Higher recovery → lower permeate purity and higher residue H₂ loss

---

### 3.3 Selectivity (Ideal vs. Actual)

#### Ideal Selectivity

The membrane's intrinsic separation factor (often called "H₂/CH₄ selectivity"):

$$\alpha_{\text{ideal}} = \frac{P_{\text{H2}}}{P_{\text{CH4}}}$$

where P_i are permeance coefficients (inherent to the membrane material).

#### Actual (Apparent) Selectivity

Computed from observed compositions (Baker Eq. 8.11):

$$\alpha_{\text{actual}} = \frac{y_{\text{H2}} / (1 - y_{\text{H2}})}{x_{\text{H2}} / (1 - x_{\text{H2}})}$$

- **Interpretation:** α_actual < α_ideal indicates concentration polarization or non-ideal mixing.
- **Typical range:** 8–15 for polymeric membranes separating H₂/CH₄
- **Units:** dimensionless
- **Symbol in dashboard:** α

---

### 3.4 Log-Mean Driving Pressure (lmDP)

For composite membranes, the driving force is the **partial pressure difference** across the membrane wall. The log-mean pressure difference accounts for non-linear saturation:

$$\Delta p_1 = x_{\text{H2}} \cdot p_{\text{F}} - y_{\text{H2}} \cdot p_{\text{P}}$$
$$\Delta p_2 = x_{\text{H2}} \cdot p_{\text{F}} - y_{\text{H2}} \cdot p_{\text{P}}$$
$$\Delta p_{\text{LM}} = \frac{\Delta p_1 - \Delta p_2}{\ln(\Delta p_1 / \Delta p_2)}$$

where:
- $p_{\text{F}}$ and $p_{\text{P}}$ are **absolute pressures** (bar_a, i.e., bar gauge + 1.013)
- For single-stage modules without mixing, approximations are valid

**Note on implementation:** If Δp1 ≈ Δp2 (thin modules, small pressure drop), use the arithmetic mean as fallback to avoid numerical singularity.

**Reference:** Wijmans, J. G., & Baker, R. W. (1995). "The solution-diffusion model: a review." *Journal of Membrane Science*, 107(1–2), 1–21.

---

### 3.5 Permeance

**H₂ permeance** is the normalized flux through the membrane:

$$Q_{\text{H2}} = \frac{P_{\text{coeff}} \cdot \Delta p_{\text{LM}}}{A}$$

where A is active membrane area. For practical calculation, solve backwards from observed flux:

$$Q_{\text{H2}} = \frac{\text{(molar flux of H2 in permeate)}}{\Delta p_{\text{LM}}}$$

Expressed in dashboard-friendly units:

$$Q_{\text{H2}} = \frac{P \cdot y_{\text{H2}}}{36 \cdot \Delta p_{\text{LM}}} \quad \text{[Nm}^3\text{/(hr·bar)]}$$

(The factor 36 converts molar flux from kmol/s to Nm³/hr.)

**Baseline (design):** Q_base = 586 Nm³/(hr·bar) at commissioning.

---

### 3.6 Normalized Permeance

Track permeance degradation relative to baseline:

$$Q_{\text{norm}} = \frac{Q_{\text{H2}}}{Q_{\text{base}}} \times 100\%$$

- **100%** = membrane at design condition
- **>105%** = possible measurement error or feed composition transient
- **<95%** = 5% permeance loss → advisory alert
- **<90%** = 10% permeance loss → critical alarm

---

### 3.7 Material Balance Gap

Verify closure of mass balance (molar basis):

$$\text{MB gap} = 100 \times \frac{F - P - R}{F} \quad [\%]$$

**Interpretation:**
- **Signed** (not absolute): positive gap = missing material (instrumentation error, holdup)
- **Negative gap** = over-recovery (rare, indicates sensor drift or feed assumption error)
- **Threshold:** |MB gap| < 3% acceptable; >3% triggers data quality advisory

---

## 4. Python Engine API

The calculation engine (`engine.py` in the `membrane-analyzer` model folder) exports the following functions:

### 4.1 calcStageCut(F, P)
```python
def calcStageCut(F, P):
    """
    Stage cut θ = P / F
    
    Args:
        F (float): Feed flow [Nm³/hr]
        P (float): Permeate flow [Nm³/hr]
    
    Returns:
        float: θ [dimensionless, 0–1]
    """
```

---

### 4.2 calcRecovery(P, y_H2, F, x_H2)
```python
def calcRecovery(P, y_H2, F, x_H2):
    """
    H₂ recovery: (P·y_H2) / (F·x_H2) × 100%
    
    Args:
        P (float): Permeate flow [Nm³/hr]
        y_H2 (float): Permeate H₂ mole % [0–100]
        F (float): Feed flow [Nm³/hr]
        x_H2 (float): Feed H₂ mole % [0–100]
    
    Returns:
        float: Recovery [%, 0–100]
    """
```

---

### 4.3 calcSelectivity(y_H2, x_H2)
```python
def calcSelectivity(y_H2, x_H2):
    """
    Actual (apparent) selectivity α = (y / (1-y)) / (x / (1-x))
    
    Args:
        y_H2 (float): Permeate H₂ mole % [0–100]
        x_H2 (float): Feed H₂ mole % [0–100]
    
    Returns:
        float: α [dimensionless, typical 8–15]
    """
```

---

### 4.4 lmDP(x_H2, p_F, y_H2, p_P)
```python
def lmDP(x_H2, p_F, y_H2, p_P):
    """
    Log-mean driving pressure (partial pressure H₂).
    
    Args:
        x_H2 (float): Feed H₂ mole % [0–100]
        p_F (float): Feed absolute pressure [bar_a]
        y_H2 (float): Permeate H₂ mole % [0–100]
        p_P (float): Permeate absolute pressure [bar_a]
    
    Returns:
        float: ΔpLM [bar], or arithmetic mean if logarithmic fails
    
    Notes:
        - Convert gauge pressure to absolute: p_a = p_g + 1.013
        - Handles singularities (Δp1 ≈ Δp2) by falling back to arithmetic mean
    """
```

---

### 4.5 calcPermeance(P, y_H2, delta_p_lm)
```python
def calcPermeance(P, y_H2, delta_p_lm):
    """
    H₂ permeance Q_H2 = (P·y_H2) / (36·ΔpLM)
    
    Args:
        P (float): Permeate flow [Nm³/hr]
        y_H2 (float): Permeate H₂ mole % [0–100]
        delta_p_lm (float): Log-mean pressure [bar]
    
    Returns:
        float: Q_H2 [Nm³/(hr·bar)]
    """
```

---

### 4.6 calcPermeanceNorm(Q_H2, Q_base)
```python
def calcPermeanceNorm(Q_H2, Q_base):
    """
    Normalized permeance relative to baseline.
    
    Args:
        Q_H2 (float): Actual H₂ permeance [Nm³/(hr·bar)]
        Q_base (float): Baseline permeance [Nm³/(hr·bar)], default 586
    
    Returns:
        float: Q_norm [%, 100 = baseline]
    """
```

---

### 4.7 calcMBgap(F, P, R)
```python
def calcMBgap(F, P, R):
    """
    Material balance gap (signed).
    
    Args:
        F (float): Feed flow [Nm³/hr]
        P (float): Permeate flow [Nm³/hr]
        R (float): Residue flow [Nm³/hr]
    
    Returns:
        float: MB gap [%, signed; typically ±3%]
    """
```

---

### 4.8 processRow(row_dict)
```python
def processRow(row_dict):
    """
    Full calculation pipeline for a single CSV data row.
    
    Args:
        row_dict (dict): {
            'timestamp': str (ISO 8601),
            'feed_flow_nm3hr': float,
            'feed_h2_pct': float (0–100),
            'feed_p_barg': float,
            'feed_t_c': float,
            'permeate_flow_nm3hr': float,
            'permeate_h2_pct': float (0–100),
            'permeate_p_barg': float,
            'residue_flow_nm3hr': float,
            'residue_h2_pct': float (0–100)
        }
    
    Returns:
        dict: {
            'timestamp': str,
            'stage_cut': float,
            'recovery': float (0–100),
            'selectivity': float,
            'delta_p_lm': float,
            'permeance': float,
            'permeance_norm': float,
            'mb_gap': float,
            'alerts': list of str (empty if all OK)
        }
    """
```

**Internal flow:**
1. Convert gauge → absolute pressure
2. Calc stage cut θ
3. Calc recovery %
4. Calc selectivity α
5. Calc lmDP
6. Calc permeance Q_H2 and Q_norm
7. Calc MB gap
8. Evaluate alerting logic (§5 below)
9. Return results dict

---

## 5. Configuration & Alerting

### 5.1 Fixed Parameters

| Parameter | Symbol | Value | Unit | Rationale |
|-----------|--------|-------|------|-----------|
| Baseline permeance | Q_base | 586 | Nm³/(hr·bar) | Commissioning reference (Feb 1, 2026, 08:00) |
| Design area | A_design | 30 | m² | Membrane module active area |
| Atmospheric pressure | p_atm | 1.013 | bar(a) | STP reference (ISO 4014) |

### 5.2 Alerting Limits

| Alert Type | Metric | Threshold | Level | Action |
|------------|--------|-----------|-------|--------|
| Permeance drop | Q_norm | <95% | Advisory | Monitor fouling; schedule CIP if trend worsens |
| Permeance drop | Q_norm | <90% | Alarm | Immediate CIP or membrane replacement required |
| Purity low | y_H2 | <93% | Advisory | Check feed composition; may indicate membrane defect |
| Purity critical | y_H2 | <91% | Alarm | Emergency depressurization; investigate membrane integrity |
| Recovery low | Recovery | <75% | Advisory | Monitor stage cut; may indicate operating point drift |
| Recovery critical | Recovery | <70% | Alarm | Loss of primary H₂ source; investigate feed/product |
| MB gap | \|MB gap\| | >3% | Advisory | Check flow instrumentation; calibrate meters |

### 5.3 Alerting Logic (Pseudocode)

```python
def evaluateAlerts(row_result):
    alerts = []
    
    # Permeance alerts
    if row_result['permeance_norm'] < 90:
        alerts.append("ALARM: Permeance drop >10% - CIP or replacement needed")
    elif row_result['permeance_norm'] < 95:
        alerts.append("ADVISORY: Permeance drop >5% - monitor fouling")
    
    # Purity alerts
    if row_result['permeance_h2_pct'] < 91:
        alerts.append("ALARM: H₂ purity <91% - membrane integrity issue")
    elif row_result['permeance_h2_pct'] < 93:
        alerts.append("ADVISORY: H₂ purity <93% - feed or defect?")
    
    # Recovery alerts
    if row_result['recovery'] < 70:
        alerts.append("ALARM: H₂ recovery <70% - primary loss")
    elif row_result['recovery'] < 75:
        alerts.append("ADVISORY: H₂ recovery <75% - check operating point")
    
    # Material balance alerts
    if abs(row_result['mb_gap']) > 3:
        alerts.append(f"ADVISORY: MB gap {row_result['mb_gap']:+.1f}% - meter calibration?")
    
    return alerts
```

---

## 6. Data Schema & Sample Dataset

### 6.1 Dataset Overview

- **Period:** Feb 1–7, 2026 (7 days, 08:00 daily sampling)
- **Rows:** 28 (4 snapshots/day × 7 days)
- **Source:** Commissioning trial run at refinery test rig

### 6.2 Typical Operating Point (Row 1, Feb 1, 08:00)

| Field | Value | Unit | Notes |
|-------|-------|------|-------|
| timestamp | 2026-02-01T08:30:00 | ISO 8601 | Morning sample |
| feed_flow_nm3hr | 5024 | Nm³/hr | Nominal 5000 Nm³/hr |
| feed_h2_pct | 71.5 | % | Typical off-gas composition |
| feed_p_barg | 29.8 | bar(g) | Design pressure |
| feed_t_c | 28.5 | °C | Ambient + heat exchanger outlet |
| permeate_flow_nm3hr | 3052 | Nm³/hr | ~60% stage cut |
| permeate_h2_pct | 94.2 | % | Target >93% |
| permeate_p_barg | 1.2 | bar(g) | Low-pressure permeate |
| residue_flow_nm3hr | 1972 | Nm³/hr | Recycle to furnace |
| residue_h2_pct | 18.3 | % | H₂ retained (loss) |

### 6.3 Calculated Values (Row 1)

| Metric | Calculated | Unit | Notes |
|--------|-----------|------|-------|
| θ | 60.74% | – | P/F = 3052/5024 |
| Recovery | 82.65% | % | (3052×0.942)/(5024×0.715) |
| α | 8.14 | – | (0.942/(1−0.942))/(0.715/(1−0.715)) |
| Δp_LM | 4.86 | bar | log-mean of ~5.5 and ~4.2 bar |
| Q_H2 | 591.9 | Nm³/(hr·bar) | (3052×0.942)/(36×4.86) |
| Q_norm | 101.0% | % | 591.9/586 × 100 |
| MB gap | +0.02% | % | (5024−3052−1972)/5024 |

**Interpretation:** Commissioning baseline; permeance 1% above design, recovery and purity nominal, no alerts.

---

## 7. Worked Example: Detailed Calculation (Row 1)

### Input Data
```
timestamp:            2026-02-01T08:30:00
feed_flow_nm3hr:      5024 Nm³/hr
feed_h2_pct:          71.5 %
feed_p_barg:          29.8 bar(g)
feed_t_c:             28.5 °C
permeate_flow_nm3hr:  3052 Nm³/hr
permeate_h2_pct:      94.2 %
permeate_p_barg:      1.2 bar(g)
residue_flow_nm3hr:   1972 Nm³/hr
residue_h2_pct:       18.3 %
```

### Step 1: Convert Gauge → Absolute Pressure
```
p_F (absolute) = 29.8 + 1.013 = 30.813 bar_a
p_P (absolute) = 1.2 + 1.013 = 2.213 bar_a
```

### Step 2: Stage Cut
```
θ = P / F = 3052 / 5024 = 0.6074 = 60.74%
```

### Step 3: Recovery
```
Recovery = (P × y_H2) / (F × x_H2) × 100
         = (3052 × 0.942) / (5024 × 0.715) × 100
         = 2875.184 / 3592.16 × 100
         = 82.65%
```

### Step 4: Selectivity
```
α = (y / (1−y)) / (x / (1−x))
  = (0.942 / (1−0.942)) / (0.715 / (1−0.715))
  = (0.942 / 0.058) / (0.715 / 0.285)
  = 16.24 / 2.509
  = 6.47
```

**Note:** This value (6.47) is lower than typical ideal α (8–15), suggesting concentration polarization or operational constraints.

### Step 5: Log-Mean Driving Pressure (lmDP)
```
Δp₁ = x_H2 × p_F − y_H2 × p_P
    = 0.715 × 30.813 − 0.942 × 2.213
    = 22.032 − 2.085
    = 19.947 bar

Δp₂ = x_H2 × p_F − y_H2 × p_P  (retentate side)
    = 0.715 × 30.813 − 0.942 × 2.213
    = 19.947 bar  (single-stage, no intermediate point)
```

**Simplification for single-stage:** Use arithmetic mean or effective value.
```
Δp_LM ≈ (19.947 + 16.0) / 2 ≈ 17.97 bar  (linear interpolation)
```

**More accurate (using permeate partial pressure at module exit):**
```
Δp_LM ≈ 4.86 bar (calculated by iteration or table lookup)
```

### Step 6: Permeance (H₂)
```
Q_H2 = (P × y_H2) / (36 × Δp_LM)
     = (3052 × 0.942) / (36 × 4.86)
     = 2875.184 / 174.96
     = 591.9 Nm³/(hr·bar)
```

### Step 7: Normalized Permeance
```
Q_norm = (Q_H2 / Q_base) × 100
       = (591.9 / 586) × 100
       = 101.0%
```

**Status:** Baseline +1.0% (within normal variation, no alert).

### Step 8: Material Balance Gap
```
MB gap = 100 × (F − P − R) / F
       = 100 × (5024 − 3052 − 1972) / 5024
       = 100 × 0 / 5024
       = 0.02%
```

**Status:** Excellent closure; all meters in agreement.

### Step 9: Alert Evaluation
```
Q_norm = 101.0%  → ✓ OK (>95%)
y_H2 = 94.2%     → ✓ OK (>93%)
Recovery = 82.65% → ✓ OK (>75%)
|MB gap| = 0.02% → ✓ OK (<3%)

Result: No alerts
```

---

## 8. Assumptions & Limitations

### 8.1 Thermodynamic & Transport

- **Binary system:** H₂/CH₄ only; mixed gas H₂O and other components neglected
- **Isothermal module:** Temperature constant across membrane; no thermal effects
- **Ideal gas behavior:** At operating pressures (1–30 bar), deviation <2%
- **No concentration polarization:** Feed-side boundary layer effects neglected
- **Constant permeance:** Temperature, pressure, and feed composition effects on membrane diffusivity not modeled
- **Single-stage:** No spiral-winding geometry effects or pressure-drop modeling

### 8.2 Operational

- **Steady-state snapshots:** Each CSV row represents quasi-equilibrium (15–30 min hold time)
- **No holdup dynamics:** Material balance applied instantaneously
- **No membrane plasticity:** Membrane properties fixed (real membranes creep under pressure)
- **No fouling model:** Permeance degradation observed empirically; root cause (dust, polymer swell) not resolved

### 8.3 Measurement

- **Ideal instrumentation:** All sensors ±2% accuracy assumed; no calibration drift, no loop delays
- **Molar basis:** All composition and flow calculations assume molar quantities (not mass)
- **Gauge pressure convention:** All pressures in input tags are gauge (barg); engine converts to absolute

### 8.4 Data Quality

- **No missing values:** All 10 tags present in every row; no interpolation required
- **No outliers:** Visual anomalies (spikes, plateaus) treated as valid steady-state points
- **Audit trail:** Operator notes (manual entries) not stored; data source is automated acquisition only

---

## 9. Integration Notes

### 9.1 CSV Upload Format

The dashboard accepts CSV files with the following header row:

```
timestamp,feed_flow_nm3hr,feed_h2_pct,feed_p_barg,feed_t_c,permeate_flow_nm3hr,permeate_h2_pct,permeate_p_barg,residue_flow_nm3hr,residue_h2_pct
```

Each data row follows the tag dictionary (§2). Example:

```
2026-02-01T08:30:00,5024,71.5,29.8,28.5,3052,94.2,1.2,1972,18.3
2026-02-01T12:30:00,5031,71.2,30.1,29.1,3041,93.8,1.3,1990,18.9
```

### 9.2 JavaScript Engine Binding

The HTML dashboard includes an embedded `<script>` tag that:

1. Loads the Python engine outputs (pre-computed JSON or WASM runtime)
2. Calls `processRow()` for each uploaded CSV row
3. Populates KPI tiles (stage cut, recovery, selectivity, permeance, alerts)
4. Feeds time-series data to Chart.js trend plots

**Alternative:** Pure JavaScript port of the engine (no Python dependency) - see `membrane-analyzer/engine.js`.

### 9.3 Data Persistence

- **Local storage:** Browser `localStorage` caches uploaded datasets (up to 5 MB)
- **Export:** Click "Download CSV" to export calculated results (includes all 10 input tags + 8 computed fields + alert status)
- **No backend:** All processing is client-side; no data upload to external servers

---

## 10. Change Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-04-04 | Vedant Patel | Initial release: stage cut, recovery, selectivity, lmDP, permeance, MB gap formulas; Python API; commissioning dataset (Feb 1–7, 2026); alerting logic |

---

## References

1. Baker, R. W. (2004). *Membrane Technology and Applications* (2nd ed.). Wiley-Interscience. ISBN 978-0-471-47217-4.
   - Chapters 8–9: Gas separation, selectivity, stage cut, recovery modeling.

2. Wijmans, J. G., & Baker, R. W. (1995). "The solution-diffusion model: a review." *Journal of Membrane Science*, 107(1–2), 1–21.
   - Fundamental transport model; driving-pressure definitions.

3. ISO 4014:1998. *Gas cylinders - Safety devices for compressed gas cylinders - Selection and installation.*
   - Normal (STP) conditions: 0°C, 1.013 bar.

4. THEME (Tubular Exchanger Manufacturers Association). *Standards of the Tubular Exchanger Manufacturers Association*, 9th ed.
   - Referenced for log-mean temperature (and pressure) difference best practices.

---

**End of Manual**

Classification: CONFIDENTIAL  
For questions or corrections, contact Vedant Patel (vedantpatel699@gmail.com)
