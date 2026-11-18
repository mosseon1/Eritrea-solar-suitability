# Solar PV Site Suitability Mapping for Eritrea

A simple, transparent GIS model in Python (GeoPandas, rasterio) that maps where utility-scale solar PV is most suitable in Eritrea, tests how stable that map is when the factor weights change, and checks the result at the Dekemhare 30 MW PV plant.

![Suitability map](outputs/map_suitability.png)

## Question

Where in Eritrea do high solar resource, gentle terrain, open land and nearby infrastructure overlap, and how much does that answer depend on the weights chosen for each factor?

## Data (all free and open)

| Layer | Source | Use |
|---|---|---|
| Country boundary | geoBoundaries, ERI ADM0 (gbOpen) | Study area and mask |
| Solar irradiance (GHI, kWh/m²/day) | Global Solar Atlas v2, Eritrea GIS data, long-term yearly average of daily totals (GeoTIFF, ~250 m) | Solar factor; Dekemhare check |
| Elevation | Copernicus DEM GLO-90 (AWS open data) | Slope |
| Land cover | ESA WorldCover 2021 v200, read from the cloud at a reduced-resolution overview (~160 m) | Exclusions and land-cover score |
| Power lines | Gridfinder (Arderne et al., 2020, *Scientific Data*): OpenStreetMap lines plus lines predicted from night-time lights | Distance to grid |
| Power lines (comparison) and main roads | OpenStreetMap, Geofabrik Eritrea extract (downloaded 18 Nov 2026); roads = trunk, primary, secondary | Distance to roads; grid-data comparison |
| ERA5 daily shortwave radiation | Open-Meteo historical archive (ERA5), 1999-2018 | Dekemhare check |

## Method

1. **Working grid:** UTM zone 37N (EPSG:32637), 500 m cells, clipped to Eritrea (481,526 cells, 120,382 km²).
2. **Slope:** DEM resampled to 250 m, slope computed in degrees, then averaged to 500 m.
3. **Land cover:** most common WorldCover class per 500 m cell.
4. **Distances:** power lines and roads rasterised to the grid; Euclidean distance (km) to the nearest line.
5. **Exclusions:** slope > 10°; land cover = tree cover, built-up, water, herbaceous wetland, mangroves.
6. **Scoring (linear, 0-1) and starting weights:**

| Factor | Score = 0 | Score = 1 | Weight |
|---|---|---|---|
| GHI | 5.0 kWh/m²/day | 6.5 kWh/m²/day | 0.35 |
| Slope | 10° | 0° | 0.15 |
| Land cover | – | bare 1.0, grassland 0.9, shrubland 0.8, cropland 0.4 | 0.10 |
| Distance to power lines | 50 km | 0 km | 0.25 |
| Distance to main roads | 20 km | 0 km | 0.15 |

7. **Suitability** = weighted sum of the five scores; the top 10 % of usable land is marked.
8. **Sensitivity test:** 300 scenarios (seed 42), each weight multiplied by a random factor in [0.7, 1.3] and renormalised to sum to 1. For each cell I record how often it falls in the top 10 %.
9. **Dekemhare check:** model score, percentile and robustness around Dekemhare (15.07 N, 39.05 E; cells within 3 km), and Global Solar Atlas vs ERA5 GHI with the bias in %.

## Results

**Exclusions and usable land**
- Slope > 10° excludes 20.9 % of Eritrea; land cover excludes a further 0.4 %.
- Usable land: 94,684 km² (78.7 % of the country).
- 67.7 % of Eritrea is within 50 km of a Gridfinder power line (13.3 % with OpenStreetMap lines only); 56.4 % is within 20 km of a main road.

**Suitability**
- Scores on usable land range from 0.184 to 0.961 (median 0.596).
- Top-10 % threshold: 0.837, covering 9,468 km², mainly in the western lowlands and along the highland grid corridor.

**Sensitivity (300 weight scenarios)**
- On average 94.7 % of the baseline top-10 % area stays in the top 10 % (worst scenario: 89.8 %).
- 6,946 km² is in the top 10 % in all 300 scenarios; 7,942 km² in at least 90 % of them; 12,508 km² in at least one.
- 83.9 % of the baseline top-10 % area is robust (≥ 90 % of scenarios).
- No single weight dominates: the largest correlation between a weight and the share of the top 10 % kept is for land cover (r = −0.24); GHI 0.10, slope −0.08, power lines 0.04, roads 0.00.

![Robustness map](outputs/map_robustness.png)

**Dekemhare 30 MW PV plant**
- The Dekemhare town cell itself is excluded (built-up), as expected.
- Within 3 km (84 usable cells): median score 0.874 (above the top-10 % threshold), best score 0.908 (98th percentile), and some cells are in the top 10 % in all 300 scenarios. The model independently ranks the area where the plant was built among the most suitable land in Eritrea.
- GHI at the site: Global Solar Atlas 6.37 kWh/m²/day; ERA5 (1999-2018, nearest grid point 15.0 N, 39.0 E, 2,007 m) 5.87 kWh/m²/day, a bias of −7.9 %. ERA5 also shows the summer rainy-season dip in July-August.

![Dekemhare comparison](outputs/dekemhare_gsa_vs_era5.png)

## Limitations

- **Grid data:** Gridfinder lines are partly *predicted* (reported ~70 % accuracy at 1 km) and date from 2020; OpenStreetMap alone maps only 162 km of lines in Eritrea. Changing the grid layer changes which areas count as near the grid.
- **Land cover at 500 m:** the most-common-class method undercounts small towns, rivers and reservoirs (built-up and water each show only 0.1 %).
- **Missing criteria:** protected areas, flood risk, dust and soiling, temperature, land tenure, security and substation capacity are not included.
- **Expert-chosen rules:** thresholds and weights are simple and transparent but not calibrated; the sensitivity test varies weights (±30 %) but not thresholds.
- **Dekemhare location:** exact plant coordinates are not published; the check uses the town position and a 3 km neighbourhood.
- **Solar data comparison:** ERA5 is a ~30 km reanalysis grid cell, while Global Solar Atlas is satellite-based at ~250 m, so part of the bias is a scale difference.
- **Slope at 250 m** smooths small steep features.

## Outputs

- `outputs/map_suitability.png`, `outputs/map_robustness.png`, `outputs/dekemhare_gsa_vs_era5.png`
- `outputs/results.json`: all numbers quoted above
- `outputs/scenario_weights.csv`: the 300 weight sets and the share of the top 10 % kept in each
- `outputs/suitability_500m.tif`, `outputs/robustness_top10_pct.tif` (GeoTIFF, EPSG:32637)

## How to run

1. Open `eritrea_solar_suitability.ipynb` in Google Colab (no installation needed).
2. Download the Global Solar Atlas GIS data for Eritrea (globalsolaratlas.info/download/eritrea → *LTAy_AvgDailyTotals (GeoTIFF)*) and place the .zip in `MyDrive/eritrea-solar-suitability/data/`.
3. Run the cells in order. All other data are downloaded automatically. When reopening the notebook, re-run sections 1.1-1.3 and 2.2 first.

## Data credits

geoBoundaries; Global Solar Atlas 2.0 (World Bank / Solargis); Copernicus DEM GLO-90 (ESA / Copernicus); ESA WorldCover 2021 v200; Gridfinder (Arderne, Zorn, Nicolas & Koks, 2020); © OpenStreetMap contributors (ODbL), via Geofabrik; ERA5 (Copernicus Climate Change Service) via Open-Meteo.
