# Air Blower Backend Technical Reference

**Version:** 1.0  
**Date:** 2026-04-04  
**Author:** Vedant Patel  
**Classification:** CONFIDENTIAL — INTERNAL USE ONLY

This is the definitive backend reference for the air blower model. It is not linked from the public site but is version-controlled in the git repository for review and auditing.

---

## 1. Purpose & Scope

The air blower model monitors the **performance and mechanical health** of a dual-train centrifugal air blower (Blowers A and B, one active at a time) driven by a 3-phase induction motor.

**Computed per timestamp:**
- Electrical shaft power (kW) — IEEE/NEMA 3-phase
- Pressure ratio and differential pressure (bar absolute)
- Three efficiency methods:
  - **Polytropic** — headline KPI; industry-standard per ASME PTC 10 §5.4a
  - **Isentropic** — adiabatic comparison per ASME PTC 10 §5.4
  - **Fluid-power** — incompressible-flow indicator only (non-thermodynamic)
- Maximum vibration across 4 probes per train (mm/s RMS)
- Maximum bearing temperature across 2 probes per train (°C)
- Tiered alerts (Advisory / Alarm / Trip) with cited limits

**Not included in scope:**
- Not a control system; not an SIS (Safety Instrumented System)
- Not a substitute for formal ASME PTC 10 acceptance test
- Fluid-power efficiency is a trending indicator only, not a thermodynamically correct compressor efficiency

---

## 2. Tag Dictionary — Full Data Contract

The engine accepts **29 input variables** via a standardized dict interface. Both snake_case canonical names and original CSV headers are valid aliases. Implementers who wire this to a historian (Aveva PI, OPC-UA, SQL, etc.) map each historian tag to a canonical engine variable at the boundary.

| # | Canonical Variable | CSV Alias | Unit | Required | Typical Range | Fallback Rule |
|---|---|---|---|---|---|---|
| 1 | `timestamp` | Date | ISO-8601 | yes | — | drop row if unparseable |
| 2 | `motor_current_a` | Motor Current A | A | yes | 0–250 | drop row |
| 3 | `motor_current_b` | Motor Current B | A | yes | 0–250 | drop row |
| 4 | `suction_pressure_a` | Suction Press A | kPaa | yes (if A active) | 90–125 | drop row |
| 5 | `suction_pressure_b` | Suction Press B | kPaa | yes (if B active) | 90–125 | drop row |
| 6 | `discharge_pressure_a` | Discharge Press A | kPag | yes (if A active) | 50–120 | drop row |
| 7 | `discharge_pressure_b` | Discharge Press B | kPag | yes (if B active) | 50–120 | drop row |
| 8 | `controller_sp_a` | Controller SP A | kPag | no | 50–120 | ignore advisory |
| 9 | `controller_sp_b` | Controller SP B | kPag | no | 50–120 | ignore advisory |
| 10 | `bypass_op_a` | Bypass OP A | % | no | 0–100 | ignore advisory |
| 11 | `bypass_op_b` | Bypass OP B | % | no | 0–100 | ignore advisory |
| 12 | `filter_dp_a` | Filter DP A | bar | no | 0–0.2 | ignore advisory |
| 13 | `filter_dp_b` | Filter DP B | bar | no | 0–0.2 | ignore advisory |
| 14 | `vibration_a_1` | Vibration A1 | mm/s (RMS) | no | 0–5 | exclude from max |
| 15 | `vibration_a_2` | Vibration A2 | mm/s (RMS) | no | 0–5 | exclude from max |
| 16 | `vibration_a_3` | Vibration A3 | mm/s (RMS) | no | 0–5 | exclude from max |
| 17 | `vibration_a_4` | Vibration A4 | mm/s (RMS) | no | 0–5 | exclude from max |
| 18 | `bearing_temp_a_1` | Bearing Temp A1 | °C | no | 30–90 | exclude from max |
| 19 | `bearing_temp_a_2` | Bearing Temp A2 | °C | no | 30–90 | exclude from max |
| 20 | `vibration_b_1` | Vibration B1 | mm/s (RMS) | no | 0–5 | exclude from max |
| 21 | `vibration_b_2` | Vibration B2 | mm/s (RMS) | no | 0–5 | exclude from max |
| 22 | `vibration_b_3` | Vibration B3 | mm/s (RMS) | no | 0–5 | exclude from max |
| 23 | `vibration_b_4` | Vibration B4 | mm/s (RMS) | no | 0–5 | exclude from max |
| 24 | `bearing_temp_b_1` | Bearing Temp B1 | °C | no | 30–90 | exclude from max |
| 25 | `bearing_temp_b_2` | Bearing Temp B2 | °C | no | 30–90 | exclude from max |
| 26 | `total_flow` | Total Flow | Nm³/hr | yes | 15,000–80,000 | drop row |
| 27 | `discharge_temp_a` | Discharge Temp A | °C | no (required for η_isen/η_poly) | −10–100 | skip thermodynamic efficiencies |
| 28 | `discharge_temp_b` | Discharge Temp B | °C | no (required for η_isen/η_poly) | −10–100 | skip thermodynamic efficiencies |
| 29 | `suction_temp` | Suction Temp | °C | no | −20–40 | forward-fill ≤ 4 h, else default 4.0 °C |

