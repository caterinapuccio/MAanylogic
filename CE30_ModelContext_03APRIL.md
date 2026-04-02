# CE 3.0 Model Context Document — 03APRIL
## For Claude: Read this at the start of every new session

**Model**: Circular Economy 3.0 — Industrial Symbiosis at VSIP1, Binh Duong, Vietnam
**Tool**: AnyLogic 8.9.8, file `03APRIL.alp` + `03APRIL.xlsx`
**Owner**: Caterina Puccio, Master's thesis, German university / SimTech Singapore
**Git workspace**: `/sessions/upbeat-sweet-gates/git-workspace/` (last commit: ccefc11, Phase 13)
**Active folder**: `/sessions/upbeat-sweet-gates/mnt/anylogic/03APRIL/`

---

## 1. Model Architecture Overview

The model is a **hybrid ABM + DES + SD** simulation of an IS (Industrial Symbiosis) network. Three paradigms:

- **ABM**: Agents (Provider, Receiver, Intermediate/BIWASE, GarbageTruck) each have statecharts and autonomous decision logic
- **DES**: Material processing chains inside the Intermediate agent (visualization + timing; accounting is in Java methods)
- **SD**: Not yet implemented; planned for MSW seasonal multiplier

### Agent Classes

| Class | Role | Count | Key fields |
|---|---|---|---|
| `Main` | Simulation controller, startup, NPV, LCA accumulators | 1 | npv, totalCapex, conversionEffPerMat[5], transportCost |
| `Provider` | Waste generator. Weekly generation, IS/noIS decision | 66 | myMaterial, avgSupply_tWeek, entityNPV, financialBalance |
| `Receiver` | Buys secondary material output from BIWASE | 5 (one per material) | myMaterial, entityNPV, financialBalance |
| `Intermediate` | BIWASE — sorts, stores, processes, sells secondary material | 1 | recyclingStoragePerMat[5], currentStockPerMat[5], entityNPV |
| `GarbageTruck` | Physical transport provider→BIWASE and BIWASE→landfill | 66+ jolly | dispatchPending, isTransporting, isBusy |
| `EndofLife` | Da Phuoc Landfill placeholder (currently minimal) | 1 | |

### Material Types (enum MaterialType)

```
PLASTIC_SCRAP(0), PAPER_WASTE(1), FERROUS_METAL(2), TEXTILE_WASTE(3), NON_FERROUS_METAL(4)
```

All arrays indexed by this enum: `conversionEffPerMat[5]`, `pw_landfillSharePerMat[5]`, etc.

---

## 2. IS Network Flow (Step by Step)

1. **Provider** generates `rawWaste_t` weekly (triangular distribution around `avgSupply_tWeek`)
2. If `useIS=true`: Provider sends booking request to Intermediate via `acceptBooking(mat, qty, prov)`
3. **Intermediate** checks Tier1/Tier2 storage capacity. If accepted: dispatches a GarbageTruck
4. **GarbageTruck** travels from BIWASE to Provider, loads waste, returns to BIWASE
5. At BIWASE: waste enters `recyclingStoragePerMat[m]`
6. Monthly: `recyclingProcessingEvent` fires → `recyclingProcessing()`:
   - Applies `conversionEffPerMat[m]` → produces `recycledStock` (goes to `currentStockPerMat[m]`) and `processWaste`
   - Injects DES chains for visualization (recyclStart, paper_Start, fe_Start, text_Start, nfe_Start)
   - Process waste staged in `pw_pending*` → `pwStart.inject()` for LCA accounting
7. **Receiver** of the matching material type periodically checks BIWASE stock and buys it
8. If `useIS=false` or booking rejected: Provider goes to **NonIS_Disposal** statechart state → GarbageTruck takes waste to landfill/incineration → `WasteToBeBurned` accumulates → weekly `weeklyLandfillBurnEvent` triggers combustion DES

---

## 3. DES Chains Inside Intermediate Agent

All DES chains are **visualization + timing only**. Actual mass accounting happens in Java code.

