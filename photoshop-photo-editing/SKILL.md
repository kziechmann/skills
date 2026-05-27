---
name: photoshop-photo-editing
description: >
  Assists with Adobe Photoshop photo editing by generating automation
  scripts (UXP/ExtendScript), designing batch-processing workflows, and
  providing editing recipes for travel, portrait, event, kids, and action
  (rock climbing, sports) photography. Trigger when the user mentions
  "Photoshop", "edit photos", "batch edit", "retouch", "color grade
  photos", "Photoshop script", "Photoshop action", "automate Photoshop",
  "export presets", or asks how to fix/enhance a specific photo type.
  Also trigger for /photoshop, /edit-photos.
allowed-tools:
  - Bash(osascript *)
  - Bash(open *)
  - Bash(python3 *)
  - Bash(exiftool *)
  - Bash(magick *)
---

# Photoshop Photo Editing Skill

Help the user edit photos in Adobe Photoshop by generating ready-to-run
automation scripts, editing recipes per subject type, and batch-processing
workflows. Always prefer non-destructive edits (Smart Objects, adjustment
layers, layer masks) so originals can be revisited.

## Guiding Principles

- **Non-destructive first**: Use adjustment layers and Smart Objects.
  Never apply destructive edits to the background layer.
- **Scripts over clicking**: Anything repetitive (batch export, consistent
  looks, renaming) should be automated with a script.
- **Subject-specific recipes**: Travel landscapes need different treatment
  than indoor portraits of kids; provide targeted guidance, not generic tips.
- **Show the layer stack**: For complex edits, describe the exact layer
  order so the user can reproduce it from scratch.
- **Export for purpose**: Web/social, large print, and archiving have
  different export settings. Always ask or infer intent.
- **RAW-first workflow**: For serious editing, work from RAW (CR3, ARW,
  NEF) via Camera Raw filter or Lightroom round-trip, not JPEG.

---

## Photoshop Automation: Script Types

| Script type       | When to use                                        | Runtime         |
|-------------------|----------------------------------------------------|-----------------|
| **UXP (JS)**      | Modern PS 2021+; panels, dialogs, async IO         | Photoshop UDT   |
| **ExtendScript**  | Legacy PS / broad compatibility; batch actions     | Script Editor   |
| **Action (.atn)** | Record-and-playback; no coding; shareable preset   | Actions panel   |
| **Droplet**       | Drag-and-drop batch from Finder; wraps an Action   | Standalone app  |
| **Shell + PS**    | Drive PS from terminal (headless on macOS/Win)     | `open -a PS`    |

Use **ExtendScript** for most automation requests unless the user has PS 2021+ and wants a panel/dialog.

---

## ExtendScript Boilerplate

```javascript
// ExtendScript — runs in Photoshop's Script Editor (File > Scripts > Browse)
// Or batch via File > Automate > Batch

#target photoshop
app.bringToFront();

function main() {
    if (!app.documents.length) {
        alert("Open a document first.");
        return;
    }
    var doc = app.activeDocument;
    // ... your edits here
    doc.flatten();
    doc.save();
}

main();
```

---

## Editing Recipes by Subject

### 1. Travel — Landscape & Cityscape

**Goal**: Rich colors, balanced sky and foreground, punchy but natural.

**Layer stack** (bottom to top):

```
[ Background — RAW via Camera Raw Smart Object ]
[ Levels adjustment — set black point / white point ]
[ Vibrance — +20 Vibrance, +5 Saturation ]
[ Hue/Sat — Sky: Cyan/Blue Hue shift -5, Sat +15 ]
[ Curves — S-curve: lift midtones, deepen shadows ]
[ Graduated Filter (sky) — manual darken top 30% ]
[ Local Dodge/Burn — Smart Object + High Pass 0% fill ]
[ Noise Reduction — Surface Blur r=2 on merged stamp ]
```

**ExtendScript: Add vibrance + S-curve in one shot**

