# Phase: Anaerobic Design Tools Integration

**Status:** Planning
**Created:** 2026-02-08
**Priority:** Next phase after ENV-1
**Origin:** Port anaerobic-skill calculations into the QSDsan engine as MCP tools + HTTP API + CLI commands

---

## Objective

Integrate the anaerobic design calculation scripts (heuristic sizing, chemical dosing, mixing power, rheology) from the `anaerobic-skill` into the QSDsan engine as first-class MCP tools, HTTP API endpoints, and CLI commands. This makes the engineering calculations accessible from:

1. **Claude Desktop** via MCP tools
2. **Claude Code** via MCP tools or CLI
3. **n8n workflows** via HTTP API
4. **Web UIs** via HTTP API
5. **Scripts/automation** via CLI

---

## Source Material

The calculations originate from five Python scripts in `~/.claude/skills/anaerobic-skill/scripts/`:

| Script | Purpose | Lines | Dependencies |
|--------|---------|-------|-------------|
| `heuristic_sizing.py` | Reactor volume, TSS threshold, MBR sizing, tank geometry, dewatering | ~900 | stdlib only |
| `chemical_dosing.py` | FeCl3 (sulfide/phosphate), NaOH (pH), Na2CO3 (alkalinity) | ~300 | stdlib only |
| `mixing_calculations.py` | Mechanical (impeller), pumped (recirculation), eductor systems | ~1400 | stdlib only |
| `rheology.py` | WEF MOP-8 viscosity, Baudez power-law, Slatter yield stress | ~350 | stdlib only |
| `convert_to_plantstate.py` | Legacy ADM1 state format conversion | ~300 | stdlib only |

All scripts use **only Python standard library** (math, dataclasses, json, logging) - no external dependencies required.

---

## Architecture

### New Module

Create `core/anaerobic_design.py` containing all calculation functions. This follows the existing pattern where `core/` modules hold shared logic consumed by both `server.py` (MCP) and `cli.py` (CLI).

```
core/
  ├── anaerobic_design.py     # NEW: All anaerobic design calculations
  ├── plant_state.py          # Existing
  ├── model_registry.py       # Existing
  ├── converters.py           # Existing
  ├── kinetic_params.py       # Existing
  └── ...
```

### Tool Exposure

Each calculation function is exposed through three interfaces:

```
core/anaerobic_design.py          # Pure calculation functions (sync)
    │
    ├── server.py @mcp.tool()     # MCP tools (async wrappers)
    │   └── HTTP API via MCP      # Auto-exposed by FastMCP
    │
    └── cli.py @app.command()     # CLI commands (sync)
```

---

## New MCP Tools (7 tools)

### Tool 1: `anaerobic_heuristic_sizing`

**Purpose:** Calculate digester dimensions, volume, flowsheet configuration (high-TSS CSTR vs low-TSS MBR), dewatering requirements.

```python
@mcp.tool()
async def anaerobic_heuristic_sizing(
    flow_m3_d: float,
    cod_mg_l: float,
    temperature_c: float = 35.0,
    target_srt_days: float = 30.0,
    biomass_yield: float = 0.10,
    adm1_state: Dict[str, Any] = None,       # Optional: for substrate-aware yield
    mbr_type: str = "submerged",              # "submerged" or "external_crossflow"
    tank_height_to_diameter: float = 1.2,
    tank_material: str = "concrete",          # "concrete" or "steel_bolted"
    mixing_type: str = "pumped",              # "pumped", "mechanical", "hybrid"
    biogas_application: str = "direct_utilization",
) -> Dict[str, Any]:
    """
    Calculate anaerobic digester sizing using heuristic design rules.

    Decision logic:
    - If steady-state TSS > 10,000 mg/L: High-TSS CSTR (HRT = SRT)
    - If steady-state TSS <= 10,000 mg/L: Low-TSS with MBR (SRT decoupled)

    Returns dict with: digester, operating_conditions, mbr, dewatering,
    mixing, biogas_blower, thermal_analysis_request, sizing_basis,
    flowsheet_decision.
    """
```

