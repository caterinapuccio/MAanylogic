# CE3.0 Implementation Plan — BIWASE Integration
*Last updated: 2026-03-26*

---

## Answered Questions

### Transport GWP: does attribution matter?

**Short answer: track it carefully, but keep attribution simple (always charge to the generator).**

The emission factor for diesel road freight (lorry, 7–12t, SEA context) is approximately **0.10–0.16 kg CO₂-eq per tonne-km** (ecoinvent 3.9, transport, freight, lorry >3.5t, EURO 5; EcoTransIT World 2024). For a 15 km haul of one truck load (e.g. 5t plastic), that is 5 × 15 × 0.12 = **9 kg CO₂-eq** — roughly 0.002 kg CO₂-eq per kg of plastic transported. Compare to production GWP of ~1.8 kg CO₂-eq/kg for virgin HDPE: transport is about **0.1% of production GWP**. For metals the ratio is similar.

In absolute terms transport is small. But for the IS vs non-IS comparison, transport IS the difference being measured — and the two scenarios have different transport distances (provider→BIWASE vs provider→BIWASE is the same, so the distance effect is zero for the baseline; the IS benefit comes from recycling vs landfilling, not from shorter distance). Transport attribution is therefore not decisive for the thesis conclusion but must be tracked for completeness.

**ISO 14044 / ILCD Handbook rule:** Waste treatment impacts, including transport to the treatment facility, are attributed to the waste generator (polluter-pays principle). This means: whoever generates the waste bears the transport GWP burden, regardless of who drives the truck. The model already does this on the Provider side.

**Decision: keep attribution at Provider level. Both financial cost and GWP from transport are always credited to the provider that owns/books the truck. BIWASE tracks total incoming tonnage and burning operations separately for facility-level monitoring.**

Sources:
- EcoTransIT World (2024). *Methodology Report Update 2024*. https://www.ecotransit.org/wp-content/uploads/20240308_Methodology_Report_Update_2024.pdf
- ILCD Handbook (2010). *General Guide for LCA*. JRC/EC. https://eplca.jrc.ec.europa.eu/uploads/ILCD-Handbook-General-guide-for-LCA-DETAILED-GUIDANCE-12March2010-ISBN-fin-v1.0-EN.pdf
- ISO 14044:2006. Environmental management — Life cycle assessment — Requirements and guidelines.

---

### BIWASE incineration capacity (from ESCAR PDF, ADB Project 56118-001, Nov 2022)

BIWASE Waste Treatment Complex, South Binh Duong, Ben Cat town:

| Facility | Capacity |
|---|---|
| Hazardous waste incinerators (3 units) | 1,700 + 1,700 + 4,200 kg/h |
| **Non-hazardous industrial waste incinerators (2 units)** | **1,000 + 4,200 kg/h = 5,200 kg/h = 124.8 t/day** |
| Medical waste / animal carcass incinerators | 100 + 200 kg/h |
| WtE mixed municipal + industrial (1 unit) | 8,400 kg/h = 201.6 t/day |
| Composting (3 operational lines + 1 under construction) | 1,680 tpd total |
| Total site capacity | 6,216.8 tpd |
| Incoming trucks | ~240 trucks/day |

**Model burning threshold:** The two non-hazardous industrial waste incinerators (relevant to VSIP1 industrial scrap) together process 5,200 kg/h. BIWASE runs these continuously, but for simulation purposes a batch trigger is used. **Setting `burningThreshold = 20 t`** (≈ 4 hours of non-hazardous incinerator operation) is realistic and will fire a few times per simulation week given 66 providers. Read this from Excel so it can be changed.

Source: IBIS Environmental and Social Asia Consulting (2022). *Environmental & Social Due Diligence Report on BIWASE Waste Treatment Complex Expansion, Viet Nam*. ADB Project 56118-001. p. 14, 18.

---

### Gate fees — are they realistic? What other costs exist?