```javascript
#target photoshop
var doc = app.activeDocument;

// Add Vibrance layer
var vibranceLayer = doc.artLayers.add();
vibranceLayer.kind = LayerKind.VIBRANCE;
// Note: set via descriptor for Vibrance
var desc = new ActionDescriptor();
desc.putInteger(stringIDToTypeID("vibrance"), 20);
desc.putInteger(stringIDToTypeID("saturation"), 5);
executeAction(stringIDToTypeID("make"), desc, DialogModes.NO);

// Add Curves layer with S-curve
var curvesLayer = doc.artLayers.add();
var ref = new ActionReference();
ref.putClass(charIDToTypeID("AdjL"));
var desc2 = new ActionDescriptor();
var desc3 = new ActionDescriptor();
desc3.putString(charIDToTypeID("Nm  "), "S-Curve");
// Shadows down, highlights up
var pts = new ActionList();
function addPoint(inp, out) {
    var pt = new ActionDescriptor();
    pt.putDouble(charIDToTypeID("Inpt"), inp);
    pt.putDouble(charIDToTypeID("Orpt"), out);
    pts.putObject(charIDToTypeID("Pnt "), pt);
}
addPoint(0, 0); addPoint(64, 50); addPoint(128, 135); addPoint(255, 255);
desc3.putList(charIDToTypeID("Crv "), pts);
desc2.putObject(charIDToTypeID("Type"), charIDToTypeID("BrgC"), desc3);
desc2.putObject(ref);
executeAction(charIDToTypeID("Mk  "), desc2, DialogModes.NO);
```

**Camera Raw settings to hit first (before PS):**
- Exposure: +0.3 to +0.5 (slightly bright)
- Highlights: -60 (recover sky)
- Shadows: +40 (open foreground)
- Whites: +20 | Blacks: -20
- Clarity: +15 | Dehaze: +10
- Vibrance: +25

---

### 2. Travel — Golden Hour / Sunset

**Goal**: Warm, glowing tones without blowing highlights or over-saturating.

**Key adjustments:**
- Hue/Sat layer: Reds +15 Sat, Yellows +10 Hue shift toward orange
- Curves layer: Pull warm (Red channel up, Blue channel down slightly)
- Color Balance: Midtones +10 Red, -5 Blue; Shadows +5 Yellow
- Luminosity mask to protect foreground shadows

**ExtendScript: Warm color grade**

```javascript
#target photoshop
var doc = app.activeDocument;

// Color Balance — midtones warm
var ref = new ActionReference();
ref.putClass(charIDToTypeID("AdjL"));
var makeDesc = new ActionDescriptor();
var typeDesc = new ActionDescriptor();
typeDesc.putInteger(charIDToTypeID("Rd  "), 12);   // +red
typeDesc.putInteger(charIDToTypeID("Grn "), 0);
typeDesc.putInteger(charIDToTypeID("Bl  "), -8);   // -blue
makeDesc.putObject(charIDToTypeID("Type"), charIDToTypeID("ClrB"), typeDesc);
makeDesc.putObject(ref);
executeAction(charIDToTypeID("Mk  "), makeDesc, DialogModes.NO);
```

---

### 3. Portraits (Events, Kids)

**Goal**: Clean skin, bright eyes, natural hair. Avoid over-smoothing.

**Layer stack:**

```
[ Background RAW ]
[ Frequency Separation — high-pass detail layer ]
[ Clone Stamp / Healing — only on LOW freq layer ]
[ Skin Smoothing — Gaussian Blur on LOW freq, 30% opacity ]
[ Dodge & Burn — 50% gray layer, Overlay mode ]
[ Eye Enhancement — Curves boost, Lasso masked ]
[ Color: Hue/Sat — Reds -5 Sat (reduce redness in skin) ]
[ Overall Brightness — Curves, midtones +10 ]
```

**Frequency separation setup script:**

