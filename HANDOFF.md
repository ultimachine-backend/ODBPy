# ODB++ Parsing for Ulties - Handoff Document

## Project Goal

Parse ODB++ files (alternative to Gerber) to generate PCB map renders and extract component lists for the Ulties MES system.

## Current Status: Phase 2 Complete

Successfully forked and fixed ODBPy, then integrated it into Ulties with Cairo rendering support.

---

## What Was Done

### 1. Research & Planning

- Evaluated ODB++ parsing libraries:
  - **ODBPy** (MIT) - Chosen, needed fixes
  - **OdbDesign** (AGPL) - Rejected due to copyleft
  - **Custom parser** - Fallback option
  - **KiCad scripting** - Not applicable (no ODB++ import)

- Plan documented at: `/home/ulti/.claude/plans/refactored-wandering-avalanche.md`

### 2. ODBPy Fork & Fixes

Cloned ODBPy to `/home/ulti/ODBPy/` and applied 8 bug fixes:

| # | Issue | Fix | File |
|---|-------|-----|------|
| 1 | Orientation regex rejected `0` | Changed `[1-7]` to `[0-7]` | LayerFeatureParser.py:17 |
| 2 | No space before `;` in attrs | Added `\s*` before attr group | LayerFeatureParser.py:18 |
| 3 | Missing DIELECTRIC layer type | Added to enum and map | Layers.py:85-100 |
| 4 | Rectangle pattern `r` vs `rect` | Changed to `rect?` | StandardSymbols.py:57 |
| 5 | UTF-8 encoding failures | Added latin-1 fallback | Utils.py:31-37 |
| 6 | Parsed non-CMP sections | Added `startswith("CMP")` | ComponentParser.py:103 |
| 7 | Missing import | Added `parse_attributes` | ComponentParser.py:11 |
| 8 | Float attrs parsed as int | Added `_parse_numeric()` | Attributes.py:22-30 |

### 3. POC Test Results

Tested with: `ATS_Commercial_IRLED-B_ODB.tgz` (file ID 19069)

```
Layer matrix: 45 layers including DIELECTRIC ✅
Pads: 35 top, 38 bottom, 32 drill holes ✅
Lines: Traces and board outline ✅
Components: 5 top-side, 2 bottom-side with properties ✅
Symbols: Round, Square, Rectangle ✅
```

---

## Key Files

| File | Purpose |
|------|---------|
| `/home/ulti/ODBPy/` | Forked ODBPy with fixes |
| `/home/ulti/ODBPy/ULTIES_FIXES.md` | Detailed fix documentation |
| `/home/ulti/test_odbpy.py` | POC test script |
| `/tmp/odb_test/odb/` | Extracted test ODB++ file |
| `/home/ulti/ulties/ulties/static/files/19067_ATS_Commercial_IRLED-B_ODB*.tgz` | Original test file |
| `/home/ulti/ulties/ulties/files/lib/odb/` | Ulties ODB++ integration module |

---

## How to Use Fixed ODBPy

```python
import sys
sys.path.insert(0, '/home/ulti/ODBPy')

from ODBPy.Layers import read_layers
from ODBPy.LineRecordParser import read_linerecords
from ODBPy.LayerFeatureParser import _features_decoder_options, Pad, Line
from ODBPy.ComponentParser import parse_components
from ODBPy.Decoder import run_decoder

# Read layer definitions
layers = read_layers('/path/to/odb++')
for layer in layers:
    print(f"{layer.name}: {layer.type.name}")

# Read pads and lines from a layer
linerecords = read_linerecords('/path/to/odb++/steps/pcb/layers/layer_1_top/features')
features = list(run_decoder(linerecords['Layer features'], _features_decoder_options))
pads = [f for f in features if isinstance(f, Pad) and f is not None]
lines = [f for f in features if isinstance(f, Line) and f is not None]

# Read components
comp_records = read_linerecords('/path/to/odb++/steps/pcb/layers/comp_+_top/components')
components = parse_components(comp_records)
for cid, comp in components.items():
    print(f"{comp.name}: {comp.part_name} at {comp.location}")
```

---

## Phase 2: Ulties Integration (Complete)

### ODB++ Module Created

```
ulties/files/lib/odb/
├── __init__.py      # Module exports
├── parser.py        # High-level ODB++ parsing wrapper
├── pcb.py           # ODBPCB class for rendering
├── converter.py     # ODBPy objects → pcb-tools Primitives
├── layers.py        # ODB++ layer type → Ulties file type mapping
└── render.py        # Full rendering with database integration
```

### Primitive Conversion (Implemented)

| ODBPy Type | pcb-tools Primitive | Status |
|------------|---------------------|--------|
| Pad + Round symbol | Circle | ✅ Done |
| Pad + Square symbol | Rectangle | ✅ Done |
| Pad + Rectangle symbol | Rectangle | ✅ Done |
| Pad + Oval symbol | Rectangle | ✅ Done |
| Pad + RoundDonut symbol | Circle (with hole) | ✅ Done |
| Line + Round symbol | Line (round aperture) | ✅ Done |
| Line + Rectangle symbol | Line (rect aperture) | ✅ Done |
| (TODO) PolygonCircle | Arc | Not implemented |
| (TODO) Surface | Region | Not implemented |