**Yes, gate fees are confirmed realistic and mandatory at BIWASE.** From the PDF (p. 13): *"Waste vehicles deliver waste to the site which is charged at either contracted rates or on a gate fee basis. Vehicles carrying waste are weighed at the Site entrance and again on exit via the weighbridge."*

Provider costs at BIWASE gate:
1. **Gate/weighbridge fee** — fixed per truck visit; covers site entry, weighing, administration. Typical Vietnam range: 50,000–150,000 VND/visit (~2–6 SGD). Implement as `gateFeePerTruck_SGD` (scalar, from Excel).
2. **Disposal processing fee** — per tonne charged by BIWASE for actual treatment. Already in model as `costLandfillPerMat[matIdx]` (per-tonne, per material). Keep this.
3. **Environmental protection tax** — Vietnam Law 57/2010/QH12 imposes this on certain waste categories. For ordinary industrial solid waste (non-hazardous): currently 0 VND/kg; for hazardous: significant. For the 5 materials in this model (all non-hazardous), this tax is currently zero — note this in thesis but no model change needed.

**So two distinct cost lines for the provider per truck delivery:**
- `gateFeePerTruck_SGD` (fixed, per trip) → charged once when truck enters BIWASE
- `costLandfillPerMat[matIdx] × cargoQty` (per tonne) → charged by weight at weighbridge

Both go to: `provider.financialBalance -= cost`, `provider.totalCosts += cost`.
BIWASE (Intermediate) records as revenue: `intermediate.totalRevenues += (gateFeeSGD + dispCost)`.

The existing `costLandfill` in the Baseline sheet covers the per-tonne disposal fee. **Rename it `costLandfillPerMat` (already done) and add `gateFeePerTruck_SGD` as a new row in the Baseline sheet.**

---

### Jolly trucks — complexity assessment

**Moderate complexity. Definitely implementable. Recommended.**

The jolly (shared, unassigned) truck pool is a natural extension of the existing GarbageTruck class. Implementation additions:

- Same `GarbageTruck` agent class. Add boolean `isJolly = false`. Jolly trucks have `homeProvider = null` and start at BIWASE.
- Separate count `jollyTruckCount` in TruckFleet Excel sheet.
- Provider booking logic: first check own fleet → if none free, check jolly trucks → if none free, stay in `CallingTransport` (wait).
- Extra cost: `jollyPremiumFactor` (e.g. 1.3×) applied to `truckFixedCostPerTrip_SGD` when booking a jolly.
- Add to TruckFleet sheet: `jollyTruckCount` (integer) and `jollyPremiumFactor` (decimal, 1.0–2.0).

**On the ABM behavior question:** Yes, this is exactly the scenario for which ABM is uniquely suited. With 66 providers competing for a shared jolly pool, resource contention and booking priority can produce emergent dynamics — e.g., providers with higher waste generation rates monopolize jolly trucks, causing delays for smaller generators. This is a genuinely novel contribution of the ABM approach vs. aggregate models and worth highlighting in the thesis Methods section.

**"Emergency landfill" = on-site waste accumulation.** If all trucks (own + jolly trucks) are unavailable, the provider stays in `CallingTransport`. Waste accumulates beyond the truck threshold in `wasteStoredPerMat`. No separate emergency landfill agent needed — the provider just holds the waste. Add a `wasteStorageOverflow` boolean flag (true when `wasteStoredPerMat[matIdx] > 3 × truckCapacity`) for monitoring and visualization. In reality this reflects illegal on-site dumping or open burning, which is mentioned in the BIWASE PDF as a current practice in Binh Duong.

---

### Parallel vs sequential implementation

**Sequential only for .alp changes. Parallel only for Excel.**

AnyLogic .alp XML is a single monolithic file with internal ID cross-references. There is no merge/diff workflow. Two independently-modified copies cannot be reliably integrated. Any attempt to run phases in parallel in separate model copies will require manual XML reconstruction to merge, which is error-prone and time-consuming.

**Exception:** Excel changes (Phase 0) are completely independent of .alp and can be done in any session before the relevant .alp phases begin.

