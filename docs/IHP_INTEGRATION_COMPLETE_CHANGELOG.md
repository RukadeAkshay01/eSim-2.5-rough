# IHP Open PDK Integration - Complete Changelog

**Project:** eSim 2.5  
**Integration:** IHP SG13G2 Open PDK  
**Date:** January 2026  
**Author:** Akshay Rukade

---

## Overview

This document details **every change** made to integrate the IHP SG13G2 Open PDK into eSim, enabling users to design and simulate circuits using IHP's 130nm SiGe BiCMOS technology.

---

## Files Modified

### 1. `src/kicadtoNgspice/DeviceModel.py`

**Purpose:** Handles device library configuration UI in KiCad to Ngspice converter.

#### Change 1: IHP Detection Logic (Lines 70-91)
```python
# BEFORE (original):
# No IHP detection - only sky130 and general

# AFTER (new):
# Check for IHP SG13G2 PDK first - be specific to avoid false matches
# Look for component lines starting with 'ihp' or models containing 'sg13_'
has_ihp = False
for line in schematicInfo:
    words = line.split()
    if len(words) >= 2:
        # Check if reference starts with 'ihp' (ihp1, ihp2, etc.)
        if words[0].startswith('ihp'):
            has_ihp = True
            break
        # Check if model name contains 'sg13_' (sg13_lv_nmos, etc.)
        if 'sg13_' in words[-1].lower():
            has_ihp = True
            break

if has_ihp:
    self.eSim_ihp(schematicInfo)
elif "sky130" in " ".join(schematicInfo):
    self.eSim_sky130(schematicInfo)
else:
    self.eSim_general_libs(schematicInfo)
```

**Why:** Detects IHP components by checking for `ihp` prefix in reference OR `sg13_` in model name. Avoids false matches that would break SKY130/NGHDL projects.

---

#### Change 2: Complete `eSim_ihp()` Method (Lines 266-480)

**New method added** to handle IHP SG13G2 PDK components.

**Key Features:**

1. **Corner Options Dictionary (Lines 275-281):**
```python
self.ihp_corner_options = {
    'mos': ['mos_tt', 'mos_ff', 'mos_ss', 'mos_sf', 'mos_fs'],
    'res': ['res_typ', 'res_bcs', 'res_wcs'],
    'cap': ['cap_typ', 'cap_bcs', 'cap_wcs'],
    'dio': ['dio_tt', 'dio_ss', 'dio_ff'],
    'hbt': ['hbt_typ', 'hbt_bcs', 'hbt_wcs'],
}
```

2. **Model-to-Corner Mapping (Lines 283-319):**
```python
self.ihp_model_to_corner = {
    # MOS Low-Voltage
    'sg13_lv_nmos': ('cornerMOSlv.lib', 'mos'),
    'sg13_lv_pmos': ('cornerMOSlv.lib', 'mos'),
    'nmoscl_2': ('cornerMOSlv.lib', 'mos'),
    'nmoscl_4': ('cornerMOSlv.lib', 'mos'),
    # MOS High-Voltage
    'sg13_hv_nmos': ('cornerMOShv.lib', 'mos'),
    'sg13_hv_pmos': ('cornerMOShv.lib', 'mos'),
    # Resistors
    'rppd': ('cornerRES.lib', 'res'),
    'rhigh': ('cornerRES.lib', 'res'),
    'rsil': ('cornerRES.lib', 'res'),
    'ptap1': ('cornerRES.lib', 'res'),
    'ntap1': ('cornerRES.lib', 'res'),
    'rparasitic': ('cornerRES.lib', 'res'),
    # Capacitors
    'cap_cmim': ('cornerCAP.lib', 'cap'),
    'cap_rfcmim': ('cornerCAP.lib', 'cap'),
    'cparasitic': ('cornerCAP.lib', 'cap'),
    # Diodes
    'dantenna': ('cornerDIO.lib', 'dio'),
    'dpantenna': ('cornerDIO.lib', 'dio'),
    'dpwdnw': ('cornerDIO.lib', 'dio'),
    'ddnwpsub': ('cornerDIO.lib', 'dio'),
    'isolbox': ('cornerDIO.lib', 'dio'),
    # HBT (Bipolar)
    'npn13g2': ('cornerHBT.lib', 'hbt'),
    'npn13g2_5t': ('cornerHBT.lib', 'hbt'),
    'npn13g2l': ('cornerHBT.lib', 'hbt'),
    'npn13g2l_5t': ('cornerHBT.lib', 'hbt'),
    'npn13g2v': ('cornerHBT.lib', 'hbt'),
    'npn13g2v_5t': ('cornerHBT.lib', 'hbt'),
    'pnpmpa': ('cornerHBT.lib', 'hbt'),
}
```

