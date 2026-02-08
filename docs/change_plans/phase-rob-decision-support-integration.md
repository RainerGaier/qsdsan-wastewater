# Phase: Rob's Decision Support System Integration

**Status:** Planning (Awaiting discussion with Rob)
**Created:** 2026-02-05
**Priority:** TBD after stakeholder meeting

---

## Overview

Rob has developed a decision support system that outputs structured treatment train recommendations (`qsdsan_build_spec`). This document captures the analysis and options for integrating this output with QSDsan flowsheet construction APIs.

---

## Rob's Decision Tree Output Format

Example `qsdsan_build_spec` from Rob's system:

```json
{
  "qsdsan_build_spec": {
    "treatment_train": [
      "Screening",
      "AOP/adsorption",
      "Anaerobic reactor (UASB)",
      "Aerobic bioreactor",
      "Sludge handling"
    ],
    "modules": {
      "Screening": {
        "type": "preliminary",
        "notes": "Coarse screens for large solids removal",
        "parameters": {}
      },
      "AOP/adsorption": {
        "type": "tertiary",
        "notes": "UV or ozone-based AOP recommended for PFAS destruction; GAC as polishing step",
        "parameters": {
          "target_contaminants": ["PFAS"]
        }
      },
      "Anaerobic reactor (UASB)": {
        "type": "secondary",
        "parameters": {
          "HRT_hours": {"min": 6, "max": 12},
          "OLR_kgCOD_m3_d": {"min": 5, "max": 15},
          "temperature_C": {"min": 30, "max": 38}
        }
      },
      "Aerobic bioreactor": {
        "type": "secondary",
        "parameters": {
          "HRT_hours": {"min": 4, "max": 8},
          "DO_mg_L": {"min": 1.5, "max": 3.0},
          "MLSS_mg_L": {"min": 3000, "max": 5000}
        }
      },
      "Sludge handling": {
        "type": "sludge",
        "notes": "Dewatering and disposal; consider anaerobic digestion if volume warrants",
        "parameters": {}
      }
    },
    "recommendations": [
      "Consider MBR instead of conventional aerobic process for better effluent quality",
      "UASB may generate biogas - consider energy recovery",
      "AOP placement after biological treatment recommended for PFAS"
    ],
    "uncertainty": {
      "influent_variability": "high",
      "parameter_confidence": "medium",
      "cost_estimate_accuracy": "±30%"
    }
  }
}
```

---

## Current QSDsan Flowsheet APIs

The existing MCP/CLI tools for flowsheet construction:

| Tool | Purpose |
|------|---------|
| `create_flowsheet_session` | Initialize session with model (ASM2d, mADM1) |
| `create_stream` | Add influent/recycle streams |
| `create_unit` | Add unit operations (CSTR, MBR, Clarifier, etc.) |
| `connect_units` | Wire units together |
| `build_system` | Compile to QSDsan System |
| `simulate_built_system` | Run simulation |

### Unit Registry

49 unit operations available in `core/unit_registry.py`:
- **Reactors:** CSTR, AnaerobicCSTR, PFR, ActivatedSludgeProcess
- **Separators:** CompletelyMixedMBR, AnMBR, PolishingFilter
- **Clarifiers:** FlatBottomCircularClarifier, PrimaryClarifier
- **Sludge:** Thickener, Centrifuge, SludgeDigester, DryingBed
- **Junctions:** ASM2dtoADM1, ADM1toASM2d, mADM1toASM2d

---

## Gap Analysis

### What Rob's Spec Provides
- High-level treatment train sequence (process types)
- Parameter ranges with uncertainty
- Recommendations and notes
- Target contaminants

### What QSDsan APIs Need
- Specific unit types (e.g., "CSTR" not "Anaerobic reactor")
- Exact parameter values (not ranges)
- Explicit connections between units
- Model specification (ASM2d vs mADM1)
- Stream definitions (flow, concentrations)

### Translation Challenges

1. **Module → Unit Mapping**
   - "Anaerobic reactor (UASB)" → `AnaerobicCSTR` or `UASB` unit?
   - "Aerobic bioreactor" → `CSTR` with aeration, `MBR`, or `ActivatedSludgeProcess`?
   - "Screening" → Not in current unit registry (preliminary treatment gap)
   - "AOP/adsorption" → Not in current unit registry (advanced treatment gap)

2. **Parameter Translation**
   - HRT_hours → V_max calculation: `V = Q × HRT`
   - OLR → Influent COD validation
   - Ranges → Need strategy (use midpoint? user selection?)

3. **Model Zone Transitions**
   - UASB (mADM1) → Aerobic (ASM2d) requires junction unit
   - Current system supports mixed-model flowsheets (Phase 9)

4. **Missing Units**
   - Screening/bar screens
   - AOP (UV, ozone)
   - GAC adsorption
   - UASB (specific vs generic AnaerobicCSTR)

---

## Proposed Integration Options

### Option A: n8n Workflow Extension (v10)

Add translation logic to the n8n workflow:

```
Rob's API → qsdsan_build_spec → n8n Translation Node → QSDsan APIs
```

**Pros:**
- Non-invasive to core engine
- Flexible, can iterate quickly
- n8n visual debugging

**Cons:**
- JavaScript-based translation may be complex
- Harder to unit test
- Duplicates logic if CLI also needs it

### Option B: New QSDsan API Endpoint

Create `/api/build_from_spec` that accepts Rob's format:

```python
@mcp.tool()
def build_from_spec(
    build_spec: dict,
    influent_state: dict,
    parameter_strategy: str = "midpoint"  # or "min", "max", "ask"
) -> dict:
    """Translate high-level spec to flowsheet and simulate."""
```

**Pros:**
- Reusable across MCP, CLI, n8n
- Proper Python implementation with unit tests
- Can include validation and suggestions

**Cons:**
- More development effort
- Needs to handle unknown module types gracefully

### Option C: Hybrid Approach

n8n does high-level parsing, QSDsan handles specifics:

1. n8n extracts treatment train and parameters
2. QSDsan endpoint maps modules to unit sequences
3. QSDsan returns flowsheet with gaps highlighted
4. User/AI resolves gaps interactively

---

## Questions for Rob

1. **Module Vocabulary:** Is the module naming fixed, or can it be aligned with QSDsan unit types?

2. **Parameter Ranges:** How should we handle uncertainty ranges?
   - Always use midpoint?
   - Monte Carlo sampling for sensitivity?
   - Interactive user selection?

3. **Contaminant Handling:** How do we represent target contaminants (PFAS) in simulation?
   - Add custom components?
   - Use surrogate parameters?

4. **Decision Rationale:** Can the spec include reasoning for why certain processes were selected? (Useful for reports)

5. **Iteration Support:** Will users want to modify Rob's recommendations before simulation?

---

## Next Steps

1. **Schedule meeting** with Rob to discuss integration approach
2. **Clarify module vocabulary** and mapping requirements
3. **Decide on parameter handling** strategy
4. **Identify unit registry gaps** that need filling
5. **Choose integration option** (A, B, or C)

---

## Related Documentation

- Phase 9: Mixed-Model Flowsheet Support (`docs/completed-plans/phase9-mixed-model-flowsheet-support.md`)
- Unit Registry: `core/unit_registry.py`
- n8n v9 Workflow: `n8n/n8n-qsd-test/qsdsan-simulation-v9.json`
