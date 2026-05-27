---
name: premiere-video-editing
description: >
  Assists with Adobe Premiere Pro video editing by generating ExtendScript
  automation, designing project structures, and providing editing recipes
  for travel vlogs, event videos, kids highlights, and action/rock climbing
  footage. Trigger when the user mentions "Premiere Pro", "edit video",
  "video editing", "cut footage", "color grade video", "make a highlight
  reel", "Premiere script", "export video", "sequence settings", "Lumetri",
  "audio mix", or asks how to organize a video project. Also trigger for
  /premiere, /edit-video.
allowed-tools:
  - Bash(osascript *)
  - Bash(open *)
  - Bash(python3 *)
  - Bash(ffmpeg *)
  - Bash(ffprobe *)
---

# Premiere Pro Video Editing Skill

Help the user edit videos in Adobe Premiere Pro by generating automation
scripts, designing project and bin structures, and providing editing
recipes per content type. Everything should be organized, non-destructive,
and repeatable.

## Guiding Principles

- **Project organization first**: A well-organized bin structure is worth
  more than any single edit. Set it up at the start.
- **Proxy workflow for performance**: 4K/RAW on a laptop — always set up
  proxies. Edit on proxies, export from originals.
- **Color before effects**: Establish a base Lumetri grade before adding
  any stylistic effects. Order matters in the Effects panel.
- **Music drives the cut**: For highlight reels and travel vlogs, the edit
  lives or dies by audio. Build around the beats.
- **Script what repeats**: Project setup, export queues, naming — automate
  with ExtendScript so every project starts clean.
- **Nested sequences**: Use nested sequences to group related sections
  (a climbing sequence, a dinner scene). Makes timeline readable and
  allows section-level color/audio adjustments.

---

## ExtendScript in Premiere Pro