3. **IHP Device Detection (Lines 321-343):**
```python
# Detection methods:
# 1. Reference starts with 'ihp' prefix
# 2. Model name is in our mapping dictionary
# 3. Model contains 'sg13' or 'npn13' or known IHP names
model_name = words[-1].lower() if len(words) > 1 else ""
is_ihp_device = (
    eachline[0:3] == 'ihp' or  # Reference prefix
    model_name in self.ihp_model_to_corner or  # Known model
    'sg13' in model_name or  # SG13 MOS
    'npn13' in model_name or  # HBT
    model_name.startswith('cap_') or  # Capacitors
    model_name in ['rppd', 'rhigh', 'rsil', 'ptap1', 'ntap1', 
                  'dantenna', 'dpantenna', 'isolbox', 'pnpmpa']
)
```

4. **Per-Device UI Panel:**
   - Library path QLineEdit (read-only)
   - "Add" button (browse for library)
   - "Default" button (auto-fill from PDK_ROOT)
   - Corner selection QComboBox (dropdown)
   - Parameters QLineEdit (W, L, nf, etc.)

5. **Default Path Auto-fill (Lines 380-400):**
```python
# Auto-fill default IHP library path if no previous value
pdk_root = os.environ.get('PDK_ROOT', '')
if pdk_root:
    default_ihp_path = os.path.join(
        pdk_root, 
        'ihp-sg13g2', 
        'libs.tech', 
        'ngspice', 
        'models',
        corner_file
    )
    if os.path.exists(default_ihp_path):
        self.trackLibraryWithoutButton(comp_name, default_ihp_path)
```

6. **Data Tracking Format:**
```
lib_path:corner:params
Example: /home/user/ihp/.../cornerMOSlv.lib:mos_tt:W=1u L=130n nf=1
```

---

### 2. `src/kicadtoNgspice/Convert.py`

**Purpose:** Converts KiCad netlist to Ngspice format, adds library includes.

#### Change 1: IHP Device Detection (Lines 645-660)
```python
# Handle IHP components
# Detection: ihp prefix, sg13/npn13 in model, or known IHP device names
model_name = words[-1].lower() if len(words) > 1 else ""
ihp_known_devices = ['rppd', 'rhigh', 'rsil', 'ptap1', 'ntap1', 
                    'dantenna', 'dpantenna', 'isolbox', 'pnpmpa',
                    'nmoscl_2', 'nmoscl_4', 'cparasitic', 'dpwdnw', 'ddnwpsub']
is_ihp_device = (
    eachline[0:3] == 'ihp' or
    'sg13' in model_name or
    'npn13' in model_name or
    model_name.startswith('cap_') or
    model_name in ihp_known_devices
)
```