### Recycling DES chains (one per material type)
| Chain | Steps | Source ref |
|---|---|---|
| PLASTIC (recyclStart) | Shred→Wash→Separation→Extrude | [63] |
| PAPER (paper_Start) | Baling→Loading | [64,65] |
| FERROUS (fe_Start) | Sorting→Grading→PreTreatment | [66,67] |
| TEXTILE (text_Start) | CutShredding→FibreReprocessing | [68,69] |
| NON_FERROUS (nfe_Start) | Sorting→Segregation | [70,71] |

### Process Waste DES (pwStart → pwTransport → pwLandfillDeposit → pwSink)
- Models internal movement of recycling residue to landfill truck bay
- NOT external truck transport (GarbageTruck handles that)
- LCA applied in `pwSink.onEnter` via `pw_pending*` staging variables
- Description fields added to all four blocks (Phase 13)

### Combustion DES (combustionStart → wasteDrying → thermalDecomposition → slagFormation → CombustionSink)
- Fires when `WasteToBeBurned >= burningThreshold_t` (threshold read from Intermediate B11)
- `WasteBurned`, `totalCO2EmittedKg`, `GWPActualIncineration` accumulated in CombustionSink.onEnter
- **CRITICAL GAP**: ash fractions (bottomAsh 25%, flyAsh 4%) are computed in `recyclingProcessing()` pw block but NOT in CombustionSink — they need to be added there
- Brick production from bottom ash: NOT YET IMPLEMENTED (see futurenotes.txt [13])

---

## 4. Financial Model (NPV)

### CAPEX deducted at t=0 (undiscounted)
- **Storage CAPEX**: `intTotalStorage × capexPerStorageUnit` (Excel "Intermediate" B7)
- **Recycling plant CAPEX**: `recyclingCapex_SGD` (Excel "Intermediate Investment" B3) — [ILLUSTRATIVE]
- **Fleet CAPEX**: `garbageTrucks.size() × truckCapex_SGD` (Excel TruckFleet K3) — Phase 13 addition

All three assigned to `intermediate.intermediateCapex`, deducted from `npv` and `intermediate.entityNPV`.

### OPEX (monthly, variable-cost model)
```java
monthlyIntOpex = usedCapacity × opexPerStorageUnitPerMonth
```
`usedCapacity = sum(recyclingStoragePerMat[m] + currentStockPerMat[m])` for m=0..4

**Why variable OPEX**: Labor, forklift fuel, and handling equipment scale with waste volume processed.
Current value: `opexPerStorageUnitPerMonth = 2 SGD/(t·month)` [ILLUSTRATIVE — no BIWASE-specific source]

### Transport costs
`transportCostCalc(mass_t, dist_km) = mass_t × dist_km × transportCost`
`transportCost = 0.10 SGD/(t·km)` [ASSUMED — falls within World Bank SEA range 0.08–0.15 USD/(t·km)]

**Phase 13 addition**: `truckBaseWeight_t = 8.0 t` (Excel TruckFleet M3) — for deadhead (empty-trip) cost and emission accounting. NOT YET WIRED into transportCostCalc (deadhead cost formula not yet implemented).

### Entity NPV
Each agent (Provider, Receiver, Intermediate) carries `entityNPV`. Discounted annually at `discountRate` (Excel "Intermediate" B10, default 8%). Year 10: salvage value added to Intermediate NPV.

---

## 5. LCA Model

### LCA indicators tracked
- GWP (kg CO2-eq) — main indicator
- HTP (kg 1,4-DCB-eq)
- LUP (m2·year crop-eq)
- CED (MJ)

### Where LCA is applied
- **Transport**: `GWPActualTransport += mass × dist × gwpTransport_kgCO2eqPerTkm`
- **Process waste landfill** (pwSink): applies `pw_pending*` LCA per material per cycle
- **Incineration** (CombustionSink): applies per-kg factors (`gwpFactor_drying`, `gwpFactor_thermalDecomp`, `gwpFactor_slagFormation`)
- **NoIS disposal** (NonIS_Disposal): applies material-specific LCA factors

### LCA factors — sourcing status
All material-specific recycling and disposal LCA factors should come from **ecoinvent 3.9**.