**Recommended session structure:**
1. Session A: All Phase 0 Excel work (independent)
2. Sessions B–N: Sequential .alp phases (each session verifies the previous before starting the next)

---

## Updated Implementation Plan

### Phase 0 — Excel restructuring (no .alp changes)

**0.1** Rename sheet `Companies_Location` → `Companies_Listing`.

**0.2** Add column I = `TruckCount` (integer) in `Companies_Listing`. Default value: 5 for all provider rows (rows 2–67). Leave blank for Receiver rows (68–72), Intermediate (73), EndOfLife (74–75). Mark orange.

**0.3** In `TruckFleet` sheet: remove `TruckCount` column (it is now per-agent in Companies_Listing). Keep and add:

| Column | Name | Value | Mark orange? |
|---|---|---|---|
| A | TruckVelocity_kmh | 60 | ✓ |
| B | LoadingTime_h | 2 | ✓ |
| C | UnloadingTime_h | 2 | ✓ |
| D | TruckCapacityPerMat_PLASTIC_t | 4.0 | ✓ |
| E | TruckCapacityPerMat_PAPER_t | 1.5 | ✓ |
| F | TruckCapacityPerMat_FERROUS_t | 9.0 | ✓ |
| G | TruckCapacityPerMat_TEXTILE_t | 2.0 | ✓ |
| H | TruckCapacityPerMat_NONFEROUS_t | 8.0 | ✓ |
| I | JollyTruckCount | 10 | ✓ |
| J | JollyPremiumFactor | 1.3 | ✓ |
| K | Source_capacity | WRAP (2022); EPA (2016) | — |

**0.4** In `Baseline` sheet: confirm `costLandfillPerMat` rows exist (per material, per tonne). Add new row: `gateFeePerTruck_SGD` (scalar, suggested default 4 SGD). Mark orange.

**0.5** Add `burningThreshold_t` row to `Intermediate` sheet (suggested default 20 t). Mark orange.

**0.6** Add `truckCapexSGD` row to `TruckFleet` sheet (capital cost per truck for LCC). Suggested default 30,000 SGD for a 7-tonne Vietnamese industrial truck. Mark orange. Add source column.

---

### Phase 1 — Geographic unification (1-line .alp change)

**1.1** In Main startup, after reading `intermediateLat/intermediateLon`:
```java
endOfLifeLat = intermediateLat;
endOfLifeLon = intermediateLon;
```
**1.2** Run model. Verify no crash. KPIs shift slightly (distance to "landfill" now equals distance to BIWASE).

---

### Phase 2 — Provider: read TruckCount + truck CAPEX, initialize fleet size

**2.1** Add `int providerTruckCount = 5` variable to Provider (will be read from Excel).

**2.2** In Main startup Provider loop: read column I of Companies_Listing into `p.providerTruckCount`.

**2.3** Add `double truckCapex = 0` and compute `provider.totalCosts += providerTruckCount × main.truckCapexSGD` during startup (one-time CAPEX). Also charge to `provider.financialBalance`.

**2.4** Add `double[] truckCapacityPerMat = new double[5]` to Main. Read from TruckFleet sheet columns D–H.

**2.5** Add `double truckVelocity_ms` to Main. Read from TruckFleet col A, convert km/h → m/s.

**2.6** Run model. Verify no crash. Check CAPEX in console output.

---

### Phase 3 — Restructure GarbageTruck population (home = provider, not BIWASE)

**3.1** Add `Provider homeProvider = null` and `boolean isJolly = false` variables to GarbageTruck.

**3.2** Add `double homeLat, homeLon` to GarbageTruck (home position for this truck).

**3.3** In Main startup, after initializing all providers, replace the existing single GarbageTruck replication block with two loops:
  - Loop A: for each provider `p`, create `p.providerTruckCount` trucks; set `truck.homeProvider = p`, `truck.homeLat = p.providerLat`, `truck.homeLon = p.providerLon`, `truck.isJolly = false`. JumpTo homeLat/homeLon.
  - Loop B: create `jollyTruckCount` trucks; set `truck.isJolly = true`, `truck.homeLat = intermediateLat`, `truck.homeLon = intermediateLon`. JumpTo BIWASE.