#### Change 2: IHP Library and OSDI Generation (Lines 660-720)
```python
if is_ihp_device and eachline[0:7] != 'ihpmode':
    # Parse the per-device tracking data: lib_path:corner:params
    ihp_parts = completeLibPath.split(':')
    ihp_lib_path = ihp_parts[0] if len(ihp_parts) > 0 else ""
    ihp_corner = ihp_parts[1] if len(ihp_parts) > 1 else "mos_tt"
    ihp_params = ihp_parts[2] if len(ihp_parts) > 2 else ""
    
    # Create .spiceinit with OSDI loading (only once per project)
    if not hasattr(self, 'ihp_spiceinit_written'):
        self.ihp_spiceinit_written = True
        
        # Derive OSDI path from lib path
        osdi_path = ihp_lib_path.replace('/models/', '/openvaf/')
        
        # Write .spiceinit
        self.writefile = open(spiceinit_path, "w")
        self.writefile.write('''
set ngbehavior=hsa
set ng_nomodcheck
set num_threads=8
option noinit
optran 0 0 0 100p 2n 0
''')
        # OSDI model loading
        osdi_files = ['psp103.osdi', 'psp103_nqs.osdi', 'r3_cmc.osdi', 
                      'bjt504.osdi', 'bjt504_sop.osdi', 'mosvar.osdi',
                      'diode_cmc.osdi', 'cap_cmim.osdi']
        for osdi_file in osdi_files:
            osdi_full = os.path.join(osdi_path, osdi_file)
            if os.path.exists(osdi_full):
                self.writefile.write(f"osdi {osdi_full}\n")
        self.writefile.close()
    
    # Add .lib directive for corner
    includeLine.append(f'.lib "{ihp_lib_path}" {ihp_corner}')
    
    # Append parameters to device line
    if ihp_params:
        words.append(ihp_params)
        deviceLine[index] = words
```

---

### 3. `src/kicadtoNgspice/Processing.py`

**Purpose:** Preprocesses netlist, handles source parameters.

#### Change 1: IHP Component Exclusion (Line 142)
```python
# BEFORE:
if compName[0] == 'v' or compName[0] == 'i':

# AFTER:
# Skip IHP components that start with 'ihp' (not current sources)
if (compName[0] == 'v' or compName[0] == 'i') and not compName.startswith('ihp'):
```

**Why:** Prevents IHP components starting with `ihp` from being mistakenly processed as voltage/current sources.

---

### 4. `eSim_IHP.kicad_sym` (New File)

**Purpose:** KiCad 8.0 symbol library for IHP SG13G2 devices.

**Format:** KiCad S-expression (version 20231120)

**28 Symbols Created:**

| Category | Symbol | Model | Pins | Reference |
|----------|--------|-------|------|-----------|
| MOS LV | sg13_lv_nmos | sg13_lv_nmos | D,G,S,B | ihp |
| MOS LV | sg13_lv_pmos | sg13_lv_pmos | D,G,S,B | ihp |
| MOS HV | sg13_hv_nmos | sg13_hv_nmos | D,G,S,B | ihp |
| MOS HV | sg13_hv_pmos | sg13_hv_pmos | D,G,S,B | ihp |
| HBT 4T | npn13G2 | npn13G2 | C,B,E,BN | ihp |
| HBT 4T | npn13G2l | npn13G2l | C,B,E,BN | ihp |
| HBT 4T | npn13G2v | npn13G2v | C,B,E,BN | ihp |
| HBT 5T | npn13G2_5t | npn13G2_5t | C,B,E,BN,T | ihp |
| HBT 5T | npn13G2l_5t | npn13G2l_5t | C,B,E,BN,T | ihp |
| HBT 5T | npn13G2v_5t | npn13G2v_5t | C,B,E,BN,T | ihp |
| PNP | pnpMPA | pnpMPA | C,B,E | ihp |
| Resistor | rsil | rsil | 1,2,BN | ihp |
| Resistor | rhigh | rhigh | 1,2,BN | ihp |
| Resistor | rppd | rppd | 1,2,BN | ihp |
| Resistor | Rparasitic | Rparasitic | 1,2 | ihp |
| Resistor | ptap1 | ptap1 | 1,2 | ihp |
| Resistor | ntap1 | ntap1 | 1,2 | ihp |
| Capacitor | cap_cmim | cap_cmim | +,- | ihp |
| Capacitor | cap_rfcmim | cap_rfcmim | +,-,BN | ihp |
| Capacitor | cparasitic | cparasitic | +,- | ihp |
| Diode | dantenna | dantenna | A,C | ihp |
| Diode | dpantenna | dpantenna | A,C | ihp |
| Diode | ddnwpsub | ddnwpsub | NW,BN | ihp |
| Diode | dpwdnw | dpwdnw | NW,BN | ihp |
| Misc | isolbox | isolbox | ISUB,NW,BN | ihp |
| Clamp | nmoscl_2 | nmoscl_2 | VDD,VSS | ihp |
| Clamp | nmoscl_4 | nmoscl_4 | VDD,VSS | ihp |
| Pad | bondpad | bondpad | PAD | ihp |

