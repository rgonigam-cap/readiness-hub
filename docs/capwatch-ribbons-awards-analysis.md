# CAPWATCH Ribbons & Awards — Reporting Feasibility Analysis

**Scope:** Identify CAP ribbons/awards (per **CAPR 39‑3, *Award of CAP Medals, Ribbons, and Certificates***)
for which a Readiness Hub report could be **built**, or an existing report **improved** by adding
**ribbon‑entitlement** detection, using the **CAPWATCH data already synced by the app**.

**Data reviewed:** CAPWATCH export for ORGID 513 (GLR‑IL‑282, Lake in the Hills Composite Squadron),
the folder the app's `SOURCE_FOLDER_ID` points at — 63 `*.txt` tables plus `CAPWATCH_Table_Structure.pdf`.
Table layouts below are taken from the **actual data file headers** (authoritative), since the schema PDF
ships with a non‑standard stream compression that defeats both text extraction and page rendering.

---

## 0. Status — IMPLEMENTED (2026‑06)

Six ribbon/award reports were built/improved per the agreed plan. **Design decisions** (which
supersede the "entitlement vs. awarded" recommendation that appears later in this analysis):

- **No "entitlement vs. awarded" framing.** eServices has no ribbon tracking — everything is presumed.
  Reports track **participation/attendance** (for derived ribbons) or **the recorded award itself**
  (for directly‑recorded awards). One report per ribbon/award.
- **Cadet achievement awards/ribbons are out of scope** (Wright Brothers → Spaatz) — handled elsewhere.
- **Cadet Special Activities Ribbon uses a denylist, not an allowlist.** Per CAPR 39‑3 the qualifying
  national‑activity list is non‑exhaustive and grows over time, so every `CadetActivities.Type` counts
  **except** an explicit exclusion set (`ENCAMP`, `RCLS`, `DDRX`, `RSTInPer`). Any new/unseen activity
  code qualifies automatically; malformed/blank types are guarded so the report never throws.

| # | Report (id) | New/Improve | Audience | Data source | Notes |
|---|---|---|---|---|---|
| 1 | Senior PD Level Awards (`senior-pd-awards`) | New | Senior | `SeniorAwards.txt` MBRRBN/DAVIS/LOENING/GARBER/WILSON | Full roster; one column per E&T level (I–V) + highest. Membership Ribbon = Level I. |
| 2 | Yeager AE Award (`yeager-award`) | New | Senior | `SeniorAwards.txt` YEAGER | Full roster, earned date. |
| 3 | Cadet Special Activities Ribbon (`cadet-special-activities`) | New | Cadet + Senior | `CadetActivities.txt` (denylist) | Participants only; ribbon + bronze star per extra activity. |
| 4 | Leadership Ribbon (`leadership-ribbon`) | New | Senior | `SpecTrack.txt` TrackLevel | Technician = ribbon, Senior = +bronze star, Master = +silver star (highest rating). |
| 5 | Encampment Ribbon (`encampment-status`) | Improve | Cadet + Senior | `CadetActivities.Type` ⊇ `ENCAMP` | Added senior staff + repeat‑attendance (clasp) count. |
| 6 | CAC Ribbon basis (`cac-representatives`) | (existing) | Cadet + Senior | committee/duty data | CAC service is the ribbon basis; report already lists reps/advisors. |
| 7 | Red Service Ribbon (`red-service-ribbon`) | New (rolled up from branch `2-Red-Service-Ribbon-Report`) | Cadet + Senior | `Member.txt` OrgJoined/Joined | **Longevity** ribbon (CAPR 39‑3 §21.b): ribbon at 2 yrs, bronze clasp at 5/10/15 (max 3), numbered longevity device at 20+ yrs. Service counted from earliest join (original OrgJoined persists across lapses/transfers). |

> **Correction:** an earlier draft of this analysis (Tier 2 below) speculated that the Red Service Ribbon
> tracks Senior Level II / `DAVIS`. That was wrong — the Red Service Ribbon is a **longevity** award keyed
> to years of service, as implemented in report #7. Davis remains Level II in the Senior PD Level Awards report (#1).