**Active blower detection:** Automatically selected by comparing motor currents; the train with current > 10 A wins. Can be forced via `settings["blower_mode"]` ∈ {"auto", "A", "B"}.

**Pressure convention:**
- Suction is **absolute** (kPaa)
- Discharge is **gauge** (kPag)
- Engine normalizes both to absolute bar

---

## 3. Equations & Mathematical References

All equations are computed in the Python engine (`engine.py`) and mirrored in the browser dashboard (`dashboard.html`). Every formula cites a standard.

### 3.1 Electrical Shaft Power (IEEE/NEMA)

$$P_{\text{elec}} = \frac{\sqrt{3} \cdot V \cdot I \cdot PF}{1000} \quad [\text{kW}]$$

**Source:** Standard 3-phase motor-input-power equation (IEEE/NEMA motor basics).

**Defaults:**
- V = 4000 V (3-phase line-to-line, from motor datasheet)
- PF = 0.85 (nameplate power factor; replace with measured PF if available)

**Implementation:** `shaft_power_kw(voltage_v, current_a, power_factor)`

### 3.2 Pressure Normalization

$$P_1 = \frac{P_{1,\text{kPaa}}}{100} \quad [\text{bar abs}]$$

$$P_2 = \frac{P_{2,\text{kPag}}}{100} + P_{\text{atm}} \quad [\text{bar abs}]$$

$$\Delta P = P_2 - P_1 \quad [\text{bar}], \quad r = \frac{P_2}{P_1} \quad [\text{dimensionless}]$$

**Site atmospheric pressure default:** P_atm = 0.93 bar (~93 kPa, Alberta elevation ~1000 m).

**Implementation:** `normalize_pressures(p_suction_kpaa, p_discharge_kpag, p_atm_bar)`

### 3.3 Fluid-Power (Incompressible-Flow Indicator)

$$P_{\text{fluid}} = \frac{Q \cdot \Delta P}{36} \quad [\text{kW}]$$

**Derivation of the constant 36:**
$$\frac{\text{Nm}^3/\text{hr} \cdot \text{bar}}{36} \times \frac{10^5 \text{ Pa/bar}}{3600 \text{ s/hr} \cdot 10^3 \text{ W/kW}} = 1$$

Thus the constant 36 absorbs the unit conversion from Nm³/hr · bar to kW.

$$\eta_{\text{fluid}} = \frac{P_{\text{fluid}}}{P_{\text{elec}}} \times 100\% \quad [\%]$$

**Critical caveat:** This is an incompressible-flow hydraulic-power formula used as a **trending indicator only**. It is **not** thermodynamically correct for a gas compressor. It can exceed 100 % under certain conditions; such exceedance is a data-quality signal, not a clamping error. Use polytropic efficiency (§3.5) for rigorous performance comparison.

**Implementation:** `fluid_power_kw(flow_nm3hr, dp_bar)`

### 3.4 Isentropic (Adiabatic) Efficiency

Per **ASME PTC 10, Section 5.4:**

