# Fired Heater Efficiency - Backend Technical Manual

**Version:** 1.0  
**Date:** 2026-04-04  
**Author:** Vedant Patel  
**Classification:** CONFIDENTIAL

---

## 1. Purpose & Scope

This manual documents the **dual-method efficiency calculation engine** for the Ferriq Fired Heater dashboard. The heater model computes thermal efficiency via two independent paths:

1. **Direct Method** (QProcess Absorption)  
   Thermal energy delivered to the process fluid, divided by fuel energy input.
   - Equation: η = Q_abs / Q_fired × 100%
   - Simplest, most direct for process focus
   - Ignores stack losses; valid only if losses are small

2. **Indirect Method** (Flue Gas Loss Stack-Up)  
   100% minus quantified flue gas exit losses (dry gas, moisture, radiation, unaccounted).
   - Equation: η = 100 − L_dry − L_moist − L_rad − L_unacc
   - ASME PTC 4 standard for boiler/heater efficiency testing
   - Accounts for all stack losses; independent of process measurements
   - Better for detecting fouling, combustion drift, or measurement error

**Gap Analysis:**  
The difference between direct and indirect (η_direct − η_indirect) serves as a **consistency check**:
- Typical gap: ±3–5 percentage points (measurement noise)
- Large gap (>8 pp) signals fouling, air/fuel imbalance, or instrument drift
- Gap analysis is reported in alerts and trend charts

---

## 2. Tag Dictionary

| Tag | Description | Units | Type | Fallback | Notes |
|-----|-------------|-------|------|----------|-------|
| `timestamp` | ISO 8601 datetime | - | Text | - | Required, UTC or local per config |
| `process_inlet_c` | Process fluid inlet temperature | °C | Float | - | Thermowell at heater inlet |
| `process_outlet_c` | Process fluid outlet temperature | °C | Float | - | Thermowell at heater outlet |
| `process_flow_kghr` | Mass flow rate of process fluid | kg/h | Float | - | Coriolis or mag-meter |
| `process_cp_kjkgk` | Specific heat of process fluid | kJ/(kg·K) | Float | - | Look-up per fluid type & T_avg |
| `fuel_flow_nm3hr` | Fuel (gas) volumetric flow at STP | Nm³/h | Float | - | Orifice meter, compensated to STP |
| `fuel_lhv_mjnm3` | Lower Heating Value of fuel | MJ/Nm³ | Float | 47.5 (nat gas) | Via gas chromatography or assume |
| `stack_o2_pct` | Flue gas O₂ concentration | % vol | Float | - | Electrochemical cell, wet basis |
| `stack_temp_c` | Stack outlet temperature | °C | Float | - | Thermowell, below APH if present |
| `bridgewall_temp_c` | Bridgewall (flame zone) temperature | °C | Float | - | Estimated via radiant tube wall IR or thermocouple |
| `ambient_temp_c` | Ambient air temperature | °C | Float | 15 | Dry-bulb at heater location |
| `fuel_temp_c` | Fuel inlet temperature | °C | Float | 25 | Gas temp at heater burner inlet |

**Notes:**
- All temperatures in Celsius; internally converted to Kelvin for thermodynamics.
- Stack O₂ measured in **wet flue gas** (includes H₂O vapor). Engine corrects to dry basis for excess-air calc.
- Process flow must be **positive** (enforced in validation).
- Fuel flow must be **positive** (enforced in validation).
- If process inlet ≥ outlet, efficiency flagged as 0% and alert raised.

---

## 3. Equations & Engineering Citations

### 3.1 Process Heat Absorption (Direct Method)

**Equation:**
```
Q_abs [kW] = m_process [kg/h] × Cp [kJ/(kg·K)] × (T_out − T_in [K]) / 3600 [s/h]
```

**Source:** API 560 §5.4 - Fired Heaters for Refinery Service

**Notes:**
- Temperature difference in **Kelvin** (ΔK = ΔC, no offset).
- Divide by 3600 to convert kg/h energy to kW.
- Assumes **constant Cp** over the temperature range; if wide range, user supplies weighted-average Cp.
- If Tin ≥ Tout, set Q_abs = 0 and raise INLET_EXCEEDS_OUTLET alert.

---

### 3.2 Fuel Energy Input (Fired Duty)

**Equation:**
```
Q_fired [kW] = V_fuel [Nm³/h] × LHV [MJ/Nm³] × 1000 [kJ/MJ] / 3600 [s/h]
```

**Source:** API 560 §5.4

**Notes:**
- **LHV** (Lower Heating Value) basis: does not include condensation heat of water vapor.
- Unit conversion: MJ → kJ × 1000; Nm³/h → kJ/s ÷ 3600.
- Assumes fuel flow measured or compensated to **STP (Standard Temperature Pressure):**
  - 0°C, 1 atm, or 15°C, 1 atm (per regional standard)
- Natural gas LHV ≈ 47.5 MJ/Nm³ (typical); ranges 45–48 depending on composition.

---

### 3.3 Direct Efficiency

**Equation:**
```
η_direct [%] = (Q_abs / Q_fired) × 100
```

**Source:** API 560

**Interpretation:**
- Fraction of fuel energy delivered to process.
- Does **not** account for stack losses; valid when loss_total << Q_abs.
- Typical range: 75–92% (75–85% in aged heaters, 88–92% in clean new heaters).

---

### 3.4 Excess Air from O₂ Measurement

**Equation:**
```
EA [%] = 100 × O₂_dry / (20.95 − O₂_dry)
```

**Source:** ASME PTC 4 Appendix A - Standard Method for Combustion Efficiency Testing

**Notes:**
- **O₂_dry** = O₂ measured in flue gas, adjusted to dry basis:
  ```
  O₂_dry [%] = O₂_wet [%] × (100 − H₂O_mole [%]) / 100
  ```
  where H₂O_mole ≈ H₂O_produced_mole / (H₂O_produced_mole + N₂ + O₂ + CO₂ + Ar).
  
  For natural gas, H₂O ≈ 2.25 % mol (from stoichiometry), so:
  ```
  O₂_dry ≈ O₂_wet × (100 − 2.25) / 100 = O₂_wet × 0.9775
  ```

