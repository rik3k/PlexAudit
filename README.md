# PlexAudit

A Python tool that cross-references your Plex Media Server database against your actual media files on disk, then generates a self-contained interactive HTML report. Instantly see what Plex has matched, what it scanned but couldn't identify, what files have gone missing, and — critically — what's sitting on your drives that Plex has never touched.

![image](https://i.imgur.com/WyjEfOm.png)  
---
<h4>Support the project</h4>
<sub>
If you found this tool useful, please consider supporting my BandCamp projects:<br>
<a href="https://ferropop.bandcamp.com">ferropop.bandcamp.com</a> [name your price!]
</sub>

---

## What it does

Plex only tells you what it *thinks* it has. PlexAudit tells you the truth.

It reads directly from Plex's SQLite database, walks your scan directories, and produces a four-way classification for every file:

| Status | Meaning |
|---|---|
| 🟢 **Matched** | File exists on disk and Plex has full metadata |
| 🟡 **Scanned, no metadata** | Plex found the file but couldn't match it to a show/movie |
| 🔴 **File missing** | Plex's database references a file that no longer exists on disk |
| 🔵 **Not in Plex** | File is on disk but Plex has never scanned it |

---

## Requirements

- Python 3.8+
- No external dependencies — uses only stdlib (`sqlite3`, `os`, `pathlib`, `json`, `argparse`)
- Plex Media Server (Windows, macOS, or Linux)

---

## Usage

```bash
# Auto-detects your Plex database on Windows
python plex_audit.py --scan "D:\Media" "E:\Media"

# Multiple scan directories
python plex_audit.py --scan "D:\Movies" "E:\TV Shows" "F:\Music"

# Specify database path manually
python plex_audit.py --db "C:\path\to\com.plexapp.plugins.library.db" --scan "D:\Media"

# Custom output filename
python plex_audit.py --scan "D:\Media" --out my_report.html

# Debug mode — prints sample paths to diagnose matching issues
python plex_audit.py --scan "D:\Media" --debug
```

The report is a single self-contained `.html` file. Open it in any browser — no server required.

### Auto-detected database locations (Windows)

```
C:\Users\{user}\AppData\Local\Plex Media Server\Plug-in Support\Databases\com.plexapp.plugins.library.db
C:\Users\{user}\AppData\Roaming\Plex Media Server\Plug-in Support\Databases\com.plexapp.plugins.library.db
C:\ProgramData\Plex Media Server\Plug-in Support\Databases\com.plexapp.plugins.library.db
```

---

## Report features

### Views & filtering
- **TV Shows / Movies / Music / All** tabs — each shows context-appropriate columns
- Switching tabs automatically enables only the relevant file extensions (video for TV/Movies, audio for Music)
- Filter by **library**, **search** (filename, show name, episode title, folder)
- **Status cards** at the top are clickable filters (show only missing files, only unmatched, etc.)

### Extension filters
- Every file extension found in your scan is shown as a toggleable pill, colour-coded by category (Video / Audio / Image / Other)
- Click a **category heading** to toggle the entire category on/off — if any are on, clicking clears them all (clearance behaviour)
- "all on" / "all off" quick links

### Columns
Columns are ordered for easy visual matching:

- **TV Shows:** Folder → Filename → Show → S → E → Episode Title → Library
- **Movies:** Folder → Filename → Plex Title → Library
- **Music:** Folder → Artist—Album → Filename → # → Track Title → Library
- **All:** Type badge → Folder → Filename → Show/Movie → S → E/# → Episode/Track → Library

### Column controls
- **Resize** any column by dragging its right edge — content clips cleanly at any width, even a single pixel
- **Reorder** columns by dragging headers — each view remembers its own order independently
- **Sort** by clicking any column header (resizing does not accidentally trigger a sort)

### Quality columns (toggle)
Hit **"Quality columns: OFF"** in the filter bar to append four columns to any view:

| Column | Source |
|---|---|
| Resolution | `media_items.width × height` with standard label (4K, 1080p, 720p…) |
| Bitrate | `media_items.bitrate` (converted from bps to Mbps) |
| File size | `media_parts.size` |
| Duration | `media_items.duration` |

Useful for identifying duplicate files and deciding which version to keep. A lower-resolution file with a higher bitrate is almost always the worse encode.

### Right-click on any filename
- **Copy full path** — the complete path to the file
- **Copy folder path** — just the directory
- **Copy "Open in Explorer" command** — copies `explorer /select,"D:\path\to\file.mkv"` ready to paste into **Win+R** or the Start search box, which opens Explorer with the file highlighted

### Performance
- **Chunked rendering** — first 200 rows appear immediately; remaining rows are appended in 200-row idle-callback batches so the UI stays responsive throughout
- **Loading overlay** with a progress bar on first open, showing each stage (parsing, filtering, rendering)
- **Filter status bar** — a 3px animated bar under the header pulses while filtering is in progress and flashes green on completion

---

## How the database is read

PlexAudit reads the Plex SQLite database in **read-only mode** — it never writes to or modifies your Plex database.

The key tables used:

- `media_parts` — file paths on disk
- `media_items` — resolution, bitrate, duration
- `metadata_items` — titles, episode/season/track numbers, hierarchy (show→season→episode, artist→album→track)
- `library_sections` — which Plex library each item belongs to

Files with metadata but `library_section_id = NULL` are extras and featurettes attached to a movie — these are labelled "Extras/Featurettes" rather than showing a blank library name.

---

## Interpreting results

**"Scanned, no metadata" (yellow)** means Plex found the file during a library scan but its metadata agent (TheTVDB, TMDb, MusicBrainz) couldn't identify it. Common causes:
- Non-standard filename format
- Obscure or very old content with incomplete online metadata
- Wrong library type (e.g. a TV show file in a Movies library)

**"Not in Plex" (blue)** means the file exists on disk but is not referenced anywhere in Plex's database. Common causes:
- The folder was never added as a Plex library source
- The file was added after the last library scan
- Image files (`.jpg`, `.png`) and extras that Plex intentionally ignores

Filter to "Not in Plex" and sort by Folder — this immediately groups unscanned files by directory, making it easy to spot entire folders Plex has missed.

---

## Debug mode

If matching results look wrong (e.g. everything shows as "Not in Plex"), run with `--debug`:

```
python plex_audit.py --scan "D:\Media" --debug
```

This prints 5 sample paths from both the Plex DB and the disk scan, normalized side by side, so you can immediately see if there's a path prefix mismatch (common when Plex was previously running inside Docker with different mount paths).

---

## License

MIT
