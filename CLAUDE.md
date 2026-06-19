# CLAUDE.md

Guidance for working in the **minerva-author** repository.

## What this is

Minerva Author is a desktop tool for building interactive "stories" (guided
narratives with waypoints, channel groups, and annotations) from large
multi-channel microscopy images (OME-TIFF, plain TIFF, SVS). It is the authoring
half of the Minerva ecosystem:

- **minerva-author** (this repo) — Python **Flask** backend + a *bundled* React UI.
- **minerva-author-ui** — the React frontend source (separate repo,
  `labsyspharm/minerva-author-ui`). Only its built bundle ships here under `static/`.
- **minerva-browser** — the JS viewer that renders the exported story
  (separate repo). Loaded by CDN in exported `index.html` (see version pin below).
- **minerva-story** — the GitHub Pages story template (`minerva-story/index.html`,
  vendored here; `.gitmodules` is empty — it is NOT a live submodule).

The user runs the app, imports an image, configures channel groups / rendering
settings / waypoints in the browser UI, previews live, and "saves" to produce a
self-contained Minerva Story (an `exhibit.json` config + a rendered JPEG/PNG image
pyramid + a `.story.json` save file).

The git user (Jeremiah Wala) is a maintainer; we are here to **fix bugs and make
improvements**.

## Running & environment

- Conda env (the active one is `minerva-author-new`):
  ```
  conda env create -f requirements.yml   # creates env "minerva-author-new"
  conda activate minerva-author-new
  python src/app.py                       # serves on localhost:2020, opens browser
  ```
- Python **3.13**. `python src/app.py [--dev]`. Without `--dev` it serves via
  **waitress** (`threads=num_workers`, `channel_timeout=15`); with `--dev` it uses
  Flask's dev server.
- MacOS needs `brew install openslide`; Windows needs OpenSlide DLLs copied into
  `src/`. SVS support comes from `openslide-python`.
- `distutils` is still imported in several modules (`file_util`, `DistutilsFileError`).
  It works here only because **setuptools** vendors it (stdlib removed it in 3.12+).
  Don't assume it's safe forever — a worthwhile cleanup target.

## Tests

- The real test suite was **removed** (git: "remove old tests" / "remove all code").
  The only remaining test is `src/test_vega.py` (vega chart dict generation).
- Run with `pytest` from repo root (README still claims a full suite — outdated).
- `testimages/exhibit*_in.json` / `exhibit*_out.json` and `testimages/*.ome.tif`,
  `markers.csv`, `CMU-1-Small-Region.svs` are leftover fixtures useful for manual /
  reintroduced testing. `testcharts/` feeds `test_vega.py`.

## Packaging / CI

- `package_mac.sh` and `package_win.bat` run **PyInstaller** (`-F`, one-file),
  bundling `static/` and `minerva-story/` and many hidden imports
  (imagecodecs, altair, ome_types, xmlschema, xsdata). The numerous
  `# Needed for pyinstaller` / `--hidden-import` bits exist to satisfy the frozen build.
- `resource_path()` and `get_story_dir()` branch on `sys._MEIPASS` to locate bundled
  data in both dev and frozen modes.
- `.github/workflows/pyinstaller.yml` builds on tag `v*` / PRs (currently macOS only;
  Windows steps exist but the matrix is `macos-26`). `appveyor.yml` is legacy Windows CI.

## Architecture / data flow

### `src/app.py` (~2050 lines) — the core. Everything important lives here.

**Global state** `G` (a dict from `reset_globals()`): `logger`, `import_pool`
(single-thread `ThreadPoolExecutor` for background TIFF→OME conversion),
`preview_cache` (per-session dict of lazily-rendered tiles), `image_openers`,
`mask_openers` (path→`Opener` caches), `save_progress` / `save_progress_max`
(progress bars keyed by session). Opener caches are guarded by module-level
`tiff_lock` / `mask_lock` (`multiprocessing.Lock`).

**`Opener`** — wraps an image file. For OME-TIFF it opens via `tifffile`, exposes the
pyramid as a **zarr** group, detects OME version (5 vs 6 — affects `is_ome` handling),
detects RGB(A) (3-channel uint8 or 1-channel uint8 → treated as already-rendered),
and reads tiles. Key methods: `get_shape()` → `(num_channels, num_levels, shape_x,
shape_y)`; `read_tiles()`; `get_tile()` (PNG path); `return_tile()` (composite RGB
JPEG path, applies per-channel color/low/high + gamma); `save_tile()`;
`generate_mask_tiles()` / `save_mask_tiles()` (uint32 label masks → RGBA PNG).

