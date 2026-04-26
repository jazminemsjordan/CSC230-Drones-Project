# Drones Project — v2 Changes

This update extends the v1 notebook with the features the proposal called
out as core stakeholder needs.

## What's new

### 1. EXIF / geotag fallback for JPGs
The biggest gap in v1: folders with photos but no `.MRK` or `.SRT` log
file produced no flight path. The proposal explicitly flagged this:

> *Many flights are documented only through raw videos and photos,
> meaning that critical location data is not immediately accessible
> and must be extracted from image and video metadata (such as
> geotags) rather than from dedicated flight path files.*

`extract_gps_from_jpg()` now reads GPS EXIF tags directly. When a folder
has no flight log, the ingestion pipeline orders the photos by their
EXIF capture timestamp and treats that ordered sequence as the flight
path. Verified end-to-end with synthetic data (5 geotagged JPGs in a
folder produce a 5-point polyline on the map).

### 2. Per-flight detail page with media gallery
New routes `/flight/<type>/<name>` and `/api/flight/<type>/<name>` show
flight metadata, the path on a small map, and a thumbnail gallery of
every photo/video in the folder. Search results now link directly to
the detail page. This addresses the proposal's "*click on individual
flights to retrieve associated imagery and video*."

### 3. Unified, date-filterable map
`/api/flights` now returns *both* professional and personal flights
(it was professional-only in v1 — a real bug since personal flights
were ingested into the database but never surfaced on the map). The
endpoint also accepts `date_from`, `date_to`, `imagery_type`, and `q`
query parameters. The search form's "Open map for these dates" link
forwards the user's filter to the map.

### 4. New `flight_media` table
A unified per-file inventory with these columns:

```
id, flight_type, flight_name, filename, file_path,
media_kind, lat, lon, taken_at
```

This is what powers photo/video counts in search results and the
gallery on the detail page. It also gives you a query-friendly view
of every file across both flight types.

### 5. Drone-identifier substring filter
The search form has a "Name contains" field. The `/search` and
`/api/flights` endpoints both accept `q=<substring>`. Useful for
quickly pulling up everything from a specific drone or location prefix.

### 6. SRT parser bug fix
Real DJI SRT files wrap the latitude/longitude tokens inside `<font>`
tags so the line doesn't start with `[`. The v1 parser required
`line.startswith("[")` and would silently extract zero coordinates from
real files. The new parser uses regex to find `[latitude: …]` and
`[longitude: …]` anywhere in the line.

### 7. Local test mode
Section 7 of the notebook builds a synthetic folder tree under
`/tmp/drones_synth/2024/` that exercises every code path:

| Folder | Tests |
|---|---|
| `10/DJI_202410171432_001` | Professional flight + .MRK file |
| `10/10.17Park` | Personal flight + .SRT video log |
| `10/10.18Trail` | Personal flight, **EXIF-only** (the new fallback) |

You can develop and demo without remounting Google Drive every time.

## Schema changes

```sql
-- NEW table:
CREATE TABLE flight_media (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    flight_type TEXT NOT NULL,        -- 'professional' or 'personal'
    flight_name TEXT NOT NULL,        -- references professional.name or personal.name
    filename TEXT NOT NULL,
    file_path TEXT NOT NULL,
    media_kind TEXT NOT NULL,         -- 'photo' | 'video' | 'log' | 'other'
    lat REAL,                          -- EXIF GPS lat (photos only)
    lon REAL,                          -- EXIF GPS lon (photos only)
    taken_at TEXT,                     -- EXIF capture time (ISO format)
    UNIQUE(flight_type, flight_name, filename)
);

-- NEW indexes for search performance:
CREATE INDEX idx_pro_date ON professional(date);
CREATE INDEX idx_per_date ON personal(date);
CREATE INDEX idx_pro_coords_name ON professional_coordinates(professional_name);
CREATE INDEX idx_per_coords_name ON personal_coordinates(personal_name);
CREATE INDEX idx_media_flight ON flight_media(flight_type, flight_name);
```

The `professional`, `personal`, and `*_coordinates` tables are unchanged
from v1, so old code that touches them keeps working.

## How to run

### In Colab (against real Drive data)
1. Open `drones_v2.ipynb` in Colab.
2. Run sections 1–4 (the install, schema setup, extractors, and Flask app).
3. In section 5, set your `folder_path` and call `mount_drive_if_needed()`
   then `import_folder_data(folder_path)`.
4. Call `run_server()`. Click the "Open search" link.

### Locally (no Drive needed)
1. `pip install flask pandas pillow piexif ipython`
2. Open the notebook in Jupyter.
3. Run sections 1–4 to define everything.
4. Skip to section 7. Run `build_synthetic_data()`, then the import cell,
   then `run_server()`.
5. Open the URL it prints.

## What to try next (recommended next steps)

These are the obvious next iterations, in rough order of payoff:

1. **Video GPS extraction** — DJI MP4s carry GPS in their metadata
   stream, separately from the SRT sidecar. `pymediainfo` or `ffprobe`
   can pull it. This handles the case where someone deletes the SRT but
   keeps the video.
2. **Spatial filter on search** — let users draw a box on the map and
   get all flights whose path intersects. SQLite's R*Tree extension
   makes this easy: build an `rtree` index on `(min_lat, max_lat,
   min_lon, max_lon)` per flight.
3. **Automated re-ingestion** — the proposal mentions "*automated
   ingestion as new flight data is added*". Right now `import_folder_data`
   is manual. A small daemon (or a Colab cell scheduled with a button)
   that runs nightly and only inserts new folders would close this loop.
4. **Drone serial / model tracking** — the EXIF data on DJI photos
   includes the drone's model and serial number. Parsing that out
   would let users filter by physical drone, which Kala'i mentioned as
   useful in the requirements list.
5. **Authentication** — currently anyone with the URL can browse
   everything. Even basic HTTP auth in front of Flask would be a
   reasonable step before the lab uses this on real data.

## Project timeline check-in

Per the timeline in the proposal:

- **4/26–5/2: implementation and debugging** — this update lands here.
- **5/3–5/8: finishing touches, demo by 5/7** — recommend prioritizing
  (1) video GPS and (2) spatial filter from the list above for the
  demo, since both are visible wins. (3)–(5) can be flagged as future
  work in the demo slides.
