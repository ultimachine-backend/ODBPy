# ODBPy Fixes for Ulties Integration

This fork fixes several issues found when parsing ODB++ files exported from Altium/ECAD tools.

## POC Results - ALL PASSING

| Feature | Status | Details |
|---------|--------|---------|
| Layer matrix | ✅ Works | 45 layers including DIELECTRIC |
| Pads | ✅ Works | 35 top, 38 bottom, 32 drill holes |
| Lines | ✅ Works | Traces and board outline |
| Components | ✅ Works | 5 top-side, 2 bottom-side with properties |
| Symbols | ✅ Works | Round, Square, Rectangle |

## Bugs Fixed

### 1. Pad Orientation Regex (LayerFeatureParser.py)
**Issue**: Original regex `[1-7]` rejected orientation value `0` (no rotation)
**Fix**: Changed to `[0-7]` to allow orientation 0
**File**: `ODBPy/LayerFeatureParser.py` line 17

### 2. Pad Attribute Whitespace (LayerFeatureParser.py)
**Issue**: Regex didn't allow space before semicolon in attributes
**Fix**: Added `\s*` before attribute group
**File**: `ODBPy/LayerFeatureParser.py` line 18

### 3. DIELECTRIC Layer Type (Layers.py)
**Issue**: `DIELECTRIC` layer type not in `_layer_type_map`
**Fix**: Added `Dielectric` and `PowerGround` to `LayerType` enum and map
**File**: `ODBPy/Layers.py` lines 85-100

### 4. Rectangle Symbol Pattern (StandardSymbols.py)
**Issue**: Pattern `r([\.\d]+)x([\.\d]+)` doesn't match `rect39.3701x110.2362`
**Fix**: Changed to `rect?([\.\d]+)x([\.\d]+)` to match both `r` and `rect` prefixes
**File**: `ODBPy/StandardSymbols.py` line 57-58

### 5. File Encoding (Utils.py)
**Issue**: Assumes UTF-8, fails on extended ASCII (e.g., degree symbol 0xB0)
**Fix**: Added encoding fallback to latin-1 when UTF-8 fails
**File**: `ODBPy/Utils.py` lines 31-37

### 6. Component Parser Section Filter (ComponentParser.py)
**Issue**: Parser tried to process non-CMP sections (attribute headers)
**Fix**: Added `name.startswith("CMP")` filter
**File**: `ODBPy/ComponentParser.py` line 103

### 7. Component Parser Missing Import (ComponentParser.py)
**Issue**: `parse_attributes` function not imported
**Fix**: Added import from Attributes module
**File**: `ODBPy/ComponentParser.py` line 11

### 8. Attribute Parser Numeric Values (Attributes.py)
**Issue**: `int()` failed on float values like `0.1799` (component height)
**Fix**: Added flexible `_parse_numeric()` function that tries int, then float
**File**: `ODBPy/Attributes.py` lines 22-30

## Test File

Tested with: `ATS_Commercial_IRLED-B_ODB.tgz` (Altium export)
- 2-layer board (LAYER_1_TOP, LAYER_2_BOTTOM)
- 7 components: MTG1, MTG2 (mounting holes), D1 (LED), R1, R2, R3, R4
- Component properties: Manufacturer, PN, Package, etc.
- 32 drill holes, 35+ pads per layer

## Usage

```python
import sys
sys.path.insert(0, '/path/to/ODBPy')

from ODBPy.Layers import read_layers
from ODBPy.LineRecordParser import read_linerecords
from ODBPy.LayerFeatureParser import _features_decoder_options, Pad, Line
from ODBPy.ComponentParser import parse_components
from ODBPy.Decoder import run_decoder

# Read layers
layers = read_layers('/path/to/odb++')

# Read features from a layer
linerecords = read_linerecords('/path/to/odb++/steps/pcb/layers/layer_1_top/features')
features = list(run_decoder(linerecords['Layer features'], _features_decoder_options))
pads = [f for f in features if isinstance(f, Pad)]
lines = [f for f in features if isinstance(f, Line)]

# Read components
comp_records = read_linerecords('/path/to/odb++/steps/pcb/layers/comp_+_top/components')
components = parse_components(comp_records)
```

## Next Steps

1. Add surface/polygon parsing for copper pours
2. Integrate with pcb-tools primitives for Cairo rendering