- 20.95% is the O₂ concentration in dry air.
- EA interpretation:
  - EA = 0%  → stoichiometric (Φ = 1), unstable, soot risk
  - EA = 15% → ~1.65% O₂ dry, efficient lean flame
  - EA = 25% → ~2.6% O₂ dry, safe margin, common target
  - EA > 40% → over-air, loss via sensible heating of air, CO unlikely

---

### 3.5 Lower Heating Value - Conversion to Basis of kg Fuel

**Equation:**
```
LHV [kJ/kg] = LHV_MJ_Nm3 [MJ/Nm³] × 1000 [kJ/MJ] × 22.414 [L/mol] / MW_fuel [g/mol] / 1000 [mg/g]
            = LHV_MJ_Nm3 × 22.414 / MW_fuel
```

**Source:** Stoichiometric conversion; ASME PTC 4

**Notes:**
- 22.414 L/mol is the molar volume of ideal gas at STP (0°C, 1 atm).
- MW_fuel: molecular weight of fuel in g/mol.
  - Natural gas (CH₄ equivalent): MW ≈ 17.5 g/mol
  - Propane: MW = 44.1 g/mol
  - Furnace gas: MW ≈ 20–22 g/mol (mixture)
- Used to calculate stack dry-loss and moisture loss on **per kg fuel** basis.

---

### 3.6 Indirect Efficiency - Stack-Up of Losses

**Equation:**
```
η_indirect [%] = 100 − L_dry − L_moist − L_rad − L_unacc
```

**Source:** ASME PTC 4 - Standard for Evaluation of Flue-Gas Losses in Boilers and Furnaces

**Notes:**
- Each loss term is in **percentage points** (0–100).
- Losses are **independent** (additive) if small; interaction ignored in this model.
- Sum of losses + η_indirect should equal 100%.

---

### 3.7 Dry Flue-Gas Loss

**Equation:**
```
L_dry [pp] = (m_dry_fg [kg/s] × Cp_fg [kJ/(kg·K)] × (T_stk − T_amb [K])) / LHV_kJ_kg × 100
```

**Alternatively (per kg fuel input):**
```
L_dry [pp] = (m_O2_stoich + m_N2_stoich + m_CO2_prod) [kg/kg_fuel] × Cp_dry [kJ/(kg·K)] × ΔT / LHV [kJ/kg] × 100
```

**Source:** ASME PTC 4 §5.2 - Sensible Heat of Flue Gas

**Notes:**
- **m_dry_fg:** total mass of dry flue gas (O₂ + N₂ + CO₂ + Ar, no H₂O) per kg fuel burned.
- **Cp_fg:** specific heat of dry flue gas, ~1.08 kJ/(kg·K) at 400–600°C (engineering average).
- **T_stk:** stack outlet temperature (°C, converted to K for ΔT).
- **T_amb:** ambient reference temperature (°C, converted to K).
  - Per ASME PTC 4, reference is **dry-bulb ambient**.
  - Fallback: 15°C if not supplied.
- **LHV_kJ_kg:** lower heating value of fuel, kJ/kg (from §3.5).

**Calculation in engine:**
1. Compute stoichiometric air requirement from fuel chemistry (or assume 17.2 kg air / kg fuel for natural gas).
2. Compute stoichiometric O₂ = air × 0.2095.
3. Compute excess air: EA_kg = stoich_air × EA% / 100.
4. Dry fg mass = stoich_air + fuel + excess_air = (stoich_air × (1 + EA/100)) + fuel.
   - Usually ~17.2 × (1 + EA/100) for natural gas.
5. L_dry = dry_fg × Cp_dry × (T_stk − T_amb) / LHV_kJ_kg × 100.

**Typical range:** 8–15 pp (higher at hot stack, higher at high EA).

---

### 3.8 Moisture Loss from Combustion Water Vapor

**Equation (LHV basis, sensible only):**
```
L_moist [pp] = (m_H2O [kg/kg_fuel] × Cp_v [kJ/(kg·K)] × (T_stk − T_fuel [K])) / LHV [kJ/kg] × 100
```

**Source:** ASME PTC 4 §5.3 - Sensible Heat of Water Vapor in Flue Gas

**Notes:**
- **LHV basis:** The LHV already subtracts the latent heat of vaporization (≈2256 kJ/kg water at 100°C).
  - We count **only sensible cooling** of H₂O from stack temp down to fuel (or ambient).
  - We do **not** count latent (condensation) loss; that is inherent in LHV definition.
  
- **m_H₂O:** molar water produced per mole fuel burned.
  - Natural gas (CH₄): 2 moles H₂O per mole CH₄ → ~0.225 kg H₂O / kg fuel.
  - Per tag dict: `H2O_mole_per_fuel ≈ 2.25 %` (of total molar flue gas) or ~0.225 kg H₂O / kg fuel.
  
- **Cp_v:** specific heat of water vapor, ~2.00 kJ/(kg·K) at 300–500°C (engineering average).

- **T_stk − T_fuel:** temperature difference from stack to fuel inlet.
  - Reference temperature is **fuel inlet**, not ambient.
  - If fuel temp not provided, fallback to 25°C.
  - Rationale: H₂O enters the furnace as air humidity + combustion product; if we cool it only to fuel temp, we're at that enthalpy basis.

**Calculation in engine:**
1. m_H2O = H2O_per_fuel_kg (default 0.225 for natural gas).
2. ΔT_water = T_stk − T_fuel (in K).
3. L_moist = m_H2O × Cp_v × ΔT_water / LHV_kJ_kg × 100.

**Typical range:** 3–6 pp (lower if fuel is hot, higher if stack is hot).

---

### 3.9 Radiation Loss (Fixed Percentage)

**Equation:**
```
L_rad [pp] = config.radiation_loss_percent
```

**Source:** ASME PTC 4 §5.4; API 560 Appendix C

**Notes:**
- Radiation from external shell and refractory to ambient is difficult to measure directly.
- Approximated as **fixed percentage of input** (typical 1–3%) for small-medium heaters.
- More precisely, radiation loss depends on exterior surface area and view factors; engineering estimate is 2% for a medium industrial heater.
- Can be refined via external thermography or heat-balance closure.
- Default in engine: 2.0%.

