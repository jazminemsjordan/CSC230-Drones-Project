# Drone Imagery Search System

> A web-based tool for cataloging, mapping, and browsing drone flight
> data for the Spatial Analysis Laboratory at CEEDS.

**CSC230 Final Project · Smith College · Spring 2025**

---

## The problem we set out to solve

The Spatial Analysis Laboratory at CEEDS runs drone flights for a wide
range of research purposes — capturing imagery and video that
researchers later analyze for everything from land-use studies to
ecological monitoring. The lab has been steadily accumulating flight
data over multiple semesters and field seasons, and the result is a
shared Drive folder full of flights organized by month and named in
inconsistent ways.

When our stakeholder, Kala'i Ellis, walked us through the data, the
core challenge became clear quickly: this isn't really a storage
problem, it's a **discovery** problem. There are professional flights
from DJI drones with proper `.MRK` flight log files, personal flights
with video and `.SRT` subtitle logs, and a long tail of flights that
exist only as a folder of geotagged JPGs with no log file at all. A
researcher who wants to find "all the flights from October over the
south meadow" has no way to query that without manually opening folder
after folder.

Our job was to build a system that ingests this messy, heterogeneous
data and surfaces it through a single clean interface — something a
researcher can open in their browser, narrow down by date, click on a
flight, and see exactly what was captured.

## What we built

`drones_v2.ipynb` is a Jupyter notebook that sets up a SQLite database,
ingests every flight folder it can find on the lab's Drive, extracts
geographic coordinates from whatever sources are available, and serves
a small Flask web app for searching and visualizing the results.

When you open the search page, you can filter flights by date range, by
type (professional or personal), and by a substring of the flight's
name. Results are color-coded by type and show how many photos and
videos each flight has, with a link to a detail page. The detail page
renders the flight's path on a Leaflet map and lays out every photo
and video in the folder as a thumbnail gallery — clicking a thumbnail
opens the full file, and videos play inline. There's also a combined
map view that draws every flight matching your current filter as a
polyline, so you can see at a glance where the lab has been flying
recently.

The piece we're proudest of is the **geotag fallback**. The proposal
called out that "many flights are documented only through raw videos
and photos" without flight logs, and we wanted to make sure those
flights weren't second-class citizens in the system. So when the
ingestion pipeline encounters a folder with no `.MRK` or `.SRT`, it
reads GPS EXIF tags from each JPG, sorts the photos by their capture
timestamps, and uses that ordered sequence as the flight path. The
result is that a folder full of geotagged photos shows up on the map
exactly the way a folder with a proper flight log does. This was the
single highest-leverage feature we added — without it, a real chunk of
the lab's existing data would have been invisible.

## How it works

The system has three layers, each doing one thing well.

**The ingestion layer** walks the year-folder of the lab's Drive (for
example `2025/`) and recurses into each month folder (`1-January`,
`2-February`, …) to find flight folders. It classifies each flight by
its folder name: professional flights look like `DJI_202410171432_001`
(the timestamp is right there in the name), and personal flights
follow the `mm.ddLocation` convention like `10.17Park` or `1.5Lake`.
For each flight, the pipeline inventories every file, classifies it as
photo / video / log / other, and then tries three coordinate-extraction
strategies in order: parse a `.MRK` if one exists, parse an `.SRT` if
one exists, otherwise read EXIF GPS tags from the JPGs. Whichever
strategy succeeds populates a coordinate table ordered by sequence so
the polyline draws correctly.

**The storage layer** is a SQLite database with five tables: one each
for professional and personal flights, one each for their coordinate
sequences, and a `flight_media` table that holds a per-file inventory
of every photo and video across all flights. We kept the split between
professional and personal because that's how the lab actually thinks
about the data, but `flight_media` unifies them when we need to count
files or serve them to the browser. Indexes on the date columns and
the foreign-key columns keep search queries fast as the dataset grows.

**The web layer** is a single-file Flask application. The same filter
parameters (`date_from`, `date_to`, `imagery_type`, `q`) flow through
both the search and the map endpoints, so the search form's "Open map
for these dates" link carries the user's current filter into the map
view. The seven routes are:

| Route | What it does |
|---|---|
| `/` | Search form |
| `/search` | JSON results, with photo / video counts per flight |
| `/flight/<type>/<name>` | Per-flight detail page with map + media gallery |
| `/api/flight/<type>/<name>` | JSON for the detail page |
| `/map` | Combined flight-path map, respects search filters |
| `/api/flights` | JSON path data for the map |
| `/media/<type>/<name>/<file>` | Serves a media file by DB lookup (no path traversal) |

## The schema

We went with a normalized relational design because the queries Kala'i
cares about — filter by date, retrieve all media for a flight, render
path coordinates in order — all map cleanly to joins and range scans.

```sql
CREATE TABLE professional (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT UNIQUE,
    date TEXT,
    folder_path TEXT
);

CREATE TABLE personal (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT UNIQUE,
    date TEXT,
    folder_path TEXT
);

CREATE TABLE professional_coordinates (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    professional_name TEXT,
    lat REAL,
    lon REAL,
    sequence INTEGER,
    FOREIGN KEY(professional_name) REFERENCES professional(name)
);

CREATE TABLE personal_coordinates (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    personal_name TEXT,
    lat REAL,
    lon REAL,
    sequence INTEGER,
    FOREIGN KEY(personal_name) REFERENCES personal(name)
);

CREATE TABLE flight_media (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    flight_type TEXT NOT NULL,        -- 'professional' or 'personal'
    flight_name TEXT NOT NULL,
    filename TEXT NOT NULL,
    file_path TEXT NOT NULL,
    media_kind TEXT NOT NULL,         -- 'photo' | 'video' | 'log' | 'other'
    lat REAL, lon REAL, taken_at TEXT,
    UNIQUE(flight_type, flight_name, filename)
);
```