$$\eta_{\text{s}} = \frac{T_1 \cdot \left[ \left(\frac{P_2}{P_1}\right)^{(k-1)/k} - 1 \right]}{T_2 - T_1} \times 100\% \quad [\%]$$

with T in Kelvin. For air: k = 1.40 (ASHRAE Handbook, Fundamentals).

**Returns:** Decimal (0.75 = 75 %), or NaN if inputs are infeasible (e.g., P₂ ≤ P₁ or T₂ ≤ T₁).

**Implementation:** `isentropic_efficiency(t1_k, t2_k, p1_bar, p2_bar, k)`

### 3.5 Polytropic Efficiency (Headline KPI)

Per **ASME PTC 10, Section 5.4a** and **API 617:**

$$\sigma = \frac{\ln(T_2/T_1)}{\ln(P_2/P_1)} \quad [\text{polytropic temperature exponent}]$$

$$\eta_{\text{p}} = \frac{(k-1)/k}{\sigma} \times 100\% \quad [\%]$$

**Why polytropic?** This is the **industry-standard efficiency metric** for centrifugal machines. It is always ≥ isentropic efficiency for a compressor, and it is **independent of compression ratio**, making it suitable for cross-machine comparison and degradation tracking over time.

**Returns:** Decimal, or NaN if inputs are infeasible.

**Implementation:** `polytropic_efficiency(t1_k, t2_k, p1_bar, p2_bar, k)`

---

## 4. Python Engine API Reference

The engine module (`engine.py`, v1.0, 2026-04-04) is the single source of truth for all calculations. The browser dashboard replicates this math for display-time calculations.

### 4.1 Utility Functions

#### `_lookup(row, canonical) → value`
```python
def _lookup(row, canonical):
    """Return the first populated alias value for a canonical tag name."""
```
Searches `row` dict for any alias of `canonical` and returns the first non-None, non-empty value found. Supports both snake_case engine names and original CSV headers.

#### `_num(val) → float`
```python
def _num(val):
    """Safe numeric parse. Returns float('nan') on blank/invalid input."""
```
Converts any value (None, str, int, float) to a float. Strips commas. Returns NaN on invalid input.

#### `_parse_ts(ts) → datetime or None`
```python
def _parse_ts(ts):
    """Parse ISO-8601 or common date-time formats. Returns None on failure."""
```
Accepts datetime objects, ISO-8601 strings, or common patterns (mm/dd/yyyy, yyyy-mm-dd, etc.). Returns None on unparseable input.

### 4.2 Core Math Functions

#### `shaft_power_kw(voltage_v, current_a, power_factor) → float`
```python
def shaft_power_kw(voltage_v, current_a, power_factor):
    """
    Electrical input power to a 3-phase induction motor.
      P = sqrt(3) * V * I * PF / 1000   [kW]
    Ref: IEEE/NEMA motor basics.
    """
    return math.sqrt(3.0) * voltage_v * current_a * power_factor / 1000.0
```

#### `normalize_pressures(p_suction_kpaa, p_discharge_kpag, p_atm_bar) → (p1_bar, p2_bar)`
```python
def normalize_pressures(p_suction_kpaa, p_discharge_kpag, p_atm_bar):
    """
    Convert mixed-unit plant measurements to absolute bar.
      P1 = P_suction_kpaa / 100                  [bar absolute]
      P2 = P_discharge_kpag / 100 + P_atm_bar    [bar absolute]
    Returns: (p1_bar, p2_bar)
    """
    p1_bar = p_suction_kpaa / 100.0
    p2_bar = p_discharge_kpag / 100.0 + p_atm_bar
    return p1_bar, p2_bar
```

#### `fluid_power_kw(flow_nm3hr, dp_bar) → float`
```python
def fluid_power_kw(flow_nm3hr, dp_bar):
    """
    Indicator-grade hydraulic power. Valid for incompressible flow; used as
    a trending indicator for gas blowers, NOT a thermodynamic efficiency.
      P_fluid = Q * dP / 36             [kW]
    Derivation: (Nm^3/hr * bar) * (1e5 Pa/bar) / (3600 s/hr * 1e3 W/kW) = 1/36
    """
    return flow_nm3hr * dp_bar / 36.0
```

