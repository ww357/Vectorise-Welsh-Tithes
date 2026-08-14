# Welsh Tithe Map Downloader

Downloads high-resolution tithe map scans (1830s–1850s) from the
[National Library of Wales](https://places.library.wales/), georeferences them,
and produces a point file of every apportionment parcel on each map — field
number, land use, occupier, landowner, acreage, and tithe rent, with both
map-pixel and real-world coordinates.

## Setup (one-off)

1. Install [Miniconda](https://docs.conda.io/en/latest/miniconda.html) if you
   don't already have conda.
2. Download or `git clone` this repository, then open a terminal
   (**Anaconda Prompt** on Windows) in the repository folder.
3. Create and activate the environment:

   ```bash
   conda env create -f environment.yml
   conda activate tithe-maps
   ```

That's it — you can now run the commands below. In every later session, just
run `conda activate tithe-maps` first.

The optional `--warp` step (QGIS-ready GeoTIFFs) additionally needs
[QGIS](https://qgis.org/) installed — the script finds and uses the GDAL tools
QGIS ships with. Everything else works without it.

## Being polite to the NLW servers

The script rate-limits itself (one request at a time, with delays between
requests) — please keep it that way. If you plan to download a large share of
the collection, consider letting the National Library of Wales know first.

## `tithe_downloader.py`

`tithe_downloader.py` is the entire toolkit: a single Python script with subcommands that builds a local dataset of georeferenced Welsh tithe maps from the National Library of Wales.

Everything it produces is tracked in a SQLite database (`tithe_maps/tithe_maps.db`), making every command resumable and ensuring nothing is downloaded twice.

### Commands

```bash
python tithe_downloader.py <command>
```

| Command | What it does |
|---------|--------------|
| `discover` | One-off: catalogues every NLW tithe map (~1,074 maps, ~876,000 parcel records) into the database. No images are downloaded—only names, PIDs, parcel counts, etc. Takes a few hours; resumable if interrupted. |
| `list` | Browse or search the catalogue, e.g. `list --county Radnor` or `list --search llan`. Search is punctuation-insensitive (`Llanbedr` matches `Llan-Bedr`). `--sort parcels` or `--sort area` ranks smallest-first (add `--desc` for largest), `--limit N` caps the rows, and `--urls` appends each map's online viewer link. |
| `coverage` | Compute each map's covered ground area (hectares) from its cached parcel points, so `list --sort area` works. Runs automatically the first time you sort by area; `--force` recomputes. |
| `metadata` | Fetches titles, dates, and image dimensions from IIIF manifests. Optional, as `download` retrieves this automatically for targeted maps. |
| `download` | Main workhorse. Downloads and stitches the map scan, fetches parcel points, and georeferences each map. Use `--pids "Llangynllo,4634773"` or `--from-file targets.txt` to specify maps. Add `--warp` to produce QGIS-ready GeoTIFFs, `--scale` to reduce resolution (see below). |
| `parcels` | Fetch parcel-point files only — no map images. With no filter it fetches points for **every** catalogued map (a few hours, resumable); or use `--pid` / `--county`. `--refetch` ignores the cache. |
| `georeference` | Re-run georeferencing for maps already on disk. Supports `--refetch` and `--warp`. |
| `geopackage` | Bundle every downloaded parcel-point file into `tithe_maps/parcels.gpkg` for QGIS (one layer per county). Add `--split` for one file per county, or `--county X` for a subset. Includes computed `area_hectares` and `rent_decimal_pounds` columns alongside the original imperial units. |
| `export-toolkit` | Export sheet(s) for the Cadastral Map Vectorisation Toolkit: a north-up **EPSG:27700** GeoTIFF at 0.5 m/px plus a matching seed-point GeoPackage. See below. |
| `quality` | Mark maps as `high`, `low`, or `excluded`. `download` skips `low` and `excluded` maps by default. |
| `status` / `export` | Show download progress or export the database as CSV. |

### Output per map

Each map is stored in:

```text
tithe_maps/downloads/{County}/{Parish}_{pid}/
```

(or alongside pre-registered images)

Files produced include:

- `.jpg` — stitched map scan (full resolution unless `--scale` was used).
- `.parcels.geojson` — parcel points with:
  - WGS84 coordinates for GIS.
  - `pixel_x` / `pixel_y` for SAM prompts (always matched to the resolution
    of the downloaded image).
  - Field number, farm/field name, land use, occupier, landowner.
  - Acreage (acres / roods / perches) and tithe rent (£ / s / d).
- `_warped.tif` (with `--warp`) — north-up georeferenced GeoTIFF with pyramids, ready for QGIS.
- `.vrt`, `.gcps.vrt`, `.jgw`/`.tfw`, `.prj` — georeferencing sidecar files and warp inputs.

### Resolution / `--scale`

By default `download` fetches maps at native full resolution — typically around
20,000 × 16,000 px and 100–400 MB per map. If you don't need that much detail
(or want a quick preview sweep), the `--scale` flag downloads at reduced
resolution; the NLW server does the resampling, so bandwidth shrinks too:

| Flag | Resolution | Approx. size vs full |
|------|------------|----------------------|
| `--scale 1` (default) | native | 100 % |
| `--scale 2` | half | ~25 % |
| `--scale 4` | quarter | ~6 % |
| `--scale 8` | eighth | ~1.5 % |

Only powers of two are allowed (tiles divide evenly, so no seam artefacts).
The scale is recorded in the database per map, and the `pixel_x`/`pixel_y`
values in that map's `.parcels.geojson` and the georeferencing control points
are automatically adjusted to match the downloaded image. If you re-download a
map at a *different* scale, delete its old image first and run
`parcels --refetch --pid <pid>` afterwards so the point file is regenerated.

### Feeding sheets into the vectorisation toolkit

`export-toolkit` hands a downloaded sheet to the Cadastral Map Vectorisation
Toolkit, converting from this project's WGS84 outputs to the EPSG:27700 /
0.5 m-per-pixel convention that toolkit expects:

```bash
python tithe_downloader.py export-toolkit --pid 4634773 \
    --toolkit-dir "C:\path\to\Claude Toolkit Development"
```

Writes two files into the toolkit, named after the parish (override with
`--sheet-name`):

| File | Contents |
|------|----------|
| `data/raw/<SHEET>/<SHEET>.tif` | North-up GeoTIFF, EPSG:27700, 0.5 m/px, tiled + DEFLATE, with overview pyramids |
| `data/parcel_points/<SHEET>_points.gpkg` | Single point layer, EPSG:27700, one seed per parcel, with a `rowid` column for the toolkit's attribute join |

The toolkit picks the points file up automatically from the sheet name — no
`config.yaml` change is needed. Then run its pipeline as normal, starting with
`python steps/01_patchify/patchify.py --sheet <SHEET> --mask`.

**Why the seed points are not simply reprojected.** The polynomial
georeferencing fit has 6–56 m of residual scatter. Reprojecting each parcel's
WGS84 position straight to EPSG:27700 would misplace watershed seeds by a median
of ~27 m and up to ~187 m (55–374 px at 0.5 m/px) — routinely landing a seed in
the *wrong parcel*. Instead each point is placed by pushing its **pixel**
position (NLW's own record of where the parcel sits on the scan) through the
*same* GCP transform `gdalwarp` uses for the image, via `gdaltransform`. Seeds
therefore land exactly where that pixel landed in the warped raster and the
residual cancels out. Absolute positional accuracy is still limited by the
georeferencing, but image and seeds are internally consistent, which is what the
watershed step needs.

Requires QGIS or OSGeo4W (for `gdalwarp` / `gdaltransform` / `gdaladdo`).

### Typical session

```bash
python tithe_downloader.py discover                      # only run once (takes a few hours)
python tithe_downloader.py list --search "parish name"   # find target maps
python tithe_downloader.py download --pids "..." --warp  # download and georeference
python tithe_downloader.py status                        # check progress
python tithe_downloader.py geopackage                    # bundle all points for QGIS
```

To grab the parcel points for the whole of Wales without downloading any
map images (a few hundred MB total):

```bash
python tithe_downloader.py parcels        # all catalogued maps, resumable
python tithe_downloader.py geopackage
```

### Repository layout

- `tithe_downloader.py` — the entire toolkit.
- `environment.yml` — conda environment definition (see Setup above).
- `tithe_maps/` — created on first run: the SQLite database, downloads, logs,
  and generated GeoPackages. Not committed to git (see `.gitignore`).
  - `tithe_maps/land_use_terms.csv` — reference glossary of every land-use
    term recorded in the apportionments (count, share, style class, meaning).
- `legacy/` — old v2 script, previous database, and Excel tracking sheet.
  Superseded and retained only for reference (also not committed).