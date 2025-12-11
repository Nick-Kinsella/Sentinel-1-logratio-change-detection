# S1 Log-Ratio Change Detection

Jupyter notebook for simple two-date change detection with **Sentinel-1** SAR data using the **log-ratio** method:

$\log_{10}\!\left(\frac{\text{newer image}}{\text{older image}}\right)$

The workflow is a Python / openEO reimplementation of the classic “log-ratio scaling” approach often demonstrated in QGIS tutorials, but:

- selects an appropriate **Sentinel-1 GRD** pair via the Copernicus **STAC** API,
- enforces **same track, orbit direction, platform and polarisation**,
- downloads clipped images via **openEO**, and
- visualises the log-ratio as an interactive **Folium** map over OSM / satellite basemaps.

> ⚠️ Note: this example currently uses **S1 GRD**, not full terrain-flattened RTC. It is best suited to relatively flat areas; mountains will show terrain-related artefacts.

---

## Contents

- `S1_LogRatio_ChangeDetection.ipynb` (or similar)  
  Main notebook demonstrating:
  - STAC search for Sentinel-1 GRD acquisitions,
  - openEO download and clipping,
  - log-ratio computation,
  - Folium visualisation.

You can rename the notebook as you like; the repo is intended as a minimal, readable example rather than a polished package.

---

## Method overview

1. **User parameters**  
   You specify:
   - AOI centre (`user_latitude`, `user_longitude`)
   - AOI half-size (`buffer_degrees`)
   - Two target dates (`older_date`, `newer_date`)
   - Polarisation (`VV` or `VH`)

2. **STAC date finder**  
   The notebook queries the **Copernicus Data Space STAC API** to find the **nearest Sentinel-1 GRD acquisitions** to each target date, filtered by:
   - collection: `sentinel-1-grd`
   - instrument mode: `IW`
   - polarisation: `VV` or `VH`  

   It:
   - first finds an “older” acquisition,
   - extracts its **relative orbit**, **orbit direction** (ascending/descending), and **platform** (S1A/S1B),
   - then finds the “newer” acquisition constrained to the **same** orbit / direction / platform.

3. **Download via openEO**

   Using the selected dates and AOI, the notebook:

   - loads `SENTINEL1_GRD` via the **openEO** backend,
   - clips to the AOI,
   - optionally reduces over time (median) if there are multiple acquisitions,
   - downloads two single-band GeoTIFFs (older and newer), and
   - saves them with filenames based on the **actual acquisition dates**.

4. **Log-ratio change detection**

   The two images are:

   - read with `rasterio`,
   - converted to `float32`,
   - combined as:

     ```python
     ratio = newer_arr / older_arr
     logratio = np.log10(ratio)
     ```

   - invalid pixels (division by zero, non-finite values) are masked as `NaN`,
   - the result is saved as a georeferenced GeoTIFF.

5. **Visualisation**

   The notebook:

   - prints basic stats (min, max, 2nd / 98th percentile) for older, newer, and log-ratio,
   - stretches the log-ratio to 0–255 using the 2nd / 98th percentiles,
   - writes a greyscale PNG,
   - overlays the PNG on a **Folium** map over:
     - OpenStreetMap, and
     - Esri World Imagery (satellite).

   Because CRS transformation is broken in the reference environment, the overlay is mapped to the **requested AOI bounds in WGS84** rather than transformed UTM bounds. This is usually close enough for flat areas but is an approximation.

---

## Interpreting the map

The Folium layer shows **log10(newer / older)** backscatter:

- **Mid-grey (~0):** little or no change; scattering behaviour is similar on both dates.
- **Bright (positive):** newer image is brighter:
  - possible infrastructure development, vegetation removal, rougher surfaces, drying of wet areas, etc.
- **Dark (negative):** newer image is darker:
  - possible vegetation growth, wetter surfaces, flooding, surface smoothing, or removal of bright scatterers.

This is **change in scattering behaviour**, not a land-cover classification. Always interpret in context with the satellite basemap and local knowledge.

Because Sentinel-1 is **side-looking SAR** and the basemap is **near-nadir optical**:

- building shapes and tall structures will not line up perfectly,
- steep terrain will show layover and shadow (especially without full terrain-flattened RTC),
- some small spatial misalignment is expected.

---

## Requirements

- Python 3.x
- Jupyter Notebook / JupyterLab

Python packages (typical):

- `openeo`
- `requests`
- `numpy`
- `rasterio`
- `matplotlib`
- `folium`
- `Pillow`

Install example (conda style):

```bash
conda create -n s1_logratio python=3.11
conda activate s1_logratio
conda install -c conda-forge rasterio folium pillow matplotlib
pip install openeo requests
