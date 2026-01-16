# IHP Open PDK Integration - File Changes Summary

**Date:** 2026-01-16  
**Author:** Akshay Rukade  
**Project:** eSim IHP SG13G2 PDK Integration  

---

## Overview

This document summarizes all file modifications made to integrate IHP Open PDK (SG13G2) support into the eSim KiCad-to-Ngspice conversion pipeline.

---

## Files Modified

### 1. `src/kicadtoNgspice/DeviceModel.py`

**Purpose:** Device model library management and GUI generation

#### Changes Made:

| Line(s) | Change Description |
|---------|-------------------|
| 71-77 | Added IHP detection logic before SKY130 check |
| 252-456 | Added `eSim_ihp()` method for IHP PDK handling |
| 458-470 | Added `trackDefaultIHPLib()` helper method |
| 472-494 | Added `trackIHPLibrary()` helper method |
| 913-924 | Updated `textChange()` to handle `ihpmode` and `ihp` components |

#### Code Additions:

**Detection Logic (lines 71-77):**
```python
# Check for IHP SG13G2 PDK first
if "ihp" in " ".join(schematicInfo) or "sg13g2" in " ".join(schematicInfo):
    self.eSim_ihp(schematicInfo)
elif "sky130" in " ".join(schematicInfo):
    self.eSim_sky130(schematicInfo)
else:
    self.eSim_general_libs(schematicInfo)
```

**`eSim_ihp()` method:** Creates GUI for:
- IHP PDK models path selection
- Corner selection (mos_tt, mos_ff, etc.)
- Component parameter entry (W, L, nf)

**`trackDefaultIHPLib()`:** Sets default path to `~/ihp/IHP-Open-PDK/ihp-sg13g2/libs.tech/ngspice/models`

**`trackIHPLibrary()`:** File browser for custom IHP PDK location

**`textChange()` additions:**
```python
if self.deviceName[0:7] == 'ihpmode':
    # IHP mode: path:corner format
    self.obj_trac.deviceModelTrack[self.deviceName] = \
        self.entry_var[self.widgetObjCount].text() + \
        ":" + str(self.entry_var[self.widgetObjCount + 1].text())
elif self.deviceName[0:3] == 'ihp':
    # IHP component: store parameters directly
    self.obj_trac.deviceModelTrack[self.deviceName] = str(
        self.entry_var[self.widgetObjCount].text())
```

---

### 2. `src/kicadtoNgspice/Convert.py`

**Purpose:** Netlist conversion and library inclusion

#### Changes Made:

| Line(s) | Change Description |
|---------|-------------------|
| 705-752 | Added IHP mode handling (`ihpmode` and `ihp` components) |

#### Code Additions:

**`ihpmode` handling (lines 705-741):**
- Creates `.spiceinit` file with IHP compatibility settings
- Includes corner library files with `.lib` directive:
  - `cornerMOSlv.lib` → Low-voltage MOS (mos_tt)
  - `cornerMOShv.lib` → High-voltage MOS
  - `cornerRES.lib` → Resistors (res_typ)
  - `cornerCAP.lib` → Capacitors (cap_typ)
  - `cornerDIO.lib` → Diodes (dio_typ)
  - `cornerHBT.lib` → HBT (hbt_typ)

**`ihp` component handling (lines 743-749):**
```python
elif eachline[0:3] == 'ihp' and eachline[0:7] != 'ihpmode':
    # IHP component: convert ihp1 to xihp1
    words[0] = words[0].replace('ihp', 'xihp')
    # Append parameters from completeLibPath
    if completeLibPath:
        words.append(completeLibPath)
    deviceLine[index] = words
```

---

### 3. `src/kicadtoNgspice/Processing.py`

**Purpose:** Netlist preprocessing and parsing

#### Changes Made:

| Line(s) | Change Description |
|---------|-------------------|
| 140-141 | Added IHP exclusion from source processing |

#### Code Additions:

**IHP exclusion (line 141):**
```python
# Skip IHP components that start with 'ihp' (not current sources)
if (compName[0] == 'v' or compName[0] == 'i') and not compName.startswith('ihp'):
```

**Reason:** `ihpmode1` starts with 'i' but is NOT a current source. Without this fix, the code tries to access `words[3]` which doesn't exist, causing IndexError.

---

## New Files Created

### 1. `Examples/IHP_Inverter/IHP_Inverter.cir`

**Purpose:** Test netlist for IHP integration

```spice
* Simple CMOS Inverter with IHP SG13G2 PDK

ihpmode1  gnd sg13_lv_nmos
ihp1  out in vdd vdd sg13_lv_pmos
ihp2  out in gnd gnd sg13_lv_nmos
v1  vdd gnd dc
v2  in gnd pulse
U1  out plot_v1
U2  in plot_v1
U3  vdd plot_v1

.end
```

### 2. `Examples/IHP_Inverter/analysis`

**Purpose:** Transient analysis configuration

