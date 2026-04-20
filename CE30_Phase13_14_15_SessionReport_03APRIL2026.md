# CE 3.0 Model — Development Session Report
**Date:** 3 April 2026 (SGT +0800), 21:36–02:02  
**Model:** Circular Economy 3.0 — AnyLogic 8.9.8, VSIP1 Industrial Symbiosis, Binh Duong, Vietnam  
**File baseline:** 31MARZ.alp / 31MARZ.xlsx  
**File output:** 03APRIL.alp (17,207 lines) / 03APRIL.xlsx  
**Commits:** ccefc11 (Phase 13), f399c5f (folder init), 8a4140f (Phase 14), ec26623 (Phase 15)  
**Session paused:** 3 April 2026 at ~02:02 SGT — resumed 20 April 2026

---

## Overview

Three successive implementation phases were completed in this session, advancing the CE 3.0 model from a functional five-material recycling simulation (Phase 12) to a more complete industrial symbiosis representation incorporating incineration-linked manufacturing, a sixth material stream with composting, and informal sector agent behaviour. The session also closed a long-standing NPV accounting gap (fleet CAPEX) and corrected a technical limitation in the Excel export function.

---

## Phase 13 — DES Descriptions, Disposal Array Alignment, Fleet CAPEX, Truck Base Weight

**Commit:** ccefc11 · 2026-04-02 21:36 SGT  
**Files modified:** 31MARZ.alp, 31MARZ.xlsx, futurenotes.txt (created, items [1]–[15])

### DES Block Descriptions

Description fields were added to the paper waste DES chain (`pwStart`, `pwTransport`, `pwLandfillDeposit`, `pwSink`). These fields are displayed in the AnyLogic Properties panel when a block is selected and serve as inline documentation for thesis reviewers unfamiliar with the model architecture.

### Disposal Array Alignment

The `noIS_landfillShare` and `noIS_incinerationShare` arrays for PLASTIC_SCRAP (index 0) and PAPER_WASTE (index 1) were aligned to the corresponding `pw_*` withered-IS baseline values. Metal streams (FERROUS, NON_FERROUS) retain differentiated noIS values based on literature. This alignment affects the GWP counterfactual computation in the noIS scenario and was noted as an assumption subject to sensitivity analysis.

### Fleet CAPEX — NPV Gap Closed

A structural gap identified in Phase 3 was closed: the garbage truck fleet capital expenditure is now deducted at simulation time zero from both `Main.npv` and `Intermediate.entityNPV`. The formula is:

```
fleetCapex = garbageTrucks.size() × truckCapex_SGD
```

`truckCapex_SGD` is read from the TruckFleet sheet. Prior to this fix, the NPV calculation omitted the most significant capital item for the collection infrastructure, systematically overstating the IS pathway NPV.

### Truck Base Weight Parameter

`truckBaseWeight_t` was added as a new model parameter (default 8.0 t, read from TruckFleet sheet column M, row 3), tagged `[ASSUMED-LIT]`. This parameter supports future deadhead (empty-trip) cost and emission accounting, which requires the vehicle's unladen weight to compute fuel consumption on return legs. The parameter is loaded but the deadhead formula has not yet been implemented (tracked in futurenotes.txt as a critical gap).

### futurenotes.txt Created

The deferred-tasks register was created in this phase with items [1]–[15], covering: grading of recycled output quality, non-ferrous metal separation DES, MSW provider ABM agent, ferrous informal sector emissions, ash fraction differentiation, material-specific recycling OPEX, and Excel sheet consolidation.

---

## Phase 14 — Brick Plant DES Extension, Brick CAPEX in NPV

**Commit:** 8a4140f · 2026-04-03 01:26 SGT  
**Files modified:** 03APRIL.alp (new model folder), 03APRIL.xlsx

This phase extended the existing combustion DES chain in the Intermediate agent to model bottom-ash valorisation as a secondary construction material (compressed earth/ash bricks), a practice documented for the BIWASE facility in Binh Duong.

### Combustion DES Extended

Three new process blocks were inserted between `slagFormation` and `CombustionSink`:

| Block | Distribution | Purpose |
|---|---|---|
| `ashTransportToBrickKiln` | tri(1, 2, 4) h | Transport delay from combustion bay to brick production unit |
| `brickProduction` | tri(2, 4, 8) h | Pressing and curing cycle for compressed ash–earth bricks |
| `flyAshToLandfill` | tri(0.5, 1, 2) h | Hazardous fly ash fraction routed to controlled landfill cell |

### Mass Accounting

A staging variable (`stagingAshCycleMass_t`) captures the total waste mass at the point of `wasteDrying.onEnter`, before the DES cycle resets the pending-mass accumulator. This ensures the brick yield calculation references the correct cycle input regardless of concurrent injections.

At `brickProduction.onExit`:
- Bottom ash fraction: `stagingAshCycleMass_t × 0.25 × brickYield_tPerTAsh`
- Revenue credited to `financialBalance` and discounted into `entityNPV`

