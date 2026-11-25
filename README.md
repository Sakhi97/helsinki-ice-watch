# Helsinki Ice Watch 🛰️❄️

**Mapping snow on land and ice on the Gulf of Finland around Helsinki through a whole winter, using free Copernicus Sentinel-2 satellite images.**

[![CI](https://github.com/Sakhi97/helsinki-ice-watch/actions/workflows/ci.yml/badge.svg)](https://github.com/Sakhi97/helsinki-ice-watch/actions/workflows/ci.yml)

🌍 **Live map:** https://Sakhi97.github.io/helsinki-ice-watch/

## Why

Every winter, people in Helsinki ask whether the sea is frozen yet and whether the snow will last. Satellites answer that every few days, for free, but the raw data comes as 110 km tiles of 13-band imagery that nobody outside remote sensing can read.

This project turns that data into a slider map and one chart: **how much of the land is snowy, and how much of the sea is frozen, on each clear day of the winter.**

## What it does

1. **Finds images.** Searches the Sentinel-2 Level-2A catalogue (STAC API) for every scene over Helsinki in the chosen winter.
2. **Downloads only what it needs.** Reads just the Helsinki window from Cloud-Optimized GeoTIFFs (HTTP range requests), resampled to 40 m. That's a few MB per date instead of ~1 GB.
3. **Masks clouds.** Uses ESA's Scene Classification Layer (SCL) to ignore cloud, cloud shadow and no-data pixels.
4. **Finds the sea.** Frozen sea looks like snow, so winter images can't tell you where the sea is. Instead, a pixel counts as sea if it was water in most clear **summer** images.
5. **Detects snow and ice.** Uses the Normalised Difference Snow Index:
   `NDSI = (green − SWIR) / (green + SWIR) > 0.4` **and** green reflectance > 0.10.
   The brightness check matters: open water also has a high NDSI, but it is dark.
6. **Reports.** Per-date map layers, a time-series chart, CSV/JSON results and an interactive Leaflet map, published to GitHub Pages by GitHub Actions.

```
STAC search ──► read COG windows ──► cloud mask (SCL) ──► NDSI + brightness ──► land / sea split ──► map + chart
                                                   summer scenes ──► sea mask ──┘
```

## Results

*Run the "Analyse satellite data and publish map" workflow, then summarise what you found here. For example: when the sea started freezing, when ice cover peaked, how many dates were usable, and how often clouds blocked the view.*

### Self-check against ESA's own snow class
For each date I also report the pixel agreement between my snow/ice map and Sen2Cor's SCL class 11 ("snow/ice"). These are two independent methods. High agreement is a sanity check, and the dates where they disagree are the interesting ones to inspect.

## Limitations (honest list)

- **Darkness and clouds.** Sentinel-2 is an optical camera. In December–January the sun is very low at 60°N, and Finnish winters are cloudy, so mid-winter dates are rare. **This is exactly the problem radar (SAR) solves**: Sentinel-1, and commercial SAR constellations like ICEYE's, see through cloud and darkness. Adding Sentinel-1 is the top item on the roadmap.
- **Thin, dark ice** (newly formed ice, or ice with water on top) can be darker than the brightness threshold and gets counted as open water.
- **Snow under forest** is partly hidden by trees, so land snow cover is under-estimated in forested areas.
- **No ground truth yet.** Results should be compared against the Finnish Meteorological Institute's ice charts (see roadmap).

## Run it yourself

```bash
git clone https://github.com/Sakhi97/helsinki-ice-watch.git
cd helsinki-ice-watch
python -m venv .venv && source .venv/bin/activate     # Windows: .venv\Scripts\activate
pip install -r requirements-dev.txt

pytest                                   # 15 tests, no internet needed
python -m snowice demo                   # offline demo site from synthetic data -> demo/
python -m snowice run                    # real data, winter 2025-26 -> docs/  (needs internet, ~10-30 min)
python -m snowice run --winter 2024-11-01/2025-04-30 --summer 2024-06-01/2024-08-31
python -m http.server -d docs 8000       # then open http://localhost:8000
```

Publish your own live map:
1. GitHub repo → **Settings → Pages → Source: GitHub Actions**
2. **Actions → "Analyse satellite data and publish map" → Run workflow**

## Project structure

| File | What it does |
|---|---|
| `snowice/config.py` | Area of interest, thresholds, SCL class codes |
| `snowice/stac.py` | Catalogue search with pagination, same-day tile grouping, scale/offset |
| `snowice/raster.py` | Reads remote COG windows warped onto one shared lon/lat grid |
| `snowice/indices.py` | The science: NDSI, cloud mask, sea mask, per-scene classification (pure NumPy) |
| `snowice/analysis.py` | Runs the season and skips cloudy or broken scenes |
| `snowice/report.py` | PNG layers, chart, CSV/JSON, web page |
| `.github/workflows/` | CI tests on every push; on-demand analysis + Pages deploy |

## Design decisions

- **Science code is pure NumPy, separate from downloading.** So the classification logic is unit-tested with small synthetic arrays, with no internet or GIS install needed in CI.
- **One shared grid for every band and date.** Bands come at 10 m and 20 m, and Helsinki straddles tile edges. Warping everything to one lon/lat grid means pixel `[i, j]` is the same spot on the ground everywhere, and the PNGs drop straight onto a web map.
- **Scale/offset read from metadata.** Since ESA's processing baseline 04.00 (2022), L2A data has a −0.1 reflectance offset. The code reads it from the catalogue (`raster:bands`) instead of hard-coding it, so old and new scenes are handled correctly.
- **Static website, no server.** GitHub Pages hosts it for free and it never goes down.

## Roadmap

- [ ] **Sentinel-1 SAR** ice detection, which works through clouds and polar darkness
- [ ] Validate sea-ice extent against Finnish Meteorological Institute ice charts
- [ ] Compare several winters (ice seasons are getting shorter; how does Helsinki look?)
- [ ] Per-area stats: Helsinki vs Espoo coast vs outer archipelago

## Data and credits

Contains modified Copernicus Sentinel data, processed with this project. Sentinel-2 L2A Cloud-Optimized GeoTIFFs are accessed through Earth Search (Element 84) on AWS Open Data. Base map © OpenStreetMap contributors © CARTO.

**Tech:** Python · NumPy · rasterio/GDAL · STAC · Cloud-Optimized GeoTIFF · Matplotlib · Leaflet · GitHub Actions · GitHub Pages