#### `isentropic_efficiency(t1_k, t2_k, p1_bar, p2_bar, k) → float`
```python
def isentropic_efficiency(t1_k, t2_k, p1_bar, p2_bar, k):
    """
    Isentropic (adiabatic) efficiency for a compressor.
    Ref: ASME PTC 10 Section 5.4.
      eta_s = T1 * [(P2/P1)^((k-1)/k) - 1] / (T2 - T1)
    
    Returns decimal (0.75 = 75 %), or NaN if inputs are infeasible.
    """
    if not (p2_bar > p1_bar and t2_k > t1_k):
        return float("nan")
    exponent = (k - 1.0) / k
    ideal_dt = t1_k * (math.pow(p2_bar / p1_bar, exponent) - 1.0)
    actual_dt = t2_k - t1_k
    return ideal_dt / actual_dt
```

#### `polytropic_efficiency(t1_k, t2_k, p1_bar, p2_bar, k) → float`
```python
def polytropic_efficiency(t1_k, t2_k, p1_bar, p2_bar, k):
    """
    Polytropic efficiency for a compressor.
    Ref: ASME PTC 10 Section 5.4a; API 617.

    Using the polytropic temperature exponent:
      sigma = ln(T2/T1) / ln(P2/P1)
      eta_p = ((k-1)/k) / sigma

    Industry-standard efficiency metric for centrifugal machines. Always
    >= isentropic efficiency for a compressor.

    Returns decimal, or NaN if inputs are infeasible.
    """
    if not (p2_bar > p1_bar and t2_k > t1_k):
        return float("nan")
    sigma = math.log(t2_k / t1_k) / math.log(p2_bar / p1_bar)
    if sigma <= 0:
        return float("nan")
    return ((k - 1.0) / k) / sigma
```

### 4.3 Active Blower Detection

#### `detect_active_blower(curr_a, curr_b, mode, min_a) → 'A' | 'B' | None`
```python
def detect_active_blower(curr_a, curr_b, mode, min_a):
    """
    Return 'A', 'B', or None. None = no blower running.
    
    mode='auto': train with current > min_a wins; higher current tiebreaker.
    mode='A'|'B': forced selection (returns None if that current < min_a).
    """
```

**Logic:**
- If `mode == "A"`: return "A" if curr_a > min_a, else None
- If `mode == "B"`: return "B" if curr_b > min_a, else None
- If `mode == "auto"`:
  - If both > min_a: return the one with higher current
  - If only one > min_a: return that one
  - If neither > min_a: return None

**Default:** min_a = 10.0 A (from DEFAULT_SETTINGS)

### 4.4 Alert Engine

#### `build_alerts(vib_max, brg_max, filter_dp, dp_bar, bypass_op, p_discharge_kpag, controller_sp, lim) → list[dict]`
```python
def build_alerts(vib_max, brg_max, filter_dp, dp_bar, bypass_op,
                 p_discharge_kpag, controller_sp, lim):
    """Return a list of active alerts (unordered; status is rolled up separately)."""
```

Returns a list of alert dicts, each with keys:
- `"severity"`: "advisory", "alarm", or "trip"
- `"message"`: human-readable description
- `"source"`: standard or plant setpoint

See §5 (Alert Logic) for full tiered rules and sources.

#### `roll_up_status(alerts) → "ok" | "advisory" | "alarm" | "trip"`
```python
def roll_up_status(alerts):
    """Collapse alert list into a single status level."""
    if any(x["severity"] == "trip" for x in alerts):     return "trip"
    if any(x["severity"] == "alarm" for x in alerts):    return "alarm"
    if any(x["severity"] == "advisory" for x in alerts): return "advisory"
    return "ok"
```

### 4.5 Single-Row Processor

#### `process_single_row(row, settings=None, limits=None) → dict`
```python
def process_single_row(row, settings=None, limits=None):
    """
    Process one timestamped row of blower tags.
    Returns a dict of KPIs + alerts + status, or {"drop": True, "reason": ...}.
    """
```