```
tran
10e-12
100e-09
0
```

### 3. `docs/ESIM_KICAD_TO_NGSPICE_PIPELINE_DOCUMENTATION.md`

**Purpose:** Comprehensive pipeline documentation (~21 KB)

Contents:
- Complete pipeline flow diagram
- File format transformations
- SKY130 PDK implementation details
- Key source files explained
- IHP integration requirements
- Implementation roadmap

### 4. `docs/IHP_PDK_INTEGRATION_CHECKLIST.md`

**Purpose:** Step-by-step implementation guide (~15 KB)

Contents:
- Code snippets for all modifications
- Model XML templates
- KiCad symbol requirements
- Testing checklist

---

## Component Naming Convention

| Designator | Component Type | Example |
|------------|---------------|---------|
| `ihpmode1` | IHP mode marker | `ihpmode1 gnd sg13_lv_nmos` |
| `ihp1`, `ihp2` | IHP transistors | `ihp1 out in vdd vdd sg13_lv_pmos` |
| `v1`, `v2` | Voltage sources | `v1 vdd gnd dc` |
| `U1`, `U2` | Plot markers | `U1 out plot_v1` |

---

## Expected Output Format

### Input (`.cir`):
```spice
ihpmode1  gnd sg13_lv_nmos
ihp1  out in vdd vdd sg13_lv_pmos
ihp2  out in gnd gnd sg13_lv_nmos
```

### Output (`.cir.out`):
```spice
.lib "/path/to/cornerMOSlv.lib" mos_tt
.lib "/path/to/cornerRES.lib" res_typ
...

*ihpmode
xihp1 out in vdd vdd sg13_lv_pmos W=2u L=130n nf=1
xihp2 out in gnd gnd sg13_lv_nmos W=1u L=130n nf=1
```

---

## IHP PDK Path Structure

```
$PDK_ROOT/ihp-sg13g2/libs.tech/ngspice/models/
├── cornerMOSlv.lib      ← .lib "..." mos_tt
├── cornerMOShv.lib      ← .lib "..." mos_tt
├── cornerRES.lib        ← .lib "..." res_typ
├── cornerCAP.lib        ← .lib "..." cap_typ
├── cornerDIO.lib        ← .lib "..." dio_typ
├── cornerHBT.lib        ← .lib "..." hbt_typ
├── sg13g2_moslv_mod.lib ← (included by corner files)
└── sg13g2_moshv_mod.lib ← (included by corner files)
```

---

## Available Corners

| Corner File | Corners Available |
|-------------|-------------------|
| `cornerMOSlv.lib` | mos_tt, mos_ss, mos_ff, mos_sf, mos_fs, *_mismatch |
| `cornerRES.lib` | res_typ, res_bcs, res_wcs, *_mismatch |
| `cornerCAP.lib` | cap_typ, cap_bcs, cap_wcs |
| `cornerDIO.lib` | dio_typ, dio_bcs, dio_wcs |
| `cornerHBT.lib` | hbt_typ, hbt_bcs, hbt_wcs |

---

## Testing Instructions

1. Open eSim
2. Navigate to `Examples/IHP_Inverter/`
3. Click **KiCad to Ngspice** converter
4. Verify IHP mode panel appears
5. Set PDK path: `/home/<user>/ihp/IHP-Open-PDK/ihp-sg13g2/libs.tech/ngspice/models`
6. Set corner: `mos_tt` (or leave empty for default)
7. Enter parameters for `ihp1` and `ihp2`: `W=1u L=130n nf=1`
8. Click **Convert**
9. Check `.cir.out` file for correct output

---

## Future Enhancements (Phase 2+)

- [ ] Create KiCad symbol library for IHP components
- [ ] Add model XML files for IHP device parameters
- [ ] Integrate with IHP KLayout/Magic for layout
- [ ] Add IHP standard cell library support
- [ ] Automated corner sweep functionality

---

## Phase 1 Status: ✅ COMPLETE

**Tested:** 2026-01-16  
**Result:** Simulation runs successfully with IHP SG13G2 PDK

### Verified Working:
- IHP detection triggers correctly
- `.lib` includes generated for all corner files
- `.spiceinit` includes OSDI loading
- `ihp1` → `xihp1` prefix conversion
- Raw data file generated (`ihp_test.raw`)

### Correct IHP Corner Names:
| File | Section |
|------|---------|
| cornerMOSlv.lib | mos_tt |
| cornerMOShv.lib | mos_tt |
| cornerRES.lib | res_typ |
| cornerCAP.lib | cap_typ |
| cornerDIO.lib | **dio_tt** |
| cornerHBT.lib | hbt_typ |

### OSDI Files Required:
- `psp103.osdi` - PSP MOS model
- `psp103_nqs.osdi` - PSP NQS model
- `r3_cmc.osdi` - Resistor model
- `mosvar.osdi` - MOS varactor model

---

*Document Version: 1.1*  
*Last Updated: 2026-01-16 01:43*