**Returns:**
```json
{
  "flowsheet_type": "high_tss|low_tss_mbr",
  "digester": {
    "liquid_volume_m3": 2000,
    "vapor_volume_m3": 200,
    "total_volume_m3": 2200,
    "hrt_days": 20,
    "srt_days": 30,
    "diameter_m": 12.5,
    "height_m": 15.0,
    "type": "CSTR|AnMBR"
  },
  "mbr": {
    "required": true,
    "type": "submerged",
    "total_area_m2": 833,
    "number_of_modules": 70,
    "flux_lmh": 5
  },
  "dewatering": {
    "flow_m3_h": 2.5,
    "dry_solids_kg_h": 25.0
  },
  "sizing_basis": {
    "feed_flow_m3d": 100,
    "cod_mg_l": 7000,
    "cod_load_kg_d": 700,
    "biomass_yield_kg_tss_kg_cod": 0.10,
    "biomass_production_kg_d": 70,
    "steady_state_tss_mg_l": 700,
    "target_srt_days": 30
  },
  "flowsheet_decision": {
    "selected": "low_tss_mbr",
    "reason": "Steady-state TSS (700 mg/L) below threshold (10,000 mg/L)",
    "tss_threshold_mg_l": 10000
  }
}
```

### Tool 2: `anaerobic_chemical_dosing`

**Purpose:** Calculate chemical doses for sulfide removal, phosphate removal, pH control, and alkalinity adjustment.

```python
@mcp.tool()
async def anaerobic_chemical_dosing(
    sulfide_mg_l: float = 0.0,
    phosphate_mg_p_l: float = 0.0,
    alkalinity_meq_l: float = 0.0,
    ph_current: float = 7.0,
    ph_target: float = 7.0,
    alkalinity_target_meq_l: float = 0.0,
    sulfide_removal_target: float = 0.90,
    phosphate_removal_target: float = 0.80,
    sulfide_safety_factor: float = 1.2,
    phosphate_safety_factor: float = 1.5,
    temperature_c: float = 35.0,
) -> Dict[str, Any]:
    """
    Calculate chemical dosing requirements for anaerobic digesters.

    Calculates doses for:
    - FeCl3 for sulfide removal (2Fe3+ + 3S2- -> Fe2S3)
    - FeCl3 for phosphate removal (Fe3+ + PO43- -> FePO4)
    - NaOH for pH adjustment
    - Na2CO3 for alkalinity supplementation

    Returns dict with dose calculations for each chemical.
    """
```

**Returns:**
```json
{
  "fecl3_for_sulfide": {
    "fecl3_dose_mg_l": 145.2,
    "fe3_added_mg_l": 50.0,
    "cl_added_mg_l": 95.2,
    "sulfide_removed_mg_l": 90.0,
    "stoichiometry": "2Fe3+ + 3S2- -> Fe2S3"
  },
  "fecl3_for_phosphate": {
    "fecl3_dose_mg_l": 85.3,
    "phosphate_removed_mg_p_l": 8.0
  },
  "naoh_for_ph": {
    "naoh_dose_mg_l": 0.0,
    "na_added_mg_l": 0.0,
    "ph_change": 0.0
  },
  "na2co3_for_alkalinity": {
    "na2co3_dose_mg_l": 0.0,
    "alkalinity_increase_meq_l": 0.0
  },
  "combined_ion_impact": {
    "total_fe_added_mg_l": 50.0,
    "total_cl_added_mg_l": 95.2,
    "total_na_added_mg_l": 0.0
  }
}
```

### Tool 3: `anaerobic_mixing_power`

**Purpose:** Calculate mixing power requirements for mechanical, pumped, or eductor systems.