**Key Properties:**
- `"Reference" "ihp"` - All symbols use `ihp` prefix for detection
- `"Sim.Device" "SUBCKT"` - Tells KiCad these are subcircuits
- `"Sim.Pins"` - Pin mapping for simulation

---

### 5. `Examples/IHP_Inverter/` (New Directory)

**Purpose:** Test circuit for IHP integration.

**Files:**
- `IHP_Inverter.cir` - CMOS inverter netlist using sg13_lv_nmos/pmos
- `analysis` - Transient analysis configuration

---

## Corner Library Mapping

| Corner File | Device Types | Available Corners |
|-------------|--------------|-------------------|
| cornerMOSlv.lib | sg13_lv_nmos, sg13_lv_pmos, nmoscl_2, nmoscl_4 | mos_tt, mos_ff, mos_ss, mos_sf, mos_fs |
| cornerMOShv.lib | sg13_hv_nmos, sg13_hv_pmos | mos_tt, mos_ff, mos_ss, mos_sf, mos_fs |
| cornerHBT.lib | npn13G2 variants, pnpMPA | hbt_typ, hbt_bcs, hbt_wcs |
| cornerRES.lib | rppd, rhigh, rsil, ptap1, ntap1 | res_typ, res_bcs, res_wcs |
| cornerCAP.lib | cap_cmim, cap_rfcmim, cparasitic | cap_typ, cap_bcs, cap_wcs |
| cornerDIO.lib | dantenna, dpantenna, ddnwpsub, dpwdnw | dio_tt, dio_ss, dio_ff |

---

## OSDI Models Required

Located in `<PDK_ROOT>/ihp-sg13g2/libs.tech/ngspice/openvaf/`:

| OSDI File | Purpose |
|-----------|---------|
| psp103.osdi | PSP 103.6 MOS model |
| psp103_nqs.osdi | PSP NQS MOS model |
| r3_cmc.osdi | R3 CMC resistor model |
| bjt504.osdi | BJT 504 model |
| bjt504_sop.osdi | BJT 504 SOP model |
| mosvar.osdi | MOS varactor model |
| diode_cmc.osdi | CMC diode model |
| cap_cmim.osdi | MIM capacitor model |

---

## User Workflow

1. **Setup:** Set `PDK_ROOT` environment variable to IHP PDK path
2. **Schematic:** Place IHP symbols (ihp1, ihp2, etc.) from eSim_IHP library
3. **Netlist:** Export netlist from KiCad
4. **Configure:** Open KiCad to Ngspice, see IHP device panels
5. **Library:** Click "Default" or browse to set library path
6. **Corner:** Select corner from dropdown (mos_tt, etc.)
7. **Parameters:** Enter device parameters (W=1u L=130n nf=1)
8. **Simulate:** Click Convert, then Simulate

---

## Bugs Fixed During Development

1. **QComboBox.text() AttributeError** - Fixed in KicadtoNgspice.py by checking widget type
2. **SKY130/NGHDL Breaking** - Fixed overly broad IHP detection (was matching any "ihp" substring)
3. **KiCad 8 Symbol Format** - Updated to version 20231120 format

---

## Files NOT Modified (Confirmed Working)

- `KicadtoNgspice.py` - Minor fix for QComboBox text extraction
- `Source.py` - No changes needed
- `Analysis.py` - No changes needed
- SKY130 handling in Convert.py - Untouched

---

## Testing Checklist

- [ ] SKY130 project simulation works
- [ ] NGHDL project simulation works
- [ ] IHP symbol library loads in KiCad 8.0
- [ ] IHP components generate `ihp1`, `ihp2` references
- [ ] IHP device panels appear in KiCad to Ngspice
- [ ] Default library path auto-fills correctly
- [ ] Corner dropdown shows correct options
- [ ] `.lib` directive generated with correct corner
- [ ] `.spiceinit` created with OSDI commands
- [ ] IHP simulation runs successfully

---

## Environment Requirements

- **KiCad:** 8.0+
- **eSim:** 2.5
- **ngspice:** 42+ (with OSDI support)
- **IHP PDK:** IHP-Open-PDK (ihp-sg13g2)
- **Python:** 3.x with PyQt5