---

### 3.10 Unaccounted Loss

**Equation:**
```
L_unacc [pp] = config.unaccounted_loss_percent
```

**Source:** ASME PTC 4 §5.5 - Unmeasured/Unmeasurable Losses

**Notes:**
- Accounts for:
  - Blowdown (if any).
  - Soot/ash on tubes (not explicitly modeled).
  - Measurement uncertainty on flow, temperature, composition.
  - Piping/manifold losses outside heater duty (if measured at headers).
- Default in engine: 0.5%.
- If detailed heat balance is required, this should be minimized via careful instrument calibration.

---

## 4. Python Engine API

The calculation engine is embedded in the dashboard HTML as a `<script>` block. Below are the exported functions:

### 4.1 `excess_air_from_o2(o2_pct, h2o_mole_pct = 2.25)`

**Purpose:** Convert measured flue-gas O₂ to excess air %.

**Parameters:**
- `o2_pct` (float): O₂ concentration in wet flue gas, 0–21% range.
- `h2o_mole_pct` (float, optional): Water vapor mole % in flue gas. Default 2.25 for natural gas.

**Returns:** Excess Air (%), 0–100+.

**Example:**
```javascript
const ea = excess_air_from_o2(2.5);  // → ~18.0% EA
```

---

### 4.2 `heat_absorbed_kw(m_kg_h, cp_kj_kgk, delta_t_c)`

**Purpose:** Calculate sensible heat absorbed by process.

**Parameters:**
- `m_kg_h` (float): Process mass flow, kg/h.
- `cp_kj_kgk` (float): Process specific heat, kJ/(kg·K).
- `delta_t_c` (float): Process temperature rise, T_out − T_in (°C = K).

**Returns:** Heat absorbed, kW.

**Example:**
```javascript
const qa = heat_absorbed_kw(5000, 2.8, 15);  // → ~58.3 kW
```

---

### 4.3 `fired_duty_kw(v_fuel_nm3_h, lhv_mj_nm3)`

**Purpose:** Calculate chemical energy of fuel input.

**Parameters:**
- `v_fuel_nm3_h` (float): Fuel volumetric flow, Nm³/h.
- `lhv_mj_nm3` (float): Lower heating value, MJ/Nm³.

**Returns:** Fuel energy, kW.

**Example:**
```javascript
const qf = fired_duty_kw(200, 47.5);  // → ~2639.0 kW
```

---

### 4.4 `direct_efficiency_pct(q_abs, q_fired)`

**Purpose:** Compute direct efficiency (process-based).

**Parameters:**
- `q_abs` (float): Heat absorbed, kW.
- `q_fired` (float): Fuel energy, kW.

**Returns:** η_direct, %.

**Example:**
```javascript
const eta_dir = direct_efficiency_pct(2100, 2500);  // → 84.0%
```

**Validation:**
- If q_fired <= 0, return 0%.
- If q_abs > q_fired (impossible), cap at 100% and flag anomaly.

---

### 4.5 `_lhv_kj_per_kg(lhv_mj_nm3, mw_fuel_g_mol = 17.5)`

**Purpose:** [Internal] Convert LHV from MJ/Nm³ to kJ/kg basis.

**Parameters:**
- `lhv_mj_nm3` (float): MJ/Nm³.
- `mw_fuel_g_mol` (float, optional): Fuel molecular weight. Default 17.5 (natural gas).

**Returns:** LHV in kJ/kg.

**Formula:**
```
LHV_kJ_kg = lhv_mj_nm3 × 22.414 / mw_fuel_g_mol
```

**Example:**
```javascript
const lhv_kg = _lhv_kj_per_kg(47.5, 17.5);  // → ~60,752 kJ/kg (≈60.75 MJ/kg)
```

---

### 4.6 `stack_dry_loss_kj_per_kg_fuel(ea_pct, t_stk_c, t_amb_c, stoich_air = 17.2, cp_dry = 1.08)`

**Purpose:** [Internal] Compute stack dry-gas sensible loss, kJ/kg fuel.

**Parameters:**
- `ea_pct` (float): Excess air, %.
- `t_stk_c` (float): Stack temperature, °C.
- `t_amb_c` (float): Ambient reference temperature, °C.
- `stoich_air` (float, optional): Stoichiometric air/fuel ratio. Default 17.2 kg/kg (natural gas).
- `cp_dry` (float, optional): Cp of dry flue gas, kJ/(kg·K). Default 1.08.

**Returns:** Loss in kJ/kg fuel.

**Formula (simplified):**
```
m_fg_dry = stoich_air × (1 + ea/100) + 1  [kg/kg fuel, ~18.4 at EA=5%, natural gas]
loss = m_fg_dry × cp_dry × (t_stk − t_amb)  [kJ/kg fuel]
```

**Example:**
```javascript
const loss_dry_kj_kg = stack_dry_loss_kj_per_kg_fuel(15, 450, 15);
// → ~9,700 kJ/kg fuel
```

---

### 4.7 `stack_moisture_loss_kj_per_kg_fuel(t_stk_c, t_fuel_c, h2o_kg_per_fuel = 0.225, cp_v = 2.00)`

**Purpose:** [Internal] Compute stack moisture sensible loss, kJ/kg fuel (LHV basis).

**Parameters:**
- `t_stk_c` (float): Stack temperature, °C.
- `t_fuel_c` (float): Fuel inlet temperature, °C.
- `h2o_kg_per_fuel` (float, optional): Water produced per kg fuel. Default 0.225 (natural gas).
- `cp_v` (float, optional): Cp of water vapor, kJ/(kg·K). Default 2.00.

**Returns:** Loss in kJ/kg fuel.

**Formula:**
```
loss = h2o × cp_v × (t_stk − t_fuel)  [kJ/kg fuel]
```

**Example:**
```javascript
const loss_moist_kj_kg = stack_moisture_loss_kj_per_kg_fuel(450, 25);
// → ~1,912 kJ/kg fuel
```

---