At `flyAshToLandfill.onExit`:
- Fly ash fraction: `stagingAshCycleMass_t × 0.04 × 1000` (kg)
- LCA impact applied: `GWPActualIncineration += flyAshMass × 0.005` `[ILLUSTRATIVE]`

### New Variables in Intermediate

| Variable | Default | Source |
|---|---|---|
| `stagingAshCycleMass_t` | 0.0 | internal |
| `brickRevenues` | 0.0 | computed |
| `brickProduced_t` | 0.0 | computed |
| `brickYield_tPerTAsh` | read from Excel | Intermediate Investment B5 |
| `brickPrice_SGD_t` | read from Excel | Intermediate Investment B6 |
| `brickPlantCapex_SGD` | read from Excel | Intermediate Investment B7 |

### NPV: Brick Plant CAPEX

`brickPlantCapex_SGD` is included in `totalInvestment` at simulation time zero alongside storage CAPEX and recycling CAPEX:

```java
totalInvestment = intCapex + recyclingCapex_SGD + intermediate.brickPlantCapex_SGD + fleetCapex
```

---

## Phase 15 — ORGANIC_WASTE Extension, Composting DES, VeChai Statechart, CellStyle Fix

**Commit:** ec26623 · 2026-04-03 02:02 SGT  
**Files modified:** 03APRIL.alp, 03APRIL.xlsx, CE30_EcoinventSearchStrings.md, futurenotes.txt

This is the most architecturally significant phase of the session. It adds a sixth material type, a full composting discrete-event process chain, a new Receiver agent class, an informal-sector agent with quality-based price negotiation logic, and closes a technical debt item in the Excel reporting function.

### MaterialType Enum Extended

`ORGANIC_WASTE` was added as index 5 to the `MaterialType` option list. All 18 arrays previously declared as `new double[5]` were extended to `new double[6]`, and all loop bounds of the form `for (int m = 0; m < 5; m++)` were replaced with `for (int m = 0; m < MaterialType.values().length; m++)` to make future extensions loop-safe.

### Disposal Fractions for ORGANIC_WASTE

The noIS counterfactual scenario assigns a 50/50 split between incineration and landfill for organic MSW in the absence of an IS pathway. This is flagged with an `xxx` comment in the source code and registered in futurenotes.txt as item [17], pending confirmation against Binh Duong province waste statistics and thesis supervisor consensus.

```java
noIS_landfillShare[5]      = 0.50;  // xxx: 50/50 organic MSW split — futurenotes [17]
noIS_incinerationShare[5]  = 0.50;  // xxx: same
```

### Composting DES Chain

Four blocks were added to the Intermediate agent to model the composting process from organic waste reception to compost product output:

| Block | Distribution | Purpose |
|---|---|---|
| `compost_Start` | — | Injection source |
| `compost_Sorting` | tri(4, 6, 8) h | Incoming organic waste sorting and contamination removal |
| `compost_Maturation` | tri(168, 252, 336) h | Aerobic composting maturation (7–14 days) |
| `compost_Sink` | — | Process completion sink |

The conversion efficiency `conversionEffPerMat[5] = 0.20` reflects the 5:1 mass reduction ratio documented in the ESCAR composting study (pp. 34–39). At `compost_Maturation.onExit`, compost mass, revenue (`compostPrice_SGD_t = 80 SGD/t [ILLUSTRATIVE]`), and GWP impact are all credited to the Intermediate agent's accounting variables.

### Agricultural Receiver (6th Receiver Agent)

The `Receivers` EmbeddedObject replication count was incremented from 5 to 6. The 6th instance (`Receiver_Organic_Agri`) is initialised from `Companies_Listing` row 73 with material type `ORGANIC_WASTE`, positioned at coordinates representing an agricultural cooperative node in Binh Duong province (Lat 10.945, Lon 106.710, avg demand 8 t/week `[ILLUSTRATIVE]`).

### Receiver Class: Quality Threshold Variables

Two new parameters were added to the Receiver agent class to support the VeChai quality decision mechanism:

| Variable | Default | Meaning |
|---|---|---|
| `minAcceptableQuality` | 0.70 | Minimum material quality index (0–1) for full IS price |
| `lowerQualityPriceFactor` | 0.50 | Price multiplier applied when quality falls below threshold |

These defaults represent HMS1/HMS2 ferrous scrap quality requirements and are tagged `[ASSUMED]`.

### VeChai Agent — Quality Decision Statechart

A new active object class `VeChai` (Id 8888000000001) was created to represent the informal waste-picker/aggregator operating within the VSIP1 catchment area. The agent's core behaviour is a quality-discriminating price negotiation cycle, implemented as a five-state statechart.

**Statechart structure:**

```
Idle ──(7 DAY timeout)──► Collecting ──(scrapThisCycle_t > 0)──► Negotiating
                                                                       │
                              ┌────── quality ≥ threshold ────────────┤
                              ▼                                        │
                        SoldAtISPrice                       SoldAtInformal ◄── quality < threshold
                              │                                        │
                              └─────────────────(true)────────────────┘
                                                    ▼
                                                  Idle
```