**Input:**
- `row`: dict with tag aliases (either snake_case or CSV headers)
- `settings`: override DEFAULT_SETTINGS
- `limits`: override DEFAULT_LIMITS

**Output on success (drop=False):**
```python
{
    "drop": False,
    "timestamp": "2021-12-22T04:00:00",      # ISO-8601
    "active_blower": "A" | "B",
    "power_kw": float,                        # shaft power
    "pressure_ratio": float,                  # P2 / P1
    "dp_bar": float,                          # P2 - P1
    "p1_bar_abs": float,                      # P1 normalized
    "p2_bar_abs": float,                      # P2 normalized
    "flow_nm3hr": float,
    "fluid_power_kw": float,
    "efficiency_fluid_pct": float,
    "efficiency_isentropic_pct": float,       # or None if T2 missing
    "efficiency_polytropic_pct": float,       # or None if T2 missing
    "efficiency_headline_pct": float,         # selected by settings["efficiency_method"]
    "efficiency_method": "polytropic" | "isentropic" | "fluid",
    "max_vibration_mms": float or None,
    "max_bearing_temp_c": float or None,
    "filter_dp_bar": float or None,
    "bypass_op_pct": float or None,
    "t1_c_used": float,                       # T₁ value actually used
    "t1_source": "measured" | "forward-filled" | "default",
    "alerts": [list of alert dicts],
    "status": "ok" | "advisory" | "alarm" | "trip",
}
```

**Output on drop (drop=True):**
```python
{
    "drop": True,
    "reason": "unparseable timestamp" | "no active blower" | "missing critical tag ..."
}
```

**Drop reasons:**
- "unparseable timestamp"
- "no active blower (both currents < min)"
- "missing critical tag (P1, P2, flow, or current)"

### 4.6 Batch Processor

#### `process_batch(rows, settings=None, limits=None) → dict`
```python
def process_batch(rows, settings=None, limits=None):
    """
    Process many rows in timestamp order. Implements bounded forward-fill
    on suction_temp per STANDARD.md §4.
    
    Returns:
        {
            "rows_processed": int,
            "rows_dropped": int,
            "drop_reasons": {reason_str: count, ...},
            "results": [list of output dicts from process_single_row]
        }
    """
```

**Suction temperature forward-fill:**
- Sorts input rows by timestamp
- Tracks the last measured suction temperature and its timestamp
- When suction_temp is missing: if a measured value exists and is ≤ 4 hours old, use it; otherwise use the default (4.0 °C)
- Injects the forward-filled value into settings via `_t1_forward_filled` key

This ensures continuous efficiency calculations even during minor instrument dropouts.

### 4.7 CSV Loader

#### `load_csv(path) → list[dict]`
```python
def load_csv(path):
    """Load a CSV file as a list of dicts (one row per dict)."""
    try:
        with open(path, encoding="utf-8-sig", newline="") as f:
            return list(csv.DictReader(f))
    except Exception as e:
        print(f"[engine] CSV load error: {e}")
        return []
```

Automatically handles UTF-8 BOM and comma delimiters. Returns an empty list on error.

---

## 5. Configuration Reference

### 5.1 DEFAULT_SETTINGS

```python
DEFAULT_SETTINGS = {
    # Engineering parameters
    "motor_voltage_v": 4000.0,        # 3-phase line-to-line, from motor datasheet
    "power_factor": 0.85,             # nameplate; replace with measured PF if available
    "gamma_k": 1.40,                  # isentropic exponent for air (ASHRAE)
    "atm_pressure_bar": 0.93,         # site atmospheric, ~93 kPa (Alberta ~1000 m)
    
    # Data-quality rules
    "active_current_min_a": 10.0,     # below this, blower treated as off
    "suction_temp_fallback_c": 4.0,   # default when live value missing
    "suction_temp_ff_max_hours": 4,   # forward-fill window (batch mode only)
    
    # Blower mode: "auto" | "A" | "B"
    "blower_mode": "auto",
    
    # Efficiency method to report as headline KPI:
    # "polytropic" | "isentropic" | "fluid"
    "efficiency_method": "polytropic",
}
```

### 5.2 DEFAULT_LIMITS