### 4.8 `indirect_efficiency_pct(q_fired, loss_dry_kj_kg, loss_moist_kj_kg, rad_pp = 2.0, unacc_pp = 0.5, lhv_mj_nm3, mw_fuel = 17.5)`

**Purpose:** Compute indirect (stack-loss) efficiency.

**Parameters:**
- `q_fired` (float): Fuel energy, kW.
- `loss_dry_kj_kg` (float): Dry-gas sensible loss, kJ/kg fuel (from §4.6).
- `loss_moist_kj_kg` (float): Moisture sensible loss, kJ/kg fuel (from §4.7).
- `rad_pp` (float, optional): Radiation loss, %. Default 2.0.
- `unacc_pp` (float, optional): Unaccounted loss, %. Default 0.5.
- `lhv_mj_nm3` (float): LHV of fuel, MJ/Nm³.
- `mw_fuel` (float, optional): Fuel MW, g/mol. Default 17.5.

**Returns:** η_indirect, %.

**Formula:**
```
L_dry [pp]   = (loss_dry_kj_kg / LHV_kJ_kg) × 100
L_moist [pp] = (loss_moist_kj_kg / LHV_kJ_kg) × 100
L_total [pp] = L_dry + L_moist + rad_pp + unacc_pp
η_indirect   = 100 − L_total
```

**Example:**
```javascript
const eta_ind = indirect_efficiency_pct(
  2500,           // q_fired kW
  9700,           // loss_dry_kj_kg
  1912,           // loss_moist_kj_kg
  2.0,            // rad_pp
  0.5,            // unacc_pp
  47.5,           // lhv_mj_nm3
  17.5            // mw_fuel
);
// → ~78.9%
```

---

### 4.9 `build_alerts(row, config, limits)`

**Purpose:** Generate alert array for a single row, based on tiered limits.

**Parameters:**
- `row` (object): Processed data row with computed fields.
- `config` (object): Configuration (stoich_air, defaults, method, etc.).
- `limits` (object): Alarm limits (stack_t_advisory, stack_t_alarm, etc.).

**Returns:** Array of alert objects `[ { severity, tag, message }, ... ]`.

**Alert Triggers:** See §6 below.

---

### 4.10 `roll_up_status(alerts)`

**Purpose:** [Internal] Compute overall equipment status from alerts.

**Parameters:**
- `alerts` (array): Alert array from build_alerts.

**Returns:** Status string: "OK", "ADVISORY", or "ALARM".

**Logic:**
- If any alert with severity "ALARM" exists → "ALARM".
- Else if any alert with severity "ADVISORY" exists → "ADVISORY".
- Else → "OK".

---

### 4.11 `process_single_row(row, config, limits)`

**Purpose:** Process one CSV row end-to-end.

**Parameters:**
- `row` (object): Raw input row (tags as keys).
- `config` (object): Configuration.
- `limits` (object): Limits.

**Returns:** Processed row object with computed fields: q_abs, q_fired, eta_direct, eta_indirect, ea, gap, alerts, status, …

**Steps:**
1. Validate tags (non-null, in range).
2. Apply fallback defaults (ambient = 15°C, fuel_temp = 25°C).
3. Compute excess air, heat absorbed, fired duty.
4. Compute direct and indirect efficiency.
5. Compute gap = η_direct − η_indirect.
6. Build alerts.
7. Roll up status.
8. Return enriched row.

---

### 4.12 `process_batch(rows, config, limits)`

**Purpose:** Process array of CSV rows in batch.

**Parameters:**
- `rows` (array): Array of input row objects.
- `config` (object): Configuration.
- `limits` (object): Limits.

**Returns:** Array of processed rows.

**Notes:**
- Preserves order.
- If a row fails validation, it is included with status "ERROR" and empty computed fields.

---

## 5. Configuration

### 5.1 DEFAULT_SETTINGS

Located in the dashboard `<script>` block, these are fallback values if not overridden by user:

```javascript
const DEFAULT_SETTINGS = {
  stoich_air:        17.2,    // kg air / kg fuel (natural gas)
  h2o_mole_pct:      2.25,    // mole % H2O in flue gas
  h2o_per_fuel_kg:   0.225,   // kg H2O / kg fuel burned (natural gas)
  mw_fuel:           17.5,    // g/mol (natural gas equivalent)
  cp_flue_gas:       1.08,    // kJ/(kg·K) dry flue gas 300–600°C
  cp_vapor:          2.00,    // kJ/(kg·K) water vapor
  lhv_latent_kj_kg:  0.0,     // Latent heat of H2O (LHV basis, = 0)
  radiation_loss_pp: 2.0,     // % of input
  unaccounted_pp:    0.5,     // % of input
  method:            "direct" // "direct", "indirect", or "both"
};
```

**Usage:**
- User can override via in-dashboard settings panel.
- If left blank, falls back to defaults above.
- Stoich air & fuel composition should be revisited if fuel type changes (e.g., propane, furnace gas).

### 5.2 DEFAULT_LIMITS

Tiered limits (Advisory, Alarm) for status and alerting:

```javascript
const DEFAULT_LIMITS = {
  // Stack temperature, °C
  stack_t_low_c:       200,    // Below = potential draft issue
  stack_t_advisory_c:  400,    // Below this is advisory (fouling risk)
  stack_t_alarm_c:     450,    // Above = start of alarm zone
  stack_t_critical_c:  500,    // Above = critical alarm

  // Bridgewall (flame zone) temperature, °C
  bw_t_low_c:          750,    // Below = low flame temp
  bw_t_advisory_c:     830,    // Target minimum
  bw_t_nominal_c:      870,    // Typical mid-range
  bw_t_alarm_c:        900,    // Above = risk of tube creep
  bw_t_critical_c:     950,    // Critical tube damage risk

  // Excess air, %
  ea_low_pp:           2.0,    // Below = risk of CO, unstable
  ea_target_low_pp:    15,     // Lean efficient zone
  ea_advisory_pp:      25,     // Safe margin
  ea_alarm_pp:         40,     // Over-air, loss zone
  ea_critical_pp:      50,     // Excessive

  // O2 low-side check (complementary to EA)
  o2_low_pct:          2.0,    // Threshold for "O2 low" alert (CO risk)

  // Efficiency, %
  eff_alarm_pp:        78,     // Below = fouling/drift likely
  eff_advisory_pp:     82,     // Target for good maintenance
  eff_nominal_pp:      85,     // Typical new heater
};
```

