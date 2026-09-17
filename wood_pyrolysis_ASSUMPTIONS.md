# Wood pyrolysis — assumptions behind the demonstration session

**File reviewed:** `SFF/wood pyrolysis.txt` — *Wood Pyrolysis to Bio Oil & Biochar*, SuperPro
Designer export, SFF v0.0.2, TEA year 2021.
**Plant:** 30 MT/h wet woody biomass (≈45% moisture), 8,400 operating h/yr.
**Session file:** `wood_pyrolysis_session_<date>.json` — open it with **Load session**; the SFF
does not need to be uploaded separately.

> **This is a workflow demonstration, not a validated TEA.** Several numbers below are
> assumptions made to get the review to a complete state with no blanks. Every one is listed
> here, and every one is visible and editable in the tool. Replace the flagged ones before
> quoting any result.

---

## 1. What the file gave us, and what it did not

| The file states | The file is silent on |
|---|---|
| 22 units, each with a purchase **and** an installed cost | any equipment sizes beyond 6 volumes/diameters |
| 39 streams with flows, temperatures and compositions | prices for everything except the feedstock |
| feedstock price, $0.055/kg | every utility — `heat_utilities`, `power_utilities` and `other_utilities` are all empty |
| its own NPV, IRR, ROI and payback, for cross-checking | any duty per unit |

The unit **types** are SuperPro's generic labels — `Generic`, `Reaction`, `Gas Flow` — so the
matcher's fuzzy scorer finds nothing for any of them. Every unit below was picked out of the
library by hand, through **Search the full library**. That is the honest outcome for this file
and is worth seeing: the fuzzy matcher earns its keep on descriptive unit names, not on these.

---

## 2. Reading the flowsheet

The SFF carries no unit names, so what each unit *is* was read off the topology:

| Unit | Read as | Evidence |
|---|---|---|
| U-1 | Shredder | takes the 30,000 kg/h wet-wood feed |
| U-2 | Hammer mill (grinding after drying) | sits between the dryer and the reactor |
| U-3 | Fluidised-bed pyrolysis reactor | takes dried biomass + hot gas + recycle gas; $6.17M, by far the largest item |
| U-4 | Aqueous bio-oil storage tank | 153.5 m³, on the aqueous product |
| U-5, U-6, U-12, U-15 | Flow splitters | one inlet, two-plus outlets, flows sum exactly; **$0 in the source model** |
| U-7 | Rotary biomass dryer, 3 units | 201 m³, on the wet feed |
| U-8, U-14 | Mixers | two inlets, one outlet; **$0 in the source model** |
| U-9 | Off-gas burner | 78.4 m³, 2.15 m dia., burns the non-condensable gas |
| U-10 | Cyclone | takes the 3,300 kg/h char out of the reactor effluent |
| U-11, U-13, U-19, U-21 | Condensers / gas coolers | each paired with a cooling-water stream |
| U-16, U-17 | Fans (recycle gas, combustion air) | `Gas Flow` type, gas in = gas out |
| U-18, U-20 | Knock-out / flash vessels | split a gas and a liquid out of one inlet |
| U-22 | Bio-oil storage tank | 129.2 m³, on the main product |

The six splitter/mixer nodes are **skipped** in the matcher, with the reason recorded: they carry
no cost in the source model and are not equipment.

---

## 3. Prices — the boundary streams

| Stream | kg/h | Category | Price | Basis |
|---|---|---|---|---|
| Woody Biomass | 30,000 | Raw Material | **$0.055/kg** | **the file's own figure** — the only price it states |
| Bio Oil | 5,688 | Product Sale | **$0.50/kg** | ⚠ **ASSUMPTION, but backed out of the file itself.** The file states `annual_revenue_usd = 30,720,949`. Take biochar at $0.25/kg ($6.93M/yr) and the remaining $23.79M over 47.78 Mkg/yr of bio-oil is **$0.498/kg**, rounded to $0.50. That is inside the $0.30–0.60/kg band used for raw fast-pyrolysis bio-oil in published work (the low end is energy parity with heavy fuel oil — bio-oil ≈17 MJ/kg against ≈40 MJ/kg). **Not a quote. Replace it.** |
| Bio Char | 3,300 | Product Sale | **$0.25/kg** | ⚠ **ASSUMPTION.** $250/MT, a solid-fuel-grade figure. Soil-amendment biochar fetches several times this; power-station co-firing char rather less. |
| Bio Oil Aqueous | 5,878 | Product Sale | **$0.00/kg** | 80% water. Assumed to leave at no value and at no cost, following the file's own `annual_waste_treatment_usd = 0`. In practice it is burned on site or treated. |
| Exhaust Gas | 11,433 | Waste Treatment | $0.00 | stack gas, no charge |
| Dryer Exhaust | 34,242 | Waste Treatment | $0.00 | stack gas, no charge |
| Hot Water | 363,500 | Waste Treatment | $0.00 | the cooling-water return; charged once, as a utility |
| Water (in) | 363,500 | Raw Material | $0.00 | the same cooling loop entering; **priced as a utility, not here, to avoid double-counting** |
| Air In Burner | 30,542 | Raw Material | $0.00 | ambient air |