**3.4** In GarbageTruck `home` state entry action: change `jumpTo(main.intermediateLat, main.intermediateLon)` → `jumpTo(homeLat, homeLon)`.

**3.5** Fix GarbageTruck `VelocityCode`: `10 MPS` → `main.truckVelocity_ms` (read from Excel).

**3.6** Run model. Confirm all trucks positioned at their home locations on GIS. Confirm jolly trucks at BIWASE.

---

### Phase 4 — Provider: waste accumulation variable

**4.1** Add `double[] wasteStoredPerMat = new double[5]` to Provider.

**4.2** In `GeneratingWaste` entry action (existing statechart), after existing logic: `wasteStoredPerMat[matIdx] += rawWaste;`

**4.3** Add `boolean wasteStorageOverflow = false` to Provider. Set true when `wasteStoredPerMat[matIdx] > 3 × main.truckCapacityPerMat[matIdx]`.

**4.4** Run model a few weeks. Verify accumulation via traceln. Existing behavior (NonIS_Disposal etc.) unchanged.

---

### Phase 5 — Provider: WasteAccumulator statechart (second statechart)

**5.1** Add second statechart to Provider with single `Idle_WA` state. Entry: traceln confirming start.

**5.2** Add `CallingTransport` state. Entry action: traceln only.

**5.3** Add transition `Idle_WA → CallingTransport`:
```
Condition: wasteStoredPerMat[main.matIndex(myMaterial)] >= main.truckCapacityPerMat[main.matIndex(myMaterial)]
```

**5.4** Add transition `CallingTransport → Idle_WA` (immediate for now, loopback). Run model. Verify threshold fires.

---

### Phase 6 — Provider: truck booking logic in CallingTransport

**6.1** In `CallingTransport` entry action:
```java
// Find available truck: own fleet first, then jolly trucks
GarbageTruck t = null;
for (GarbageTruck truck : main.garbageTrucks) {
    if (!truck.busy && truck.homeProvider == this) { t = truck; break; }
}
if (t == null) { // try jolly
    for (GarbageTruck truck : main.garbageTrucks) {
        if (!truck.busy && truck.isJolly) { t = truck; break; }
    }
}
if (t != null) {
    int matIdx = main.matIndex(myMaterial);
    double loadQty = Math.min(main.truckCapacityPerMat[matIdx], wasteStoredPerMat[matIdx]);
    t.cargoMaterial = myMaterial;
    t.cargoQty = loadQty;
    t.sourceProvider = this;
    t.targetLat = providerLat;
    t.targetLon = providerLon;
    t.busy = true;
    wasteStoredPerMat[matIdx] -= loadQty;
    // Cost: transport + gate fee (gate fee paid on arrival, not here)
    double tripCost = t.isJolly
        ? main.truckFixedCostPerTrip_SGD * main.jollyPremiumFactor
        : main.truckFixedCostPerTrip_SGD;
    this.financialBalance -= tripCost;
    this.totalCosts += tripCost;
    t.send("Go"); // trigger truck statechart
    // transition back to Idle_WA
} else {
    // No truck available — stay in CallingTransport, retry on next weekly cycle
    // wasteStorageOverflow check
    if (wasteStoredPerMat[main.matIndex(myMaterial)] > 3 * main.truckCapacityPerMat[main.matIndex(myMaterial)])
        wasteStorageOverflow = true;
}
```

**6.2** Add `double truckFixedCostPerTrip_SGD` and `double jollyPremiumFactor` to Main. Read from TruckFleet sheet.

**6.3** Add `MaterialType cargoMaterial` and `double cargoQty` to GarbageTruck.

**6.4** Modify `CallingTransport → Idle_WA` transition: only fire when a truck was found (`bookingAccepted = true` flag). If no truck found, stay in `CallingTransport` until next weekly cycle.