```python
DEFAULT_LIMITS = {
    # Vibration (ISO 10816-3, Group 1: large machines >300 kW, rigid foundation,
    # measured in mm/s RMS on non-rotating parts)
    "vib_advisory_mms": 4.5,        # Zone B/C boundary
    "vib_alarm_mms":    7.1,        # Zone C/D boundary
    "vib_trip_mms":     11.0,       # Zone D (long operation not permissible)
    
    # Bearing temperature (manufacturer-typical for oil-lubed journal bearings)
    "brg_advisory_c":   70.0,
    "brg_alarm_c":      85.0,
    "brg_trip_c":       95.0,
    
    # Process advisories (plant-specific; user-tunable)
    "filter_dp_max_bar":   0.10,    # replace filter advisory
    "blower_dp_max_bar":   0.45,    # fouling advisory
    "bypass_open_max_pct": 60.0,    # recirculation advisory
}
```

---

## 6. Alert Logic — Complete Tiered Rules

The alert engine implements **tiered advisories** aligned with industrial best practice (ISA-101 / HPHMI gray-default). Each alert has a severity, message, and source citation.

### 6.1 Vibration Alerts (ISO 10816-3, Group 1)

**Applies to:** Equipment > 300 kW, rigid foundation, measured on non-rotating parts (mm/s RMS).

| Condition | Severity | Threshold | Zone | Action |
|-----------|----------|-----------|------|--------|
| vib_max < 4.5 | — | — | A–B | OK (normal) |
| 4.5 ≤ vib_max < 7.1 | **advisory** | 4.5 mm/s | B/C boundary | Monitor |
| 7.1 ≤ vib_max < 11.0 | **alarm** | 7.1 mm/s | C/D boundary | Investigate soon |
| 11.0 ≤ vib_max | **trip** | 11.0 mm/s | D | Shutdown recommended |

**Message format:**
```
Vibration {vib_max:.2f} mm/s [exceeds | above] [trip limit 11.0 | alarm limit 7.1 | advisory 4.5] mm/s 
(ISO 10816 Zone [D | C/D | B/C]).
```

**Source:** ISO 10816-3, Group 1

### 6.2 Bearing Temperature Alerts

| Condition | Severity | Threshold | Action |
|-----------|----------|-----------|--------|
| brg_max < 70.0 | — | — | OK (normal) |
| 70.0 ≤ brg_max < 85.0 | **advisory** | 70 °C | Monitor; check cooling |
| 85.0 ≤ brg_max < 95.0 | **alarm** | 85 °C | Investigate soon; reduce load |
| 95.0 ≤ brg_max | **trip** | 95 °C | Shutdown recommended |

**Message format:**
```
Bearing temperature {brg_max:.1f} C [exceeds | above] [trip limit 95 | alarm limit 85 | advisory 70] C.
```

**Source:** bearing datasheet (manufacturer-typical for oil-lubed journal bearings)

### 6.3 Filter Differential Pressure

**Threshold:** filter_dp > 0.10 bar  
**Severity:** advisory  
**Message:** `Filter dP {filter_dp:.3f} bar above 0.10 bar - replace filter.`  
**Source:** plant setpoint

### 6.4 Blower Differential Pressure (Fouling Indicator)

**Threshold:** dp_bar > 0.45 bar  
**Severity:** advisory  
**Message:** `Blower dP {dp_bar:.3f} bar above design 0.45 bar - possible internal fouling.`  
**Source:** plant setpoint

**Interpretation:** Exceeding design ΔP suggests internal passage blockage or mechanical wear.

### 6.5 Bypass Valve Opening

**Threshold:** bypass_op > 60.0 %  
**Severity:** advisory  
**Message:** `Bypass valve {bypass_op:.1f}% above 60% - recycling excess flow.`  
**Source:** plant setpoint

**Interpretation:** High bypass opening indicates flow exceeds demand; consider reducing inlet guide vane opening or reducing system pressure.

### 6.6 Capacity Limit (Bypass Closed + Discharge Below Setpoint)

**Conditions (all true):**
- bypass_op < 5.0 % (bypass is closed)
- p_discharge_kpag < controller_sp
- Neither value is NaN