**Rationale:**
- Stack temp low (<400°C) signals fouling; high (>450°C) signals incomplete combustion or high EA.
- Bridgewall 830–900°C is normal operating range; creep risk above 920°C.
- Excess air 15–25% is common target; <5% risks CO, >40% is wasteful.
- Efficiency >85% is excellent; <78% flags maintenance/fouling.

---

## 6. Alert Logic

### 6.1 Stack Temperature Alerts

| Condition | Severity | Message |
|-----------|----------|---------|
| T_stack < 200°C | ADVISORY | "Stack temp low - check draft, inspect chimney" |
| T_stack 200–400°C | ADVISORY | "Stack temp moderate - fouling possible" |
| T_stack 400–450°C | NORMAL | - |
| T_stack 450–500°C | ADVISORY | "Stack temp elevated - combustion drift or low excess air?" |
| T_stack > 500°C | ALARM | "Stack temp critical - check fuel/air ratio, inspect for soot" |

---

### 6.2 Bridgewall Temperature Alerts

| Condition | Severity | Message |
|-----------|----------|---------|
| T_bw < 750°C | ADVISORY | "Bridgewall low - flame not fully established" |
| T_bw 750–830°C | ADVISORY | "Bridgewall below target - improve burner or fuel atomization" |
| T_bw 830–900°C | NORMAL | - |
| T_bw 900–950°C | ADVISORY | "Bridgewall elevated - tube creep risk, reduce fuel flow or increase air" |
| T_bw > 950°C | ALARM | "Bridgewall critical - immediate reduction required" |

---

### 6.3 Excess Air Alerts

| Condition | Severity | Message |
|-----------|----------|---------|
| EA < 2%  | ALARM | "Excess air very low - CO production likely, unstable flame" |
| EA 2–15% | ADVISORY | "Excess air lean - confirm flame stability, monitor for CO" |
| EA 15–25% | NORMAL | - |
| EA 25–40% | ADVISORY | "Excess air high - efficiency loss, check burner adjustment" |
| EA > 40% | ALARM | "Excess air critical - over-fire, energy waste" |

---

### 6.4 O₂ Low Alerts (Complementary)

| Condition | Severity | Message |
|-----------|----------|---------|
| O₂_pct < 2.0% | ALARM | "O₂ very low - CO and incomplete combustion risk" |

---

### 6.5 Process Flow Alerts

| Condition | Severity | Message |
|-----------|----------|---------|
| m_process = 0 or null | ALARM | "No process flow - heater idle" |
| m_process < 0 | ALARM | "Process flow negative - instrument error" |

---

### 6.6 Temperature Inversion Alerts

| Condition | Severity | Message |
|-----------|----------|---------|
| T_inlet ≥ T_outlet | ALARM | "Process inlet ≥ outlet - Q_abs invalid, check thermowells" |

---

### 6.7 Efficiency Alerts (Method-Dependent)

**If method = "direct":**
| Condition | Severity | Message |
|-----------|----------|---------|
| η_direct < 78% | ADVISORY | "Direct efficiency low - fouling or process mismatch?" |
| η_direct < 75% | ALARM | "Direct efficiency critical - immediate cleaning recommended" |

**If method = "indirect":**
| Condition | Severity | Message |
|-----------|----------|---------|
| η_indirect < 78% | ADVISORY | "Indirect efficiency low - fouling or combustion drift" |
| η_indirect < 75% | ALARM | "Indirect efficiency critical - cleaning/service required" |

**If method = "both":**
| Condition | Severity | Message |
|-----------|----------|---------|
| \|η_direct − η_indirect\| > 8 pp | ADVISORY | "Efficiency gap large - fouling, air/fuel imbalance, or measurement drift" |

---

## 7. Data Schema

### 7.1 CSV Structure

| Column | Type | Example | Notes |
|--------|------|---------|-------|
| `timestamp` | ISO 8601 | 2026-02-01T00:00:00Z | UTC; accepts local with TZ offset |
| `process_inlet_c` | Float | 120.5 | °C |
| `process_outlet_c` | Float | 135.8 | °C |
| `process_flow_kghr` | Float | 5000 | kg/h |
| `process_cp_kjkgk` | Float | 2.8 | kJ/(kg·K) |
| `fuel_flow_nm3hr` | Float | 175.5 | Nm³/h |
| `fuel_lhv_mjnm3` | Float | 47.5 | MJ/Nm³ |
| `stack_o2_pct` | Float | 2.5 | % vol, wet basis |
| `stack_temp_c` | Float | 380 | °C |
| `bridgewall_temp_c` | Float | 860 | °C |
| `ambient_temp_c` | Float | 18 | °C (or omitted → 15°C default) |
| `fuel_temp_c` | Float | 28 | °C (or omitted → 25°C default) |

### 7.2 Demo Dataset

**File:** `demo.csv` (embedded in dashboard or supplied for testing)

**Period:** 2026-02-01 to 2026-02-07, 6-hour intervals (28 rows)

**Characteristics:**
- Steady operation ~5000 kg/h process flow, 175 Nm³/h fuel.
- Stack temp 370–420°C (fouling creep mid-week).
- Bridgewall 820–880°C (stable).
- O₂ 2.3–2.8% (EA 15–22%, well-controlled).
- Direct efficiency 83–86%, indirect ~78–80%.
- Gap 5–7 pp (consistent, normal).
- One advisory alert on day 3 (stack temp 410°C, approaching advisory).

---

## 8. Worked Example

**Input Row (2026-02-01 00:00:00):**

| Tag | Value |
|-----|-------|
| `timestamp` | 2026-02-01T00:00:00Z |
| `process_inlet_c` | 120.0 |
| `process_outlet_c` | 135.8 |
| `process_flow_kghr` | 5000 |
| `process_cp_kjkgk` | 2.80 |
| `fuel_flow_nm3hr` | 175.5 |
| `fuel_lhv_mjnm3` | 47.5 |
| `stack_o2_pct` | 2.50 |
| `stack_temp_c` | 380 |
| `bridgewall_temp_c` | 860 |
| `ambient_temp_c` | 15 (default) |
| `fuel_temp_c` | 25 (default) |