**ecoinvent search strings to run** (document this for thesis):
```
"market for waste plastic, mixture | waste treatment" → plastic landfill/incineration GWP
"treatment of waste plastic, municipal incineration" → plastic incineration LCA
"market for waste paper | waste treatment" → paper landfill GWP
"treatment of waste paper, sanitary landfill" → paper landfill LCA
"treatment of ferrous scrap, scrap metal facility" → ferrous IS recycling benefit
"steel production, electric, low-alloyed | steel production" → EAF formal recycling GWP
"treatment of fly ash from municipal incineration, residual material landfill" → fly ash hazardous disposal
"treatment of bottom ash from municipal solid waste incineration, sanitary landfill" → bottom ash landfill
"treatment of organic fraction, biowaste, municipal incineration" → organic/MSW incineration
"composting | compost, at plant" → MSW composting LCA
"transport, freight, lorry 7.5-16 metric ton, EURO5 | transport" → truck transport GWP (or Vietnam equivalent)
```

**GWPActualProcess**: this accumulator in `pwSink.onEnter` currently uses `pw_pendingGWP`, which is computed in `recyclingProcessing()`. The GWP for process waste is `processWaste × gwp_landfill_factor_for_material_m`. These factors should be ecoinvent values for "treatment of waste [material type], sanitary landfill". Currently they are [ILLUSTRATIVE] — these are the exact ecoinvent strings you need to run next week.

---

## 6. Excel File Structure (03APRIL.xlsx)

### Key sheets and orange-cell values (read by ALP)
- **Companies_Listing**: 66 providers (rows 2-67), 5 receivers (rows 68-72). Orange: lat, lon, avgSupply_tWeek, supplyVariability, truckCount
- **TruckFleet** (row 3): velocity (A), loading/unloading times (B,C), capacities per material (D-H), jolly count (I), jolly premium (J), capex (K), source annotations (L), baseWeight (M) — all orange
- **Intermediate** (row 3+): storage, OPEX, discount rate, capacity fractions, LCA factors, ash fractions (rows 26-27 orange), HMS1 note (row 28)
- **Intermediate Investment** (row 3+): recyclingCapex_SGD (B3 orange), recyclingOpexAnnual (B4 orange)
- **Materials** (rows 3-7): conversionEffPerMat[0-4] — orange

### Important: orange cells = ALP reads these
Cells with light orange fill (FFFFCC99) are actively read by AnyLogic. Never change their style.

---

## 7. Disposal Arrays (Phase 12-13)

```java
// [0]=PLASTIC [1]=PAPER [2]=FERROUS [3]=TEXTILE [4]=NON_FERROUS
double[] pw_landfillSharePerMat     = {0.70, 0.80, 0.85, 0.50, 0.85}; // recycling residue → landfill
double[] pw_incinerationSharePerMat = {0.25, 0.10, 0.05, 0.40, 0.10}; // recycling residue → incineration
double[] noIS_landfillShare         = {0.70, 0.80, 0.15, 0.55, 0.15}; // [0][1] aligned to pw (modeling assumption)
double[] noIS_incinerationShare     = {0.25, 0.10, 0.05, 0.35, 0.05}; // [0][1] aligned to pw (modeling assumption)
```

[2] FERROUS and [4] NON_FERROUS have very low disposal shares because ~70-85% of industrial ferrous scrap is captured by informal itinerant buyers (*ve chai*) before formal disposal.

---

## 8. Open Design Questions (As of 03APRIL session)

### CRITICAL (block implementation if unresolved)

**Q1: MSW provider material type mapping**
The ESCAR composition (organic 43.2%, recyclables 1.3%, incineration/landfill 49.5%) does NOT map to the 5-material model. Options:
- A (recommended): Only the 1.3% recyclable fraction enters IS. Organic fraction handled as BIWASE-internal composting. The wasteCompositionVec[5] then contains only very small fractions.
- B: Add ORGANIC_WASTE as material type 5 — requires restructuring ALL 5-element arrays to 6 elements
- **Question for Caterina**: Which option?

**Q2: "Fraction of 49.5% is incinerated" — what fraction?**
The ESCAR says 49.5% goes to incineration OR landfill but does not specify the split. What number do you want to use?

**Q3: Ve chai agent — design parameters**
- How many ve chai agents in the model? (1 archetypal, or ~5-10?)
- Geographic coverage: do all providers have equal probability of being approached?
- Price offer: function of formal secondary material price (e.g., 60% of formal ferrous price)?
- Truck capacity (ve chai typically use 1-3t motorbike carts or small trucks)
- Does the informal smelter produce a specific receiver output, or is it a pure waste sink?