```javascript
#target photoshop
var doc = app.activeDocument;
var radius = 4; // adjust for skin texture (3-6 typical)

// Duplicate background twice
var high = doc.activeLayer.duplicate();
high.name = "High Frequency";
var low = doc.activeLayer.duplicate();
low.name = "Low Frequency";
doc.activeLayer = low;

// Blur the low frequency layer
var blurDesc = new ActionDescriptor();
blurDesc.putDouble(charIDToTypeID("Rds "), radius);
executeAction(charIDToTypeID("GsnB"), blurDesc, DialogModes.NO);

// Set high frequency to Linear Light, apply Apply Image
doc.activeLayer = high;
high.blendMode = BlendMode.LINEARLIGHT;

var applyDesc = new ActionDescriptor();
applyDesc.putString(charIDToTypeID("Scr "), doc.name);
applyDesc.putString(charIDToTypeID("Chnl"), "RGB");
applyDesc.putString(charIDToTypeID("Trgt"), "RGB");
applyDesc.putEnumerated(
    charIDToTypeID("Blnd"), charIDToTypeID("BlnM"), charIDToTypeID("Sbtr")
);
applyDesc.putBoolean(charIDToTypeID("Invr"), false);
applyDesc.putDouble(charIDToTypeID("Scl "), 2);
applyDesc.putInteger(charIDToTypeID("Ofst"), 128);
executeAction(charIDToTypeID("ApIm"), applyDesc, DialogModes.NO);

alert("Frequency separation done. Paint on 'Low Frequency' layer to smooth skin.");
```

---

### 4. Kids — Bright & Cheerful

**Goal**: Punchy, warm, airy — no moody shadows. Kids photos should look joyful.

**Quick recipe:**
- Exposure: +0.4 | Shadows: +50 | Blacks: +20 (lift shadows, keep airy)
- Vibrance: +30 | Saturation: +5
- Temperature: shift warm (+200–300K)
- Curves: Lift shadow anchor to ~15 (film-like fade, avoids harsh blacks)
- Hue/Sat — Reds: +5 Hue toward orange (warmer skin), -10 Sat (less red)
- Skin tone check: Color Sampler on cheek, target L 70–80 in Lab mode

**ExtendScript: Airy lift (lift shadows)**

```javascript
#target photoshop
var doc = app.activeDocument;
// Add curves layer and lift the shadow end
var ref = new ActionReference();
ref.putClass(charIDToTypeID("AdjL"));
var desc = new ActionDescriptor();
var curvDesc = new ActionDescriptor();
var pts = new ActionList();
function pt(i, o) {
    var p = new ActionDescriptor();
    p.putDouble(charIDToTypeID("Inpt"), i);
    p.putDouble(charIDToTypeID("Orpt"), o);
    pts.putObject(charIDToTypeID("Pnt "), p);
}
pt(0, 18);    // lift shadows — airy look
pt(128, 138); // lift midtones slightly
pt(255, 255);
curvDesc.putList(charIDToTypeID("Crv "), pts);
desc.putObject(charIDToTypeID("Type"), charIDToTypeID("BrgC"), curvDesc);
desc.putObject(ref);
executeAction(charIDToTypeID("Mk  "), desc, DialogModes.NO);
```

---

### 5. Rock Climbing & Action

**Goal**: Dynamic, contrasty, punchy. Climber pops from the rock.

**Key adjustments:**
- Exposure: correct for climber face (often backlit — use grad filter)
- Shadows: +40 to open dark rock face
- Clarity: +25 (texture in rock, chalk, gear)
- Dehaze: +10 (if shot in direct sun)
- Curves: Strong S-curve — drama without blowing highlights
- Hue/Sat — Blues: +15 Sat (sky), Greens: +10 Sat (foliage)
- Vignette: -20 to draw eye to climber
- Sharpen: Amount 80, Radius 1.2, Detail 30 (for rope/rock texture)

**Selective sharpening on climber only (ExtendScript):**

