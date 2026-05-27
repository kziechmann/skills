---
name: media-organizer
description: >
  Organizes, archives, and indexes photos and videos into a searchable
  folder structure using EXIF/metadata extraction, consistent naming
  conventions, duplicate detection, and manifest files. Trigger when the
  user asks to "organize my photos", "archive photos", "sort my files",
  "rename photos", "find duplicates", "tag photos", "create a folder
  structure", "import from SD card", "back up photos", "search my photos",
  or "set up a photo archive". Also trigger for /organize-media,
  /archive-photos, /index-photos.
allowed-tools:
  - Bash(exiftool *)
  - Bash(ffprobe *)
  - Bash(ffmpeg *)
  - Bash(find *)
  - Bash(jdupes *)
  - Bash(jq *)
  - Bash(python3 *)
  - Bash(sha256sum *)
  - Bash(md5sum *)
---

# Media Organizer Skill

Build a clean, searchable archive of photos and videos organized by date,
event, category, and location. The goal is a structure you can navigate
visually AND query programmatically years from now — no locked-in app
required.

## Guiding Principles

- **Originals are sacred**: Never delete or overwrite source files. Always
  copy, then verify, then optionally delete the source.
- **Filesystem is the database**: Folder structure and file names encode
  enough context to browse without any app. Metadata files augment, not
  replace, the folder structure.
- **EXIF is ground truth**: Use capture time from EXIF/metadata, not
  file-system timestamps, which change on copy.
- **Idempotent operations**: Running the organizer twice on the same input
  produces the same result. Use checksums to skip already-imported files.
- **Category-first, date-second**: Browsing by event/trip is more natural
  than browsing by year. The folder structure reflects this.
- **Everything searchable**: Every import generates a manifest JSON so you
  can `grep`, `jq`, or query it without opening any GUI.

---

## Tool Stack

| Task                          | Tool / Library                        |
|-------------------------------|---------------------------------------|
| Photo EXIF extraction         | `exiftool` (CLI) — primary            |
| Video metadata                | `ffprobe` (part of FFmpeg)            |
| File checksums / dedup        | `sha256sum` / `jdupes` / `hashlib`    |
| Perceptual image dedup        | `imgdupes` (Python)                   |
| Perceptual video dedup        | `videohash` (Python)                  |
| Batch file operations         | Python `shutil`, `pathlib`            |
| Sidecar metadata              | XMP files via `exiftool`              |
| Thumbnail generation          | `ffmpeg` (video), `exiftool` (photo)  |
| Manifest / search index       | JSON + `jq` for querying              |
| GPS → location name           | `geopy` + OpenStreetMap Nominatim     |

### Check available tools

```bash
# Verify what's installed
exiftool -ver 2>/dev/null && echo "exiftool OK" || echo "install: brew install exiftool"
ffprobe -version 2>/dev/null | head -1 || echo "install: brew install ffmpeg"
jdupes --version 2>/dev/null | head -1 || echo "install: brew install jdupes"
python3 -c "import PIL; print('Pillow OK')" 2>/dev/null || pip install Pillow
python3 -c "import imgdupes; print('imgdupes OK')" 2>/dev/null || pip install imgdupes
python3 -c "import geopy; print('geopy OK')" 2>/dev/null || pip install geopy
```

---

## Quick Commands: exiftool One-Liners

For fast, single-command organization without Python scripts.

### Organize by date into YYYY/YYYY-MM folders (copy, preserve originals)

```bash
# Photos: copy into ~/Photos/Archive/YYYY/YYYY-MM/
exiftool -r \
  -ext jpg -ext jpeg -ext heic -ext png -ext cr3 -ext cr2 -ext arw -ext nef -ext dng \
  -o "$HOME/Photos/Archive" \
  '-Directory<DateTimeOriginal' -d '%Y/%Y-%m' \
  /path/to/source

# Videos: same pattern using CreateDate
exiftool -r \
  -ext mp4 -ext mov -ext mts -ext m2ts \
  -o "$HOME/Photos/Archive" \
  '-Directory<CreateDate' -d '%Y/%Y-%m' \
  /path/to/source
```