```python
@mcp.tool()
async def anaerobic_mixing_power(
    tank_volume_m3: float,
    tank_diameter_m: float,
    mixing_type: str = "pumped",              # "mechanical", "pumped", "eductor"
    # Mechanical parameters
    impeller_type: str = "pitched_blade_turbine",
    d_t_ratio: float = 0.33,
    target_power_w_m3: float = 7.0,
    # Pumped/eductor parameters
    recirculation_rate_m3_h: float = 0.0,
    pump_head_m: float = 5.0,
    pump_efficiency: float = 0.65,
    entrainment_ratio: float = 5.0,           # For eductor: total/motive
    # Fluid properties
    fluid_density_kg_m3: float = 1010.0,
    fluid_viscosity_pa_s: float = 0.01,
) -> Dict[str, Any]:
    """
    Calculate mixing power for anaerobic digesters.

    Supports three modes:
    - mechanical: Impeller-based (Reynolds, power number correlations)
    - pumped: Simple recirculation
    - eductor: Jet pump with entrainment (CRITICAL: sizes pump for motive flow only)

    Returns power calculations, Reynolds number, flow regime.
    """
```

**Returns:**
```json
{
  "mixing_type": "mechanical",
  "power_total_kw": 14.0,
  "power_intensity_w_m3": 7.0,
  "impeller_speed_rpm": 45,
  "impeller_diameter_m": 4.1,
  "reynolds_number": 168000,
  "flow_regime": "turbulent",
  "tip_speed_m_s": 4.8,
  "warnings": []
}
```

### Tool 4: `anaerobic_sludge_rheology`

**Purpose:** Estimate sludge viscosity and non-Newtonian parameters from total solids and temperature.

```python
@mcp.tool()
async def anaerobic_sludge_rheology(
    total_solids_percent: float,
    temperature_c: float = 35.0,
    sludge_type: str = "digested",   # "digested" or "raw"
) -> Dict[str, Any]:
    """
    Estimate sludge rheological properties.

    Uses WEF MOP-8 correlations for viscosity, Baudez et al. (2011) for
    power-law parameters, and Slatter (2001) for yield stress.

    Returns viscosity, power-law K and n, yield stress.
    """
```

**Returns:**
```json
{
  "viscosity_pa_s": 0.038,
  "viscosity_cp": 38.0,
  "power_law": {
    "K_pa_s_n": 5.30,
    "n": 0.392,
    "valid_range_ts_percent": [3, 8],
    "reference": "Baudez et al. 2011"
  },
  "yield_stress_pa": 0.64,
  "yield_stress_reference": "Slatter 2001",
  "fluid_behavior": "non-newtonian",
  "temperature_c": 35.0,
  "total_solids_percent": 4.0,
  "sludge_type": "digested",
  "warnings": []
}
```

### Tool 5: `anaerobic_validate_composites`

**Purpose:** Validate that mADM1 state variables match target bulk parameters (COD, TSS, VSS, TKN, TP).

```python
@mcp.tool()
async def anaerobic_validate_composites(
    state: Dict[str, Any],
    targets: Dict[str, float],
    tolerance: float = 0.10,
) -> Dict[str, Any]:
    """
    Validate mADM1 state composites against measured bulk parameters.

    Computes COD, TSS, VSS, TKN, TP from individual components and
    compares against user-supplied targets within tolerance.

    Args:
        state: PlantState dict (model_type, flow_m3_d, temperature_K, concentrations)
        targets: e.g. {"cod_mg_l": 7000, "tss_mg_l": 1650, "tkn_mg_l": 625}
        tolerance: Fractional tolerance (0.10 = 10%)

    Returns validation results with calculated vs target comparisons.
    """
```

**Returns:**
```json
{
  "valid": true,
  "calculated": {
    "cod_mg_l": 6934.0,
    "tss_mg_l": 1659.4,
    "vss_mg_l": 1252.2,
    "tkn_mg_l": 625.4,
    "tp_mg_l": 40.2
  },
  "targets": {"cod_mg_l": 7000, "tss_mg_l": 1650, "tkn_mg_l": 625},
  "deviations": {"cod_mg_l": 0.0094, "tss_mg_l": 0.0057, "tkn_mg_l": 0.0006},
  "max_deviation": 0.0094,
  "tolerance": 0.10,
  "message": "All composite targets met within tolerance"
}
```