**`ZarrWrapper`** — translates a logical `[level, x, y, z, channel, t]` index into the
zarr group's native dimension order. Handles non-power-of-2 / missing pyramid levels
via `sampling_ratio`, `to_group_level`, `from_group_level`. `MissingLevel` is raised
when a requested level isn't backed by a real array. This dimension-order logic is
subtle and a common source of bugs.

**Rendering convention:** channel "Low"/"High" sliders are stored as **fractions 0–1**
and multiplied by the dtype max at render time. `gamma_correct*` apply gamma. Composite
math is in `render_jpg.composite_channel`.

**HTTP endpoints** (all `@cross_origin @nocache`):
- `GET /` — serve the bundled UI (`static/index.html`).
- `POST /api/import` — open image (or load a `.dat`/`.json`/`.story.json` save),
  read OME-XML pixel size + channel marker names (`yield_labels`: CSV → OME-XML →
  numeric fallback), optionally auto-generate channel groups via `story.auto_minerva`
  (GMM auto-thresholding). Returns channels, dimensions, session id, etc. Lots of
  branching for autosave-vs-save and "ASK ERR" sentinel error strings the UI parses.
- `POST /api/import/groups` — import just `groups`/`defaults` from another save.
- `GET /api/u16/<key>/<channel>/<level>_<x>_<y>.png` — single 16-bit channel tile.
- `GET /api/u32/<key>/<level>_<x>_<y>.png` — 32-bit mask tile.
- `GET /api/validate/u32/<key>` — kicks off background TIFF→OME-TIFF mask conversion
  (`convert_mask` → `pyramid_assemble.main`) and reports readiness.
- `GET /api/mask_subsets/<key>` — parse a mask-state CSV (`CellID` + `State`/`State1..`)
  into cell-id subsets.
- `POST /api/preview/<session>` — build `preview_cache[session]`: a dict mapping output
  filenames (`exhibit.json`, `index.html`, `bundle.js`, every tile path, vega CSVs) to
  lazy render thunks. **Does not render anything yet.**
- `GET /story/<session>/<path>` — serve a file from `preview_cache` by invoking its
  thunk on demand (live preview). 404 page tells user to restart.
- `POST /api/save/<session>` — write the `.story.json` save file (handles autosave
  bookkeeping + timestamps via `copy_saved_states` / `is_new_autosave`), copy vis CSVs.
- `POST /api/render/<session>` — the real export: `create_story_base`, write
  `exhibit.json`, render all channel-group JPEG tiles (`render_color_tiles`), render a
  thumbnail (`thumbnail.py`), render all uint32 mask PNG tiles (`render_u32_tiles`).
- `GET /api/render/<session>/progress` — progress bar polling.
- `GET /api/filebrowser` — local FS browser for the UI (handles Windows drive letters).

**`exhibit.json` shape** is produced by `make_exhibit_config` + the `make_*` helpers
(`make_stories`/`make_waypoints`, `make_groups`, `make_channels`, `make_rows`,
`make_subgroups`, `make_mask_yaml`). These helpers are **shared/imported by `render.py`
and `exhibit.py`**, so changing their output affects both the live app and the CLI tools.

### Supporting modules
- `story.py` — `auto_minerva`: GMM-based per-channel auto-threshold (`auto_threshold`)
  + heuristic channel grouping (`to_group_starts` looks for DNA/Hoechst keywords and
  even group sizes). Run in a thread pool.
- `render_jpg.py` — `render_color_tiles` (writes the JPEG pyramid, caches via per-group
  `config.json` to skip unchanged tiles), `composite_channel`, `_calculate_total_tiles`.
- `render_png.py` — `render_tile` (live PNG tiles with sanity checks → `MissingTilePNG`),
  `render_u32_tiles`, mask colorization (`hsv2rgba`/`spike`/`colorize_*`).
