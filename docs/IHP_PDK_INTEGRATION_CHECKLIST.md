# IHP Open PDK Integration - Implementation Checklist

## Quick Summary

This document provides a step-by-step checklist for integrating IHP Open PDK (ihp-sg13g2) into the eSim KiCad-to-Ngspice pipeline, following the pattern established by SKY130 PDK.

---

## Prerequisites ✓

- [x] IHP Open PDK installation script created (`ihp/ihp-install-script.sh`)
- [x] KiCad symbols for IHP devices added
- [ ] Ngspice compiled with OSDI support (`--enable-osdi`)
- [ ] OpenVAF installed and configured
- [ ] IHP Verilog-A models compiled to OSDI

---

## Phase 1: DeviceModel.py Modifications

### File: `src/kicadtoNgspice/DeviceModel.py`

#### 1.1 Add IHP Detection (around line 71)
```python
# Current:
if "sky130" in " ".join(schematicInfo):
    self.eSim_sky130(schematicInfo)
else:
    self.eSim_general_libs(schematicInfo)

# Modified:
if "ihp" in " ".join(schematicInfo) or "sg13g2" in " ".join(schematicInfo):
    self.eSim_ihp(schematicInfo)
elif "sky130" in " ".join(schematicInfo):
    self.eSim_sky130(schematicInfo)
else:
    self.eSim_general_libs(schematicInfo)
```