At each weekly cycle, material quality is sampled from `Normal(qualityMean=0.60, qualityStdDev=0.15)` and clamped to [0, 1]. The threshold is read from the ferrous Receiver's `minAcceptableQuality` at runtime, allowing it to be varied experimentally.

- **SoldAtISPrice**: credits `ferrousReceiver.totalReceivedFromIS`, `Main.financialBalance`, and records the IS transaction.
- **SoldAtInformal**: credits only `veChaiRevenue` at `lowerQualityPriceFactor × priceSecondaryPerMat[FERROUS]`; the IS exchange is not credited, and the material enters the informal recycling chain rather than the tracked IS network.

### Item [6] — CellStyle POI Fix

The `exportValidationReport` function in Main was creating a new `XSSFCellStyle` object via `cloneStyleFrom()` for every truck row in the summary table. Apache POI enforces a hard limit of 64,000 cell styles per workbook. For large fleets or long simulation runs, this caused a `java.lang.IllegalStateException` at export time. The fix replaces per-row style creation with reuse of a pre-created style object:

```java
// Before (Phase 14 and earlier):
XSSFCellStyle tsNs = wb.createCellStyle(); tsNs.cloneStyleFrom(numStyle); tmc.setCellStyle(tsNs);

// After (Phase 15):
tmc.setCellStyle(numStyle);  // CE3.0 Phase 15: reuse pre-created style
```

### Excel Changes

| Sheet | Row | Content added |
|---|---|---|
| Materials | 8 | ORGANIC_WASTE: convEff=0.20, GWP/HTP/LUP/CED `[ILLUSTRATIVE]`, compostPrice=80 SGD/t |
| Companies_Listing | 73 | Receiver_Organic_Agri (ORGANIC_WASTE, lat 10.945, lon 106.710, 8 t/wk) |
| recycling_and processingfactors | 14–15 | ORGANIC_WASTE: convEff=0.20, procEff=0.90 |

### CE30_EcoinventSearchStrings.md — Section E

Section E was fully rewritten to document the two ecoinvent 3.9 process entries required for the ORGANIC_WASTE IS pathway:
- `treatment of biowaste, composting` — process-level GWP/HTP for compost production
- `market for compost, at plant` — product output for LCA burden allocation

The section includes ALP variable wiring, geographic preference (`RoW` or `VN` if available), and five recommended search strings for the ecoinvent database query interface.

---

## Summary Table — Changes by Phase

| Item | Phase | Type | Status |
|---|---|---|---|
| DES block descriptions (pwStart etc.) | 13 | Documentation | Done |
| noIS disposal array alignment PLASTIC/PAPER | 13 | Model logic | Done |
| Fleet CAPEX at t=0 in NPV | 13 | NPV accounting | Done |
| truckBaseWeight_t parameter | 13 | Parameter | Done (formula pending) |
| futurenotes.txt created ([1]–[15]) | 13 | Documentation | Done |
| Combustion DES → brick production chain | 14 | DES extension | Done |
| Brick plant CAPEX in totalInvestment | 14 | NPV accounting | Done |
| stagingAshCycleMass_t mass snapshot | 14 | Model logic | Done |
| ORGANIC_WASTE MaterialType(5) + array extension | 15 | Architecture | Done |
| Composting DES chain (Sort→Mature) | 15 | DES extension | Done |
| Agricultural Receiver (6th agent) | 15 | Agent | Done |
| Receiver: minAcceptableQuality + lowerQualityPriceFactor | 15 | Parameter | Done |
| VeChai agent class + quality statechart | 15 | Agent / statechart | Done |
| veChai EmbeddedObject in Main | 15 | Deployment | Done |
| POI 64k CellStyle fix (exportValidationReport) | 15 | Bug fix | Done |
| Excel Materials row 8 ORGANIC_WASTE | 15 | Data | Done |
| Excel Companies_Listing row 73 Receiver_Organic_Agri | 15 | Data | Done |
| futurenotes.txt item [17] (organic 50/50 split) | 15 | Documentation | Done |

---

## Open Items After This Session

The following gaps remain open and are tracked in futurenotes.txt:

- **[17]** Organic MSW 50/50 incineration/landfill split requires literature confirmation (Binh Duong DONRE data or Vietnam national MSW statistics).
- **Critical gap:** `truckBaseWeight_t` loaded but deadhead cost/emission formula not yet implemented.
- **Critical gap:** All LCA factors for ORGANIC_WASTE are `[ILLUSTRATIVE]` — ecoinvent 3.9 database lookup required (see CE30_EcoinventSearchStrings.md Section E).
- **Item [11]:** MSW provider as a full ABM agent (organic waste generator) not yet implemented.
- All other items in futurenotes.txt [1]–[16] remain deferred.

---

*Report generated: 20 April 2026. Model state: 03APRIL.alp / 20APRIL.alp, Phase 15, commit ec26623.*