## 4. Utilities — all four utility blocks in the file are empty, so these are hand-entered

| Utility | Amount | Basis |
|---|---|---|
| Electricity | **2,800 kW** | ⚠ **ASSUMPTION**, built from the flowsheet: recycle fan U-16 ≈1,400 kW (326,700 m³/h at ~10 kPa, 65% efficient), shredder ≈600 kW (20 kWh/MT × 30 MT/h), mill ≈540 kW (30 kWh/MT × 17.9 MT/h), combustion-air fan U-17 ≈220 kW, cooling-water pumps ≈45 kW. Priced from `utility_costs.json` (EIA industrial). |
| Cooling water | **363,500 kg/h** | the circulating flow the flowsheet itself states (25 → 80 °C) |
| Purchased fuel | **none** | the burner runs on the process off-gas (S-115), so no fuel is bought. This is a real feature of the flowsheet, not an omission. |

## 5. CapEx treatment

- Where the library has a correlation the unit genuinely fits, the unit is **matched** and costed
  from the correlation; the SFF-reported cost is logged beside it and the delta shown.
- Where it does not — the pyrolysis reactor (the library has **no** Reactors entries at all), the
  rotary dryer, the burner, and the two very small knock-out vessels — the unit is **bootstrapped**
  from the SFF-reported *installed* cost with a six-tenths-rule exponent, which is what that path
  exists for. Those rows carry `bm = 1`, `inst = 1` so the already-installed figure is not
  factored a second time, and they are flagged as provisional single-source entries. One
  consequence worth knowing: for those five rows the "purchased cost" column reads the same as
  the installed cost, because the only figure available to fit against was an installed one.
- **A cooling tower was added that the source model does not cost.** The flowsheet circulates
  363.5 m³/h of cooling water but SuperPro charges nothing for the cooling system. One
  field-erected cooling tower with pumps (101 L/s) was added from the library, flagged as a
  reviewer addition.

  > ⚠ **This session double-counts that cooling system, and Plant Economics now says so.**
  > The cooling water on the OpEx tab is priced by the Ulrich & Vasudevan correlation,
  > `C = a·CEPCI + b·C_fuel`, whose `a·CEPCI` term *is* the cooling plant's capital recovery and
  > O&M — **95.7%** of the $0.332/m³ price. So of the **$1.01M/yr** of cooling water,
  > about **$970k/yr** is already paying for a cooling system, and the **$478k** tower in
  > Σ C_BM pays for it again through FCI and the 0.18·FCI fixed-cost term.
  >
  > It was left in deliberately, because it is the clearest possible demonstration of the flag
  > added in Round 22. **To correct it, do one of two things:** delete the cooling-tower row in
  > the CapEx Estimator (the correlated price already covers it), or keep the tower and override
  > the cooling-water price to the marginal `b·C_fuel` term, **$0.0143/m³**, plus make-up water
  > and treatment chemicals. The first is the smaller edit and is what the source model assumed.
- CEPCI: the file states `TEA_year = 2021`, so SFF-derived costs are escalated from the 2021
  index to the current one. Escalation beyond five years is outside the recommended window and
  the tool says so.

## 6. Economics

| Setting | Value | Basis |
|---|---|---|
| Operating hours | 8,400 h/yr | the file's own process description |
| Convention | Turton (all three are computed and the spread reported) | the labour model and the COM closed form are Turton's |
| Operators | **23, entered by hand** | ⚠ **an override, and the most important one here.** Turton's correlation is fitted for **P ≤ 2** and this flowsheet's census gives **P = 3** (shredder, mill, cyclone). At P = 3 the 31.7·P² term dominates: N_OL = √(6.29 + 31.7·9 + 0.23·6) = 17.1 operators/shift, ×4.5 = **78 people** for a 30 t/h plant, and $5.85M/yr of labour. The file states its own labour bill — `annual_labor_cost_usd = 1,738,800` — so the headcount is pinned to that instead: 1,738,800 ÷ $75,000/operator-year = 23. The tool shows the correlation's own answer beside the override. |
| Tax | 21% | US federal corporate rate |
| Discount rate | 10% | ⚠ assumption, a common TEA hurdle |
| Plant life | 20 operating years | ⚠ assumption |
| Construction | 2 years | ⚠ assumption |
| Depreciation | MACRS 7-year | standard for chemical plant |
| MSP solved for | Bio Oil | the main product by revenue |

## 7. What came out

| | This review | The file's own figure | |
|---|---|---|---|
| Σ C_BM (17 line items) | **$16.11M** | $11.53M installed, 22 units | we add a cooling tower and cost 2 parallel trains where a correlation was out of range |
| FCI (Turton C_GR) | **$27.06M** | $52.29M direct fixed capital | SuperPro's DFC sits on a different factor chain; the spread across the three conventions is on screen |
| TCI | **$31.12M** | — | |
| Cost of manufacture | **$30.37M/yr** | $30.58M/yr annual operating cost | **within 0.7%** — the closest agreement anywhere in this review, and it only appears once labour is pinned to the file's own figure. Note it contains the cooling-system double count described in §5: correcting it moves the COM by roughly $0.2–1.0M/yr depending on which of the two fixes you take |
| Revenue | **$30.82M/yr** | $30.72M/yr | by construction — the bio-oil price was backed out of it |
| NPV @ 10%, 20 yr | **−$23.8M** | +$1.19M | |
| IRR | **−6.2%** | 3.36% after tax | |
| **Minimum selling price, bio oil** | **$0.585/kg** | — | **this is the number to look at.** At the $0.50/kg the file's own revenue implies, the plant does not clear a 10% hurdle; it needs $0.585/kg. |