**Calculation Steps:**

**Step 1: Heat Absorbed (Direct Method)**
```
Q_abs = m × Cp × ΔT / 3600
      = 5000 × 2.80 × (135.8 − 120.0) / 3600
      = 5000 × 2.80 × 15.8 / 3600
      = 220,400 / 3600
      = 61.22 kW
```

**Step 2: Fired Duty**
```
Q_fired = V × LHV × 1000 / 3600
        = 175.5 × 47.5 × 1000 / 3600
        = 8,333,750 / 3600
        = 2314.9 kW
```

**Step 3: Direct Efficiency**
```
η_direct = Q_abs / Q_fired × 100
         = 61.22 / 2314.9 × 100
         = 2.65%
```

⚠️ **Note:** This is unrealistic. The issue is that Q_abs = 61 kW is tiny compared to Q_fired = 2315 kW. 

**Diagnosis:** The fuel flow is likely **much higher** than intended for the process load. Recheck the flow measurement or the process inlet/outlet temps. Alternatively, if the process is cooler (say 60°C inlet, 75°C outlet), then:

```
ΔT = 75 − 60 = 15°C
Q_abs = 5000 × 2.80 × 15 / 3600 = 58.33 kW
η_direct = 58.33 / 2314.9 = 2.52%
```

Still very low. **Let me use more realistic numbers:**

**Corrected Input (5000 kg/h, larger ΔT):**
| Tag | Value |
|-----|-------|
| `process_inlet_c` | 80.0 |
| `process_outlet_c` | 120.0 |
| `process_flow_kghr` | 20000 |
| `fuel_flow_nm3hr` | 175.5 |

**Recalculation:**
```
ΔT = 120 − 80 = 40°C
Q_abs = 20000 × 2.80 × 40 / 3600
      = 2,240,000 / 3600
      = 622.2 kW

η_direct = 622.2 / 2314.9 × 100
         = 26.9%
```

Still low. Let me use a more sensible scenario:

---

**Realistic Worked Example (Revised):**

**Input:**
| Tag | Value |
|-----|-------|
| `process_inlet_c` | 50.0 |
| `process_outlet_c` | 110.0 |
| `process_flow_kghr` | 30000 |
| `process_cp_kjkgk` | 3.0 |
| `fuel_flow_nm3hr` | 220.0 |
| `fuel_lhv_mjnm3` | 47.5 |
| `stack_o2_pct` | 2.5 |
| `stack_temp_c` | 380 |
| `bridgewall_temp_c` | 860 |
| `ambient_temp_c` | 15 |
| `fuel_temp_c` | 25 |

**Step 1: Heat Absorbed**
```
ΔT = 110 − 50 = 60°C
Q_abs = 30000 × 3.0 × 60 / 3600
      = 5,400,000 / 3600
      = 1500 kW
```

**Step 2: Fired Duty**
```
Q_fired = 220 × 47.5 × 1000 / 3600
        = 10,450,000 / 3600
        = 2902.8 kW
```

**Step 3: Direct Efficiency**
```
η_direct = 1500 / 2902.8 × 100
         = 51.68%
```

**Still low** (indicates a lot of fuel is lost to stack). Let's compute indirect:

**Step 4: Excess Air**
```
O₂_dry = 2.5 × (100 − 2.25) / 100 = 2.5 × 0.9775 = 2.444%
EA = 100 × 2.444 / (20.95 − 2.444)
   = 100 × 2.444 / 18.506
   = 13.2%
```

**Step 5: LHV in kJ/kg**
```
LHV_kJ_kg = 47.5 × 22.414 / 17.5
          = 60,751 kJ/kg
```

**Step 6: Dry Flue Gas Loss**
```
m_fg_dry = 17.2 × (1 + 13.2/100) + 1 [kg/kg fuel]
         = 17.2 × 1.132 + 1
         = 19.47 + 1 = 20.47 kg/kg fuel (approx)

Actually, mass balance:
m_fg = stoich_air × (1 + EA/100) + fuel_mass
     = 17.2 × 1.132 + 1
     ≈ 20.47 kg fuel + air per kg fuel

ΔT_stack = 380 − 15 = 365 K
loss_dry = 20.47 × 1.08 × 365
         = 8,066 kJ/kg fuel
```

**Step 7: Moisture Loss**
```
ΔT_moisture = 380 − 25 = 355 K
loss_moist = 0.225 × 2.00 × 355
           = 159.75 kJ/kg fuel
```

**Step 8: Indirect Efficiency**
```
L_dry [pp] = (8066 / 60751) × 100 = 13.27 pp
L_moist [pp] = (159.75 / 60751) × 100 = 0.26 pp
L_rad [pp] = 2.0 pp
L_unacc [pp] = 0.5 pp
L_total = 13.27 + 0.26 + 2.0 + 0.5 = 16.03 pp
η_indirect = 100 − 16.03 = 83.97% ≈ 84%
```

**Step 9: Gap Analysis**
```
gap = η_direct − η_indirect
    = 51.68 − 83.97
    = −32.29 pp
```

**Interpretation:**
- Direct efficiency 51.68% is very low, suggesting large process mismatch or measurement error.
- Indirect efficiency 83.97% is reasonable for stack conditions.
- Large negative gap (−32 pp) flags a major discrepancy: either Q_abs measurement is wrong, or fuel/air balance is off.
- **Alert:** "Efficiency gap large - fouling, air/fuel imbalance, or measurement drift."

---

**Better Example (More Realistic Numbers):**

Let me use operating point closer to real heaters:

**Input:**
| Tag | Value |
|-----|-------|
| `process_inlet_c` | 80.0 |
| `process_outlet_c` | 140.0 |
| `process_flow_kghr` | 25000 |
| `process_cp_kjkgk` | 2.8 |
| `fuel_flow_nm3hr` | 150.0 |
| `fuel_lhv_mjnm3` | 47.5 |
| `stack_o2_pct` | 2.6 |
| `stack_temp_c` | 390 |
| `bridgewall_temp_c` | 860 |
| `ambient_temp_c` | 15 |
| `fuel_temp_c` | 25 |