The split between two flight tables (rather than one with a `type`
column) was a deliberate call: the two flight categories already have
different naming conventions, different metadata expectations, and
different file formats. Keeping them separate makes the ingestion
logic clearer and lets us evolve them independently if the lab's
conventions change. `flight_media` then provides the unified view we
need at query time.

## How to run it

The intended runtime is Google Colab, since that's where the lab's
Drive folder is most easily accessible.

Before running anything, the lab's `2025` folder needs to be available
in your My Drive. The simplest way is to add a shortcut: open
[the folder in Drive](https://drive.google.com/drive/folders/1NNjvn5rWcKxhFDbOd0rnf5oyIdLky-U3),
right-click the `2025` folder, choose **Organize → Add shortcut to
Drive**, and pick **My Drive** as the location. After that, the
folder will be accessible at `/content/drive/MyDrive/2025` once Colab
mounts Drive.

With the shortcut in place, open `drones_v2.ipynb` in Colab and run
the cells in order. Sections 1 through 6 install dependencies, set up
the schema, define the three coordinate extractors, build the Flask
app, and provide a database-inspection helper. They don't touch any
data, so they're safe to run blind.

Section 7 connects to the lab's actual Drive in five steps. The first
cell mounts Drive (you'll get the standard Colab OAuth prompt). The
second cell points at the shortcut and prints a preview of what it
found — number of months, number of flight folders per month, sample
folder names. This is a quick sanity check before the longer import.
The third cell runs the full ingestion, which can take a few minutes
the first time because EXIF is read from every JPG over the Drive
FUSE mount. The fourth cell prints a summary of what landed in the
database. The last cell starts the web server and prints a clickable
proxy URL.

## What to try once it's running

Open the search URL and run a query across the full year: `2025-01-01`
to `2025-12-31` with type set to All. You should see every ingested
flight, badged blue for professional and green for personal, with
photo and video counts beside each.

Click any flight name. You'll land on the detail page with the flight
path drawn on a small map and a thumbnail gallery beneath. Clicking a
thumbnail opens the full image at native resolution; videos play
inline.

From the search page, click "Open map for these dates." The combined
map opens with every flight in your current filter drawn as a
polyline, color-coded the same way as the search results. Clicking
any path opens a popup with the flight name and a direct link to its
detail page.

Try narrowing to a single day, or use the name-contains box to filter
by location keyword ("Park", "Lake") or by drone identifier prefix
("DJI_202410").

## What we learned

The biggest takeaway for us was how much of database work happens
*before* the database. The schema we landed on is straightforward and
the queries are simple, but most of our development time went into
the pipeline that turns inconsistent real-world folder layouts into
consistent rows. We rewrote our SRT parser after discovering that real
DJI subtitle files wrap their coordinates inside `<font>` tags rather
than starting lines with `[`. We added the EXIF fallback after
realizing how many lab folders had no flight log at all. Each of
these discoveries required actually trying the code on the lab's
data, not just on our assumptions about it. If we did this again,
we'd build the synthetic-data pipeline first and the real-data
pipeline second — but make sure we tested against real data sooner.

The other thing we'd flag for any team picking this up: keep the
ingestion idempotent. We use `INSERT OR IGNORE` on flights and
delete-then-reinsert on coordinates and media, which means re-running
the importer is always safe. That property turned out to be more
valuable than we expected during development.

## What's next

We deliberately scoped this iteration to make sure we delivered
something working by the demo. There are several natural extensions
that the next team or ongoing lab use could build on.

The most useful next feature would be **video GPS extraction**. DJI
MP4 files carry GPS in their metadata stream separately from the SRT
sidecar, and `pymediainfo` or `ffprobe` can pull it out. This would
handle the case where someone deletes the SRT but keeps the video.

A close second is **spatial filtering** — letting the user draw a box
on the map and retrieve all flights whose paths intersect it.
SQLite's R*Tree extension makes this straightforward: build an R-tree
index on the (min_lat, max_lat, min_lon, max_lon) bounding box of
each flight.

Beyond those, we'd suggest **automated re-ingestion** so the lab
doesn't have to manually trigger imports as new flights arrive,
**drone serial / model tracking** by parsing additional EXIF tags,
and basic **authentication** before the system is exposed to a wider
audience.

## Project structure

```
drones_v2.ipynb    # the entire system, organized into 7 sections
README.md          # this file
```

The notebook is intentionally self-contained. Everything — the
schema, the three extractors, the Flask app, the HTML templates, the
Drive connection — lives in one place so the stakeholder can step
through it cell by cell, see what each piece does, and modify it as
the lab's needs evolve.

## Acknowledgments

This project builds on work from a previous CSC230 student team —
particularly the date-range search and the map view, which we
extended significantly but didn't have to design from scratch. The
iterative, multi-semester nature of the project was one of the things
we appreciated most about it.

Huge thanks to **Kala'i Ellis** at the Spatial Analysis Laboratory
for being a generous and hands-on stakeholder, walking us through
the data multiple times and clarifying scope whenever we asked.

Thank you to our CSC230 instructor and TAs for the structure and
feedback that got us to this point.

---

**Team:** *[add your names here]*

**Stakeholder:** Kala'i Ellis · Spatial Analysis Laboratory · CEEDS · Smith College