**6.5** Run model. Verify providers booking trucks. Watch console for booking events.

---

### Phase 7 — GarbageTruck: Loading + Unloading states

**7.1** Restructure GarbageTruck statechart:
```
home → (on "Go") → Booked → Loading (2h timeout) → transit → Unloading (2h timeout) → home
```

**7.2** `Booked` entry: `lorry.setVisible(true);`

**7.3** `Loading` entry: `jumpTo(targetLat, targetLon);` — truck appears at provider site.

**7.4** `transit` entry: `moveTo(main.intermediateLat, main.intermediateLon);` — truck drives to BIWASE. (Remove old jumpTo from transit — it now happens in Loading.)

**7.5** `Unloading` entry: full arrival accounting (Phase 8 below).

**7.6** `home` entry: `busy = false; lorry.setVisible(false); jumpTo(homeLat, homeLon);` Notify provider if needed.

**7.7** Run model. Confirm GIS animation: truck appears at provider → drives to BIWASE → disappears.

---

### Phase 8 — Arrival accounting: GWP credit + BIWASE accumulation

**8.1** Add `double[] materialArrivedAtBIWASE = new double[5]` to Intermediate.

**8.2** In GarbageTruck `Unloading` entry action:
```java
Provider p = (Provider) sourceProvider;
int matIdx = main.matIndex(cargoMaterial);
// Transport GWP — credited to provider
double dist = main.haversineKm(p.providerLat, p.providerLon, main.intermediateLat, main.intermediateLon);
double transportGWP = main.transportGWPCalc(cargoQty, dist);
p.GWPActualTransport += transportGWP;
p.GWPActualTotal     += transportGWP;
main.totalTransportGWP += transportGWP;
main.totalTransportCost += /* already charged at booking */;
main.networkTotalTransportCostNonISProcesses += /* cost booked in Phase 6 */;
// Gate fee — paid to BIWASE at arrival
double gateFee = main.gateFeePerTruck_SGD;
p.financialBalance -= gateFee;
p.totalCosts       += gateFee;
main.intermediate.totalRevenues  += gateFee;
main.intermediate.financialBalance += gateFee;
// Disposal cost (per tonne)
double dispCost = cargoQty * main.costLandfillPerMat[matIdx];
p.financialBalance -= dispCost;
p.totalCosts       += dispCost;
main.intermediate.totalRevenues  += dispCost;
main.intermediate.financialBalance += dispCost;
// Waste arrives at BIWASE
main.intermediate.materialArrivedAtBIWASE[matIdx] += cargoQty;
main.landfill.totalWasteReceived += cargoQty;  // keep existing KPI
```

**8.3** Run model. Verify `materialArrivedAtBIWASE` grows. Verify provider is charged gate fee + disposal cost.

---

### Phase 9 — BIWASE burning statechart + LCA

**9.1** Add `double burningThreshold_t` to Intermediate. Read from Excel Intermediate sheet.

**9.2** Add `double burningGWPTotal = 0` and `double burningCostTotal = 0` to Intermediate.

**9.3** Add burning statechart to Intermediate: `Idle_Burn → Burning (10h timeout) → Idle_Burn`.

**9.4** Transition `Idle_Burn → Burning` condition:
```java
java.util.Arrays.stream(materialArrivedAtBIWASE).sum() >= burningThreshold_t
```

**9.5** `Burning` entry action:
```java
double totalGWP = 0;
double totalCost = 0;
for (int m = 0; m < 5; m++) {
    double kgBurned = materialArrivedAtBIWASE[m] * 1000.0;
    double gwp = kgBurned * main.GWPWasteToLandfill[m];
    totalGWP  += gwp;
    totalCost += materialArrivedAtBIWASE[m] * main.costLandfillPerMat[m]; // operating cost to BIWASE
    // Attribute GWP back to providers (proportional split — Phase 10 for precise tracking)
    // For now: accumulate at BIWASE facility level only
    burningGWPTotal  += gwp;
    burningCostTotal += materialArrivedAtBIWASE[m] * main.costLandfillPerMat[m];
}
// Charge burning operating cost to BIWASE
this.totalCosts += totalCost;
this.financialBalance -= totalCost;
// Update network-level KPI
main.landfillCostPerMat[m] already updated at arrival (Phase 8)
// Reset accumulator
java.util.Arrays.fill(materialArrivedAtBIWASE, 0.0);
```