**Severity:** advisory  
**Message:** `Bypass closed and discharge pressure below setpoint - machine at capacity limit.`  
**Source:** control narrative

**Interpretation:** Machine is running fully open but cannot meet the control setpoint; system demand exceeds blower maximum capacity.

---

## 7. Data Schema — demo.csv

The reference dataset spans **28 rows** (rows 2–29 of the CSV; row 1 is headers) from **2021-12-22 04:00 to 2026-01-24 08:00** (note: date range spans multiple months; see data/demo.csv for full extent).

**Key characteristics:**
- **Active blower:** B throughout (Motor Current B ≈ 110–124 A; Motor Current A ≈ 0 A)
- **Suction temperature:** All 28 rows have blank Suction Temp; engine defaults to 4.0 °C
- **Discharge temperatures:** Range −28 °C to +84 °C (reflecting seasonal variation)
- **Vibration:** All probes < 1.0 mm/s (healthy machine)
- **Bearing temperatures:** Range 23–76 °C (normal operation)
- **Flow:** 10,000–22,000 Nm³/hr (typical centrifugal blower turndown)

**First row (2021-12-22 04:00):**
```
Motor Current B:    114.42 A
Suction Press B:    120.18 kPaa
Discharge Press B:  92.01 kPag
Total Flow:         20,049.42 Nm³/hr
Discharge Temp B:   74.00 °C
Vibration B1–B4:    0.06, 0.04, 0.77, 0.77 mm/s (max = 0.77)
Bearing Temp B1–B2: 54.39, 68.29 °C (max = 68.29)
Suction Temp:       (blank → default 4.0 °C)
```

---

## 8. Worked Validation Example

**Test case:** Row 1 of demo.csv (timestamp 2021-12-22 04:00), active blower = B, motor current B = 114.42 A.

**Inputs:**
| Tag | Value |
|-----|-------|
| Motor Current B (I) | 114.42 A |
| Suction Press B (P₁_kPaa) | 120.18 kPaa |
| Discharge Press B (P₂_kPag) | 92.01 kPag |
| Total Flow (Q) | 20,049.42 Nm³/hr |
| Discharge Temp B (T₂) | 74.00 °C |
| Suction Temp (T₁) | blank → fallback 4.0 °C |

**Settings:** V = 4000 V, PF = 0.85, P_atm = 0.93 bar, k = 1.40.

**Step 1 — Shaft power (§3.1):**
$$P_{\text{elec}} = \frac{\sqrt{3} \times 4000 \times 114.42 \times 0.85}{1000} = 673.82 \text{ kW}$$

**Step 2 — Pressure normalization (§3.2):**
$$P_1 = 120.18 / 100 = 1.2018 \text{ bar abs}$$
$$P_2 = 92.01 / 100 + 0.93 = 1.8501 \text{ bar abs}$$
$$\Delta P = 0.6483 \text{ bar}, \quad r = P_2 / P_1 = 1.5394$$

**Step 3 — Fluid power (§3.3):**
$$P_{\text{fluid}} = 20,049.42 \times 0.6483 / 36 = 361.06 \text{ kW}$$
$$\eta_{\text{fluid}} = 361.06 / 673.82 \times 100\% = 53.58\%$$

**Step 4 — Isentropic efficiency (§3.4):**
$$T_1 = 4.0 + 273.15 = 277.15 \text{ K}$$
$$T_2 = 74.0 + 273.15 = 347.15 \text{ K}$$
$$(P_2/P_1)^{(k-1)/k} = 1.5394^{0.2857} = 1.1312$$
$$\text{ideal } \Delta T = 277.15 \times (1.1312 - 1) = 36.37 \text{ K}$$
$$\text{actual } \Delta T = 347.15 - 277.15 = 70.00 \text{ K}$$
$$\eta_s = 36.37 / 70.00 \times 100\% = 51.94\%$$

**Step 5 — Polytropic efficiency (§3.5):**
$$\sigma = \ln(347.15 / 277.15) / \ln(1.5394) = 0.2252 / 0.4317 = 0.5217$$
$$\eta_p = (0.2857 / 0.5217) \times 100\% = 54.74\%$$