### Tool 6: `anaerobic_validate_ion_balance`

**Purpose:** Validate charge balance and calculate equilibrium pH using the engine's PCM solver.

```python
@mcp.tool()
async def anaerobic_validate_ion_balance(
    state: Dict[str, Any],
    target_ph: float = 7.0,
    max_ph_deviation: float = 0.5,
) -> Dict[str, Any]:
    """
    Validate ion balance and equilibrium pH for mADM1 state.

    Uses the engine's native PCM (physicochemical module) to calculate
    true equilibrium pH from the charge balance of all ionic species.

    CRITICAL: WasteStream.pH is just a stored attribute - this tool
    calculates the REAL pH the simulation will produce.

    Returns equilibrium pH, deviation from target, and pass/fail.
    """
```

**Returns:**
```json
{
  "equilibrium_ph": 7.70,
  "target_ph": 7.0,
  "ph_deviation": 0.70,
  "max_ph_deviation": 0.5,
  "balanced": false,
  "charge_balance": {
    "cation_meq_l": 42.5,
    "anion_meq_l": 41.8,
    "imbalance_meq_l": 0.7
  },
  "adjustment_hints": {
    "direction": "lower_ph",
    "suggestion": "Add anions (S_Cl, S_SO4) or reduce cations"
  },
  "message": "pH deviation (0.70) exceeds limit (0.5). Equilibrium pH: 7.70, Target: 7.0"
}
```

### Tool 7: `anaerobic_full_design`

**Purpose:** Orchestrate the complete anaerobic design workflow in a single call. Combines validation, sizing, and dosing calculations.

```python
@mcp.tool()
async def anaerobic_full_design(
    state: Dict[str, Any],
    targets: Dict[str, float] = None,
    target_srt_days: float = 30.0,
    temperature_c: float = 35.0,
    h2s_threshold_mg_l: float = 150.0,
    mixing_type: str = "pumped",
    tolerance: float = 0.10,
) -> Dict[str, Any]:
    """
    Run complete anaerobic design workflow (Steps 3-6, 9 of the skill).

    Performs in sequence:
    1. Validate composites (if targets provided)
    2. Validate ion balance
    3. Calculate heuristic sizing
    4. Estimate sludge rheology
    5. Calculate mixing power
    6. Calculate chemical dosing (if H2S exceeds threshold)

    Does NOT run simulation (use simulate_system separately).

    Returns combined design package.
    """
```

**Returns:**
```json
{
  "validation": {
    "composites": { "...": "..." },
    "ion_balance": { "...": "..." }
  },
  "sizing": { "...": "..." },
  "rheology": { "...": "..." },
  "mixing": { "...": "..." },
  "dosing": { "...": "..." },
  "design_summary": {
    "flowsheet_type": "high_tss",
    "reactor_volume_m3": 2000,
    "hrt_days": 20,
    "srt_days": 30,
    "mixing_power_kw": 14.0,
    "fecl3_dose_mg_l": 145.2,
    "estimated_biogas_m3_d": 850,
    "warnings": []
  }
}
```

---

## CLI Commands (7 commands)

Map each MCP tool to a CLI command under an `anaerobic` subcommand group:

```bash
# Heuristic sizing
python cli.py anaerobic sizing \
  --flow 100 --cod 7000 --temperature 35 --srt 30 --json-out

# Chemical dosing
python cli.py anaerobic dosing \
  --sulfide 100 --phosphate 10 --json-out

# Mixing power
python cli.py anaerobic mixing \
  --volume 2000 --diameter 12.5 --type pumped --json-out

# Sludge rheology
python cli.py anaerobic rheology \
  --total-solids 5.0 --temperature 35 --json-out

# Validate composites
python cli.py anaerobic validate-composites \
  --state plant_state.json \
  --targets '{"cod_mg_l": 7000, "tss_mg_l": 1650}' \
  --tolerance 0.10 --json-out

# Validate ion balance
python cli.py anaerobic validate-ion-balance \
  --state plant_state.json \
  --target-ph 7.0 --max-ph-deviation 0.5 --json-out

# Full design workflow
python cli.py anaerobic full-design \
  --state plant_state.json \
  --targets '{"cod_mg_l": 7000}' \
  --srt 30 --temperature 35 --json-out
```

**Note:** The existing CLI commands `validate-composites`, `validate-ion-balance`, and `validate-finalize` in `cli.py` already implement some of this logic. The new `anaerobic` subcommand group consolidates and extends these.

---

## HTTP API Endpoints

FastMCP auto-exposes MCP tools as HTTP endpoints. The following will be available:

| Endpoint | Method | MCP Tool |
|----------|--------|----------|
| `/api/anaerobic_heuristic_sizing` | POST | `anaerobic_heuristic_sizing` |
| `/api/anaerobic_chemical_dosing` | POST | `anaerobic_chemical_dosing` |
| `/api/anaerobic_mixing_power` | POST | `anaerobic_mixing_power` |
| `/api/anaerobic_sludge_rheology` | POST | `anaerobic_sludge_rheology` |
| `/api/anaerobic_validate_composites` | POST | `anaerobic_validate_composites` |
| `/api/anaerobic_validate_ion_balance` | POST | `anaerobic_validate_ion_balance` |
| `/api/anaerobic_full_design` | POST | `anaerobic_full_design` |

These can be called from n8n HTTP Request nodes, web UIs, or any HTTP client.

---

## n8n Integration

A future n8n workflow version can use these endpoints to automate anaerobic design:

```
Webhook Input (wastewater characteristics)
  → HTTP Request: POST /api/anaerobic_full_design
  → HTTP Request: POST /api/simulate_system (anaerobic_cstr_madm1)
  → Poll: GET /api/get_job_status
  → HTTP Request: GET /api/get_job_results
  → Generate Report (Code node)
  → Convert to PDF (Gotenberg)
  → Upload to Supabase
```

The n8n workflow version number is TBD (not necessarily v10).

---

## Implementation Plan

### Step 1: Create `core/anaerobic_design.py`

Port the five scripts into a single module with clean function signatures:

```python
# core/anaerobic_design.py

from dataclasses import dataclass
from typing import Dict, Any, Optional, Literal
import math

# --- Heuristic Sizing ---

def calculate_biomass_production(
    flow_m3_d: float,
    cod_mg_l: float,
    biomass_yield: float = 0.10,
    adm1_state: Optional[Dict[str, float]] = None,
) -> Dict[str, float]:
    """Calculate biomass production and steady-state TSS."""

def calculate_digester_sizing(
    flow_m3_d: float,
    cod_mg_l: float,
    target_srt_days: float = 30.0,
    biomass_yield: float = 0.10,
    adm1_state: Optional[Dict[str, Any]] = None,
    mbr_type: str = "submerged",
    tank_height_to_diameter: float = 1.2,
    tank_material: str = "concrete",
    mixing_type: str = "pumped",
    biogas_application: str = "direct_utilization",
    temperature_c: float = 35.0,
) -> Dict[str, Any]:
    """Full heuristic sizing with flowsheet decision."""

# --- Chemical Dosing ---

def calculate_fecl3_sulfide_dose(
    sulfide_mg_l: float,
    target_removal: float = 0.90,
    safety_factor: float = 1.2,
) -> Dict[str, float]:
    """FeCl3 dosing for sulfide removal."""

def calculate_fecl3_phosphate_dose(
    phosphate_mg_p_l: float,
    target_removal: float = 0.80,
    safety_factor: float = 1.5,
) -> Dict[str, float]:
    """FeCl3 dosing for phosphate removal."""

def calculate_naoh_dose(
    alkalinity_meq_l: float,
    ph_current: float,
    ph_target: float,
) -> Dict[str, float]:
    """NaOH dosing for pH adjustment."""

def calculate_na2co3_dose(
    alkalinity_current_meq_l: float,
    alkalinity_target_meq_l: float,
) -> Dict[str, float]:
    """Na2CO3 dosing for alkalinity supplementation."""

def calculate_chemical_dosing(
    sulfide_mg_l: float = 0.0,
    phosphate_mg_p_l: float = 0.0,
    alkalinity_meq_l: float = 0.0,
    ph_current: float = 7.0,
    ph_target: float = 7.0,
    alkalinity_target_meq_l: float = 0.0,
    **kwargs,
) -> Dict[str, Any]:
    """Combined dosing calculation."""

# --- Mixing Power ---

def calculate_mechanical_mixing(
    tank_volume_m3: float,
    tank_diameter_m: float,
    impeller_type: str = "pitched_blade_turbine",
    d_t_ratio: float = 0.33,
    target_power_w_m3: float = 7.0,
    fluid_density_kg_m3: float = 1010.0,
    fluid_viscosity_pa_s: float = 0.01,
) -> Dict[str, Any]:
    """Mechanical (impeller) mixing calculation."""

def calculate_pumped_mixing(
    tank_volume_m3: float,
    recirculation_rate_m3_h: float,
    pump_head_m: float = 5.0,
    pump_efficiency: float = 0.65,
    fluid_density_kg_m3: float = 1010.0,
    mixing_mode: str = "simple",
    entrainment_ratio: float = 5.0,
) -> Dict[str, Any]:
    """Pumped/eductor mixing calculation."""

# --- Rheology ---

def estimate_sludge_viscosity(
    total_solids_percent: float,
    temperature_c: float = 35.0,
    sludge_type: str = "digested",
) -> Dict[str, float]:
    """WEF MOP-8 viscosity estimation."""

def estimate_power_law_parameters(
    total_solids_percent: float,
    temperature_c: float = 35.0,
) -> Dict[str, float]:
    """Baudez et al. 2011 power-law parameters."""

def estimate_yield_stress(
    total_solids_percent: float,
) -> float:
    """Slatter 2001 yield stress."""

def calculate_rheology(
    total_solids_percent: float,
    temperature_c: float = 35.0,
    sludge_type: str = "digested",
) -> Dict[str, Any]:
    """Combined rheology calculation."""

# --- Composite Validation ---

def validate_composites(
    concentrations: Dict[str, float],
    targets: Dict[str, float],
    tolerance: float = 0.10,
) -> Dict[str, Any]:
    """Validate mADM1 composites against bulk parameters."""

# --- Full Design ---

def full_anaerobic_design(
    state: Dict[str, Any],
    targets: Dict[str, float] = None,
    target_srt_days: float = 30.0,
    temperature_c: float = 35.0,
    h2s_threshold_mg_l: float = 150.0,
    mixing_type: str = "pumped",
    tolerance: float = 0.10,
) -> Dict[str, Any]:
    """Orchestrate complete design workflow."""
```

### Step 2: Add MCP Tools to `server.py`

Add 7 new `@mcp.tool()` functions after the existing TEA tools (after ~line 2830). Each tool is a thin async wrapper around the `core/anaerobic_design.py` functions.

### Step 3: Add CLI Commands to `cli.py`

Add `anaerobic` subcommand group with 7 commands using `typer.Typer()` sub-app pattern.

### Step 4: Write Tests

Create `tests/test_anaerobic_design.py`:

| Test Category | Count (est.) | Description |
|---------------|-------------|-------------|
| Sizing calculations | 10 | Volume, TSS threshold, MBR sizing, geometry |
| Chemical dosing | 8 | FeCl3 sulfide/phosphate, NaOH, Na2CO3 |
| Mixing power | 8 | Mechanical, pumped, eductor, Reynolds |
| Rheology | 6 | Viscosity, power-law, yield stress |
| Composite validation | 6 | COD/TSS/TKN/TP calculation and tolerance |
| Full design workflow | 4 | End-to-end orchestration |
| MCP tool contracts | 7 | Tool signature validation |
| Edge cases | 5 | Zero inputs, extreme values, warnings |

**Estimated total: ~54 tests**

### Step 5: Update Documentation

- Update CLAUDE.md tool count and categories
- Add anaerobic design tools to NOTES_FOR_SKILLS.md
- Update API_REFERENCE.md with new endpoints

### Step 6: Docker Build & Deploy

No new dependencies needed (stdlib only). Rebuild Docker image to include `core/anaerobic_design.py`.

---

## Composite Validation Formulas

The composite validation requires computing bulk parameters from individual mADM1 components. These formulas must be implemented in `core/anaerobic_design.py`:

### COD Calculation

```python
# All concentrations in kg/m3; convert to mg/L by * 1000
COD_COMPONENTS = {
    # Soluble substrates (COD = concentration for organic species)
    'S_su': 1.0, 'S_aa': 1.0, 'S_fa': 1.0,
    'S_va': 1.0, 'S_bu': 1.0, 'S_pro': 1.0, 'S_ac': 1.0,
    'S_h2': 1.0, 'S_ch4': 1.0, 'S_I': 1.0,
    # Particulate substrates
    'X_ch': 1.0, 'X_pr': 1.0, 'X_li': 1.0, 'X_I': 1.0,
    # Biomass (1.42 g COD/g VSS for heterotrophs)
    'X_su': 1.0, 'X_aa': 1.0, 'X_fa': 1.0, 'X_c4': 1.0,
    'X_pro': 1.0, 'X_ac': 1.0, 'X_h2': 1.0,
    'X_PAO': 1.0, 'X_PHA': 1.0,
    'X_hSRB': 1.0, 'X_aSRB': 1.0, 'X_pSRB': 1.0, 'X_c4SRB': 1.0,
}
cod_mg_l = sum(conc * COD_COMPONENTS.get(comp, 0) for comp, conc in concentrations.items()) * 1000
```

### TSS Calculation

```python
# TSS = all particulate (X_*) components
TSS_COMPONENTS = [
    'X_ch', 'X_pr', 'X_li',
    'X_su', 'X_aa', 'X_fa', 'X_c4', 'X_pro', 'X_ac', 'X_h2', 'X_I',
    'X_PHA', 'X_PP', 'X_PAO',
    'X_hSRB', 'X_aSRB', 'X_pSRB', 'X_c4SRB',
    # Minerals
    'X_CCM', 'X_ACC', 'X_ACP', 'X_HAP', 'X_DCPD', 'X_OCP',
    'X_struv', 'X_newb', 'X_magn', 'X_kstruv',
    'X_FeS', 'X_Fe3PO42', 'X_AlPO4',
    # HFO
    'X_HFO_H', 'X_HFO_L', 'X_HFO_old', 'X_HFO_HP', 'X_HFO_LP',
    'X_HFO_HP_old', 'X_HFO_LP_old',
]
tss_mg_l = sum(concentrations.get(comp, 0) for comp in TSS_COMPONENTS) * 1000
```

### TKN Calculation

```python
# TKN = S_IN + organic-N in amino acids/proteins + biomass-N
tkn = concentrations.get('S_IN', 0)  # Inorganic N [kg N/m3]
tkn += concentrations.get('S_aa', 0) * 0.11  # 11% N in amino acids
tkn += concentrations.get('X_pr', 0) * 0.11  # 11% N in proteins
# Biomass: ~8% N content
for comp in ['X_su', 'X_aa', 'X_fa', 'X_c4', 'X_pro', 'X_ac', 'X_h2',
             'X_PAO', 'X_hSRB', 'X_aSRB', 'X_pSRB', 'X_c4SRB']:
    tkn += concentrations.get(comp, 0) * 0.08
tkn_mg_l = tkn * 1000  # Convert to mg N/L
```