**9.6** Run model end-to-end (at least 4 simulation weeks). Confirm burning fires, GWP computed.

---

### Phase 10 — Disconnect old NonIS_Disposal path

Only after Phases 1–9 verified stable.

**10.1** In Provider existing statechart, disable `NonIS_Disposal` entry action (replace with comment stub).

**10.2** Disable `IS_Rejected → landfill` entry action (replace with comment stub — truck-based path now handles this).

**10.3** Run full model useIS=false. Confirm all waste now flows through truck → BIWASE → burning.

**10.4** Run full model useIS=true. Confirm IS path still works via acceptBooking/fulfillDemand.

---

### Phase 11 — IS pathway at BIWASE (separate planning session)

Detailed planning after Phases 1–10 are stable. Topics:
- IS pricing negotiation between Provider and BIWASE
- Material quality check at BIWASE gate
- High-value material: BIWASE sends own trucks to pick up (uses BIWASE truck fleet)
- Low-value material: Provider sends own trucks (same as baseline)
- Integration with existing tier1/tier2 storage and fulfillDemand logic
- Receiver integration

---

## What to do NOW (next session)

**Start with Phase 0 (Excel only). Open 22MARZ.xlsx and:**
1. Rename sheet `Companies_Location` → `Companies_Listing`
2. Add column I = `TruckCount` with value 5 for provider rows 2–67, blank for rows 68+
3. In `TruckFleet` sheet: restructure as described in Phase 0.3 table above (remove old TruckCount, add velocity, capacity per material, jolly count, premium, capex)
4. In `Baseline` sheet: confirm `costLandfillPerMat` rows; add `gateFeePerTruck_SGD = 4`
5. In `Intermediate` sheet: add `burningThreshold_t = 20`

Excel changes are zero-risk — the model will not crash if new columns are added that are not yet read by the .alp. They just get read in Phase 2.

**Do NOT touch the .alp until the Excel is done and confirmed correct.**

---

## Variables reference (all new, all from Excel)

| Variable | Location | Sheet | Default | Notes |
|---|---|---|---|---|
| `providerTruckCount` | Provider | Companies_Listing col I | 5 | Per agent |
| `truckCapacityPerMat[5]` | Main | TruckFleet cols D–H | see table | Per material |
| `truckVelocity_ms` | Main | TruckFleet col A | 16.67 m/s | 60 km/h |
| `loadingTime_h` | GarbageTruck | TruckFleet col B | 2 | Fixed |
| `unloadingTime_h` | GarbageTruck | TruckFleet col C | 2 | Fixed |
| `jollyTruckCount` | Main | TruckFleet col I | 10 | Shared pool |
| `jollyPremiumFactor` | Main | TruckFleet col J | 1.3 | 30% premium |
| `truckCapexSGD` | Main | TruckFleet col K | 30,000 | Per truck |
| `truckFixedCostPerTrip_SGD` | Main | TruckFleet (new row) | 45 | ~15km @ 3 SGD/km |
| `gateFeePerTruck_SGD` | Main | Baseline sheet | 4 | Fixed per visit |
| `burningThreshold_t` | Intermediate | Intermediate sheet | 20 | Total mass trigger |
| `wasteStoredPerMat[5]` | Provider | — | 0 | Accumulator |
| `materialArrivedAtBIWASE[5]` | Intermediate | — | 0 | Accumulator |
| `isJolly` | GarbageTruck | — | false | Jolly flag |
| `homeLat / homeLon` | GarbageTruck | — | from owner | Home position |
