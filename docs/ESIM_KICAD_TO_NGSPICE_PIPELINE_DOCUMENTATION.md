# eSim KiCad to Ngspice Pipeline Documentation

## Table of Contents
1. [Overview](#overview)
2. [Complete Pipeline Flow](#complete-pipeline-flow)
3. [File Formats and Transformations](#file-formats-and-transformations)
4. [SKY130 PDK Integration (Current Implementation)](#sky130-pdk-integration-current-implementation)
5. [Key Source Files and Their Responsibilities](#key-source-files-and-their-responsibilities)
6. [IHP Open PDK Integration Requirements](#ihp-open-pdk-integration-requirements)
7. [Implementation Roadmap for IHP PDK](#implementation-roadmap-for-ihp-pdk)

---

## Overview

eSim (Electronic Simulation) is an open-source EDA tool developed by FOSSEE, IIT Bombay. It integrates:
- **KiCad** for schematic capture
- **Ngspice** for circuit simulation
- **Python/PyQt5** for the GUI and conversion pipeline

The core workflow converts a KiCad schematic into a Ngspice-compatible netlist (`.cir.out` file), adding device models, subcircuits, analysis commands, and control statements.

---

## Complete Pipeline Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        eSim KiCad to Ngspice Pipeline                       │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│   KiCad ESchema  │ -> │  Netlist Export  │ -> │   .cir File      │
│   (Schematic)    │    │  (Spice Format)  │    │   (Raw Netlist)  │
└──────────────────┘    └──────────────────┘    └──────────────────┘
                                                        │
                                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     KiCad to Ngspice Converter GUI                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        Tab-Based Interface                           │   │
│  │  ┌────────────┐ ┌──────────────┐ ┌─────────────┐ ┌────────────────┐ │   │
│  │  │  Analysis  │ │Source Details│ │Ngspice Model│ │Device Modeling │ │   │
│  │  └────────────┘ └──────────────┘ └─────────────┘ └────────────────┘ │   │
│  │  ┌─────────────┐ ┌─────────────────┐                                 │   │
│  │  │ Subcircuits │ │ Microcontroller │                                 │   │
│  │  └─────────────┘ └─────────────────┘                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Convert.py Processing                              │
│  1. addSourceParameter()    - Add voltage/current source values            │
│  2. addModelParameter()     - Add Ngspice model statements                  │
│  3. addMicrocontrollerParameter() - Add NGHDL/Verilog models               │
│  4. addDeviceLibrary()      - Add device model libraries (.lib/.spice)     │
│  5. addSubcircuit()         - Add subcircuit references (.sub)             │
│  6. analysisInsertor()      - Create analysis file (.ac/.dc/.tran)         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│ .cir.out File    │ -> │   Ngspice        │ -> │ Simulation       │
│ (Final Netlist)  │    │   Execution      │    │ Results/Plots    │
└──────────────────┘    └──────────────────┘    └──────────────────┘
```

---

## File Formats and Transformations

### 1. Input: `.cir` File (KiCad-exported Netlist)

```spice
* /home/user/eSim-Workspace/RC/RC.cir

* EESchema Netlist Version 1.1 (Spice format)

* Sheet Name: /
R1  in out 1k		
C1  out GND 10u		
v1  in GND pwl		
U1  in plot_v1		
U2  out plot_v1		

.end
```

**Key Observations:**
- First line is the file path (becomes info line)
- Component lines follow SPICE format: `<RefDesignator> <Nodes...> <Value/Model>`
- `U` prefix components are special (plot markers, ICs, subcircuits)
- Source types (pwl, sine, pulse, etc.) need parameter completion

### 2. Output: `.cir.out` File (Ngspice-ready Netlist)

```spice
* /home/fossee/esim-workspace/rc/rc.cir

r1  in out 1k
c1  out gnd 10u
v1  in gnd pwl(0m 0 0.5m 5 50m 5 50.5m 0 100m 0)
* u1  in plot_v1
* u2  out plot_v1
.tran 10e-03 100e-03 0e-03

* Control Statements 
.control
run
print allv > plot_data_v.txt
print alli > plot_data_i.txt
plot v(in)
plot v(out)
.endc
.end
```

**Transformations Applied:**
1. All text converted to lowercase
2. Source parameters filled in (pwl values)
3. Plot markers commented out and converted to control statements
4. Analysis commands added (`.tran`, `.ac`, `.dc`)
5. Control block added with `run` and `print` commands

---

## SKY130 PDK Integration (Current Implementation)

### Detection Mechanism
In `DeviceModel.py`, SKY130 mode is detected by checking if "sky130" appears in the schematic:

```python
# Line 71 in DeviceModel.py
if "sky130" in " ".join(schematicInfo):
    self.eSim_sky130(schematicInfo)
else:
    self.eSim_general_libs(schematicInfo)
```

### Component Designator Prefixes

| Prefix | Component Type | SKY130 Usage |
|--------|---------------|--------------|
| `sc` | SKY130 Component | MOSFET/devices with parameters |
| `scmode` | SKY130 Mode Marker | Library path and corner selection |
| `x` | Subcircuit Instance | Standard subcircuit calls |
| `u` | IC/Special Block | Plot markers, transformers, etc. |
| `v`, `i` | Sources | Voltage/Current sources |
| `a` | Analog Block | Ngspice analog model instances |

### SKY130 Library Path Configuration

**Default Paths:**
```python
# Windows
path_name = "library/sky130_fd_pr/models/sky130.lib.spice"

# Linux
path_name = "/usr/share/local/sky130_fd_pr/models/sky130.lib.spice"
```

### SKY130-Specific Ngspice Configuration

When SKY130 is detected, a `.spiceinit` file is created in the project directory:

```spice
set ngbehavior=hsa     ; set compatibility for reading PDK libs
set ng_nomodcheck      ; don't check the model parameters
set num_threads=8      ; CPU hardware threads available
option noinit          ; don't print operating point data
optran 0 0 0 100p 2n 0 ; don't use dc operating point, but transient op
```

### SKY130 Library Includes

```spice
.lib "<path>/sky130.lib.spice" tt

.include "<path>/sky130_fd_pr__model__diode_pd2nw_11v0.model.spice"
.include "<path>/sky130_fd_pr__model__diode_pw2nd_11v0.model.spice"
.include "<path>/sky130_fd_pr__model__inductors.model.spice"
.include "<path>/sky130_fd_pr__model__linear.model.spice"
.include "<path>/sky130_fd_pr__model__pnp.model.spice"
.include "<path>/sky130_fd_pr__model__r+c.model.spice"
```

### SKY130 Component Conversion

In `Convert.py`, component lines are processed:

```python
# Lines 700-703 in Convert.py
elif eachline[0:2] == 'sc' and eachline[0:6] != 'scmode':
    words[0] = words[0].replace('sc', 'xsc')  # Convert sc1 to xsc1
    words.append(completeLibPath)  # Add library path
    deviceLine[index] = words
```

**Example Transformation:**
```
Input:  sc1 vdd vss gate drain sky130_fd_pr__nfet_01v8
Output: xsc1 vdd vss gate drain sky130_fd_pr__nfet_01v8
```

---

## Key Source Files and Their Responsibilities

### 1. `src/kicadtoNgspice/KicadtoNgspice.py`
**Main Entry Point for Conversion**

| Function | Purpose |
|----------|---------|
| `__init__()` | Initialize processing, read netlist, detect models |
| `createMainWindow()` | Create tabbed conversion interface |
| `callConvert()` | Execute all conversion steps and save to XML |
| `createNetlistFile()` | Write final `.cir.out` file |
| `createSubFile()` | Generate `.sub` file for subcircuit export |

### 2. `src/kicadtoNgspice/Processing.py`
**Netlist Preprocessing**

| Function | Purpose |
|----------|---------|
| `readNetlist()` | Read `.cir` file and split into lines |
| `readParamInfo()` | Extract `.param` definitions |
| `preprocessNetlist()` | Replace parameters, handle multiline |
| `separateNetlistInfo()` | Separate options from schematic |
| `insertSpecialSourceParam()` | Handle special source types |
| `convertICintoBasicBlocks()` | Parse U-components, load model XML |

### 3. `src/kicadtoNgspice/Convert.py`
**Conversion Logic**

| Function | Purpose |
|----------|---------|
| `addSourceParameter()` | Add sine/pulse/pwl/ac/dc parameters |
| `addModelParameter()` | Add Ngspice model statements |
| `addMicrocontrollerParameter()` | Add NGHDL microcontroller models |
| `addDeviceLibrary()` | Include device model libraries |
| `addSubcircuit()` | Include subcircuit files |
| `analysisInsertor()` | Write analysis commands |

### 4. `src/kicadtoNgspice/DeviceModel.py`
**Device Library Management**

| Function | Purpose |
|----------|---------|
| `eSim_sky130()` | Handle SKY130 PDK components |
| `eSim_general_libs()` | Handle standard device libraries |
| `trackLibrary()` | Track user-selected libraries |
| `trackDefaultLib()` | Set default SKY130 library path |
| `textChange()` | Handle parameter text changes |

### 5. `src/kicadtoNgspice/SubcircuitTab.py`
**Subcircuit Management**

| Function | Purpose |
|----------|---------|
| `trackSubcircuit()` | Validate and track subcircuit paths |
| `trackSubcircuitWithoutButton()` | Auto-load from previous values |

### 6. `library/modelParamXML/`
**Model Parameter Definitions (XML)**

Example model XML structure:
```xml
<model>
  <name>gain</name>
  <type>Analog</type>
  <node_number>2</node_number>
  <title>Add Parameters for model Gain</title>
  <split>None</split>
  <param>
    <in_offset default="0.0">Enter offset for input (default=0.0)</in_offset>
    <gain vector="1" default="1.0">Enter gain (default=1.0)</gain>
  </param>
</model>
```

---

## IHP Open PDK Integration Requirements

### 1. PDK Structure Understanding

**IHP Open PDK Directory Structure:**
```
$PDK_ROOT/ihp-sg13g2/
├── libs.tech/
│   ├── ngspice/
│   │   ├── .spiceinit       # Ngspice initialization
│   │   ├── osdi/            # Compiled OSDI modules
│   │   │   ├── psp103_nmos.osdi
│   │   │   ├── psp103_pmos.osdi
│   │   │   └── ...
│   │   └── models/          # SPICE model files
│   ├── klayout/             # KLayout technology files
│   └── verilog-a/           # Verilog-A source files
├── libs.ref/
│   ├── sg13g2_stdcell/      # Standard cells
│   └── sg13g2_io/           # I/O cells
└── tech/                    # Technology files
```

### 2. Key Differences from SKY130

| Feature | SKY130 | IHP SG13G2 |
|---------|--------|------------|
| Model Format | BSIM3/BSIM4 SPICE | PSP/Verilog-A OSDI |
| Library Extension | `.lib.spice` | `.osdi` + `.lib` |
| Ngspice Requirement | Standard | OSDI-enabled (--enable-osdi) |
| Corner Selection | tt, ss, ff, etc. | mos_tt, res_typ, etc. |
| Include Method | `.lib` / `.include` | `osdi` command |

### 3. Required Modifications

#### A. New Component Designator Prefix
Suggested prefix for IHP: `ihp` (e.g., `ihp1`, `ihpmode`)

#### B. New Detection Method in DeviceModel.py

```python
# Proposed addition to DeviceModel.py
def __init__(self, schematicInfo, clarg1):
    ...
    if "ihp" in " ".join(schematicInfo) or "sg13g2" in " ".join(schematicInfo):
        self.eSim_ihp(schematicInfo)
    elif "sky130" in " ".join(schematicInfo):
        self.eSim_sky130(schematicInfo)
    else:
        self.eSim_general_libs(schematicInfo)
```

#### C. IHP-Specific Handler Method

```python
def eSim_ihp(self, schematicInfo):
    """
    Handle IHP SG13G2 PDK components.
    Similar to eSim_sky130 but with:
    - OSDI model loading
    - IHP-specific corners
    - Different library structure
    """
    # Create IHP mode configuration
    ihpbox = QtWidgets.QGroupBox()
    ihpgrid = QtWidgets.QGridLayout()
    ihpbox.setTitle("Add parameters of IHP SG13G2 library")
    
    # Path entry for OSDI directory
    self.parameterLabel[self.count] = QtWidgets.QLabel("Enter OSDI path")
    # Default: $PDK_ROOT/ihp-sg13g2/libs.tech/ngspice/osdi
    
    # Corner selection (mos_tt, mos_ff, mos_ss, etc.)
    self.cornerLabel = QtWidgets.QLabel("Enter corner (e.g., mos_tt)")
    ...
```

#### D. IHP Library Include Generation in Convert.py

```python
# Proposed addition to Convert.py addDeviceLibrary()
elif eachline[0:3] == 'ihp' and eachline[0:7] != 'ihpmode':
    # Convert ihp1 to xihp1
    words[0] = words[0].replace('ihp', 'xihp')
    words.append(completeLibPath)
    deviceLine[index] = words

elif eachline[0:7] == 'ihpmode':
    # Generate .spiceinit for IHP
    self.writefile = open(self.Fileopen.replace('analysis', '.spiceinit'), "w")
    self.writefile.write('''
set ngbehavior=hsa
set ng_nomodcheck
set num_threads=8
option noinit
''')
    
    # Add OSDI loading commands
    osdi_path = tempStr[0]  # Path to OSDI directory
    for osdi_file in ['psp103_nmos.osdi', 'psp103_pmos.osdi', ...]:
        includeLine.append(f'osdi "{osdi_path}/{osdi_file}"')
    
    # Add library include with corner
    includeLine.append(f'.lib "{libAbsPath}" {tempStr[1]}')
```

#### E. IHP .spiceinit Configuration

```spice
* IHP SG13G2 PDK Configuration
set ngbehavior=hsa
set ng_nomodcheck
set num_threads=8
option noinit

* Load OSDI models
osdi "$PDK_ROOT/ihp-sg13g2/libs.tech/ngspice/osdi/psp103_nmos.osdi"
osdi "$PDK_ROOT/ihp-sg13g2/libs.tech/ngspice/osdi/psp103_pmos.osdi"
osdi "$PDK_ROOT/ihp-sg13g2/libs.tech/ngspice/osdi/HICUM_L2V3p0p0.osdi"
osdi "$PDK_ROOT/ihp-sg13g2/libs.tech/ngspice/osdi/DIODE_CMC.osdi"
```

### 4. KiCad Symbol Library for IHP

**Current State:** Symbols have been added to KiCad (as mentioned by user)

**Required Symbol Naming Convention:**
- Reference designator: `ihp1`, `ihp2`, etc.
- Value field: `sg13_lv_nmos`, `sg13_lv_pmos`, etc.
- Footprint: Link to IHP KLayout/Magic cells

**Example Symbol Value Mapping:**
```
sg13_lv_nmos    -> IHP low-voltage NMOS
sg13_lv_pmos    -> IHP low-voltage PMOS
sg13_hv_nmos    -> IHP high-voltage NMOS
sg13_hv_pmos    -> IHP high-voltage PMOS
rppd            -> Poly resistor (P-doped)
npn13G2         -> NPN bipolar transistor
```

---

## Implementation Roadmap for IHP PDK

### Phase 1: Core Infrastructure (Week 1-2)

1. **Create IHP Component Prefix Handler**
   - Add `eSim_ihp()` method to `DeviceModel.py`
   - Implement OSDI path configuration UI
   - Add corner selection dropdown

2. **Modify Detection Logic**
   - Update `__init__` in `DeviceModel.py` to detect IHP
   - Add "ihp" prefix recognition throughout pipeline

3. **Create IHP Library Directory Structure**
   ```
   library/
   ├── ihp_sg13g2/
   │   ├── models/
   │   │   └── sg13g2.lib.spice
   │   ├── osdi/
   │   │   └── <symlinks to installed OSDI files>
   │   └── .spiceinit
   └── modelParamXML/
       └── IHP/
           ├── sg13_lv_nmos.xml
           ├── sg13_lv_pmos.xml
           └── ...
   ```

### Phase 2: Conversion Logic (Week 2-3)

1. **Update Convert.py**
   - Add IHP-specific library include logic
   - Generate OSDI loading commands
   - Handle IHP parameter format (W, L, nf, etc.)

2. **Create IHP Model XML Files**
   - Define parameter structure for each IHP device
   - Include defaults and validation rules

3. **Update .spiceinit Generation**
   - Create IHP-specific initialization
   - Include all required OSDI files

### Phase 3: KiCad Integration (Week 3-4)

1. **Verify KiCad Symbols**
   - Ensure correct pin ordering
   - Validate value field format

2. **Create Reference Projects**
   - Simple inverter with IHP devices
   - Amplifier circuits
   - Digital standard cell examples

3. **Test Full Pipeline**
   - Schematic → `.cir` → `.cir.out` → Ngspice

### Phase 4: Documentation & Testing (Week 4-5)

1. **Create User Documentation**
   - Installation guide
   - Usage tutorial
   - Troubleshooting guide

2. **Automated Testing**
   - Regression test suite
   - Corner case validation

3. **Integration with Install Script**
   - Update `ihp-install-script.sh`
   - Add eSim configuration for IHP

---

## Quick Reference: File Modification Summary

| File | Modification Required |
|------|----------------------|
| `src/kicadtoNgspice/DeviceModel.py` | Add `eSim_ihp()` method, detection logic |
| `src/kicadtoNgspice/Convert.py` | Add IHP library include logic, OSDI commands |
| `src/kicadtoNgspice/Processing.py` | Handle `ihp` prefix in component parsing |
| `src/kicadtoNgspice/KicadtoNgspice.py` | No changes needed (uses existing pipeline) |
| `library/modelParamXML/` | Create `IHP/` folder with device XMLs |
| `library/ihp_sg13g2/` | Create directory with model/OSDI links |

---

## Appendix A: Example IHP .cir File

```spice
* /home/user/eSim-Workspace/IHP_Inverter/IHP_Inverter.cir

* EESchema Netlist Version 1.1 (Spice format)

* Sheet Name: /
ihpmode1 GND sg13_lv_nmos
ihp1 out in vss vss sg13_lv_nmos
ihp2 out in vdd vdd sg13_lv_pmos
v1 vdd GND dc
v2 in GND pulse
U1 out plot_v1

.end
```

## Appendix B: Expected IHP .cir.out File

```spice
* /home/user/esim-workspace/ihp_inverter/ihp_inverter.cir

* OSDI Model Loading
osdi "$PDK_ROOT/ihp-sg13g2/libs.tech/ngspice/osdi/psp103_nmos.osdi"
osdi "$PDK_ROOT/ihp-sg13g2/libs.tech/ngspice/osdi/psp103_pmos.osdi"

* Library Include
.lib "$PDK_ROOT/ihp-sg13g2/libs.tech/ngspice/models/sg13g2.lib" mos_tt

* ihpmode1 gnd sg13_lv_nmos
xihp1 out in vss vss sg13_lv_nmos W=2u L=130n nf=1
xihp2 out in vdd vdd sg13_lv_pmos W=4u L=130n nf=1
v1 vdd gnd dc 1.2
v2 in gnd pulse(0 1.2 1n 100p 100p 5n 10n)
* u1 out plot_v1

.tran 1e-12 20e-09 0

* Control Statements 
.control
run
print allv > plot_data_v.txt
print alli > plot_data_i.txt
plot v(out) v(in)
.endc
.end
```

---

*Document created: 2026-01-15*
*Author: eSim Research Documentation*
*For: IHP Open PDK Integration Project*