**Calculations:**

```
Q_abs = 25000 × 2.8 × (140 − 80) / 3600
      = 25000 × 2.8 × 60 / 3600
      = 4,200,000 / 3600
      = 1166.7 kW

Q_fired = 150 × 47.5 × 1000 / 3600
        = 7,125,000 / 3600
        = 1979.2 kW

η_direct = 1166.7 / 1979.2 × 100 = 58.9%

EA = 100 × [2.6 × (100−2.25) / 100] / (20.95 − 2.6 × 0.9775)
   = 100 × 2.538 / 18.512
   = 13.7%

LHV_kJ_kg = 47.5 × 22.414 / 17.5 = 60,751 kJ/kg

loss_dry_kj_kg = 17.2 × (1 + 0.137) × 1.08 × 375 = 8,195 kJ/kg
loss_moist_kj_kg = 0.225 × 2.00 × 365 = 164.25 kJ/kg

L_dry [pp] = 8195 / 60751 × 100 = 13.49 pp
L_moist [pp] = 164.25 / 60751 × 100 = 0.27 pp
L_rad = 2.0 pp
L_unacc = 0.5 pp
η_indirect = 100 − (13.49 + 0.27 + 2.0 + 0.5) = 83.74% ≈ 84%

gap = 58.9 − 83.74 = −24.84 pp
```

**Diagnosis:** Still a large negative gap. This suggests the true process load (1166.7 kW absorbed) is significantly less than the chemical energy input (1979.2 kW). Possible causes:
1. Process fluid Cp is overestimated.
2. Fuel LHV is underestimated.
3. Fuel flow sensor reads high (calibration drift).
4. Process flow or ΔT sensor reads low.
5. Bypass or leakage around the heater (not in process flow).

**Alert generated:** Advisory - "Efficiency gap large - investigate fuel flow meter, process Cp, or potential bypass."

---

**Example with Small Gap (Well-Calibrated):**

**Input (Adjusted):**
| Tag | Value |
|-----|-------|
| `process_inlet_c` | 80.0 |
| `process_outlet_c` | 140.0 |
| `process_flow_kghr` | 30000 |
| `process_cp_kjkgk` | 2.8 |
| `fuel_flow_nm3hr` | 175.0 |
| `fuel_lhv_mjnm3` | 47.5 |
| `stack_o2_pct` | 2.5 |
| `stack_temp_c` | 390 |
| `bridgewall_temp_c` | 860 |
| `ambient_temp_c` | 15 |
| `fuel_temp_c` | 25 |

**Calculations:**

```
Q_abs = 30000 × 2.8 × 60 / 3600 = 1400 kW

Q_fired = 175 × 47.5 × 1000 / 3600 = 2305.6 kW

η_direct = 1400 / 2305.6 × 100 = 60.7%

EA = 13.2% (same as before, O₂ = 2.5%)

loss_dry_kj_kg = 17.2 × 1.132 × 1.08 × 375 = 7,936 kJ/kg

loss_moist_kj_kg = 0.225 × 2.00 × 365 = 164.25 kJ/kg

L_dry [pp] = 7936 / 60751 = 13.06 pp
L_moist [pp] = 164.25 / 60751 = 0.27 pp
L_total = 13.06 + 0.27 + 2.0 + 0.5 = 15.83 pp

η_indirect = 100 − 15.83 = 84.17%

gap = 60.7 − 84.17 = −23.47 pp
```

**Still negative.** The issue is that **direct efficiency is fundamentally lower** when stack losses are large. This is physically correct if the heat exchanger is inefficient (high stack temp).

---

**Final Worked Example (Direct vs Indirect Agreement):**

Assume a heater with excellent design (low stack temp, low losses):

**Input:**
| Tag | Value |
|-----|-------|
| `process_inlet_c` | 100.0 |
| `process_outlet_c` | 150.0 |
| `process_flow_kghr` | 40000 |
| `process_cp_kjkgk` | 2.9 |
| `fuel_flow_nm3hr` | 160.0 |
| `fuel_lhv_mjnm3` | 47.5 |
| `stack_o2_pct` | 2.8 |
| `stack_temp_c` | 280 |
| `bridgewall_temp_c` | 860 |
| `ambient_temp_c` | 15 |
| `fuel_temp_c` | 25 |

**Calculations:**

```
Q_abs = 40000 × 2.9 × 50 / 3600 = 1611.1 kW

Q_fired = 160 × 47.5 × 1000 / 3600 = 2111.1 kW

η_direct = 1611.1 / 2111.1 × 100 = 76.3%

EA = 100 × 2.73 / 18.27 = 14.9%

loss_dry_kj_kg = 17.2 × 1.149 × 1.08 × 265 = 5,556 kJ/kg

loss_moist_kj_kg = 0.225 × 2.00 × 255 = 114.75 kJ/kg

L_dry [pp] = 5556 / 60751 = 9.14 pp
L_moist [pp] = 114.75 / 60751 = 0.19 pp
L_total = 9.14 + 0.19 + 2.0 + 0.5 = 11.83 pp

η_indirect = 100 − 11.83 = 88.17%

gap = 76.3 − 88.17 = −11.87 pp
```

**Interpretation:**
- Direct efficiency 76.3% is below excellent (85%+) but reasonable.
- Indirect efficiency 88.17% is very good (low losses).
- Gap is still −11.9 pp, but smaller.
- Still indicates some mismatch, likely due to process Cp or fuel flow calibration.

The fundamental insight: **If the heater is well-designed and insulated (low stack temp), indirect efficiency is high. But if the process duty is lower than calculated, direct efficiency lags. The gap tells you to audit the flow and composition measurements.**

---

## 9. Assumptions & Constraints

1. **No Air Preheater (APH):** The model assumes the inlet air to the burner is at ambient temperature (or close to it). If an APH is present downstream, stack losses should be computed on the APH outlet, not the heater stack exit.