#### 1.2 Create `eSim_ihp()` Method (after `eSim_sky130()`, around line 246)
```python
def eSim_ihp(self, schematicInfo):
    """Handle IHP SG13G2 PDK components."""
    
    # IHP Mode Configuration Group Box
    ihpbox = QtWidgets.QGroupBox()
    ihpgrid = QtWidgets.QGridLayout()
    self.count = self.count + 1
    self.row = self.row + 1
    self.devicemodel_dict_beg["ihpmode1"] = self.count
    beg = self.count
    self.deviceDetail[self.count] = "ihpmode1"
    ihpbox.setTitle("Add parameters of IHP SG13G2 library")
    
    # OSDI Path Entry
    self.parameterLabel[self.count] = QtWidgets.QLabel("Enter OSDI path")
    self.row = self.row + 1
    ihpgrid.addWidget(self.parameterLabel[self.count], self.row, 0)
    self.entry_var[self.count] = QtWidgets.QLineEdit()
    self.entry_var[self.count].setReadOnly(True)
    
    # Set default path
    for child in self.root:
        if child.tag == "ihpmode1":
            if child[0].text and os.path.exists(child[0].text):
                self.entry_var[self.count].setText(child[0].text)
                path_name = child[0].text
            else:
                if os.name == 'nt':
                    path_name = os.path.abspath("library/ihp_sg13g2/osdi")
                else:
                    pdk_root = os.environ.get('PDK_ROOT', 
                        os.path.expanduser('~/ihp/IHP-Open-PDK'))
                    path_name = os.path.join(pdk_root, 
                        "ihp-sg13g2/libs.tech/ngspice/osdi")
                self.entry_var[self.count].setText(path_name)
    
    ihpgrid.addWidget(self.entry_var[self.count], self.row, 1)
    
    # Add button
    self.addbtn = QtWidgets.QPushButton("Add")
    self.addbtn.setObjectName("%d" % beg)
    self.addbtn.clicked.connect(self.trackIHPLibrary)
    ihpgrid.addWidget(self.addbtn, self.row, 2)
    
    # Add Default button
    self.adddefaultbtn = QtWidgets.QPushButton("Add Default")
    self.adddefaultbtn.setObjectName("%d" % beg)
    self.adddefaultbtn.clicked.connect(self.trackDefaultIHPLib)
    ihpgrid.addWidget(self.adddefaultbtn, self.row, 3)
    
    self.count = self.count + 1
    
    # Corner Selection Entry
    self.parameterLabel[self.count] = QtWidgets.QLabel(
        "Enter corner (e.g., mos_tt)")
    self.row = self.row + 1
    ihpgrid.addWidget(self.parameterLabel[self.count], self.row, 0)
    self.entry_var[self.count] = QtWidgets.QLineEdit()
    self.entry_var[self.count].setText("mos_tt")  # Default corner
    self.entry_var[self.count].setMaximumWidth(150)
    self.entry_var[self.count].setObjectName("%d" % beg)
    
    for child in self.root:
        if child.tag == "ihpmode1":
            if child[1].text:
                self.entry_var[self.count].setText(child[1].text)
    
    ihpgrid.addWidget(self.entry_var[self.count], self.row, 1)
    self.entry_var[self.count].textChanged.connect(self.textChange)
    
    ihpbox.setLayout(ihpgrid)
    ihpbox.setStyleSheet("""
        QGroupBox { border: 1px solid gray; border-radius: 9px; 
                    margin-top: 0.5em; }
        QGroupBox::title { subcontrol-origin: margin; left: 10px; 
                          padding: 0 3px 0 3px; }
    """)
    self.grid.addWidget(ihpbox)
    
    self.row = self.row + 1
    self.devicemodel_dict_end["ihpmode1"] = self.count
    self.count = self.count + 1
    
    # Process IHP components (ihp prefix)
    for eachline in schematicInfo:
        words = eachline.split()
        
        # Skip non-IHP components
        if eachline[0:3] != 'ihp' and eachline[0] != 'u' \
                and eachline[0] != 'x' and eachline[0] != '*' \
                and eachline[0] != 'v' and eachline[0] != 'i' \
                and eachline[0] != 'a':
            print(f"Component {words[0]} may not be compatible with IHP mode")
            continue
        
        # Handle IHP devices
        if eachline[0:3] == 'ihp' and eachline[0:7] != 'ihpmode':
            self.devicemodel_dict_beg[words[0]] = self.count
            self.deviceDetail[self.count] = words[0]
            
            ihpcompbox = QtWidgets.QGroupBox()
            ihpcompgrid = QtWidgets.QGridLayout()
            beg = self.count
            
            ihpcompbox.setTitle(f"Parameters for {words[0]} : {words[-1]}")
            
            # Width
            self.parameterLabel[self.count] = QtWidgets.QLabel(
                f"Width of {words[0]} (default=1u):")
            ihpcompgrid.addWidget(self.parameterLabel[self.count], 0, 0)
            self.entry_var[self.count] = QtWidgets.QLineEdit()
            self.entry_var[self.count].setText("")
            self.entry_var[self.count].setMaximumWidth(150)
            ihpcompgrid.addWidget(self.entry_var[self.count], 0, 1)
            self.count = self.count + 1
            
            # Length
            self.parameterLabel[self.count] = QtWidgets.QLabel(
                f"Length of {words[0]} (default=130n):")
            ihpcompgrid.addWidget(self.parameterLabel[self.count], 1, 0)
            self.entry_var[self.count] = QtWidgets.QLineEdit()
            self.entry_var[self.count].setText("")
            self.entry_var[self.count].setMaximumWidth(150)
            ihpcompgrid.addWidget(self.entry_var[self.count], 1, 1)
            self.count = self.count + 1
            
            # Number of fingers
            self.parameterLabel[self.count] = QtWidgets.QLabel(
                f"Fingers (nf) of {words[0]} (default=1):")
            ihpcompgrid.addWidget(self.parameterLabel[self.count], 2, 0)
            self.entry_var[self.count] = QtWidgets.QLineEdit()
            self.entry_var[self.count].setText("")
            self.entry_var[self.count].setMaximumWidth(150)
            ihpcompgrid.addWidget(self.entry_var[self.count], 2, 1)
            
            ihpcompbox.setLayout(ihpcompgrid)
            ihpcompbox.setStyleSheet("""
                QGroupBox { border: 1px solid gray; border-radius: 9px;
                            margin-top: 0.5em; }
                QGroupBox::title { subcontrol-origin: margin; left: 10px;
                                  padding: 0 3px 0 3px; }
            """)
            
            # Load from previous values
            try:
                for child in self.root:
                    if child.tag == words[0]:
                        i = beg
                        for field in child:
                            if i <= self.count:
                                self.entry_var[i].setText(field.text or "")
                            i += 1
            except Exception:
                pass
            
            self.grid.addWidget(ihpcompbox)
            self.devicemodel_dict_end[words[0]] = self.count
            self.count = self.count + 1
        
        self.show()
```