### Rename files to timestamp + auto-counter (safe, collision-free)

```bash
# Rename to 20240315_092045_001.jpg format
exiftool -r \
  '-FileName<DateTimeOriginal' \
  -d '%Y%m%d_%H%M%S%%-c.%%e' \
  /path/to/photos

# Rename with event prefix
exiftool '-FileName<DateTimeOriginal' \
  -d '2024-Japan_%Y%m%d_%H%M%S.%%e' \
  /path/to/album
```

### Export metadata to CSV for search indexing

```bash
exiftool -csv -r \
  -DateTimeOriginal -GPSLatitude -GPSLongitude \
  -Make -Model -ImageWidth -ImageHeight -Subject \
  ~/Photos/Archive > ~/Photos/index.csv
```

### Read / write XMP keyword tags (non-destructive)

```bash
# Read keywords on a file
exiftool -Subject -XMP:Rating photo.jpg

# Write keywords (embeds in file)
exiftool -XMP:Subject="rock climbing,yosemite,2024" photo.jpg

# Write to XMP sidecar (leaves RAW file untouched — preferred for RAW)
exiftool -XMP:Subject="travel,japan,kyoto" -o photo.xmp photo.cr3

# Star rating
exiftool -XMP:Rating=5 photo.jpg

# Batch: tag all JPEGs in a folder
exiftool -XMP:Subject="climbing,red-rocks" /path/to/album/*.jpg
```

### Checksum archive for integrity verification

```bash
# Generate SHA-256 checksums for all media in archive
find ~/Photos/Archive -type f \( -iname "*.jpg" -o -iname "*.mp4" -o -iname "*.cr3" \) \
  | sort | xargs sha256sum > ~/Photos/checksums.sha256

# Verify later (returns errors only if files have changed)
sha256sum --check --quiet ~/Photos/checksums.sha256
```

### Video thumbnails (for previews / contact sheets)

```bash
# Single thumbnail at 5 seconds
ffmpeg -i video.mp4 -ss 00:00:05 -frames:v 1 "${video%.mp4}_thumb.jpg"

# Batch thumbnails for all MP4s in a folder
for f in /path/to/videos/*.mp4; do
  ffmpeg -i "$f" -ss 00:00:05 -frames:v 1 "${f%.mp4}_thumb.jpg" 2>/dev/null
done

# Contact sheet (4×4 grid of frames)
ffmpeg -i video.mp4 -vf "fps=1/15,scale=320:-1,tile=4x4" contact_sheet.jpg
```

---

## Folder Structure Convention

```
~/Photos/
├── Archive/                    ← permanent, organized archive
│   ├── 2024/
│   │   ├── 2024-03_Travel_Japan-Kyoto/
│   │   │   ├── RAW/            ← original camera files (.CR3, .ARW, .NEF)
│   │   │   ├── Edited/         ← Photoshop/Lightroom exports
│   │   │   ├── Video/          ← video clips (.MP4, .MOV)
│   │   │   ├── Selects/        ← hero shots (symlinks or copies)
│   │   │   └── .manifest.json  ← metadata index for this album
│   │   ├── 2024-06_Event_Birthday-Emma/
│   │   ├── 2024-08_Climbing_Red-Rocks/
│   │   ├── 2024-09_Kids_First-Day-School/
│   │   └── 2024-12_Family_Christmas/
│   └── 2025/
│       └── ...
├── Inbox/                      ← unprocessed imports, staging area
├── Working/                    ← active editing projects
└── index.json                  ← cross-archive search index
```

### Album naming convention

```
YYYY-MM_[Category]_[Descriptor]
```

**Category keywords** (use consistently for filtering):

| Category     | Examples                                    |
|--------------|---------------------------------------------|
| `Travel`     | `Travel_Japan-Kyoto`, `Travel_Peru-Machu`   |
| `Event`      | `Event_Birthday-Emma`, `Event_Wedding-Smith`|
| `Climbing`   | `Climbing_Red-Rocks`, `Climbing_Yosemite`   |
| `Kids`       | `Kids_Soccer-Season`, `Kids_School-Play`    |
| `Family`     | `Family_Christmas`, `Family_Thanksgiving`   |
| `Everyday`   | `Everyday_Backyard`, `Everyday_Hike-Local`  |