- `thumbnail.py` — blends top-level group tiles into a single preview JPEG.
- `pyramid_assemble.py` — standalone-capable: assemble multiple single-channel TIFFs
  into a tiled pyramidal OME-TIFF; also used in-process to convert masks (`--mask` =
  nearest-neighbor downsampling). Writes OME-XML by hand + patches it into the file.
- `storyexport.py` — output path/dir/dedup helpers (`get_story_folders`,
  `label_to_dir`, `deduplicate*`, `mask_path_from_index`, etc.) and `create_story_base`.
  `get_story_folders` returns `(out_dir, exhibit.json, save.json, log)`; prefers
  `<title>.json`, falls back to `<title>.story.json` (post-1.6.0 naming).
- `create_vega.py` — Altair → Vega-Lite specs + CSV munging for the 3 chart types
  (scatterplot, barchart, matrix) embeddable in waypoints.
- `exhibit.py` / `render.py` — **CLI** re-export tools (`python src/exhibit.py ...`,
  `python src/render.py ...`) that turn an OME-TIFF + author `.json` into a hosted
  exhibit (config-only, or config + rendered pyramid + thumbnail respectively). They
  import the `make_*` helpers from `app.py`.
- `convert_omero_channels.py` — CLI to convert exported OMERO channel settings into a
  Minerva Author save file.
- `fit_render_settings.py` — CLI wrapper around `auto_minerva` to dump auto settings.

## Conventions & gotchas (read before editing)

- **`get_shape()` order is `(channels, levels, shape_x=width, shape_y=height)`** — but
  it's unpacked many different ways across files. Double-check width/height when you
  touch tiling math.
- **Low/High are 0–1 fractions**, scaled by dtype max at render time, in *both* the
  live `Opener.return_tile` and `render.to_one_tile`. Keep these two in sync.
- **Two render paths** must stay consistent: live preview (`render_png`/`Opener.get_tile`
  /`return_tile`) and exported pyramid (`render_jpg.render_color_tiles`). Bugs often come
  from fixing one and not the other.
- **Caching by config equality**: both JPEG (`render_jpg`, per-group `config.json`) and
  mask (`render_png`, `is_up_to_date`) skip re-rendering when settings + output file are
  unchanged. If output looks stale after a change, this is why — delete the output dir.
- **Error strings are an API contract.** The UI parses sentinel prefixes like
  `IMAGE ASK ERR`, `OUT ASK ERR`, `AUTO ASK ERR`, `AUTO MISSING ERR`. Don't reword
  these without checking minerva-author-ui.
- `custom_log_warning` (app.py) **monkeypatches `tifffile.log_warning` to raise** — so
  any tifffile warning becomes an exception. Intentional, but surprising.
- RGB(A) images (3ch or 1ch uint8) are treated as already-rendered: no auto-grouping,
  gamma-only adjustment.
- Save files: legacy `.dat` (pickle) and `.json`/`.story.json`. `load_saved_file`
  handles both. Autosave is nested under `"autosave"` with its own timestamp.
- `minerva-browser` is pinned to **3.20.0** in the exported HTML (hardcoded CDN URL in
  `render.py` and `exhibit.py` `json_to_html`, and in `minerva-story/index.html`).
  Bump all together.
- Known stray/cruft: `src/T3-HE.log` (untracked log artifact from a `FileHandler` run —
  safe to ignore/delete); duplicate `"defaults"` key in the `/api/import` JSON response;
  `crender.so`/`crender.dll` shipped binaries.

## Where to start for common tasks

- **Tile rendering / color / threshold bug** → `Opener.return_tile`/`get_tile` +
  `render_jpg.composite_channel` (+ mirror in `render.to_one_tile`).
- **Pyramid level / dimension-order bug** → `ZarrWrapper`, `Opener.get_shape`/
  `get_level_tiles`.
- **Import / channel-name / auto-group bug** → `api_import`, `yield_labels`, `story.py`.
- **Export / exhibit.json content** → `make_exhibit_config` + `make_*` helpers (affects
  app, `render.py`, `exhibit.py`).
- **Masks** → `convert_mask`/`pyramid_assemble`, `generate_mask_tiles`,
  `render_u32_tiles`, `load_mask_state_subsets`.
- **UI-visible behavior** lives partly in the separate **minerva-author-ui** repo; the
  bundle in `static/` is a build artifact, not editable source here.