**Skipped:** Unit Citation / Squadron of Merit (no usable data — `OrgSquadron_Of_Merit.txt` is a legacy
stats snapshot, verified by base64 decode). Standalone Membership Ribbon report (folded into #1 as Level I).

Code touch‑points: constants in `ConfigConstants.html` (`PD_LEVEL_AWARDS`,
`CSA_RIBBON_EXCLUDED_ACTIVITIES`, `CADET_ACTIVITY_NAMES`, `LEADERSHIP_RIBBON_DEVICES`); generators +
`reportCatalog` + PDF `columnConfigs`/orientation in `Index.html`; render blocks in `AppReports.html`.

---

## 1. How CAPWATCH actually stores awards (the key finding)

CAPWATCH does **not** carry every ribbon a member is entitled to. The award data splits three ways:

| Where | Table(s) | Columns | What it holds |
|---|---|---|---|
| **Explicit award records** | `SeniorAwards.txt`, `CadetAwards.txt` | `CAPID,Award,AwardNo,Completed,UsrID,DateMod` | A **narrow, milestone‑only** set of PD/AE awards (see §2). `AwardNo` is the national serial #, not a clasp count. |
| **Activity participation** (entitlement source) | `CadetActivities.txt` | `CAPID,Type,Location,Completed,UsrID,DateMod` | Every encampment / NCSA attendance — the *basis* for activity ribbons, even though the ribbon itself is **not** in the awards tables. |
| **Free‑text catch‑all** | `Training.txt` | `CAPID,TypeCrs,HowComplete,CrsID,Completed,UsrID,DateMod` | Some ribbons/decorations appear only as free‑text rows with `HowComplete` = `Awards` or `Historical Ribbons` (e.g. *"Disaster Relief Ribbon"*, *"Meritorious Service Award …"*). Unstructured and inconsistent. |

**Evidence that the awards tables are milestone‑only:** this unit's `CadetAwards.txt` has just **3 rows**
(`WRIGHT BROTHERS`, `MITCHELL`) for two legacy members — yet `CadetActivities.txt` shows **dozens** of
`ENCAMP` completions across the current cadet roster. If activity ribbons were stored in `CadetAwards.txt`,
we would see many encampment rows there. They are not. **Therefore activity/service‑ribbon entitlement must
be *derived* from `CadetActivities.txt` et al. — it is not handed to us pre‑computed.** This is exactly the
"addition of ribbon entitlements" opportunity the task calls out.

### Observed award codes (code → friendly name)
A report needs this dictionary because the tables store codes, not names:

| Code (`Award`) | Ribbon / Award | Audience |
|---|---|---|
| `MBRRBN` | Membership Ribbon | Senior |
| `YEAGER` | Gen. Chuck Yeager Aerospace Education Award | Senior |
| `DAVIS` | Benjamin O. Davis Award (Level II) | Senior |
| `LOENING` | Grover Loening Aerospace Award (Level III) | Senior |
| `GARBER` | Paul E. Garber Award (Level IV) | Senior |
| `WILSON` | Gill Robb Wilson Award (Level V) *(not present in this small unit, standard code)* | Senior |
| `WRIGHT BROTHERS` | Wright Brothers Award (Achv 4 / Phase I) | Cadet |
| `MITCHELL` | Gen. Billy Mitchell Award (Achv 10 / Phase II) | Cadet |
| `EARHART` | Amelia Earhart Award (Achv 14 / Phase III) | Cadet |
| `EAKER` | Gen. Ira C. Eaker Award (Achv 20 / Phase IV) | Cadet |
| `SPAATZ` | Gen. Carl A. Spaatz Award (Achv 21 / Phase V) | Cadet |

`CadetActivities.Type` values seen (the entitlement source for activity ribbons): `ENCAMP`, `NCSA`, `COS`,
`RCLS`, `NFAG`, `NFAJ Staff`, `CFA-G`, `CFA-P`, `CFAP Staff`, `SpaceOps`, `SpaceOps Staff`, `NESA`,
`SUPTCFC`, `AFSPCFC Staff`, `DDRX`, `RSTInPer`.

---

## 2. What the app already does with award data

| Asset | File / location | Status |
|---|---|---|
| Loads `cadetAwards`, `seniorAwards`, `esMbrAchievements`, `cadetActivities` | `Code.gs` fileMap (`priority 12/25/29/31`); parsed in `ServicesDataService.html`, `ServicesCadetDataService.html` | ✅ synced & in memory |
| `buildSeniorAwards(member)` | `ServicesDataService.html:252` | ⚠️ **Computed but never rendered** — `ModalSeniorProfile.html` shows Specialty Tracks, not awards. |
| `getMilestoneAwards(capid)` + `MilestoneAwardsCard` | `ServicesCadetDataService.html:812`, `ComponentsCadetComponents.html:518` | ⚠️ Renders **only the 5 phase milestones**; uses `cadetAwards` only as a supplement to achievement number. |
| Encampment **completion** (not framed as a ribbon) | report `encampment-status` → `generateEncampmentReport` (`Index.html:4366`) | ✅ exists; no ribbon/clasp framing |
| CAC representatives | report `cac-representatives` (`Index.html:4447`) | ✅ exists; not framed as CAC Ribbon entitlement |
| Yeager used only for QUA gate | `ConfigConstants.html:1125` `QUA_YEAGER_AWARD_NAME` | ✅ |

**Net:** there is **no awards/ribbons report** in the report catalog (`Index.html:4274` `reportCatalog`),
and the richest award data the app already holds is largely unsurfaced.

---

## 3. Feasibility tiers — which ribbons can a report cover?

### Tier 1 — Directly recorded → *surface + entitlement‑gap check* (low effort, high value)
Data is already loaded; only a code→name map and a render surface are missing.

| Ribbon / Award | Source | Report idea |
|---|---|---|
| Membership Ribbon, Yeager, Davis/Loening/Garber/Wilson (Lvl II–V) | `SeniorAwards.txt` | **"Member Awards & Ribbons Roster"** — per‑member earned awards with dates; **gap check**: member completed Level III (`SeniorLevel.LV3`) but no `LOENING` row ⇒ award not posted. |
| 5 Cadet milestones | `CadetAwards.txt` + `CadetAchv.txt` | Already partly shown; add to the same roster and flag milestone earned in `CadetAchv` but missing an award record. |

### Tier 2 — Derivable entitlement (the "add ribbon entitlements" sweet spot)
Not in the awards tables, but computable from data the app already syncs.

| Ribbon | Derivation | Build vs. improve |
|---|---|---|
| **Encampment Ribbon** | `CadetActivities.Type == 'ENCAMP'` (≥1 = entitled; row count ⇒ clasp/oak‑leaf count). Seniors who staff encampment also qualify. | **Improve** `encampment-status`: add "Ribbon entitled / # awards (clasps)" columns. |
| **Cadet Special Activities (CSA) Ribbon** | `CadetActivities.Type` ∈ national‑activity set (`NCSA, COS, RCLS, NFAG, NFAJ, CFA‑*, SpaceOps, NESA, SUPTCFC, AFSPCFC…`); ≥1 attendance = entitled. | **Build** new "Cadet Activity Ribbons" report. |
| **CAC Ribbon** | CAC service already resolved by `generateCACRepresentativesReport` (`Index.html:4447`) from committee/duty data. | **Improve**: flag current/!past CAC reps as ribbon‑entitled. |
| **Red Service Ribbon** | ~~Senior Level II complete~~ — **incorrect**; it is a longevity award (years of service from `Member.txt` OrgJoined/Joined). See §0 report #7. | **Built** (`red-service-ribbon`). |
| **Leadership Ribbon (Technician)** | `SpecTrack.TrackLevel ∈ {TECHNICIAN, SENIOR, MASTER}` (per CAPP‑series guidance: Tech rating → Leadership Ribbon; bronze/silver star for Senior/Master). | **Build/extend** Tier‑1 roster. |
| IACE / National Cadet Competition / National Color Guard ribbons | `CadetActivities.Type` if the corresponding code is present (none in this unit's sample). | Low priority; data‑dependent. |

> **Recommended headline feature:** a single **"Ribbon Entitlement vs. Awarded"** report that, per member,
> lists each derivable ribbon, whether they're **entitled** (from activity/level/track/CAC data), and whether
> a matching **award record exists**. The delta is a commander's to‑do list ("these cadets earned the
> Encampment Ribbon — make sure it's posted in eServices"). This is the highest‑leverage, low‑risk build.

### Tier 3 — Partial / low‑confidence
| Ribbon | Limitation |
|---|---|
| **Recruiter Ribbon** (7 recruits) | `Member.txt` has `Joined`/`OrgJoined` but **no sponsor/recruiter linkage** → cannot attribute recruits to a sponsor. Not reliably derivable. |
| **Disaster Relief / Air SAR / "Find" / Homeland Security** | Mission‑participation based; appear only as **free‑text** rows in `Training.txt` (`HowComplete='Awards'/'Historical Ribbons'`). A scrape is possible but fragile (inconsistent strings). Flag as "informational only." |

### Tier 4 — Not trackable (out of scope for entitlement reports)
Nomination/discretionary decorations — Medal of Valor (Silver/Bronze), Distinguished/Exceptional/Meritorious
Service, Commander's Commendation, Achievement Award, Lifesaving Award, Unit Citation. No structured CAPWATCH
basis; surface only if already present as a `Training.txt` text row.

### Unit‑level awards (bonus)
- **QCUA** and **QUA** already have reports.
- `OrgSquadron_Of_Merit.txt` is synced → a **Unit Citation / Squadron of Merit history** panel is feasible.

---

## 4. Recommended next steps (in priority order)

1. **Add the code→name dictionary** (a `RIBBON_AWARDS` map in `ConfigConstants.html`) covering the §1 codes
   plus `CadetActivities.Type` → ribbon mapping. Everything else depends on it.
2. **Render senior awards** in `ModalSeniorProfile.html` (data already built at `ServicesDataService.html:252`)
   and broaden the cadet `MilestoneAwardsCard` to list all recorded cadet awards.
3. **Build the "Ribbon Entitlement vs. Awarded" report** (Tier 1 + Tier 2) in `reportCatalog`
   (`Index.html:4274`), following the existing `generate(members)` pattern (icon `Award`, tags
   `['cadet','senior','awards','readiness','compliance']`).
4. **Improve `encampment-status`** to add Encampment‑Ribbon entitlement + clasp count.
5. (Optional) Tier‑3 informational panel that surfaces `Training.txt` "Awards/Historical Ribbons" text rows.

**Effort/risk:** Steps 1–2 are trivial (render already‑loaded data). Step 3–4 reuse the established report
generator + PDF export plumbing, so no new data plumbing is required — only new derivation logic over tables
the app already parses.