### TP Calculation

```python
# TP = S_IP + organic-P in biomass + X_PP + mineral-P
tp = concentrations.get('S_IP', 0)  # Inorganic P [kg P/m3]
tp += concentrations.get('X_PP', 0)  # Polyphosphate
# Biomass: ~2% P content
for comp in ['X_su', 'X_aa', 'X_fa', 'X_c4', 'X_pro', 'X_ac', 'X_h2', 'X_PAO']:
    tp += concentrations.get(comp, 0) * 0.02
tp_mg_l = tp * 1000  # Convert to mg P/L
```

---

## Relationship to Existing Validation

The existing `cli.py` already has `validate-composites`, `validate-ion-balance`, and `validate-finalize` commands. These call into `core/converters.py` functions. The new anaerobic tools should:

1. **Reuse** existing charge balance and mass balance logic from `core/converters.py`
2. **Add** the composite calculation (COD/TSS/TKN/TP from individual components) which is new
3. **Wrap** the existing PCM pH solver for ion-balance validation
4. **Not duplicate** - the existing `validate_state` MCP tool handles component presence and charge balance; the new tools add target-based composite validation

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| COD calculation disagrees with QSDsan | Medium | High | Validate against QSDsan's `WasteStream.composite()` |
| Substrate-aware yield model oversimplifies | Low | Medium | Default to constant yield; substrate-aware is opt-in |
| Non-Newtonian rheology out of calibration range | Low | Low | Add warnings for TS% outside 3-8% |
| Eductor entrainment ratio misunderstood | Medium | High | Add explicit warnings and documentation |
| Large server.py file (already 2889 lines) | Medium | Low | Core logic in `core/anaerobic_design.py`; server.py only has thin wrappers |

---

## Acceptance Criteria

- [ ] All 7 MCP tools return correct results for known test cases
- [ ] CLI commands produce identical results to MCP tools
- [ ] HTTP API accessible from n8n (test with curl)
- [ ] Composite validation matches existing CLI `validate-composites` output
- [ ] Ion-balance validation matches existing CLI `validate-ion-balance` output
- [ ] Sizing calculations match original `heuristic_sizing.py` output
- [ ] Chemical dosing matches original `chemical_dosing.py` output
- [ ] Mixing power matches original `mixing_calculations.py` output
- [ ] Rheology matches WEF MOP-8 reference values (within 5%)
- [ ] ~54 new tests passing
- [ ] Docker image builds and deploys successfully
- [ ] Existing 450+ tests still pass (no regressions)

---

## Estimated Effort

| Task | Estimate |
|------|----------|
| `core/anaerobic_design.py` | Main effort - port and refactor 5 scripts |
| `server.py` additions | 7 thin async wrappers |
| `cli.py` additions | 7 CLI commands with Typer |
| `tests/test_anaerobic_design.py` | ~54 tests |
| Documentation updates | CLAUDE.md, API_REFERENCE.md, NOTES_FOR_SKILLS.md |
| Docker rebuild & test | Minimal (no new deps) |

---

## Dependencies

- No new Python packages required (all stdlib)
- Existing `core/converters.py` for charge balance logic
- Existing `core/plant_state.py` for PlantState parsing
- Existing `models/madm1.py` for PCM pH solver (ion-balance validation)

---

## Future Extensions

1. **n8n workflow integration** - Use the HTTP API endpoints in a future n8n workflow version
2. **Web UI** - The HTTP API can serve a React/Streamlit design calculator
3. **Monte Carlo sensitivity** - Add parameter uncertainty analysis using ranges from the skill
4. **Rob's decision support** - Map `qsdsan_build_spec` modules to these design tools (see `docs/change_plans/phase-rob-decision-support-integration.md`)