**Q4: Industrial vs. municipal plastic — same DES chain?**
Municipal plastic (mixed, contaminated) has much lower recyclability than industrial scrap.
Options:
- A: Same recyclStart chain, different `conversionEffPerMat` value for MSW provider
- B: Separate DES chain (msw_plasticStart) with explicit contamination/sorting steps
- **ecoinvent note**: Industrial plastic scrap → "market for waste plastic, production waste"; Municipal mixed plastic → "market for waste plastic, mixture" — different datasets, different GWP

---

## 9. Future Tasks List (from futurenotes.txt)

Items 1-8: existing bugs and design issues
**Item 9**: Ferrous grading (HMS1 assumption, implement later)
**Item 10**: Non-ferrous material differentiation (Al, Cu, Pb)
**Item 11**: MSW provider — Binh Duong municipality
**Item 12**: Ferrous informal sector — ve chai and noIS emission factor (IS quality benefit)
**Item 13**: Ash material-specific fractions + brick production
**Item 14**: Material-specific recycling OPEX
**Item 15**: Excel sheet consolidation (Intermediate + Intermediate Investment)

---

## 10. Critical Gaps (for thesis defense)

1. **truckBaseWeight_t** (8.0 t) is read at startup but NOT yet used in any cost or emission calculation — deadhead cost formula not implemented
2. **Ash fractions in CombustionSink**: bottomAsh_massFraction/flyAsh_massFraction are computed in `recyclingProcessing()` pw block but NOT in CombustionSink — both should produce ash quantities
3. **Brick production**: bottom ash → transport → brick production DES steps not yet implemented
4. **Hazardous fly ash**: goes to landfill but NO separate LCA factor applied (should use ecoinvent hazardous ash disposal CF)
5. **conversionEffPerMat** applies equally to ALL providers including future MSW — MSW recyclability is 20-40% vs. 85-95% for industrial streams
6. **Truck velocity**: `truckVelocity_kmh` read from Excel but trucks use hardcoded 10 m/s (36 km/h)
7. **Provider truck CAPEX**: Providers have no CAPEX in their NPV — depending on model scope, they may need a capital contribution
8. **No ecoinvent values confirmed**: all LCA factors currently [ILLUSTRATIVE] — need ecoinvent 3.9 database search next week

---

## 11. Git History

| Commit | Phase | Key content |
|---|---|---|
| ae27b04 | 12 | Material-specific disposal arrays + 4 new DES chains |
| ccefc11 | 13 | DES descriptions, disposal array alignment, fleet CAPEX, baseline truck weight |
| Active | 03APRIL | Context document, OPEX comment, ecoinvent search strings list |

---

## 12. How DES Chains Know Their Batch

**Short answer**: They don't — DES entities are anonymous tokens. The batch is identified by which `inject()` call triggered the chain.

In `recyclingProcessing()`:
```java
if (recyc_pendingRaw[0] > 0) recyclStart.inject();  // PLASTIC
if (recyc_pendingRaw[1] > 0) paper_Start.inject();  // PAPER
// etc.
```
The material identity is implicit in which chain fires. Mass values are pre-staged in `recyc_pendingRaw[m]` BEFORE inject. If two materials happen to fire simultaneously, each chain tracks its own accumulator independently.

For MSW: the same pattern would apply. Stage MSW-specific values in `msw_pending*` variables, then inject into an MSW-specific DES chain. The DES chain "knows" it's processing MSW because only MSW staging variables are loaded at that point.

---

## 13. Sources Tagged in Model

- `[LIT]` = peer-reviewed literature (citable in thesis)
- `[TRAINING]` = Claude training knowledge (needs verification via Consensus or ecoinvent)
- `[ASSUMED]` = modeling assumption without specific source
- `[GREY]` = grey literature (industry reports, non-peer-reviewed)
- `[THESIS-SRC]` = from your own thesis supervisor / direct BIWASE data
- `[ILLUSTRATIVE]` = placeholder to be replaced before thesis submission

All ecoinvent values are currently either [ILLUSTRATIVE] or [TRAINING]. These MUST be replaced with verified ecoinvent 3.9 values before thesis submission.