Run scripts via: **File > Project Settings > Scratch Disks** (no — that's wrong) → 
Actually: **File > Scripts > Run Script File** (or the ExtendScript Toolkit in older versions).

Modern Premiere Pro (2019+): use **Window > Extensions** for UXP panels, or
**File > Project Settings > Run Script** via the QE DOM.

```javascript
// ExtendScript boilerplate for Premiere Pro
#target premierepro

var project = app.project;
var sequence = project.activeSequence;

if (!project) { alert("No project open."); }
```

---

## Project Setup: Bin Structure

Create this structure at the start of every project.

```
[Project Name]
├── 00_Footage/
│   ├── A-Roll/          ← primary footage (main action, interviews)
│   ├── B-Roll/          ← supplementary shots (scenery, detail, cutaways)
│   ├── Drone/           ← aerial footage
│   └── GoPro/           ← action cam, POV footage
├── 01_Audio/
│   ├── Music/
│   ├── SFX/             ← sound effects
│   └── Voiceover/
├── 02_Graphics/
│   ├── Titles/
│   ├── LUTs/            ← color lookup tables
│   └── Overlays/
├── 03_Sequences/
│   ├── MASTER/          ← final deliverable sequence
│   ├── Rough_Cuts/      ← versioned rough cuts
│   └── Sections/        ← nested sub-sequences
├── 04_Exports/          ← reference to exported files (not imported)
└── 05_Project_Files/    ← saved project versions, backups
```

**ExtendScript: Create bin structure automatically**

```javascript
#target premierepro
var project = app.project;

var topBins = [
    ["00_Footage",    ["A-Roll", "B-Roll", "Drone", "GoPro"]],
    ["01_Audio",      ["Music", "SFX", "Voiceover"]],
    ["02_Graphics",   ["Titles", "LUTs", "Overlays"]],
    ["03_Sequences",  ["MASTER", "Rough_Cuts", "Sections"]],
    ["04_Exports",    []],
    ["05_Project_Files", []],
];

var root = project.rootItem;

topBins.forEach(function(pair) {
    var topName = pair[0];
    var subNames = pair[1];
    var topBin = root.createBin(topName);
    subNames.forEach(function(sub) {
        topBin.createBin(sub);
    });
});

alert("Bin structure created.");
```

---

## Sequence Settings by Project Type

### Travel Vlog / Documentary

```
Resolution:    3840 × 2160 (4K) if camera supports; 1920 × 1080 for YouTube
Frame rate:    24 fps (cinematic) or 30 fps (social-media-native)
Pixel aspect:  Square (1.0)
Audio:         48 kHz, Stereo
Work area:     Extend to full duration
Preview:       H.264 (GPU), use proxies during edit
```

### Short-form (Reels, TikTok, YouTube Shorts)

```
Resolution:    1080 × 1920 (9:16 vertical)
Frame rate:    30 fps
Audio:         48 kHz, Stereo (will be mono on most platforms)
Max duration:  60–90 sec
```

### Climbing / Action

```
Resolution:    4K or match GoPro (2704 × 1520 at 60fps)
Frame rate:    60 fps for slow-motion → 24 fps timeline (2.5× slow)
               120 fps (GoPro) → 24 fps timeline (5× slow)
Pixel aspect:  Square (1.0)
```

**ExtendScript: Create a new sequence with correct settings**

```javascript
#target premierepro
var project = app.project;

// Create 4K 24fps sequence
var seqPreset = "path/to/preset.sqpreset"; // export from existing sequence
// Or use built-in preset GUID — easier to set manually via New Sequence dialog
// Script below creates via the New Sequence dialog preset name:

var presetList = app.encoder.getExporterSettings("H.264");
// For sequence creation, use the Sequence Preset API:
var presets = project.getInsertableItems(qe.project.getSequenceSettingsPresets());

// Simpler: duplicate an existing sequence as template
if (project.sequences.numSequences > 0) {
    var template = project.sequences[0];
    var newSeq = project.createNewSequence("MASTER", "");
    alert("Created sequence: MASTER (adjust settings in Sequence > Sequence Settings)");
}
```

---

## Editing Recipes by Content Type

### 1. Travel Highlight Reel (2–5 min)

**Structure:**
```
[HOOK — 10–15 sec]  Best single shot/moment, no context needed
[Title card — 3 sec] Destination name
[Act 1 — Arrival/Setting] Establishing drone, wide shots, first impressions
[Act 2 — The Experience] Activities, food, people, culture — bulk of the reel
[Act 3 — Emotional Peak] Sunset, meaningful moment, best sequence
[Outro — 10 sec] Credit/logo or quiet fade
```

**Music-to-cut workflow:**
1. Import music first, place on A1 (lock it)
2. Listen and drop **markers** at every beat or phrase change
3. Rough cut B-roll to markers — don't overthink it
4. Refine: trim to motion, lead cuts on action (pan starts, subject moving)
5. Add A-roll / voiceover after structure is locked

**B-roll shot checklist per location:**
- [ ] Wide establishing (drone if possible)
- [ ] Medium activity shots
- [ ] Close-up detail (food, textures, signage, hands)
- [ ] Reaction shot / people / local life
- [ ] Movement shot (walking toward camera, sliding door reveal)
- [ ] Money shot (the main attraction — temple, summit, coastline)

---

### 2. Event Video (Birthday, Wedding, Concert)

**Structure:**
```
[Pre-event prep / atmosphere — 30 sec]
[The moment (ceremony / performance / candles) — core]
[Celebration / reception — energy, dancing, candids]
[Emotional anchor — speech, first dance, toast]
[Credits / thank you slide]
```

**Technical notes:**
- Indoor events: color grade for mixed lighting (candles + overhead)
  → Lumetri: Temp -300 to +200 to neutralize; check skin tone
- Dynamic mic on camera for speeches; ambient second angle for music
- Sync multi-camera via audio waveform (right-click clips → Synchronize > Audio)

**Multi-cam sync script:**

```javascript
#target premierepro
// Select clips in bin, then run: creates a merged clip synced by audio
var project = app.project;
var bin = project.rootItem.children[0]; // point to correct bin
// Multi-cam merge via ExtendScript is limited; use Sequence > Create Multi-Camera Source Sequence
// Script below automates the menu action:
var qeProject = qe.project;
qeProject.mergeClipsAudio(); // merge selected clips by audio sync
```

---

### 3. Kids Highlights

**Goal**: Fun, energetic, warm. Parents want to see the kids' faces clearly — not artsy composition.

**Quick cut rules:**
- Max 3 sec per clip unless it's a reaction shot or funny moment
- Energy: cut on motion, keep pace brisk (especially for under-5 content)
- Music: upbeat, bright, familiar (Disney-adjacent works well)
- Always include at least one close-up face per section

**Color look for kids:**
- Lumetri Creative: Faded Film 20 (airy, warm)
- Temperature: +200 (warm)
- Tint: +5
- Vibrance: +15
- Shadow Tint: warm yellow (move HSL wheel toward yellow in shadows)
- Vignette: very light, -0.3 only to soften edge

---

### 4. Rock Climbing Edit

**Goal**: Dynamic pacing, technical clarity, visceral energy. Viewer should feel the exposure and effort.

**Structure:**
```
[Approach — quick] 5–15 sec of hiking / rack setup
[The Route Overview] Drone or wide showing the full wall / problem
[Key Sequences] Crux section, dynamic moves, resting shake-out
[Summit / Top-out] Money shot; take time here, slow the pace
[Post-climb] Gear, team, landscape — decompress
```

**Slow motion workflow (GoPro 120fps → 24fps timeline):**

1. Import 120fps clips normally
2. Right-click clip in timeline → **Speed/Duration**
3. Set Speed to **20%** (120fps ÷ 24fps × 100% × inverse = 20%)
4. Check "Ripple Edit" if needed
5. Or: in sequence, clip interpreted at 24fps → right-click → **Modify > Interpret Footage** → set frame rate to 24fps for smooth slo-mo

**ExtendScript: Apply 20% slo-mo to all selected clips**

```javascript
#target premierepro
var seq = app.project.activeSequence;
if (!seq) { alert("No active sequence."); }
else {
    for (var t = 0; t < seq.videoTracks.numTracks; t++) {
        var track = seq.videoTracks[t];
        for (var c = 0; c < track.clips.numItems; c++) {
            var clip = track.clips[c];
            if (clip.isSelected()) {
                clip.getSpeed(); // read current
                clip.setSpeed(20, true, true, false);
                // params: speed%, rippleEdit, maintainAudioPitch, useFrameSampling
            }
        }
    }
    alert("Slo-mo applied to selected clips.");
}
```

**Lumetri grade for climbing:**
- Basic: Contrast +20, Highlights -30, Shadows +20, Clarity +15
- Curves: Strong S-curve (deeper blacks, bright highlights)
- Color Wheels: slightly cool shadows (blue), neutral midtones, warm highlights
- Creative: look "Fuji F125 Kodak 2393" as starting point
- Vignette: -0.8 (draw eye to climber/route)
- Sharpen: +30 (rock texture)

---

## Audio Workflow

### Mixing order
1. **Sync**: all clips at correct sync points
2. **Cleanup**: remove noise (Essential Sound > Clean Up Audio)
3. **Levels**: dialogue/VO at -12 dBFS average; music at -18 dBFS under VO, -10 dBFS when no VO
4. **Music ducking**: Auto-duck music under dialogue (Essential Sound > Music > Ducking)
5. **SFX**: add room tone, ambient, foley at -20 to -25 dBFS
6. **Master**: Loudness Normalization to -14 LUFS for YouTube, -16 LUFS for film

**ExtendScript: Set clip volume**

```javascript
#target premierepro
var seq = app.project.activeSequence;
for (var t = 0; t < seq.audioTracks.numTracks; t++) {
    var track = seq.audioTracks[t];
    for (var c = 0; c < track.clips.numItems; c++) {
        var clip = track.clips[c];
        if (clip.isSelected()) {
            // Set volume component (0 = -∞, 1.0 = 0dB, ~0.25 = -12dB)
            var volumeComp = clip.components.getComponentAt(0);
            // Component 0 is typically Volume — verify in Effects panel
            if (volumeComp) {
                var volumeParam = volumeComp.params.getParamForName("Level");
                if (volumeParam) {
                    volumeParam.setValue(-12, true); // dBFS
                }
            }
        }
    }
}
```

---

## Color Grading with Lumetri

### Grading order (always in this sequence)

1. **White balance** (Basic panel: Temp, Tint) — neutralize first
2. **Exposure** (Exposure, Highlights, Shadows, Whites, Blacks)
3. **Contrast** (Curves) — S-curve for punch
4. **Creative Look** (Creative panel, or apply a LUT)
5. **HSL / Color Wheels** — selective color push
6. **Vignette** — frame and guide eye
7. **Sharpen / Noise** — last step

### Apply a LUT to all clips via ExtendScript

```javascript
#target premierepro
var seq = app.project.activeSequence;
var lutPath = "/path/to/your/LUT.cube"; // absolute path

for (var t = 0; t < seq.videoTracks.numTracks; t++) {
    var track = seq.videoTracks[t];
    for (var c = 0; c < track.clips.numItems; c++) {
        var clip = track.clips[c];
        // Add Lumetri Color effect and set Input LUT
        var lumetriEffect = clip.components.addComponent("Lumetri Color");
        if (lumetriEffect) {
            var inputLUT = lumetriEffect.params.getParamForName("Input LUT");
            if (inputLUT) {
                inputLUT.setValue(lutPath, true);
            }
        }
    }
}
```

### Color grading looks by content type

| Content       | Look style                        | Lumetri preset starting point      |
|---------------|-----------------------------------|------------------------------------|
| Travel (warm) | Warm, saturated, cinematic        | SL Cross Process Warm              |
| Travel (cool) | Teal-orange (classic cinematic)   | SL Teal and Orange                 |
| Event         | Natural, clean, slight warmth     | None — correct only                |
| Kids          | Bright, airy, warm                | Faded Film 20 + warm shift         |
| Climbing      | Punchy, dramatic, cool shadows    | Fuji F125 Kodak 2393               |
| Night/low     | Moody, high contrast, desaturated | SL Horror                          |

---

## Export Settings

### Adobe Media Encoder presets

| Destination         | Format    | Preset / Settings                         |
|---------------------|-----------|-------------------------------------------|
| YouTube 4K          | H.264     | YouTube 2160p 4K Ultra HD; VBR 2-pass    |
| YouTube 1080p       | H.264     | YouTube 1080p HD; VBR 2-pass; 16 Mbps    |
| Instagram Reels     | H.264     | 1080×1920; 30fps; 30 Mbps; AAC 320       |
| Archive / master    | ProRes    | Apple ProRes 422 HQ; full res; PCM audio |
| Client delivery     | H.264     | Vimeo 1080p HD or custom high bitrate    |
| Rough cut review    | H.264     | Match Source – High Bitrate (fast)        |

**Send to Media Encoder via script:**

```javascript
#target premierepro
var project = app.project;
var seq = project.activeSequence;
var outputPath = "/Users/you/Desktop/Export/";

// Use AMEBatchExporter
var exporter = app.encoder;
exporter.launchEncoder(); // opens AME if not running

var format = "H.264";
var preset = "/Applications/Adobe Media Encoder 2024/MediaIO/systempresets/58444341_4d584658/YouTube 1080p HD.epr";
var outputFile = outputPath + seq.name + "_1080p.mp4";

exporter.encodeSequence(seq, outputFile, preset, RemoveFromQueue.NO, 0);
alert("Added to Media Encoder queue: " + seq.name);
```

---

## Proxy Workflow Setup

For 4K footage on slower machines:

1. **Create proxies**: Right-click footage in Project panel → **Proxy > Create Proxies**
   - Format: H.264 or ProRes Proxy
   - Preset: 1280×720 or 1920×1080 (half-res)
2. **Attach proxies**: Right-click → Proxy > Attach Proxies (if created externally)
3. **Toggle during edit**: Button in Program Monitor → Toggle Proxies (wrench icon)
4. **Export**: Premiere automatically uses originals — proxies only affect playback

**ExtendScript: Create proxies for all items in a bin**

```javascript
#target premierepro
var project = app.project;
var bin = project.rootItem.findItemsMatchingMediaPath("", false)[0]; // adjust

// Select all clips in bin
var clips = project.rootItem.findItemsMatchingMediaPath("", false);
clips.forEach(function(clip) {
    if (clip.type === ProjectItemType.CLIP) {
        clip.createProxy(
            "/path/to/proxy/folder/",
            "H.264",          // format
            "",               // preset path — blank uses default
            false             // don't replace original
        );
    }
});
```

---

## Keyboard Shortcuts Reference

| Action                    | Mac shortcut          |
|---------------------------|-----------------------|
| Ripple trim (in)          | Q                     |
| Ripple trim (out)         | W                     |
| Add edit (split)          | Cmd + K               |
| Lift / Extract            | ; / '                 |
| Match frame               | F                     |
| Step forward/back 1 frame | ←/→                   |
| Step 5 frames             | Shift + ←/→           |
| Set In / Out              | I / O                 |
| Mark clip                 | X                     |
| Render in/out             | Enter                 |
| Toggle proxy              | Opt + P               |
| Lumetri panel             | Shift + Cmd + F7      |

---

## Quality Checks

Before finishing any Premiere Pro workflow, verify:
- [ ] Project bin structure is organized before import
- [ ] Sequence settings match camera specs (frame rate, resolution)
- [ ] Proxies are enabled during editing and will be bypassed on export
- [ ] Audio levels are set: dialogue -12 dBFS, music -18 dBFS under VO
- [ ] Lumetri grades are applied in the correct order (white balance → exposure → contrast → look)
- [ ] Color correction was done on a dedicated Lumetri layer or adjustment layer, not baked into clips
- [ ] Export preset matches the intended platform (YouTube / Instagram / archive)
- [ ] Media Encoder queue reviewed before starting export — check output path
- [ ] Any ExtendScript was tested on one clip before running across the full sequence
- [ ] The master sequence is named, saved, and backed up before export