### Layer Type Mapping (Implemented)

| ODB++ Layer Type | pcb-tools Layer Class | Ulties File Type |
|------------------|----------------------|------------------|
| Signal (top) | top | gerber_top |
| Signal (bottom) | bottom | gerber_bottom |
| Signal (internal) | internal | gerber_internal |
| SilkScreen (top) | topsilk | gerber_silk_top |
| SilkScreen (bottom) | bottomsilk | gerber_silk_bottom |
| SolderMask (top) | topmask | gerber_mask_top |
| SolderMask (bottom) | bottommask | gerber_mask_bottom |
| SolderPaste (top) | toppaste | gerber_paste_top |
| SolderPaste (bottom) | bottompaste | gerber_paste_bottom |
| Drill | drill | gerber_nc_drill |
| Document | outline | gerber_outline |

### Integration Points (Remaining)

1. **File detection**: `ulties/files/lib/files.py`
   - Detect ODB++ by `matrix/matrix` file presence
   - Add file types: `odb_folder`, `odb_archive`

2. **Async task**: `ulties/events/celery/tasks.py`
   - Add `render_odb` Celery task

3. **Database**: Migration for new file types

---

## Known Gaps

### Not Yet Implemented in ODBPy

1. **Surface/Polygon parsing** - Copper pours show as "unrecognized" (~1000-3000 per layer)
   - S, OB, OS, OE records need parsing
   - Would convert to pcb-tools Region primitive

2. **Arc parsing** - Curved traces
   - OC records in polygon parser exist but not integrated

### Test Coverage

- Tested with 1 Altium export
- Should test with: KiCad, Eagle, OrCAD exports
- May need additional fixes for other EDA tools

## Important Notes

### Symbol Dimensions vs Coordinate Units

**Critical finding**: In ODB++ files, the `UNITS` header (INCH, MIL, MM) only applies to **coordinates**. Symbol dimensions encoded in symbol names (e.g., `r98.4252` for a round pad) are **always in mils**, regardless of the file's unit setting.

This means:
- Coordinates: Use file's UNITS (e.g., INCH)
- Symbol dimensions: Always convert from MIL

The converter handles this automatically.

---

## Running the POC Test

```bash
# Activate environment
source /home/ulti/ulties/env/bin/activate

# Run test
python /home/ulti/test_odbpy.py /tmp/odb_test/odb

# Or with a new ODB++ file
tar -xzf your_design.tgz -C /tmp/new_test/
python /home/ulti/test_odbpy.py /tmp/new_test/your_design
```

---

## Using the Ulties ODB++ Module

### Quick Parse and Render (Standalone)

```python
import sys
sys.path.insert(0, '/home/ulti/ODBPy')  # Add ODBPy fork to path

from ulties.files.lib.odb import parse_odb
from ulties.files.lib.odb.pcb import ODBPCB
from gerber.render.cairo_backend import GerberCairoContext
from gerber.render.theme import THEMES

# Parse ODB++ folder
odb_data = parse_odb('/path/to/odb++/folder')
print(f"Layers: {len(odb_data.layers)}, Components: {len(odb_data.components)}")

# Convert to renderable PCB
pcb = ODBPCB(odb_data, units='MIL')

# Render top view
ctx = GerberCairoContext()
ctx.render_layers(pcb.top_layers, '/tmp/top.png', theme=THEMES['default'])

# Render bottom view
from ulties.files.lib.odb.render import bottom_theme
ctx.render_layers(pcb.bottom_layers, '/tmp/bottom.png', theme=bottom_theme)
```

### Extract Component List

```python
from ulties.files.lib.odb import parse_odb

odb_data = parse_odb('/path/to/odb++/folder')

for name, comp in odb_data.components.items():
    print(f"{name}: {comp.part_name} ({comp.side})")
    print(f"  Location: {comp.location}, Rotation: {comp.rotation}")
    print(f"  Properties: {comp.properties}")
```

### Full Ulties Integration (with Database)

```python
# From within Ulties views/tasks:
from ulties.files.lib.odb.render import render_odb, get_odb_components

# Render to file records (requires request context)
records = render_odb(request, '/path/to/extracted/odb', recipe)

# Get component list for BOM
components = get_odb_components('/path/to/extracted/odb')
```

---

## References

- [ODBPy GitHub](https://github.com/ulikoehler/ODBPy) - Original repo (MIT license)
- [ODB++ v7 Spec](https://odbplusplus.com/wp-content/uploads/sites/2/2020/03/ODB_Format_Description_v7.pdf) - Format documentation
- Existing Gerber code: `/home/ulti/ulties/ulties/files/lib/gerbers/`
- pcb-tools primitives: `/home/ulti/ulties/src/pcb-tools/gerber/primitives.py`