### File naming convention

```
YYYYMMDD_HHMMSS_[seq].ext           ← standard rename
YYYYMMDD_HHMMSS_[seq]_orig.ext      ← original preserved alongside edit
```

---

## Core Python Toolkit

### Setup

```python
import os
import json
import shutil
import hashlib
from pathlib import Path
from datetime import datetime
import subprocess

ARCHIVE_ROOT = Path.home() / 'Photos' / 'Archive'
INBOX = Path.home() / 'Photos' / 'Inbox'

PHOTO_EXTS = {'.jpg', '.jpeg', '.png', '.heic', '.heif',
              '.cr3', '.cr2', '.arw', '.nef', '.orf', '.rw2', '.dng'}
VIDEO_EXTS = {'.mp4', '.mov', '.mts', '.m2ts', '.avi', '.mkv'}
ALL_MEDIA_EXTS = PHOTO_EXTS | VIDEO_EXTS
```

### 1 — Extract metadata with exiftool

```python
def get_exif(file_path: Path) -> dict:
    """Return metadata dict for any photo or video using exiftool."""
    result = subprocess.run(
        ['exiftool', '-json', '-DateTimeOriginal', '-CreateDate',
         '-GPSLatitude', '-GPSLongitude', '-Make', '-Model',
         '-ImageWidth', '-ImageHeight', '-Duration',
         str(file_path)],
        capture_output=True, text=True
    )
    if result.returncode != 0:
        return {}
    data = json.loads(result.stdout)
    return data[0] if data else {}


def parse_capture_time(meta: dict) -> datetime | None:
    """Extract capture datetime from exiftool metadata dict."""
    for field in ('DateTimeOriginal', 'CreateDate'):
        raw = meta.get(field)
        if raw:
            try:
                return datetime.strptime(raw[:19], '%Y:%m:%d %H:%M:%S')
            except ValueError:
                continue
    return None


def get_video_meta(file_path: Path) -> dict:
    """Get video duration and creation time via ffprobe."""
    result = subprocess.run(
        ['ffprobe', '-v', 'quiet', '-print_format', 'json',
         '-show_format', '-show_streams', str(file_path)],
        capture_output=True, text=True
    )
    if result.returncode != 0:
        return {}
    return json.loads(result.stdout)
```

### 2 — Checksum-based deduplication

```python
def file_checksum(path: Path, chunk_size: int = 65536) -> str:
    """SHA-256 of file content. Used to detect duplicates."""
    h = hashlib.sha256()
    with open(path, 'rb') as f:
        while chunk := f.read(chunk_size):
            h.update(chunk)
    return h.hexdigest()


def load_seen_checksums(archive_root: Path) -> set[str]:
    """Walk existing archive and collect all known checksums."""
    index_path = archive_root.parent / 'index.json'
    if index_path.exists():
        idx = json.loads(index_path.read_text())
        return {e['checksum'] for e in idx.get('files', [])}
    # Fallback: compute on the fly (slow on first run)
    seen = set()
    for f in archive_root.rglob('*'):
        if f.suffix.lower() in ALL_MEDIA_EXTS and f.is_file():
            seen.add(file_checksum(f))
    return seen
```

### 3 — Rename and copy into archive

```python
def make_dest_path(src: Path, capture_time: datetime, album_dir: Path,
                   subfolder: str, seq: int) -> Path:
    """Build the destination path using naming convention."""
    timestamp = capture_time.strftime('%Y%m%d_%H%M%S')
    new_name = f'{timestamp}_{seq:04d}{src.suffix.lower()}'
    dest_dir = album_dir / subfolder
    dest_dir.mkdir(parents=True, exist_ok=True)
    return dest_dir / new_name


def safe_copy(src: Path, dest: Path) -> bool:
    """Copy src to dest, verify with checksum, return True on success."""
    if dest.exists():
        return True  # already imported
    shutil.copy2(src, dest)
    if file_checksum(src) == file_checksum(dest):
        return True
    dest.unlink()  # failed verification
    return False
```