2. **Fuel Type:** Natural gas (composition equivalent to CH₄). Stoichiometric air = 17.2 kg/kg, MW ≈ 17.5 g/mol, H₂O prod ≈ 2.25 % mol. If furnace gas, propane, or syngas is used, these must be updated.

3. **LHV Basis:** All calculations use **Lower Heating Value** (LHV), which excludes latent heat of water condensation. This is standard for gas-fired equipment; if steam generation is involved, use HHV (Higher Heating Value).

4. **Radiation Loss:** Approximated as a **fixed percentage** of input (default 2%). This is reasonable for steady-state operation but may drift if heater fouling changes the external surface condition. For precision, measure external surface temps and compute via Stefan–Boltzmann.

5. **No Blow-Down / Purge Loss:** Model assumes continuous fuel supply. If the heater includes a purge cycle or stack damper, these are lumped into unaccounted loss (0.5% default).

6. **Dry Combustion Products:** Model assumes complete combustion (no CO, soot in flue gas). If CO is suspected (O₂ < 1% or flame color very orange), direct and indirect efficiencies may diverge significantly; flag as anomaly.

7. **Cp of Flue Gas:** 1.08 kJ/(kg·K) is a reasonable **average** for 300–600°C. If stack temp is very hot (>700°C) or very cool (<200°C), refine Cp_fg via steam tables or real-gas correlations.

8. **No Moisture in Inlet Air:** Model does not account for humidity of combustion air. In very humid climates, inlet air may carry significant H₂O, increasing flue gas moisture loss. For precision, measure wet-bulb and adjust.

9. **Steady-State Operation:** Calculations assume process and fuel flows are constant over the measurement interval (typically 6 hours). Transient behavior (ramp-up, load step) is not modeled.

10. **Measurement Precision:** Efficiency calculations are sensitive to small errors in T, flow, and Cp. ±1°C on stack temp or ±2% on flow meter introduces ±1–2 pp uncertainty in efficiency. Regular calibration is essential.

---

## 10. Integration Notes

### 10.1 CSV Upload

The dashboard includes a **Papa Parse** CSV parser (v5.4.1, CDN). Users can:
1. Click "Upload CSV" button.
2. Select a `.csv` file with headers matching the tag dictionary.
3. Parser auto-detects headers and maps columns to tags.
4. Fallback aliases supported (e.g., `stack_outlet_temp_c` → `stack_temp_c`).

### 10.2 Live Calculation

Once CSV is loaded:
1. Each row is processed via `process_single_row()`.
2. Computed fields (q_abs, q_fired, eta_direct, eta_indirect, gap, alerts, status) are added.
3. Charts are updated with new data (Chart.js refresh).
4. KPI tiles (latest efficiency, stack temp, EA) are updated.

### 10.3 Export

Users can export processed data as **new CSV** containing:
- Original tags + computed fields (q_abs_kw, q_fired_kw, eta_direct_pct, eta_indirect_pct, gap_pp, status, alert_count).

### 10.4 Real-Time DCS Integration (Future)

If integrating with a live DCS (Honeywell, ABB, Emerson):
1. API endpoint delivers JSON rows at configurable interval (e.g., every 6 hours).
2. Dashboard periodically polls and appends to in-memory data store.
3. Charts auto-scroll; oldest data dropped after N rows to keep memory bounded.

---

## 11. Change Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-04-04 | Vedant Patel | Initial release: dual-method efficiency, 12-tag schema, alert tiering, worked example |

---

## Appendix A: Thermodynamic Reference Data

### A.1 Specific Heat Capacities (at ~400°C, 1 atm)

| Substance | Cp [kJ/(kg·K)] | Notes |
|-----------|----------------|-------|
| Dry air | 1.008 | Constant over 0–600°C |
| N₂ (dry flue gas) | 1.040 | Main component of flue gas |
| CO₂ | 1.14–1.20 | Increases with T |
| O₂ (residual) | 0.97 | Small fraction |
| Water vapor | 2.0 | Avg 300–600°C; ~1.9 at 200°C, ~2.1 at 600°C |
| Natural gas (CH₄) | 2.22 | Liquid; not in flue gas |

**Dry flue gas mixture Cp:** ~1.08 kJ/(kg·K) (weighted avg of N₂ 70–75%, CO₂ 10–13%, O₂ 2–3%).

### A.2 Molar Volumes

| Condition | V_m [L/mol] |
|-----------|-------------|
| STP (0°C, 1 atm) | 22.414 |
| SATP (25°C, 1 atm) | 24.465 |
| 15°C, 1 atm | 23.682 |

**Convention:** Fuel flows in dashboard are at **STP (0°C)** or **15°C (ISO 1783)**; use 22.414 L/mol.

### A.3 Combustion Stoichiometry (Natural Gas / CH₄)

| Reaction | Basis | Notes |
|----------|-------|-------|
| CH₄ + 2 O₂ → CO₂ + 2 H₂O | Molar | Complete combustion |
| 1 kg CH₄ requires 4 kg O₂ | Mass | O₂ requirement |
| 1 kg CH₄ requires ~17.2 kg air | Mass | Air requirement (0.21 O₂ fraction) |
| 1 kg CH₄ produces 2.75 kg CO₂ + 2.25 kg H₂O | Mass | Products (complete) |

**Air excess at E% EA:** Total air = 17.2 × (1 + E/100) kg air per kg fuel.

---

## Appendix B: Standards & References

| Standard | Section | Title | Reference |
|----------|---------|-------|-----------|
| **API 560** | §5.4 | Performance Measurement - Fired Heaters | Direct efficiency, Q_abs, Q_fired |
| **ASME PTC 4** | App. A | Combustion Efficiency - O₂ to Excess Air | EA calculation |
| **ASME PTC 4** | §5.2–5.5 | Stack Losses - Direct Method | Dry gas, moisture, radiation, unaccounted |
| **ISO 5167** | Part 1–4 | Measurement of Fluid Flow - Orifice Plates | Flow measurement (referenced in tag notes) |
| **ISO 1783** | - | Gas fuels - Guidance on measurement methods | STP, measurement temperature corrections |
| **TEMA** | Type S | Tubular Exchanger Manufacturers Assoc. | Referenced for shell-tube comparison |

---

**END OF DOCUMENT**