**Consistency check:** η_p (54.74 %) > η_isen (51.94 %) ✓ (thermodynamically sound for a compressor)

**Step 6 — Alerts:**
- **Vibration:** max(0.06, 0.04, 0.77, 0.77) = 0.77 mm/s < 4.5 mm/s → **OK**
- **Bearing temp:** max(54.39, 68.29) = 68.29 °C < 70.0 °C → **OK**
- **Blower ΔP:** 0.6483 bar > 0.45 bar → **ADVISORY: possible internal fouling**
- **Overall status:** **ADVISORY**

**Verification:** These results match `engine.process_single_row()` output exactly (verified 2026-04-04).

---

## 9. Assumptions & Limitations

1. **Gas composition:** Air only (k = 1.40). Do not reuse this model for process gases (natural gas, nitrogen, CO₂, etc.) without updating γ_k per the respective fluid's thermodynamic properties.

2. **Power factor:** Nameplate value (0.85) is used. If the site has a recent power-factor measurement, replace with measured PF for improved accuracy.

3. **Flow accuracy:** Flow measurement assumed accurate per ISO 5167 (orifice plate or venturi). Typical ±2 % full-scale uncertainty; no uncertainty propagation is implemented in this PoC.

4. **Suction temperature:** Not instrumented in the provided demo.csv (all 28 rows blank). Engine forward-fills a measured value up to 4 hours old, then falls back to 4.0 °C. Adding a dedicated suction-temperature transmitter would improve thermodynamic-efficiency accuracy significantly.

5. **Fluid-power efficiency:** Can exceed 100 % under certain conditions (typically when flow is measured at high resolution but motor current is relatively low). Such exceedance is a **data-quality signal**, not an error. Never clamp it; investigate the outlier.

6. **Vibration interpretation:** Limits use ISO 10816-3 Group 1 (machines > 300 kW, rigid foundation). If the installed machines are on a flexible foundation or are smaller, update the thresholds per ISO 10816-1 or -2.

7. **Alerting scope:** No rate-of-change alarms, no hysteresis, no machine-learning outlier detection. These are PoC scope.

---

## 10. Integration Notes — Historian Boundary

The engine reads plain Python dicts. For a live historian integration, the implementer who wires this to Aveva PI, OPC-UA, SQL, or a real-time database should:

1. **Fetch raw historian tags** by their plant naming convention (e.g., `1645-IT-A01.PV` for Motor Current A).
2. **At the boundary**, transpose the historian response into a list of dicts with keys matching the engine's **canonical tag names** (snake_case):
   ```python
   # Example: Aveva PI REST API response
   resp = historian_api.get("1645-IT-A01.PV", start_time, end_time)
   rows = [
       {
           "timestamp": "2021-12-22 04:00:00",
           "motor_current_a": 0.00,
           "motor_current_b": 114.42,
           # ... other 27 tags ...
       },
       # ... more rows ...
   ]
   ```
3. **Call the engine:**
   ```python
   from engine import process_batch, load_csv
   result = process_batch(rows)
   ```
4. **Publish the results** to your dashboard, alerting system, or data lake. The engine output includes:
   - All KPIs (power, pressures, efficiencies, vibration, temperature)
   - Alerts with severity and message
   - Status rollup

**Key point:** The tag mapping from historian → engine variable happens at this boundary. Nothing downstream needs to know the plant's specific tag naming scheme.

---

## 11. Change Log

| Version | Date | Author | Change |
|---------|------|--------|--------|
| 1.0 | 2026-04-04 | Vedant Patel | Initial backend reference. Rebuilt against STANDARD.md. Added polytropic efficiency (ASME PTC 10 §5.4a). Tiered Advisory/Alarm/Trip alerts using ISO 10816-3 Group 1 vibration zones and manufacturer-typical bearing limits. Removed 0–100 % efficiency clamp. Added bounded forward-fill on suction temperature (≤4 hours, else default 4.0 °C). Complete tag dictionary, full API reference, configuration glossary, alert rule matrix, worked validation example, and integration notes for historian boundary. |

---

**Document end.** For implementation support or technical questions, contact the ChemDash maintainer.