The NPV and IRR disagree with the file's because the capital bases disagree ($27.1M against $52.3M) and
because a 10% hurdle over 20 years is this review's assumption, not the file's. The COM agreement is
the part worth trusting: two independent builds of the operating cost land within $200k of each other.

## 8. LCA

- Functional unit: **per kg of product**.
- ⚠ **Nine of the ten impact categories are empty in the shipped factor library**, so this is a
  carbon footprint and nothing else. The tab reports the other nine as "no factors" rather than
  as zero, which is the point — but do not read the absence of an acidification number as an
  absence of acidification.
- Biogenic CO₂: **reported separately** (GWP 0 in the total), which is the usual convention for
  a biomass feedstock. Switch it to *included* on the tab to see the fossil-equivalent figure.
- Fates: both stack gases **released to air**; the cooling-water return **not an emission**; all
  three product streams **not an emission**; the wood feed and combustion air **not an emission**.
- The impact library currently populates **GWP only** — the other nine categories report
  "no factors" rather than zero, which is the point.

**Result: 8,224 t CO₂e/yr, or 0.0659 kg CO₂e per kg of product** — and **all of it is the
purchased electricity**, 23.52 GWh/yr × 0.34967 kg CO₂e/kWh, to the kilogram. The **cooling water
contributes nothing**: `impact_factors.json` has an entry for it, so the LCA tab recognises the
name, but that entry carries no populated factor, so its 3.05 Mm³/yr is counted as uncharacterised
rather than as zero. (The Appendix's **Impact Factors** panel, added in Round 21, shows this at a
glance — 21 of the 31 entries in that file have no factor at all.) Direct stack emissions
contribute nothing either, for two further reasons, and it matters which is which:

1. **Every direct CO₂ here is biogenic** — the carbon comes from the wood. Under the
   report-separately convention it is excluded from the GWP total by design, so its absence from
   the total is *correct*. Switch `Biogenic CO₂` to *included* on the tab to see it counted as
   fossil.
2. **But the toolkit could not characterise it either way.** ⚠ **This is a real gap, found by
   this review.** Both stack gases give their composition as **mole fractions only**
   (`mol_fraction`, no `mass_fraction` and no `component_mass_flows_kg_h`), so
   `computeComposition()` in `streams_matcher.html` returns components with `kg_per_h = null`
   and the LCA tab has nothing to multiply a factor by. Their 384 Mkg/yr lands in
   **uncounted mass** on the Coverage panel rather than in the total — flagged, not silently
   zeroed — but the two streams are not named in the *uncharacterised* list the way an unmatched
   substance would be. See §9.

## 9. Two things this review found in the toolkit

**a. Mole-fraction compositions are not converted to mass.** Described above. The file's own
`chemicals` block carries molecular weights for Nitrogen, Oxygen, Water and others, so the
conversion is available for most components — but **`Carb. Dioxide` is `mw_source: "unresolved"`**,
which is exactly the component that carries the GWP. A fix would be: convert mol% → mass% in
`computeComposition()` when every component in the stream has a molecular weight, and leave the
stream uncharacterised (as now) when any is missing. Worth a round of its own; not done here.

**b. The MSP product pick did not survive a session save — fixed.** This was already the top
open item in `TODO.md` ("**Do this first**"), and building this session reproduced it on real
data. `productPick` is a plain variable; restoring the `<select>`'s value did not set it, so the
first `opex-update` after a load found no pick to keep and fell back to `productCandidates[0]`.
That broadcast arrives *before* the OpEx tab has finished restoring its prices, so every
candidate still read zero revenue and the ranking was by nothing in particular — this session
came back with the **3.05 billion kg/yr cooling-water return** as the MSP denominator and a
minimum selling price of **$0.0013/kg** against $0.585/kg. One line in
`plant_economics.html`'s restore handler fixes it, and `test_session.js` now pins it (four new
checks, including one that picks a stream which is *not* top of the ranking so a fallback cannot
pass). Without this the session file attached here would have opened wrong.

## 10. The cross-check to look at first

Plant Economics' **SFF-reported totals** panel compares this bottom-up build against the numbers
SuperPro itself put in the file (NPV $1.19M, IRR 3.36% after tax, ROI 8.98%, payback 11.1 yr,
direct fixed capital $52.3M, annual operating cost $30.58M, annual revenue $30.72M). They will
not agree closely — the revenue figure alone depends entirely on the bio-oil price assumed in §3,
which the file does not state. **Where they disagree, the disagreement is the finding**, not an
error to tune away.