#### 1.3 Add Helper Methods

```python
def trackDefaultIHPLib(self):
    """Set default IHP OSDI library path."""
    sending_btn = self.sender()
    self.widgetObjCount = int(sending_btn.objectName())
    
    pdk_root = os.environ.get('PDK_ROOT', 
        os.path.expanduser('~/ihp/IHP-Open-PDK'))
    path_name = os.path.join(pdk_root, 
        "ihp-sg13g2/libs.tech/ngspice/osdi")
    
    self.entry_var[self.widgetObjCount].setText(path_name)
    self.trackLibraryWithoutButton(self.widgetObjCount, path_name)

def trackIHPLibrary(self):
    """Browse for IHP OSDI library directory."""
    sending_btn = self.sender()
    self.widgetObjCount = int(sending_btn.objectName())
    
    self.libfile = QtCore.QDir.toNativeSeparators(
        QtWidgets.QFileDialog.getExistingDirectory(
            self, "Select IHP OSDI Directory",
            os.path.expanduser("~/ihp")
        )
    )
    
    if not self.libfile:
        return
    
    self.entry_var[self.widgetObjCount].setText(self.libfile)
    self.deviceName = self.deviceDetail[self.widgetObjCount]
    
    if self.deviceName[0:7] == 'ihpmode':
        self.obj_trac.deviceModelTrack[self.deviceName] = self.libfile + \
            ":" + str(self.entry_var[self.widgetObjCount + 1].text())
```

#### 1.4 Update `textChange()` Method (around line 706)

```python
def textChange(self):
    ...
    elif self.deviceName[0:7] == 'ihpmode':
        self.obj_trac.deviceModelTrack[self.deviceName] = \
            self.entry_var[self.widgetObjCount].text() + \
            ":" + str(self.entry_var[self.widgetObjCount + 1].text())
        print(self.obj_trac.deviceModelTrack[self.deviceName])
    elif self.deviceName[0:3] == 'ihp':
        # Track IHP component parameters (W, L, nf)
        width = str(self.entry_var[self.widgetObjCount].text()) or "1u"
        length = str(self.entry_var[self.widgetObjCount + 1].text()) or "130n"
        nf = str(self.entry_var[self.widgetObjCount + 2].text()) or "1"
        self.obj_trac.deviceModelTrack[self.deviceName] = \
            f"W={width} L={length} nf={nf}"
    ...
```

---

## Phase 2: Convert.py Modifications

### File: `src/kicadtoNgspice/Convert.py`

#### 2.1 Update `addDeviceLibrary()` Method (around line 623)

Add IHP handling after SKY130 block:

```python
# After the elif for scmode (around line 700)
elif eachline[0:7] == 'ihpmode':
    # Generate .spiceinit file for IHP
    (filepath, filemname) = os.path.split(self.clarg1)
    self.Fileopen = os.path.join(filepath, ".spiceinit")
    print("Writing .spiceinit for IHP SG13G2 compatibility")
    
    self.writefile = open(self.Fileopen, "w")
    self.writefile.write('''
* IHP SG13G2 PDK Configuration
set ngbehavior=hsa
set ng_nomodcheck
set num_threads=8
option noinit
''')
    self.writefile.close()
    
    # Get OSDI path and corner from tempStr
    osdi_path = tempStr[0]
    corner = tempStr[1] if len(tempStr) > 1 else "mos_tt"
    
    # Add OSDI loading commands
    osdi_files = [
        'psp103_nmos.osdi',
        'psp103_pmos.osdi',
        'r_nplus.osdi',
        'r_pplus.osdi',
        'HICUM_L2V3p0p0.osdi',
        'DIODE_CMC.osdi'
    ]
    
    for osdi_file in osdi_files:
        full_path = os.path.join(osdi_path, osdi_file)
        if os.path.exists(full_path):
            includeLine.append(f'osdi "{full_path}"')
    
    deviceLine[index] = "*ihpmode"

elif eachline[0:3] == 'ihp' and eachline[0:7] != 'ihpmode':
    # Convert ihp1 to xihp1 and add parameters
    words[0] = words[0].replace('ihp', 'xihp')
    
    # Get parameters from deviceModelTrack
    device_key = eachline.split()[0]
    if device_key in self.obj_track.deviceModelTrack:
        params = self.obj_track.deviceModelTrack[device_key]
        words.append(params)  # Append W=... L=... nf=...
    
    words.append(completeLibPath)  # Append model name
    deviceLine[index] = words
```