```javascript
#target photoshop
// Smart Sharpen on a merged stamp layer, then mask to subject
var doc = app.activeDocument;

// Create merged stamp
var stamp = doc.artLayers.add();
stamp.name = "Sharpened Stamp";

var ref = new ActionReference();
ref.putEnumerated(charIDToTypeID("Lyr "), charIDToTypeID("Ordn"), charIDToTypeID("Trgt"));
var desc = new ActionDescriptor();
desc.putReference(charIDToTypeID("null"), ref);
desc.putEnumerated(charIDToTypeID("Usng"), charIDToTypeID("MrgV"), charIDToTypeID("MrgV"));
executeAction(charIDToTypeID("Dplc"), desc, DialogModes.NO);

// Apply Smart Sharpen
var ssDesc = new ActionDescriptor();
ssDesc.putDouble(charIDToTypeID("Amnt"), 80);
ssDesc.putDouble(charIDToTypeID("Rds "), 1.2);
ssDesc.putInteger(charIDToTypeID("Thsh"), 0);
ssDesc.putEnumerated(charIDToTypeID("Rmvl"), charIDToTypeID("IRmd"),
    charIDToTypeID("GsnB"));
executeAction(stringIDToTypeID("smartSharpen"), ssDesc, DialogModes.NO);

// Add black mask (user paints in sharpening with white brush)
app.activeDocument.activeLayer.grouped = false;
var mask = app.activeDocument.activeLayer;
var blackMask = new ActionDescriptor();
blackMask.putBoolean(charIDToTypeID("HdFX"), false);
blackMask.putEnumerated(charIDToTypeID("Add "), charIDToTypeID("Chnl"), charIDToTypeID("Msk "));
blackMask.putEnumerated(charIDToTypeID("Clr "), charIDToTypeID("Clr "), charIDToTypeID("Blck"));
executeAction(charIDToTypeID("Add "), blackMask, DialogModes.NO);

alert("Black mask added. Paint white over the climber to apply sharpening selectively.");
```

---

## Batch Processing Workflow

### Batch export: folder of RAW → resized JPEGs for sharing

```javascript
#target photoshop
// File > Scripts > Browse — point to this file
// Processes all supported files in a source folder

var sourceFolder = Folder.selectDialog("Select source folder (RAW files)");
var destFolder = Folder.selectDialog("Select destination folder (JPEGs)");

if (!sourceFolder || !destFolder) { alert("Cancelled."); }
else {
    var files = sourceFolder.getFiles(/\.(cr3|cr2|arw|nef|dng|jpg|jpeg)$/i);
    var longSideMax = 2048; // for web/social; change to 4096 for print

    for (var i = 0; i < files.length; i++) {
        var f = files[i];
        var doc = app.open(f);

        // Flatten and resize
        doc.flatten();
        var w = doc.width.as("px");
        var h = doc.height.as("px");
        if (Math.max(w, h) > longSideMax) {
            if (w >= h) {
                doc.resizeImage(UnitValue(longSideMax, "px"), undefined, 300, ResampleMethod.BICUBICSHARPER);
            } else {
                doc.resizeImage(undefined, UnitValue(longSideMax, "px"), 300, ResampleMethod.BICUBICSHARPER);
            }
        }

        // Save as JPEG
        var jpegName = decodeURI(f.name).replace(/\.[^.]+$/, '.jpg');
        var destFile = new File(destFolder + "/" + jpegName);
        var opts = new JPEGSaveOptions();
        opts.quality = 11;        // 0–12; 11 = high quality, ~1–3 MB
        opts.formatOptions = FormatOptions.STANDARDBASELINE;
        opts.matte = MatteType.NONE;
        doc.saveAs(destFile, opts, true, Extension.LOWERCASE);
        doc.close(SaveOptions.DONOTSAVECHANGES);
    }
    alert("Done! " + files.length + " files exported to " + destFolder.fsName);
}
```

### Apply an action to a folder (via Batch)

Instead of scripting adjustments, record a Photoshop **Action** with the edits,
then run via `File > Automate > Batch`. To script the Batch dialog:

```javascript
#target photoshop
var batchDesc = new ActionDescriptor();
var ref = new ActionReference();
ref.putName(charIDToTypeID("ASet"), "My Action Set");
ref.putName(charIDToTypeID("Actn"), "My Action Name");
batchDesc.putReference(charIDToTypeID("null"), ref);
batchDesc.putEnumerated(charIDToTypeID("Srce"), charIDToTypeID("BMdS"), charIDToTypeID("Fldr"));
batchDesc.putPath(charIDToTypeID("From"), new Folder("/path/to/source"));
batchDesc.putBoolean(charIDToTypeID("SprS"), true);           // suppress color warnings
batchDesc.putBoolean(charIDToTypeID("SprD"), true);           // suppress dialogs
batchDesc.putEnumerated(charIDToTypeID("Dstn"), charIDToTypeID("BMdS"), charIDToTypeID("Fldr"));
batchDesc.putPath(charIDToTypeID("To  "), new Folder("/path/to/dest"));
batchDesc.putBoolean(charIDToTypeID("Ovrd"), true);           // override save-as
executeAction(charIDToTypeID("Btch"), batchDesc, DialogModes.NO);
```

---

## Export Settings by Purpose

| Purpose              | Format | Long side | Quality | Color space |
|----------------------|--------|-----------|---------|-------------|
| Instagram / social   | JPEG   | 2048 px   | 85–90%  | sRGB        |
| Website / web embed  | JPEG   | 1920 px   | 80–85%  | sRGB        |
| Print (lab)          | TIFF   | Full res  | —       | AdobeRGB    |
| Print (home inkjet)  | JPEG   | Full res  | 95–100% | AdobeRGB    |
| Archive master       | PSD    | Full res  | —       | ProPhoto RGB|
| Email / quick share  | JPEG   | 1200 px   | 75–80%  | sRGB        |
| Premiere Pro import  | ProRes 4444 / TIFF | Full res | — | AdobeRGB |

**Always embed a color profile.** Untagged files look different across devices.

---

## Camera Raw: Fast Global Settings

Use these as starting points, not final settings — adjust per image.

```
Landscape (travel):      Highlights -60, Shadows +40, Clarity +15, Dehaze +10
Golden hour:             Temp +300, Tint +5, Highlights -40, Vibrance +20
Indoor portrait:         Temp -200, Exposure +0.3, Shadows +30, Noise Red Lum 30
Outdoor portrait (sun):  Highlights -50, Shadows +50, Clarity -5 (soften)
Kids (bright, airy):     Exposure +0.4, Shadows +50, Blacks +20, Vibrance +30
Climbing / action:       Clarity +25, Dehaze +10, Shadows +30, Contrast +20
Night / low light:       Exposure +1.0, Noise Red Lum 50–70, Detail 40, Sharpening 60
```

---

## Smart Object Workflow (Non-Destructive RAW)

Always place RAW files as Smart Objects in Photoshop so Camera Raw settings
remain re-editable:

1. `File > Open As Smart Object` (or drag from Bridge with Smart Object pref)
2. Double-click the Smart Object thumbnail to reopen Camera Raw
3. Do global corrections in Camera Raw; close back to PS
4. Add all creative adjustments as Adjustment Layers above the Smart Object
5. To edit Camera Raw again: double-click thumbnail → Camera Raw reopens

For batch: `File > Scripts > Load Files into Stack` → check "Create Smart Object After Loading Files"

---

## Quality Checks

Before handing off any Photoshop workflow, verify:
- [ ] Background layer is preserved (not modified directly)
- [ ] All edits are on adjustment layers or Smart Objects
- [ ] Color profile is embedded in the export
- [ ] Export resolution and color space match the intended use
- [ ] Skin tones were checked with Color Sampler (not just eyeballed)
- [ ] The script was tested on one file before batch-running on the full set
- [ ] Any script with `doc.save()` has been reviewed — prefer `saveAs` to avoid overwriting originals
- [ ] Noise reduction was applied before sharpening (not after)