### 4 — Full import workflow

```python
def import_media(source_dir: Path, album_name: str,
                 dry_run: bool = False) -> dict:
    """
    Import all media from source_dir into a named album.

    album_name format: "2024-03_Travel_Japan-Kyoto"
    Returns a manifest dict with all imported files.
    """
    year = album_name[:4]
    album_dir = ARCHIVE_ROOT / year / album_name
    seen = load_seen_checksums(ARCHIVE_ROOT)

    manifest = {
        'album': album_name,
        'import_date': datetime.now().isoformat(),
        'source': str(source_dir),
        'files': [],
        'skipped_duplicates': 0,
        'errors': [],
    }

    files = sorted(
        f for f in source_dir.rglob('*')
        if f.suffix.lower() in ALL_MEDIA_EXTS and f.is_file()
    )

    for seq, src in enumerate(files, start=1):
        try:
            checksum = file_checksum(src)
            if checksum in seen:
                manifest['skipped_duplicates'] += 1
                continue

            is_video = src.suffix.lower() in VIDEO_EXTS
            meta = get_exif(src)
            capture_time = parse_capture_time(meta) or \
                           datetime.fromtimestamp(src.stat().st_mtime)

            subfolder = 'Video' if is_video else 'RAW'
            dest = make_dest_path(src, capture_time, album_dir, subfolder, seq)

            entry = {
                'original': str(src),
                'dest': str(dest),
                'checksum': checksum,
                'capture_time': capture_time.isoformat(),
                'camera': f"{meta.get('Make','')} {meta.get('Model','')}".strip(),
                'type': 'video' if is_video else 'photo',
                'gps_lat': meta.get('GPSLatitude'),
                'gps_lon': meta.get('GPSLongitude'),
            }

            if not dry_run:
                if safe_copy(src, dest):
                    seen.add(checksum)
                    manifest['files'].append(entry)
                else:
                    manifest['errors'].append(str(src))
            else:
                print(f'DRY RUN: {src.name} → {dest}')
                manifest['files'].append(entry)

        except Exception as e:
            manifest['errors'].append(f'{src}: {e}')

    if not dry_run:
        manifest_path = album_dir / '.manifest.json'
        manifest_path.write_text(json.dumps(manifest, indent=2))

    return manifest
```

### 5 — Build / update global search index

```python
def rebuild_index(archive_root: Path = ARCHIVE_ROOT) -> None:
    """
    Walk all album manifests and merge into a single index.json.
    Run after any import to keep search index current.
    """
    index = {'updated': datetime.now().isoformat(), 'files': []}
    for manifest_path in archive_root.rglob('.manifest.json'):
        m = json.loads(manifest_path.read_text())
        for entry in m.get('files', []):
            entry['album'] = m['album']
            index['files'].append(entry)

    idx_path = archive_root.parent / 'index.json'
    idx_path.write_text(json.dumps(index, indent=2))
    print(f'Index rebuilt: {len(index["files"])} files → {idx_path}')


def search_index(query: str, field: str = 'album',
                 archive_root: Path = ARCHIVE_ROOT) -> list[dict]:
    """
    Simple search over the JSON index.
    field: 'album', 'camera', 'capture_time', 'type', etc.
    query: substring match (case-insensitive)
    """
    idx_path = archive_root.parent / 'index.json'
    if not idx_path.exists():
        rebuild_index(archive_root)
    index = json.loads(idx_path.read_text())
    q = query.lower()
    return [
        e for e in index['files']
        if q in str(e.get(field, '')).lower()
    ]
```

---

## Step-by-Step: Organizing a New Import

### Step 1 — Ask the user three questions

Before running anything, determine:
1. **Source**: Where are the files? (SD card, phone backup, download folder, existing messy folder)
2. **Album name**: What event/trip is this? Suggest a name using the convention: `YYYY-MM_Category_Descriptor`
3. **Dry run first?**: Recommended — show what would happen before moving anything.

