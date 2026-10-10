# Herbaceous vegetation: data quality report and tech hand-off

**Project:** Savanna Monitoring Pilot · **Area:** Lewa (LW) · **Survey:** `savmon|lw|herbs|ho` (2026 baseline)

**Contents**
- [Vegetation challenge 1: data quality report](#vegetation-challenge-1-data-quality-report)
  - [a. Survey-level summary](#a-survey-level-summary)
  - [b. Plot-level summary](#b-plot-level-summary)
  - [c. Assessment of sampling effort](#c-assessment-of-sampling-effort)
  - [d. Map of transect locations](#d-map-of-transect-locations)
- [Vegetation challenge 2: hand-off to the Tech team](#vegetation-challenge-2-hand-off-to-the-tech-team)


## Vegetation challenge 1: data quality report

### a. Survey-level summary
| | |
|---|---|
| Dates | 19–27 May 2026 (9 field days) |
| Team | Sam, Wanjiku, Grace · recorders: Grace 26, Sam 6 |
| Submissions | 32, of which 2 rejected in ODK review |

**From planned locations to accepted surveys**

| Stage | Source | Count | What drops out |
|---|---|---|---|
| Planned locations | `centroids.csv` | **51** | 36 primary + 15 backups |
| Registered in the field | `vegplots.csv` | **31** | 30 primary + 1 backup (Plot_46). Never registered: primary Plots 31–36, and 14 backups |
| Viable | `vegplots.is_viable` | **30** | Plot_23 non-viable |
| Survey submissions | `herbaceous_veg_survey.csv` | **32** | Covers all 30 viable plots; Plots 18 and 21 were surveyed twice |
| Accepted surveys | `ReviewState` ≠ rejected | **30** | 2 rejected: the repeat visits to Plots 18 and 21 |

Result: **one accepted survey for every viable registered plot**: 29 primary plots plus backup Plot_46. Of the 36 primary plots, 7 were not surveyed: Plot_23 (non-viable) and Plots 31–36 (never registered).

**Duplicate surveys:** Plots 18 and 21 were each surveyed twice on 26 May. In both cases the rejected submission is the later, repeat visit, and the earlier survey was kept. The reason for rejection isn't recorded in the export.

### b. Plot-level summary
The notebook's plot table has one row per plot: quadrats, worst quadrat GPS accuracy, % of species records that are provisional, distance of the midpoint from its planned location, and new species in the final 5 quadrats.

A median of 30% of each plot's species records are provisional (`herb_XXX`), ranging from 9% to 74% across plots.

| Check | Result |
|---|---|
| 20 quadrats per plot | ✅ all plots |
| Quadrat GPS accuracy ≤ 5 m | ✅ all quadrats (worst is 5.0 m) |
| Midpoint ≤ 25 m from planned location | ❌ **Plot_14: 33.5 m** (all others ≤ 24 m) |
| "Additional species = yes" but none entered | ⚠️ 9 quadrats in 7 plots. These quadrats are probably missing species records. |
| Additional-species entry with no species selected | ❌ 1 entry |
| Same species added twice to the new-species list (two IDs) | ❌ *Aristida scabrivalvis*, *Justicia divaricata* |
| New-species entry already on the project list | ⚠️ *Pechuel-loeschea leubnitziae* |
| Typed species names with leading or trailing spaces | ℹ️ 64 entries |

### c. Assessment of sampling effort
The notebook (section c) plots species accumulation curves: the average number of species found as quadrats (or plots) are added, over random orders. A curve that flattens means extra sampling finds little new; one still rising means more species remain. To compare plots, I use the number of new species added by the final 5 quadrats (or plots).

- **Within plots (20 quadrats):** the final 5 quadrats still add a median of 2.6 new species (range 0.6–4.5). Most curves are still rising at quadrat 20, so 20 quadrats don't capture a plot's full species list.
- **Across the project (30 plots):** the final 5 plots still add about 9 new species, so more plots would probably add more.
- **Caveat:** provisional species affect these numbers. If several `herb_XXX` entries are later identified as the same species, or as species already on the list, these counts will drop.

**Recommendation:** resolve the provisional species before using the species data for reporting. If a fuller species list is needed, consider more quadrats per plot and more plots.

### d. Map of transect locations
The map is in the notebook (section d). It shows each registered transect (endpoint A to B, labelled by plot number), unused backup locations, and the planned primary plots that were never registered. The six unregistered plots are all on the eastern edge of the area. With more time, I would add an interactive map (e.g. folium) with a satellite imagery basemap, to inspect vegetation cover across the site and how the plots sit within it.

## Vegetation challenge 2: hand-off to the Tech team

**Goal:** when herbaceous survey data arrives from ODK, automatically join the tables, run the QA checks below, and show the results in a Vegetation QA/QC dashboard. Errors are flagged, never fixed. Given the time available, this is an outline of the approach rather than a full specification; [`vegetation-data-investigation.ipynb`](vegetation-data-investigation.ipynb) prototypes each step.

### 1. Joins
| Table | Links to | Using |
|---|---|---|
| `herbaceous_veg_survey` (one per plot visit) | `vegplots` (registered plot) | `selected_plot_uuid` → `vegplots.__id` |
| `vegplots` | `centroids` (planned location) | `plot_uuid` → `centroids.__id` |
| `quadrat_repeat` (20 per visit) | `herbaceous_veg_survey` | `PARENT_KEY` → `KEY` |
| `additional_species_repeat` (off-list species) | `quadrat_repeat` | `PARENT_KEY` → `KEY` |
| species on each quadrat | `species` (project list) | space-separated IDs → `species.__id` |
| additional species entries | `species_extra` (new and provisional species) | `target_extra_name` / `select_reuse_*` → `species_extra.__id` |

### 2. Steps
Re-run for a survey whenever a submission arrives, a submission is reviewed, or an entity list changes.
1. Join quadrats and species records to their plots by ID, excluding submissions rejected in ODK review.
2. Run the [checks](#3-checks) below.
3. Compute the measures from the data quality report: the [survey-level summary](#a-survey-level-summary), [plot-level summary](#b-plot-level-summary) and [sampling effort](#c-assessment-of-sampling-effort). This could feed into a dashboard.

### 3. Checks
Thresholds come from the SOPs.

| Level | Check | Rule |
|---|---|---|
| survey | Planned plot not registered | primary in `centroids` but not in `vegplots` |
| survey | Viable plot not surveyed | `is_viable = yes` but no accepted survey |
| survey | Survey of non-viable or unregistered plot | accepted survey for a plot not registered as viable |
| survey | Plot surveyed more than once | more than one submission for a plot |
| plot | Quadrat count | not 20 quadrats in a plot's accepted survey |
| plot | Midpoint moved too far | registered midpoint > 25 m from the planned location |
| quadrat | Quadrat GPS accuracy | > 5 m |
| quadrat | Additional species missing | `additional_species_present = yes` with no species entries |
| species | Entry with no species | no `species_extra` ID created or re-used |
| species | Duplicate new species | the same name on more than one `species_extra` entry |
| species | New species already on list | a new species entry whose name is on the project list |
| species | Whitespace in typed name | leading or trailing spaces in the typed name |

### 4. Dashboard
- Plots planned, registered and surveyed, and a map of them.
- A list of check failures, filterable by check and plot.
- The plot summary and the sampling-effort curves.
- Provisional species waiting for identification.

### 5. Suggested next steps
- Export ODK review comments alongside `ReviewState`, so the dashboard can show why a submission was rejected.
- Check free-text species names against a taxonomic reference (GBIF) to catch misspellings.
- Add unit tests for each check.