---

## Phase 3: Processing.py Modifications

### File: `src/kicadtoNgspice/Processing.py`

#### 3.1 Update `separateNetlistInfo()` if needed
No changes required - generic handling works for `ihp` prefix.

#### 3.2 Validate IHP component handling in `convertICintoBasicBlocks()`
The existing logic handles 'U' prefixed components. IHP devices with `ihp` prefix will be processed as regular device lines.

---

## Phase 4: Library Structure Setup

### Create IHP Library Directory
```bash
mkdir -p library/ihp_sg13g2/osdi
mkdir -p library/modelParamXML/IHP
```

### Create Model XML Files

#### `library/modelParamXML/IHP/sg13_lv_nmos.xml`
```xml
<model>
  <name>sg13_lv_nmos</name>
  <type>IHP</type>
  <node_number>4</node_number>
  <title>IHP SG13G2 Low-Voltage NMOS Parameters</title>
  <split>4-NV</split>
  <param>
    <w default="1u">Enter width (default=1u)</w>
    <l default="130n">Enter length (default=130n)</l>
    <nf default="1">Enter number of fingers (default=1)</nf>
  </param>
</model>
```

#### `library/modelParamXML/IHP/sg13_lv_pmos.xml`
```xml
<model>
  <name>sg13_lv_pmos</name>
  <type>IHP</type>
  <node_number>4</node_number>
  <title>IHP SG13G2 Low-Voltage PMOS Parameters</title>
  <split>4-NV</split>
  <param>
    <w default="2u">Enter width (default=2u)</w>
    <l default="130n">Enter length (default=130n)</l>
    <nf default="1">Enter number of fingers (default=1)</nf>
  </param>
</model>
```

---

## Phase 5: KiCad Symbol Requirements

### Symbol Naming Convention
| KiCad Reference | KiCad Value | Ngspice Model |
|-----------------|-------------|---------------|
| ihp1, ihp2, ... | sg13_lv_nmos | sg13_lv_nmos |
| ihpP1, ihpP2, ... | sg13_lv_pmos | sg13_lv_pmos |
| ihpN1, ihpN2, ... | npn13G2 | npn13G2 |
| ihpR1, ihpR2, ... | rppd | rppd |

### Symbol Pin Ordering (MOSFET)
```
Pin 1: Drain
Pin 2: Gate
Pin 3: Source
Pin 4: Body (Substrate)
```

### Sample KiCad Netlist Line
```
ihp1 out in vss vss sg13_lv_nmos
```

---

## Phase 6: Testing Checklist

### Test Cases
- [ ] Simple inverter with sg13_lv_nmos and sg13_lv_pmos
- [ ] Common source amplifier
- [ ] Ring oscillator
- [ ] Standard cell usage

### Verification Steps
1. Create schematic in KiCad with IHP symbols
2. Export netlist (`.cir` file)
3. Open in eSim → Kicad to Ngspice
4. Verify IHP mode detected automatically
5. Set OSDI path and corner
6. Convert and check `.cir.out` file
7. Run simulation in Ngspice
8. Verify plots are generated

---

## File Change Summary

| File | Changes |
|------|---------|
| `DeviceModel.py` | +200 lines (eSim_ihp method + helpers) |
| `Convert.py` | +50 lines (IHP library handling) |
| `library/modelParamXML/IHP/*.xml` | New directory with model XMLs |
| `library/ihp_sg13g2/` | New directory for local copies |

---

## Environment Variables

Add to `~/.bashrc`:
```bash
export PDK_ROOT="$HOME/ihp/IHP-Open-PDK"
export PDK="ihp-sg13g2"
```

---

*Document Version: 1.0*
*Created: 2026-01-15*
*For: eSim IHP Open PDK Integration*