### Step 2 — Scan and report

```python
def scan_source(source_dir: Path) -> dict:
    """Report what's in the source before importing."""
    photos, videos, others, total_bytes = [], [], [], 0
    for f in source_dir.rglob('*'):
        if not f.is_file():
            continue
        ext = f.suffix.lower()
        total_bytes += f.stat().st_size
        if ext in PHOTO_EXTS:
            photos.append(f)
        elif ext in VIDEO_EXTS:
            videos.append(f)
        else:
            others.append(f)

    return {
        'photos': len(photos),
        'videos': len(videos),
        'other': len(others),
        'total_gb': round(total_bytes / 1e9, 2),
        'date_range': _estimate_date_range(photos + videos),
    }

def _estimate_date_range(files: list[Path]) -> str:
    times = []
    for f in files[:50]:  # sample first 50 for speed
        meta = get_exif(f)
        t = parse_capture_time(meta)
        if t:
            times.append(t)
    if not times:
        return 'unknown'
    return f'{min(times).date()} → {max(times).date()}'
```

Report to user:
```
📁 Source: /Volumes/CANON_SD/DCIM
   📷 Photos: 347
   🎬 Videos: 23
   📦 Total: 18.4 GB
   📅 Date range: 2024-03-10 → 2024-03-22
   💡 Suggested album: 2024-03_Travel_Japan-Kyoto
```

### Step 3 — Dry run

Run `import_media(source, album_name, dry_run=True)` and show a sample of what will be created:

```
DRY RUN preview (first 10 of 370):
  IMG_0001.CR3 → Archive/2024/2024-03_Travel_Japan-Kyoto/RAW/20240310_083045_0001.cr3
  IMG_0002.CR3 → Archive/2024/2024-03_Travel_Japan-Kyoto/RAW/20240310_083102_0002.cr3
  VID_0003.MP4 → Archive/2024/2024-03_Travel_Japan-Kyoto/Video/20240310_094500_0003.mp4
  ...
  🔁 0 duplicates would be skipped
```

### Step 4 — Execute and verify

Run `import_media(source, album_name, dry_run=False)`. After completion:
- Report counts: imported / skipped duplicates / errors
- Show manifest path
- Run `rebuild_index()` to update the global search index

### Step 5 — Offer follow-up actions

After a successful import, offer:
1. **Tag selects**: "Want to mark your favorite shots from this album?"
2. **GPS → location**: "Shall I look up location names from GPS coordinates?"
3. **Create event summary**: "I can generate a one-page text summary of the album for your notes."

---

## Finding Files: Search Patterns

### By album name / event

```bash
# Find all Japan photos
python3 -c "
from pathlib import Path; import json
idx = json.loads((Path.home()/'Photos/index.json').read_text())
results = [e for e in idx['files'] if 'japan' in e['album'].lower()]
for r in results[:20]: print(r['dest'])
"
```

### By date range

```python
from datetime import datetime

def find_by_date(start: str, end: str) -> list[dict]:
    """Find files captured between two dates. Dates as 'YYYY-MM-DD'."""
    idx = json.loads((Path.home() / 'Photos' / 'index.json').read_text())
    s = datetime.fromisoformat(start)
    e = datetime.fromisoformat(end)
    return [
        f for f in idx['files']
        if s <= datetime.fromisoformat(f['capture_time'][:10]) <= e
    ]
```

### By category

```bash
# All climbing shots
jq '[.files[] | select(.album | contains("Climbing"))]' ~/Photos/index.json

# All videos from kids events
jq '[.files[] | select((.album | contains("Kids")) and .type=="video")]' ~/Photos/index.json
```

### By camera

```python
search_index('Canon EOS R5', field='camera')  # all photos from a specific body
```

---

## Duplicate Detection (Standalone)

Use a three-stage pipeline — fastest to most thorough:

### Stage 1 — Byte-identical duplicates (fastest)

