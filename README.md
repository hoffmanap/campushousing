# El Paso Campus Housing Zone — Legislative Impact Analysis

This repository contains the geospatial pre-processing engine, dataset schemas, and mathematical modeling parameters used to analyze the municipal and physical impacts of upzonint near Higher Learning Institutions. 

The end product of this analysis feeds a dynamic, interactive dashboard that models zoning optimization, identifies underutilized parcels, and projects net housing unit additions near major institutions of higher education in El Paso, Texas.

🌐 **Live Interactive Dashboard:** [https://hoffmanap.github.io/campushousing/](https://hoffmanap.github.io/campushousing/)

---

## 1. Legislative Criteria & Core Mandates

The analysis maps out precise boundaries and developer envelopes explicitly established by the statutory text of Chapter 218:

### A. Campus Eligibility Threshold
* Applies exclusively to **Institutions of Higher Education** maintaining a physical campus with a verifiable on-site enrollment of **at least 15,000 students**. 
* Core qualifying nodes modeled within El Paso include the main campus of the **University of Texas at El Paso (UTEP)** and core campuses of the **El Paso Community College (EPCC)**.

### B. Spatial Definition of the "Campus Housing Zone"
The designated impact zone is calculated automatically for each eligible campus using a mathematical bounding minimize function. The zone extends to the **lesser** of:
1. A uniform, straight-line **one-half (1/2) mile buffer** extending outward from the contiguous campus property boundary line.
2. A distance radius from the campus boundary equal to the radius ($r$) of a perfect circle whose total area matches the exact total land acreage of that specific campus ($Area = \pi r^2$).

### C. Development Standards & Prohibited Municipal Constraints
Inside the delineated Campus Housing Zone, standard restrictive municipal zoning overlays are legally superseded. Municipalities are prohibited from enforcing regulations that limit density or layout, introducing the following by-right standards:
* **By-Right Approvals:** Multifamily residential developments (defined as $\ge 3$ dwelling units per tract) and college dormitories are permitted entirely by-right without requiring discretionary variances, conditional use permits, or comprehensive plan amendments.
* **Height Protections:** Building heights cannot be restricted to less than **45 feet** (approximately 4 stories) or the maximum height allowed under base zoning, whichever is greater.
* **Elimination of Floor-Area Ratio (FAR):** No maximum FAR limits on multifamily developments permitted within the zone.
* **Dimensional Standards:** No minimum lot size, lot width, or lot depth constraints can be used to impede multifamily construction.
* **Setback Alignments:** Setbacks, stepbacks, or yard buffers cannot be stricter than those generally applicable to multifamily residential development citywide.
* **Parking Mandates Eliminated:** Municipalities are forbidden from enforcing minimum off-street parking space requirements.

---

## 2. Geospatial Pre-processing Pipeline

Because raw spatial data from municipal and central appraisal district registries is too dense and unformatted for instantaneous web rendering, a automated local pipeline was built using Python (`geopandas`, `pandas`, `shapely`) to clean, join, and optimize the data layer.

### Phase 1: Spatial Delineation & Clipping
1. **Coordinate Conversion:** All source datasets—including `campuses.geojson` (campus geometries), `Parcels.geojson` (El Paso Central Appraisal District property lines), and `FootprintHeights.geojson` (existing structures)—are systematically reprojected into a local projected coordinate system measured in feet (**NAD83 / Texas Central - EPSG:2278**) to guarantee mathematically accurate `.area` and `.buffer()` calculations.
2. **Zone Masking:** The script generates the legal bounding buffer for eligible campuses and intersects it against a municipal boundary layer (`CampusClip.geojson`) to isolate valid properties and remove slivers outside city limits.

### Phase 2: Dimensional Zoning Alignment
1. **Attribute Extraction:** Base parcel attributes are stripped of non-essential variables, mapping the core parameters: `LAND_ACRES` (parcel area) and `ZONELABEL` (existing base zoning district).
2. **Relational Database Join:** A relational merge attaches localized dimensional lookup metrics from `Zoning_Criteria_Lookup.csv` to each parcel profile based on the corresponding `District_Key`.
3. **Dynamic Boundary Offsets:** Since standard 2D vector buffers shrink polygons uniformly on all axes, an averaged setback deduction formula $\frac{\text{Front} + \text{Rear} + \text{Side} + \text{Side}}{4}$ applies a localized geometric reduction unique to each parcel's base zoning framework. This computes the `buildable_footprint_sqft` remaining once legal yard limits are satisfied.

### Phase 3: Footprint Overlays & Density Calculations
1. **Spatial Join (`sjoin`):** Structural building footprints (`FootprintHeights.geojson`) containing `HEIGHT_FT`, `SQFEET`, and `PRIM_OCC` (primary occupancy codes) are overlaid directly onto the parcel map.
2. **Density Base Calculation:** The script aggregates the existing building footprint square footages per property, contrasts it against the total parcel acreage, and explicitly computes the pre-legislation density footprint line:
$$\text{EXISTING\_FAR} = \frac{\text{Sum of Existing Structural Square Footage}}{\text{Total Parcel Square Footage}}$$

### Phase 4: Web Optimization (The Compression Loop)
To bypass GitHub Pages' file storage boundaries and prevent page latency or frame drops during live slider adjustments, a standalone optimization engine (`optimize_geojson.py`) executes a three-tiered size compression:
* **Coordinate Precision Anchor:** Limits latitude/longitude strings to a strict 5-decimal precision ceiling. This maintains sub-meter surveying accuracy while instantly trimming up to 60% of raw string character length.
* **Property Whitelisting:** Drops all residual appraisal database strings or internal temporary join variables, compiling only the precise keys needed for mapping calculations and interactive UI tooltips.
* **Topology Simplification:** Utilizes a lightweight Douglas-Peucker tolerance curve to smooth and prune redundant jagged vertices from complex parcel structures.
* *Result:* Condenses an unmanageable 11 MB tracking layer down to a highly responsive, web-ready **2 to 3 MB `campus_zone_parcels.geojson` asset**.

---

## 3. Analytical & Engineering Assumptions

To model realistic development outcomes under a clean-slate by-right zoning envelope, the following systematic assumptions were integrated into the backend calculations:

* **Average Unit Size:** Fixed at a standardized baseline of **800 sq. ft.**, matching typical "Missing Middle" housing scales (such as courtyard apartments or walk-ups).
* **Building Efficiency Factor:** Modeled at **80% net leasable area**. The remaining 20% is structurally reserved for internal building infrastructure, including stairwells, framing, egress paths, and common utility channels.
* **Soft Site Identification Metrics:** Real estate assessment values are notoriously subject to lagging data cycles or distortion. To correct for this, this analysis utilizes **Floor Area Ratio (FAR)** as the primary indicator for site underutilization. Any parcel exhibiting an $\text{EXISTING\_FAR} < 0.3$ is flagged with a boolean value (`IS_SOFT_SITE = 1`), indicating it is primed for market-driven redevelopment (such as conversion of vast surface parking lots or single-story structures).
* **Volumetric Capacity Model:** Gross new unit potential per property is derived using the maximum allowable 45-foot physical height limit (yielding 4 structural stories) multiplied by the buildable space footprint:
$$\text{gross\_units} = \left\lfloor \frac{(\text{buildable\_footprint\_sqft} \times 4 \text{ stories}) \times 0.80 \text{ Efficiency}}{\text{800 sq. ft. Unit Size}} \right\rfloor$$
* **Net New Housing Output:** Projected new value added by the bill on any given parcel subtracts the original existing housing count from the maximized envelope potential ($\text{net\_new\_units} = \text{gross\_units} - \text{existing\_units}$).

---

## 4. Repository Structure & Requirements

```text
├── .gitignore                        # Standard protection file blocking raw heavy assets and environments
├── requirements.txt                  # Standardized version locks for pipeline reproduction
├── Zoning_Criteria_Lookup.csv       # Relational lookup table mapping setbacks to base zoning
├── updateparcels.py                  # Core spatial execution and FAR calculation workflow script
├── optimize_geojson.py              # Precision and vertex compression script
├── index.html                        # Production front-end container displaying the interactive map interface
└── assets/
    └── campus_zone_parcels.geojson   # Highly optimized, lightweight output source feeding the map
