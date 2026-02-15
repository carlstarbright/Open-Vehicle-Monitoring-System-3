# Nissan Leaf CAN Bus & DTC Research

**Research Date:** 2026-02-15  
**Target Vehicle:** 2018+ Nissan Leaf ZE1 (40kWh battery)  
**Focus:** CAN bus architecture, UDS Mode 19 implementation, and DTC codes

---

## Table of Contents
1. [Overview](#overview)
2. [CAN Bus Architecture](#can-bus-architecture)
3. [Key Resources](#key-resources)
4. [UDS Mode 19 - Read DTC Information](#uds-mode-19---read-dtc-information)
5. [Specific DTC Codes](#specific-dtc-codes)
6. [CAN Message IDs](#can-message-ids)
7. [Implementation Notes](#implementation-notes)
8. [References](#references)

---

## Overview

The Nissan Leaf uses multiple CAN buses for communication between ECUs. The ZE1 generation (2018+, 40kWh/62kWh) introduced significant changes compared to earlier ZE0/AZE0 models, most notably a CAN gateway that isolates the OBD-II port when the ignition is off.

### Leaf Generations
- **Gen1 (ZE0)**: 2010-2012, 24kWh, EM61 motor
- **Gen2 (AZE0-0/1)**: 2013-2014, 24kWh, EM57 motor
- **Gen3 (AZE0-2)**: 2016-2017, 30kWh, EM57 motor
- **Gen4 (ZE1)**: 2018+, 40kWh, EM57 motor 110kW, **404V NMC battery**
- **Gen5 (ZE1 e+)**: 2019+, 62kWh, EM57 motor 160kW, 404V NMC battery

---

## CAN Bus Architecture

### Three CAN Buses (OBD-II Connector Pinout)

#### 1. Primary CAN Bus ("CAN communication circuit")
- **Pins:** CAN-L (14), CAN-H (6)
- **Speed:** 500 kbps
- **Components:** VCM, DLC, BCM, ABS, Steering, Dash/Nav
- **Purpose:** Interior operations, displays, controls

#### 2. EV CAN Bus ("EV system CAN circuit")
- **Pins:** EV CAN-L (12), EV CAN-H (13)
- **Speed:** 500 kbps
- **Components:** VCM, HV Battery (LBC), OBC, PDM, Inverter, TCU
- **Purpose:** High-voltage battery, motor, charging systems
- **Note:** VCM acts as gateway between CAN and EV-CAN

#### 3. AV CAN Bus
- **Pins:** AV-CAN-H (11), AV-CAN-L (3)
- **Components:** Audio-Video Navigation unit, multifunction switch
- **Purpose:** Infotainment system

### ZE1-Specific Changes (2018+)

**Critical Difference:** The OBD-II port is behind a **CAN Gateway Module (M101)** that:
- Powers down when ignition is off
- Isolates OBD-II port from vehicle CAN buses
- Requires direct CAN tap at gateway connector (24-pin behind instrument cluster)
- Standard OBD-II cable insufficient (needs 8-wire tap, not 6-wire)

**Gateway Location:** Behind instrument cluster  
**Gateway Connector:** 24-pin  
**Required Tap Points:** Both EV-CAN and Car-CAN at gateway connector

---

## Key Resources

### 1. Dala's GitHub - LEAF CAN Bus Messages
**URL:** https://github.com/dalathegreat/leaf_can_bus_messages

**Contents:**
- DBC (CAN Database) files for ZE0, AZE0, and ZE1
- Proper message decoding with frame positioning
- Evolution of Google Sheets documentation
- CRC/CSUM/MPRUN error checking methods

**Tool Recommendation:** Kvaser Database Editor 3  
https://www.kvaser.com/download/

**Key Files:**
- `LEAF_ZE0.dbc` - First generation (2010-2012)
- `LEAF_AZE0.dbc` - Second generation (2013-2017)
- `LEAF_ZE1.dbc` - Third generation (2018+)

**Notable Note:** "Major differences between ZE0, AZEO and ZE1 LEAFs"

### 2. MyNissanLeaf.com Forum
**Main Thread:** https://mynissanleaf.com/viewtopic.php?t=4131  
**Forum Section:** https://mynissanleaf.com/forums/leaf-canbus.44/

**Key Discussions:**
- CAN bus decoding (open discussion)
- Real-world implementations
- DTC troubleshooting experiences
- Battery swap documentation

**Legacy Resource:** Original Google Sheets:  
https://docs.google.com/spreadsheets/d/1EHa4R85BttuY4JZ-EnssH4YZddpsDVu6rUFm0P7ouwg/edit#gid=1

### 3. Open Vehicles Monitoring System (OVMS)
**Docs:** https://docs.openvehicles.com/en/latest/components/vehicle_nissanleaf/docs/index.html  
**GitHub Issue:** https://github.com/openvehicles/Open-Vehicle-Monitoring-System-3/issues/323

**ZE1 Implementation Details:**
- CAN tap cable wiring diagrams
- TCU disconnection for remote climate
- Battery capacity configurations
- Polling information

**ZE1 Polling Info:** https://drive.google.com/file/d/1jH9cgm5v23qnqVnmZN3p4TvdaokWKPjM/view

### 4. Nissan Leaf Car Hacking Wiki
**URL:** https://nissanleaf.carhackingwiki.com

**Coverage:**
- Component technical specs (LBC, PDM, TCU, etc.)
- Connector pinouts (full catalog)
- Service manual locations
- LeafSpy integration info

---

## UDS Mode 19 - Read DTC Information

### Service ID: 0x19 (ReadDTCInformation)

**Purpose:** Retrieve Diagnostic Trouble Codes (DTCs) from ECU fault code memory (FCM)

**Protocol Standard:** ISO 14229 (Unified Diagnostic Services)

### Key Sub-Functions

| Sub-Function | Name | Description |
|--------------|------|-------------|
| 0x01 | ReportNumberOfDTCByStatusMask | Count DTCs matching status mask |
| 0x02 | ReportDTCByStatusMask | List DTCs with status matching mask |
| 0x03 | ReportDTCSnapshotIdentification | Available snapshot records |
| 0x04 | ReportDTCSnapshotRecordByDTCNumber | Snapshot data for specific DTC |
| 0x06 | ReportDTCExtendedDataRecordByDTCNumber | Extended data for DTC |
| 0x0A | ReportSupportedDTC | All DTCs ECU supports |
| 0x14 | ReportDTCFaultDetectionCounter | Fault occurrence counter |

### DTC Status Byte (8 bits)

| Bit | Name | Meaning |
|-----|------|---------|
| 0 | testFailed | Diagnostic test has failed |
| 1 | testFailedThisOperationCycle | Failed during current driving cycle |
| 2 | pendingDTC | Detected but not yet confirmed |
| 3 | confirmedDTC | Fault confirmed and stored |
| 4 | testNotCompletedSinceLastClear | Test not run since last clear |
| 5 | testFailedSinceLastClear | Failed since last clear |
| 6 | testNotCompletedThisOperationCycle | Test not run this cycle |
| 7 | warningIndicatorRequested | MIL (Malfunction Indicator Lamp) requested |

### Example: Read DTCs with Status Mask

**Request (read all confirmed DTCs):**
```
0x19 0x02 0x08
```
- `0x19` = ReadDTCInformation service
- `0x02` = ReportDTCByStatusMask sub-function
- `0x08` = Status mask (bit 3 = confirmed DTC)

**Positive Response:**
```
0x59 0x02 [count] [DTC1_H] [DTC1_M] [DTC1_L] [Status1] ...
```
- `0x59` = Positive response (0x40 + 0x19)
- DTCs encoded as 3 bytes each

### DTC Format (5-character alphanumeric)

**Structure:** `[System][Type][Subsystem][Fault]`

**System (1st character):**
- `P` = Powertrain
- `B` = Body
- `C` = Chassis
- `U` = Network/Communication

**Type (2nd character):**
- `0` = SAE/ISO Generic
- `1` = Manufacturer Specific
- `3` = Combined (often battery/hybrid systems)

**Example:** `P3193`
- `P` = Powertrain
- `3` = Manufacturer specific (battery system)
- `19` = Subsystem identifier
- `3` = Specific fault

---

## Specific DTC Codes

### Codes Found in Our System

#### P3193, P3180, P317E, P31B8, P318E
**Pattern:** All P31xx codes  
**System:** Powertrain (P), Manufacturer Specific (3)  
**Subsystem:** Battery/Hybrid system (1x)

**General Meaning:** These are Nissan-specific battery/EV system codes

**Context from Forum Research:**
- P31xx codes typically relate to **Li-Ion battery modules**
- Often associated with battery management, cell voltage, or isolation issues
- May include snapshot data (temperature, SoC, voltage at time of fault)

#### U1000
**Full Code:** U1000  
**System:** Network/Communication (U)  
**Type:** SAE Generic (0)  
**Meaning:** CAN Communication Circuit fault

**Common Causes:**
- Low 12V battery voltage
- CAN bus wiring issues
- Gateway communication failure
- Multiple ECU communication loss

**Note from Forums:** "A bunch of power and communication related codes like this is usually **low power coming from the 12V accessory battery**"

### Related DTC Patterns from Research

**P0AA6** - Hybrid Battery Voltage System Isolation  
- Indicates HV leak to ground detected
- Safety circuit triggered
- Can cause restart inhibit (P31E7)

**P31E7** - EV/HEV Restart Inhibition  
- Often secondary to other battery faults
- Prevents vehicle operation until cleared

**Battery Module Codes:**
- Often require module replacement
- Service bulletins exist (NTB23-024, etc.)
- Isolation-related DTCs are critical safety issues

---

## CAN Message IDs

### Active Polling IDs (UDS Query/Response)

| Control Unit | Query ID | Response ID | Notes |
|--------------|----------|-------------|-------|
| VCM (Vehicle Control Module) | 0x797 | 0x79A | |
| LBC (Lithium Battery Controller) | 0x79B | 0x7BB | **Battery Management** |
| BCM (Body Control Module) | 0x745 | 0x765 | |
| ABS | 0x740 | 0x760 | |
| INVERTER/MC (Motor Controller) | 0x784 | 0x78C | |
| OBC (On-Board Charger) | 0x792 | 0x793 | |
| PDM (Power Delivery Module) | 0x792 | 0x793 | Contains OBC + DC-DC |
| HVAC | 0x744 | 0x764 | |
| TCU (Telematics) | 0x746 | 0x783 | |
| AIRBAG | 0x752 | 0x772 | |

### Key Passive CAN Messages (Broadcast)

From Dala's DBC files (select messages):

| ID (hex) | Name | Purpose | Bus |
|----------|------|---------|-----|
| 0x1DB | HV Battery Status | Battery voltage, current, SoC | EV-CAN |
| 0x1DC | HV Battery Temp/Stats | Temperature, power limits | EV-CAN |
| 0x1F2 | Battery Module Temps | Individual module temperatures | EV-CAN |
| 0x5BC | VCM Status | Ready state, gear position | Car-CAN |
| 0x5C0 | Battery GIDS | State of charge (GIDS units) | EV-CAN |
| 0x390 | Motor Speed | RPM, torque | EV-CAN |
| 0x3F0 | Left Blindspot Warning | (ZE1) | Car-CAN |
| 0x3F3 | Right Blindspot Warning | (ZE1) | Car-CAN |

**Important Notes:**
- GIDS = Nissan's proprietary SoC unit (~80Wh per GID)
- 40kWh battery max = ~502 GIDS
- 62kWh battery max = ~775 GIDS
- Message availability varies by generation

### UDS Mode 19 Request Example

**Query LBC for DTCs:**
```
TX: 0x79B  [02 19 02 FF 00 00 00 00]
```
- First byte: `02` = length (2 more bytes)
- Second byte: `19` = Service ID (ReadDTCInformation)
- Third byte: `02` = Sub-function (ReportDTCByStatusMask)
- Fourth byte: `FF` = Status mask (all DTCs)

**Expected Response:**
```
RX: 0x7BB  [10 XX 59 02 ...] (first frame)
RX: 0x7BB  [21 DTC1_H DTC1_M DTC1_L Status1 ...] (continuation)
```
- Multi-frame response via ISO-TP
- `59` = positive response (0x40 + 0x19)

---

## Implementation Notes

### For ZE1 Models (2018+ 40kWh)

#### 1. Hardware Access
- **Cannot use OBD-II port** for passive monitoring when ignition off
- Must tap CAN at gateway behind instrument cluster
- Requires 8-wire cable (not standard 6-wire OBD cable)
- Tap both EV-CAN and Car-CAN at gateway connector

#### 2. CAN Gateway Bypass
**Option A:** Tap at gateway (recommended)
- Allows operation when ignition off
- Access to all CAN buses
- Requires instrument cluster removal

**Option B:** Use OBD-II with limitations
- Only works when ignition on
- Gateway may filter messages
- Polling still possible

#### 3. TCU Interference
If vehicle has TCU (Telematics Control Unit):
- TCU may override CAN messages (e.g., remote climate)
- Located behind glovebox (LHD) or driver footwell (RHD)
- Can unpin CAN wires (pins 2) or fully disconnect
- **Warning:** Unpinning CAN causes permanent DTCs (expected)

#### 4. UDS Polling Strategy
**Recommended approach:**
- Use sub-function 0x02 with status mask for active faults
- Poll LBC (0x79B) for battery DTCs
- Check VCM (0x797) for system DTCs
- Use sub-function 0x04 for snapshot data when investigating specific fault

**Timing:**
- Allow 50-100ms between polling requests
- Use ISO-TP for multi-frame responses
- Implement timeout handling (recommend 1000ms)

#### 5. DTC Interpretation
**P31xx codes:**
- Always check isolation (P0AA6)
- Verify 12V battery health (affects U1000)
- Retrieve snapshot data for temperature/voltage at fault time
- Check extended data for fault counters

**U1000 troubleshooting:**
- First check 12V battery voltage (>12.4V resting)
- Verify CAN termination (120Ω each end)
- Check for loose connectors at gateway
- Scan other modules to identify which are offline

---

## References

### Primary Documentation
1. **Dala's GitHub Repository**  
   https://github.com/dalathegreat/leaf_can_bus_messages  
   - DBC files for all generations
   - CAN polling IDs
   - Error checking methods

2. **OVMS Documentation**  
   https://docs.openvehicles.com/en/latest/components/vehicle_nissanleaf/docs/index.html  
   - ZE1 implementation guide
   - CAN tap wiring diagrams
   - Battery configurations

3. **Nissan Leaf Car Hacking Wiki**  
   https://nissanleaf.carhackingwiki.com  
   - Component specifications
   - Connector pinouts
   - Service procedures

4. **MyNissanLeaf Forum - CAN Decoding Thread**  
   https://mynissanleaf.com/viewtopic.php?t=4131  
   - Community knowledge base
   - Real-world troubleshooting
   - DTC experiences

### UDS Protocol Resources
5. **UDS Service 0x19 In-Depth Guide**  
   https://www.cselectricalandelectronics.com/in-depth-guide-to-uds-service-0x19-readdtcinformation/  
   - Complete sub-function reference
   - Status byte explanation
   - Request/response examples

6. **UDS Protocol Tutorial**  
   https://piembsystech.com/read-dtc-information-service-0x19-uds-protocol/  
   - DTC properties and criteria
   - Fault detection logic
   - Aging counters

7. **ISO 14229 Standard**  
   - Unified Diagnostic Services specification
   - Official protocol definition

### Community Resources
8. **ZE1 Polling Information**  
   https://drive.google.com/file/d/1jH9cgm5v23qnqVnmZN3p4TvdaokWKPjM/view  
   - ZE1-specific polling commands

9. **Legacy Spreadsheet**  
   https://docs.google.com/spreadsheets/d/1EHa4R85BttuY4JZ-EnssH4YZddpsDVu6rUFm0P7ouwg/edit#gid=1  
   - Original community CAN decoding effort

### Tools
10. **Kvaser Database Editor 3**  
    https://www.kvaser.com/download/  
    - DBC file viewer/editor

11. **LeafSpy Pro**  
    https://play.google.com/store/apps/details?id=com.Turbo3.Leaf_Spy_Pro  
    - Android app for diagnostics
    - DTC reading/clearing
    - Battery health monitoring

### Service Manuals
12. **Nissan TechInfo (Official)**  
    https://www.nissan-techinfo.com/find.aspx  
    - Paid subscription ($35/day - $1250/year)
    - North American Market only
    - Electronic wiring diagrams

13. **Free ZE1 Service Manual**  
    https://www.mediafire.com/file/yoikgbj1l8l7qwi/SM18EA0ZE1U8.zip  
    - Interactive HTML manual
    - PDF wiring diagram

14. **Older Service Manuals (2011-2015)**  
    https://www.nicoclub.com/nissan-service-manuals  
    - Requires subscription

---

## Next Steps

### For OVMS Integration
1. Implement UDS 0x19 service handler for LBC (0x79B → 0x7BB)
2. Parse multi-frame ISO-TP responses
3. Decode 3-byte DTC format to 5-character display
4. Store status bytes for trend analysis
5. Implement snapshot data retrieval for critical DTCs
6. Add 12V battery voltage monitoring to catch U1000 early

### For Battery Swap Documentation
1. Focus on LBC CAN messages (0x1DB, 0x1DC, 0x1F2, 0x5C0)
2. Document GIDS recalibration after swap
3. Map P31xx DTCs to specific module failures
4. Test isolation resistance monitoring
5. Verify CRC/checksum methods from Dala's notes

### DTC Research Gaps
- Need specific P31xx code definitions from service manual
- Snapshot data format for battery DTCs
- Extended data record structure
- Fault confirmation criteria (how many cycles?)
- Aging counter thresholds

---

**Document Version:** 1.0  
**Last Updated:** 2026-02-15  
**Compiled By:** OVMS Subagent (can-bus-research)
