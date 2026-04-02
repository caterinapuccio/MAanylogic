# CE 3.0 — ecoinvent 3.9 Search Strings
## Run these in ecoinvent database before thesis submission

All LCA factors currently in the model are [ILLUSTRATIVE] or [TRAINING].
These must be replaced with verified ecoinvent 3.9 values.

---

## A. Landfill — per material

Search in ecoinvent 3.9 Activity Browser or SimaPro, geography = RoW or Vietnam/SE Asia if available.

| Material | Search string | ALP variable to update |
|---|---|---|
| Plastic | `treatment of waste plastic, mixture, sanitary landfill` | gwp_landfill for PLASTIC_SCRAP[0] |
| Paper | `treatment of waste paper, sanitary landfill` | gwp_landfill for PAPER_WASTE[1] |
| Ferrous metal | `treatment of ferrous metal waste, sanitary landfill` | gwp_landfill for FERROUS_METAL[2] |
| Textile | `treatment of textile waste, sanitary landfill` | gwp_landfill for TEXTILE_WASTE[3] |
| Non-ferrous | `treatment of non-ferrous metal waste, sanitary landfill` | gwp_landfill for NON_FERROUS[4] |

---

## B. Incineration — per material

| Material | Search string | Notes |
|---|---|---|
| Plastic (industrial) | `treatment of waste plastic, industrial incineration` | High calorific value |
| Plastic (municipal, mixed) | `treatment of waste plastic, municipal incineration` | For MSW provider |
| Paper | `treatment of waste paper, municipal incineration` | |
| Organic/MSW general | `treatment of organic fraction, biowaste, municipal incineration` | For MSW organic fraction |
| Textile | `treatment of textile waste, municipal incineration` | |
| General MSW | `market for municipal solid waste, for incineration` | For WasteToBeBurned aggregate |

---

## C. Recycling benefit — formal IS pathway

These represent the environmental savings from recycling instead of virgin production.

| Material | Search string | Use for |
|---|---|---|
| Recycled plastic pellets | `market for recycled plastic granulate, polypropylene` OR `polyethylene` | Output value from BIWASE plastic recycling |
| Recycled paper pulp | `market for recycled pulp` | Output of BIWASE paper chain |
| Secondary steel (EAF) | `steel production, electric, low-alloyed` | IS path ferrous recycling |
| Secondary aluminium | `aluminium production, secondary, from scrap` | If NON_FERROUS is Al-dominant |
| Secondary copper | `copper production, from sulfide ore` vs `secondary copper` | If Cu is modeled |

---

## D. Ash residues — incineration outputs

**Critical for thesis**: these are NOT yet applied in CombustionSink.

| Residue | Search string | Fraction of burned mass |
|---|---|---|
| Bottom ash (non-hazardous) | `treatment of bottom ash from municipal solid waste incineration, sanitary landfill` | 25% (bottomAsh_massFraction) |
| Fly ash (hazardous) | `treatment of fly ash from municipal incineration, residual material landfill` | 4% (flyAsh_massFraction) |

After brick production is implemented: the bottom ash fraction that goes to brick production has LOWER landfill impact and should be credited with "avoided impact" of virgin brick production.

---

## E. Composting (MSW organic fraction)

| Process | Search string | Notes |
|---|---|---|
| Composting | `treatment of biowaste, composting` OR `market for compost, at plant` | For MSW 43.2% organic fraction |
| Compost product use | `market for compost` | Revenue/credit side if BIWASE sells compost |

---

## F. Transport

| Mode | Search string | Notes |
|---|---|---|
| Waste truck | `transport, freight, lorry 7.5-16 metric ton, EURO5` | Or `EURO3` for older Vietnam fleet |
| Vietnam-specific | Check for `Vietnam` geography; if not available use `RAS` (Rest of Asia and Pacific) | |

---

## G. Informal smelting (noIS ferrous pathway)

This is needed for the IS vs. noIS GWP comparison for ferrous.

| Process | Search string | Notes |
|---|---|---|
| Formal EAF recycling | `steel production, electric, low-alloyed` | IS path |
| Informal induction furnace | No direct ecoinvent entry likely. Check: `steel production, converter, unalloyed` as proxy + add penalty factors | noIS path: higher GWP, dioxin, PM2.5 |
| Literature supplement | Nguyen et al. 2019 (Resources Conservation & Recycling) for SEA informal smelting emission factors | Use as [LIT] supplement to ecoinvent |

---

## H. Brick production (future, when implemented)

| Process | Search string | Notes |
|---|---|---|
| Brick from ash | `brick production, fly ash` OR `autoclaved aerated concrete block` | Ash-based brick is non-standard — check literature |
| Avoided virgin brick | `market for clay brick` | Credit for replacing virgin brick |

---

## Implementation notes

1. All values should be extracted as **GWP100 (kg CO2-eq / functional unit)** first
2. Also extract HTP, LUP, CED if available (needed for multi-indicator LCA)
3. Functional unit = 1 kg treated/produced (the ALP works in kg internally for LCA factors)
4. Preferred geography order: Vietnam → Southeast Asia (RAS) → Global (GLO) → RoW
5. After extracting values, update the `Materials` Excel sheet with new rows for LCA factors, and update `gwpFactor_*` variables in the ALP (currently in Intermediate sheet rows 12-17)

---

*Document created: 03APRIL session. Update after ecoinvent search is complete.*