```bash
# jdupes: finds exact byte-for-byte duplicates recursively
jdupes -r ~/Photos

# To delete (keep first copy, delete rest — review first!)
jdupes -r -d ~/Photos

# fdupes (alternative if jdupes unavailable)
fdupes -r ~/Photos
```

### Stage 2 — Perceptual image duplicates (same photo, different quality/resize)

```bash
pip install imgdupes

# Find near-duplicates using difference hash (fast)
imgdupes --dHash ~/Photos

# Perceptual hash (slower, more accurate for edited/cropped versions)
imgdupes --pHash ~/Photos
```

### Stage 3 — Near-duplicate videos

```python
from videohash import VideoHash  # pip install videohash

def are_similar_videos(path_a: str, path_b: str, threshold: int = 10) -> bool:
    """Return True if videos look nearly identical (different quality/trim)."""
    ha = VideoHash(path_a)
    hb = VideoHash(path_b)
    return ha - hb <= threshold  # Hamming distance threshold
```

### Python checksum dedup (no extra tools)

```python
def find_duplicates(search_dir: Path) -> dict[str, list[Path]]:
    """Return dict of checksum → [list of duplicate paths]."""
    seen: dict[str, list[Path]] = {}
    for f in search_dir.rglob('*'):
        if f.suffix.lower() in ALL_MEDIA_EXTS and f.is_file():
            cs = file_checksum(f)
            seen.setdefault(cs, []).append(f)
    return {cs: paths for cs, paths in seen.items() if len(paths) > 1}

# Usage
dupes = find_duplicates(Path('/Users/you/Downloads/Photos_Mess'))
for cs, paths in dupes.items():
    print(f'Duplicate ({len(paths)} copies):')
    for p in paths:
        print(f'  {p}  ({p.stat().st_size // 1024} KB)')
```

**Decision rule for duplicates**: Always keep the highest-quality copy (largest file size for same content). Move duplicates to a `_Duplicates_Review/` folder — never auto-delete without user confirmation.

---

## Category-Specific Conventions

### Travel

- Album: `YYYY-MM_Travel_[Country]-[City/Region]`
- Subfolders: `RAW/`, `Edited/`, `Video/`, `Selects/`
- Tag key shots with a `★` prefix in filename after editing: `★_20240312_091500_0042.jpg`

### Events (birthdays, weddings, concerts)

- Album: `YYYY-MM_Event_[Type]-[Name]`
- Subfolders: `Ceremony/`, `Reception/`, `Portraits/`, `Candid/`, `Video/`

### Rock Climbing

- Album: `YYYY-MM_Climbing_[Area]` (e.g., `Climbing_Yosemite`, `Climbing_Red-Rocks`)
- Tag by climb: rename selects with route name: `20240815_El-Cap-Nose_0012.jpg`
- Video clips: `Video/burns/` for training reviews

### Kids

- Album: `YYYY-MM_Kids_[Descriptor]` (e.g., `Kids_Soccer-Spring`, `Kids_School-Play`)
- Group by child if needed: subfolder `Emma/`, `Jack/`

---

## Sidecar XMP Tags (Optional)

Add searchable keywords without modifying originals:

```bash
# Tag a photo with keywords using exiftool
exiftool -Subject="travel,japan,kyoto,2024" photo.jpg

# Write to XMP sidecar (leaves original untouched)
exiftool -Subject="climbing,red-rocks,trad" -o photo.xmp photo.jpg

# Search by keyword across archive
exiftool -r -if '$Subject =~ /climbing/' -FileName -Directory ~/Photos/Archive/
```

---

## Quality Checks

Before finishing any import, verify:
- [ ] Dry run was shown and approved before executing
- [ ] File count in archive matches source (minus known duplicates)
- [ ] No files in the error list; if errors exist, diagnose and retry
- [ ] Manifest `.manifest.json` exists in the album folder
- [ ] `index.json` was rebuilt after import
- [ ] Original source files are intact (if "copy, don't move" was requested)
- [ ] Album name follows the `YYYY-MM_Category_Descriptor` convention
- [ ] At least one `jq` or Python search was run to verify searchability
