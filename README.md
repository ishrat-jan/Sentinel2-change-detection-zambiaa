# Sentinel2-change-detection-zambiaa
Sentinel-2 Change Detection — Open-Pit Mine, Zambia

Detecting land-surface change at an open-pit copper mine in Zambia from two
Sentinel-2 images taken three weeks apart (2023-08-12 and 2023-09-02). Done for
the Solafune technical assessment.

The short version: load the imagery, line the two dates up, find where the ground
changed using PCA, clean up the result, turn it into polygons, save those to a
database, and map it.

Two notebooks

I ended up doing this two ways, so there are two notebooks:


3banded.ipynb — uses the three bands the assessment gives you (B02 blue,
B03 green, B04 red). This is the main one and runs straight off the provided
data.
solafune-assignment-11bands.ipynb — the same idea but on an 11-band image
I put together myself from Copernicus (more on that below). Having the extra
bands also let me compute NDVI.


I kept them separate because the two datasets don't line up cleanly — different
CRS, different resolution, different number of bands — so mixing them into one
notebook would've been messier than it's worth.

How it works

Loading & prep. Read the bands for each date, check the CRS / transform /
size actually match, and stack them into one multi-band image. If the two dates
sit on different grids, the second one gets resampled onto the first so every
pixel is comparing the same spot on the ground.

Change detection. Two methods:


the provided Euclidean-distance baseline (example_change_detection.py), applied
as-is, and
a PCA approach. First I match the after image to the before one band by band
(so the difference reflects real change and not just one date being brighter),
then run PCA on the per-pixel band difference and take the magnitude of the
leading components as the change signal. PCA pulls the real, coherent change into
the first few components and dumps the noise into the rest, which I drop.


Cleaning it up. I threshold the change magnitude at the 98th percentile —
P95 lit up way too much (it basically flags ~5% of pixels no matter what), so P98
keeps only the strong stuff. Then a bit of morphological open/close, and I throw
out any blob smaller than 50 pixels since real change is connected and the little
specks are just noise on the bright tailings.

Storing it. The binary change gets turned into polygons and written to a
SQLite table change_features with the columns the assessment asks for
(id, date_before, date_after, area_m2, confidence, geometry). I also export a
GeoPackage so it's easy to open in QGIS.

Visualizing. A 4-panel figure (before, after, change intensity, and the
detected change in red) with the AOI polygon drawn on top, plus an interactive
Folium map you can click around in.

The 11-band image (how I made it)

The assessment only gives 3 bands, but I wanted more spectral information to work
with, so I built an 11-band version:


Downloaded the same-date Sentinel-2 scenes from Copernicus.
Clipped to the AOI in QGIS.
Stacked the bands I needed.
Took the ROI in ENVI and saved it as a georeferenced ENVI file.


That file comes out in lat/lon, so in the notebook I reproject everything to UTM
35S before measuring areas (otherwise the polygon areas come out in degrees, which
is useless). There's also a faint diagonal seam in the after image where two
satellite passes meet — I detect and blend that out before running the change
detection.

Sentinel-2 change detection at an open-pit mine in Zambia (PCA-based pipeline with spatial database and visualization).
Results


3-band: ~3.39 km² of change, 115 separate regions (~1.28% of the scene)
11-band: ~2.18 km² of change, 144 polygons


Both land in the same place — the change clusters around the active pit and the
tailings/water areas, while the surrounding forest stays basically unchanged,
which is what you'd hope to see.
