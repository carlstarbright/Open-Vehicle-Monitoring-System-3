# OVMS Nissan Leaf Vehicle Module - Code Analysis

**Analysis Date:** February 14, 2026  
**Code Location:** `/Users/carl/ovms-firmware/vehicle/OVMS.V3/components/vehicle_nissanleaf/src/`  
**Primary Files:**
- `vehicle_nissanleaf.cpp` (104,337 bytes)
- `vehicle_nissanleaf.h` (11,682 bytes)
- `nl_web.cpp` (18,926 bytes)
- `nl_types.h` (1,582 bytes)

---

## 1. Lock/Unlock Door Commands

### Overview
The Nissan Leaf module implements door lock/unlock functionality via CAN bus commands sent to the TCU (Telematics Control Unit). The implementation varies based on model year (pre-2016 vs 2016+).

### Key Functions & Line Numbers

#### RemoteCommand Enum Definition
**File:** `vehicle_nissanleaf.h`, Lines 82-89
```cpp
typedef enum
  {
  ENABLE_CLIMATE_CONTROL,
  DISABLE_CLIMATE_CONTROL,
  START_CHARGING,
  STOP_CHARGING,
  AUTO_DISABLE_CLIMATE_CONTROL,
  UNLOCK_DOORS,
  LOCK_DOORS
  } RemoteCommand;
```

#### Command Entry Points
**File:** `vehicle_nissanleaf.cpp`

**CommandLock()** - Line 2573-2576
```cpp
OvmsVehicle::vehicle_command_t OvmsVehicleNissanLeaf::CommandLock(const char* pin)
  {
  return RemoteCommandHandler(LOCK_DOORS);
  }
```

**CommandUnlock()** - Line 2578-2581
```cpp
OvmsVehicle::vehicle_command_t OvmsVehicleNissanLeaf::CommandUnlock(const char* pin)
  {
  return RemoteCommandHandler(UNLOCK_DOORS);
  }
```

#### RemoteCommandHandler()
**File:** `vehicle_nissanleaf.cpp`, Lines 2525-2548
```cpp
OvmsVehicle::vehicle_command_t OvmsVehicleNissanLeaf::RemoteCommandHandler(RemoteCommand command)
  {
  if (!cfg_enable_write) return Fail; //disable commands unless canwrite is true
  ESP_LOGI(TAG, "RemoteCommandHandler");
  // Use the configured pin to wake up GEN 1 Leaf with EV SYSTEM ACTIVATION REQUEST
  if (MyConfig.GetParamValueInt("xnl", "modelyear", DEFAULT_MODEL_YEAR) < 2013)
  {
    if (MyConfig.GetParamValueBool("xnl", "command.wakeup", true)) {
       CommandWakeupTCU();
    }
    MyPeripherals->m_max7317->Output((uint8_t)cfg_ev_request_port, 1);
    ESP_LOGI(TAG, "EV SYSTEM ACTIVATION REQUEST ON");
  } else {
    // Use wakeup only for newer cars.
    CommandWakeup();
  }
  // The GEN 2 Nissan TCU module sends the command repeatedly, so we start
  // m_remoteCommandTimer (which calls RemoteCommandTimer()) to do this
  // EV SYSTEM ACTIVATION REQUEST is released in the timer too
  nl_remote_command = command;
  nl_remote_command_ticker = REMOTE_COMMAND_REPEAT_COUNT;
  xTimerStart(m_remoteCommandTimer, 0);

  return Success;
  }
```

**Key Points:**
- Commands are disabled if `cfg_enable_write` is false
- Pre-2013 models use GPIO pin (EV SYSTEM ACTIVATION REQUEST)
- 2013+ models use `CommandWakeup()` / `CommandWakeupTCU()`
- Commands are repeated 24 times (REMOTE_COMMAND_REPEAT_COUNT = 24)

#### SendCommand() - Lock/Unlock Implementation
**File:** `vehicle_nissanleaf.cpp`, Lines 1902-1916

```cpp
    case UNLOCK_DOORS:
      ESP_LOGI(TAG, "Unlock Doors");
      data[0] = 0x11;
      data[1] = 0x00;
      data[2] = 0x00;
      data[3] = 0x00;
      break;
    case LOCK_DOORS:
      ESP_LOGI(TAG, "Lock Doors");
      data[0] = 0x60;
      data[1] = 0x80;
      data[2] = 0x00;
      data[3] = 0x00;
      break;
```

**CAN Message Transmission** - Line 1938-1942
```cpp
      result = tcuBus->WriteStandard(0x56e, length, data);
      if (result != ESP_OK)
      {
          ESP_LOGE(TAG, "CAN TX Error: 0x56e, result=%d", result);
      }
```

### CAN Message Details for Lock/Unlock

| Command | CAN ID | Data Bytes | Bus | Notes |
|---------|--------|------------|-----|-------|
| UNLOCK_DOORS | 0x56e | `11 00 00 00` | CAN1 (pre-2016) or CAN2 (2016+) | Standard frame |
| LOCK_DOORS | 0x56e | `60 80 00 00` | CAN1 (pre-2016) or CAN2 (2016+) | Standard frame |

**Model Year Differences** - Lines 1853-1863:
- **Pre-2016:** Uses CAN1 (EV Bus), 1-byte messages, old TCU
- **2016+:** Uses CAN2 (CAR Bus), 4-byte messages, new TCU

#### RemoteCommandTimer()
**File:** `vehicle_nissanleaf.cpp`, Lines 1950-1998

```cpp
void OvmsVehicleNissanLeaf::RemoteCommandTimer()
  {
  ESP_LOGI(TAG, "RemoteCommandTimer %d", nl_remote_command_ticker);
  // if lock or unlock is successful don't repeat command
  if (nl_remote_command == LOCK_DOORS && StandardMetrics.ms_v_env_locked->AsBool())
    {
    nl_remote_command_ticker = 0;
    }
  if (nl_remote_command == UNLOCK_DOORS && !StandardMetrics.ms_v_env_locked->AsBool())
    {
    nl_remote_command_ticker = 0;
    }
  if (nl_remote_command_ticker > 0)
    {
    nl_remote_command_ticker--;
    if (nl_remote_command != AUTO_DISABLE_CLIMATE_CONTROL)
      {
      SendCommand(nl_remote_command);
      }
    ...
```

**Key Features:**
- Timer decrements ticker every 100ms (10 Hz rate)
- Stops repeating if lock status changes to desired state
- For pre-2013 models, releases GPIO after ACTIVATION_REQUEST_TIME (10 × 100ms = 1 second)

---

## 2. Climate Control Commands

### Overview
Climate control (HVAC) is controlled via CAN messages sent to the TCU, with support for enable, disable, and auto-disable modes.

### Key Functions & Line Numbers

#### Command Entry Points
**File:** `vehicle_nissanleaf.cpp`

**CommandClimateControl()** - Lines 2564-2569
```cpp
OvmsVehicle::vehicle_command_t OvmsVehicleNissanLeaf::CommandClimateControl(bool climatecontrolon)
  {
  ESP_LOGI(TAG, "CommandClimateControl");
  return RemoteCommandHandler(climatecontrolon ? ENABLE_CLIMATE_CONTROL : DISABLE_CLIMATE_CONTROL);
  }
```

**CommandHomelink() - Alternate Interface** - Lines 2550-2563
```cpp
OvmsVehicle::vehicle_command_t OvmsVehicleNissanLeaf::CommandHomelink(int button, int durationms)
  {
  ESP_LOGI(TAG, "CommandHomelink");
  if (button == 0)
    {
    return RemoteCommandHandler(ENABLE_CLIMATE_CONTROL);
    }
  if (button == 1)
    {
    return RemoteCommandHandler(DISABLE_CLIMATE_CONTROL);
    }
  return NotImplemented;
  }
```

#### SendCommand() - Climate Control Implementation
**File:** `vehicle_nissanleaf.cpp`, Lines 1873-1901

```cpp
    case ENABLE_CLIMATE_CONTROL:
      m_climate_rqinprogress->SetValue(true);
      ESP_LOGI(TAG, "Enable Climate Control");
      data[0] = 0x4e;
      data[1] = 0x08;
      data[2] = 0x12;
      data[3] = 0x00;
      break;
    case DISABLE_CLIMATE_CONTROL:
      ESP_LOGI(TAG, "Disable Climate Control");
      data[0] = 0x56;
      data[1] = 0x00;
      data[2] = 0x01;
      data[3] = 0x00;
      break;
    ...
    case AUTO_DISABLE_CLIMATE_CONTROL:
      ESP_LOGI(TAG, "Auto Disable Climate Control");
      data[0] = 0x46;
      data[1] = 0x08;
      data[2] = 0x32;
      data[3] = 0x00;
      break;
```

### CAN Message Details for Climate Control

| Command | CAN ID | Data Bytes | Purpose |
|---------|--------|------------|---------|
| ENABLE_CLIMATE_CONTROL | 0x56e | `4e 08 12 00` | Turn on HVAC |
| DISABLE_CLIMATE_CONTROL | 0x56e | `56 00 01 00` | Turn off HVAC |
| AUTO_DISABLE_CLIMATE_CONTROL | 0x56e | `46 08 32 00` | Auto-disable timer |

**Auto-Disable Timer** - Lines 1969-1971
```cpp
    if (nl_remote_command_ticker == 1 && nl_remote_command == ENABLE_CLIMATE_CONTROL)
      {
      xTimerStart(m_ccDisableTimer, 0);
      }
```

When climate control is enabled, a timer (`m_ccDisableTimer`) starts to automatically disable it after a period.

#### Climate Status Detection
**File:** `vehicle_nissanleaf.cpp`, Lines 1309-1330

**CAN Message 0x54b** - HVAC Status Detection
```cpp
    case 0x54b:
      {
      uint8_t nlHvacStatus = (d[3] & 0x1e) >> 1;
      if (MyConfig.GetParamValueInt("xnl", "modelyear", DEFAULT_MODEL_YEAR) < 2013)
      {
        // this might be a bit field? So far these 6 values indicate HVAC on
        m_climate_remoteheat->SetValue(nlHvacStatus == 0x09 || nlHvacStatus == 0x12 || nlHvacStatus == 0x92);
        m_climate_remotecool->SetValue(nlHvacStatus == 0x09 || nlHvacStatus == 0x12 || nlHvacStatus == 0x92);
        m_climate_fan_only->SetValue(false);
        StandardMetrics.ms_v_env_hvac->SetValue(m_climate_remoteheat->AsBool() ||
                                                m_climate_remotecool->AsBool() ||
                                                m_climate_fan_only->AsBool());
      }
      else if (MyConfig.GetParamValueInt("xnl", "modelyear", DEFAULT_MODEL_YEAR) < 2016)
      {
        m_climate_remoteheat->SetValue(nlHvacStatus == 0x09);
        m_climate_remotecool->SetValue(nlHvacStatus == 0x12);
        m_climate_fan_only->SetValue(nlHvacStatus == 0x0c);
        StandardMetrics.ms_v_env_hvac->SetValue(m_climate_remoteheat->AsBool() ||
                                                m_climate_remotecool->AsBool() ||
                                                m_climate_fan_only->AsBool());
      }
```

**HVAC Status Values by Model Year:**

| Model Year | Heat Status | Cool Status | Fan Only | Values |
|------------|-------------|-------------|----------|---------|
| Pre-2013 | 0x09, 0x12, 0x92 | 0x09, 0x12, 0x92 | N/A | Multiple values |
| 2013-2015 | 0x09 | 0x12 | 0x0c | Distinct values |
| 2016+ | More detailed | (see line 1356+) | Advanced | Enhanced monitoring |

---

## 3. OBD Mode Handling

### Overview
The module supports OBD-II diagnostic requests using standard and extended modes. Mode 0x19 (DTC reading) is not explicitly implemented but can be accessed via the shell command interface.

### Supported OBD Modes

**File:** `vehicle_nissanleaf.cpp`, Lines 571-574
```cpp
    uint8_t mode = (req <= 0xffff) ? ((req & 0xff00) >> 8) : ((req & 0xff0000) >> 16);
    if (mode != 0x01 && mode != 0x02 && mode != 0x09 &&
        mode != 0x10 && mode != 0x1A && mode != 0x21 && mode != 0x22) {
      writer->puts("ERROR: mode must be one of: 01, 02, 09, 10, 1A, 21 or 22");
```

**Supported Modes:**
- **0x01** - Show current data
- **0x02** - Show freeze frame data
- **0x09** - Request vehicle information
- **0x10** - Manufacturer-specific (permanent DTCs)
- **0x1A** - Manufacturer-specific 
- **0x21** - Manufacturer-specific (Nissan extended)
- **0x22** - Manufacturer-specific (Read data by ID)

**Note:** Mode 0x19 (Read DTCs) is NOT in the allowed list but could be added.

### OBD Request Infrastructure

#### ObdRequest() Function
**File:** `vehicle_nissanleaf.cpp`, Lines 599-652

```cpp
bool OvmsVehicleNissanLeaf::ObdRequest(uint16_t txid, uint16_t rxid, uint32_t request, 
                                       string& response, int timeout_ms /*=3000*/, uint8_t bus)
  {
  OvmsMutexLock lock(&nl_obd_request);
  // prepare single poll:
  OvmsPoller::poll_pid_t poll[] = {
    { txid, rxid, 0, 0, { 1, 1, 1, 1 }, 0, ISOTP_STD },
    POLL_LIST_END
  };
  if (request < 0x10000) {
    poll[0].type = (request & 0xff00) >> 8;
    poll[0].pid = request & 0xff;
  } else {
    poll[0].type = (request & 0xff0000) >> 16;
    poll[0].pid = request & 0xffff;
  }
  poll[0].pollbus = bus;
  // stop default polling:
  PollSetPidList(NULL);
  vTaskDelay(pdMS_TO_TICKS(100));

  // clear rx semaphore, start single poll:
  nl_obd_rxwait.Take(0);
  nl_obd_rxbuf.clear();
  PollSetPidList(poll);

  // wait for response:
  bool rxok = nl_obd_rxwait.Take(pdMS_TO_TICKS(timeout_ms));
  if (rxok == pdTRUE) {
    response = nl_obd_rxbuf;
    nl_obd_rxbuf.clear();
    }

  // restore default polling:
  nl_obd_rxwait.Give();
  vTaskDelay(pdMS_TO_TICKS(100));
  if (cfg_ze1)
    {
    PollSetPidList(m_can1,obdii_polls_ze1);
    }
  else
    {
    PollSetPidList(m_can1,obdii_polls_aze0);
    }

  return (rxok == pdTRUE);
  }
```

**Process:**
1. Mutex lock to prevent concurrent requests
2. Create temporary poll list with requested mode/PID
3. Stop default polling (100ms delay)
4. Clear receive buffer and start single poll
5. Wait for response (3 second timeout default)
6. Restore default polling
7. Return success/failure

#### Shell Command Interface
**File:** `vehicle_nissanleaf.cpp`, Lines 530-597

```cpp
void OvmsVehicleNissanLeaf::shell_obd_request(int verbosity, OvmsWriter* writer, 
                                              OvmsCommand* cmd, int argc, const char* const* argv)
  {
    OvmsVehicleNissanLeaf* nl = GetInstance(writer);
    if (!nl)
      return;

    const char* strbus = cmd->GetParent()->GetName();
    uint16_t txid = 0, rxid = 0;
    uint32_t req = 0;
    uint8_t bus = 2; // default to CAR can
    string response;

    // parse args:
    string device = cmd->GetName();
    if (device == "device") {
      if (argc < 3) {
        writer->puts("ERROR: too few args, need: txid rxid request");
        return;
      }
      txid = strtol(argv[0], NULL, 16);
      rxid = strtol(argv[1], NULL, 16);
      req = strtol(argv[2], NULL, 16);
      bus = (strcmp(strbus,"can1") == 0 ? 1 : 2);
    } else {
      if (argc < 1) {
        writer->puts("ERROR: too few args, need: request");
        return;
      }
      req = strtol(argv[0], NULL, 16);
      if (device == "broadcast") {
        txid = BROADCAST_TXID;
        rxid = BROADCAST_RXID;
      }
    }
```

**Shell Command Examples:**
```bash
# Request via broadcast
OVMS# xnl can1 broadcast 0109

# Request to specific device
OVMS# xnl can1 device 79b 7bb 0101

# Request to charger
OVMS# xnl can2 device 797 79a 2212
```

### OBD Device IDs

**File:** `vehicle_nissanleaf.cpp`, Lines 48-59
```cpp
#define MAX_POLL_DATA_LEN         329
#define BMS_TXID                  0x79B
#define BMS_RXID                  0x7BB
#define CHARGER_TXID              0x797
#define CHARGER_RXID              0x79a
#define BROADCAST_TXID            0x7df
#define BROADCAST_RXID            0x0
// other pairs 743/763 744/764 745/765 784/78C 792/793 79D/7BD
#define VIN_PID                   0x81
#define QC_COUNT_PID              0x1203
#define L1L2_COUNT_PID            0x1205
```

| Device | TX ID | RX ID | Bus | Purpose |
|--------|-------|-------|-----|---------|
| BMS (Battery) | 0x79B | 0x7BB | CAN1 | Battery management data |
| Charger | 0x797 | 0x79a | CAN2 | Charging status, VIN |
| Broadcast | 0x7df | 0x0 | Both | General queries |

### Automatic Polling Configuration

#### ZE1 Models (2018+)
**File:** `vehicle_nissanleaf.cpp`, Lines 72-86
```cpp
static const OvmsPoller::poll_pid_t obdii_polls_ze1[] =
  {
    // BUS 2
    { CHARGER_TXID, CHARGER_RXID, VEHICLE_POLL_TYPE_OBDIIGROUP, VIN_PID, {  0, 3600, 0, 0 }, 2, ISOTP_STD },           // VIN [19] Never changes
    { CHARGER_TXID, CHARGER_RXID, VEHICLE_POLL_TYPE_OBDIIEXTENDED, QC_COUNT_PID, {  0, 0, 0, 3600 }, 2, ISOTP_STD },   // QC [2] Only changes when charging
    { CHARGER_TXID, CHARGER_RXID, VEHICLE_POLL_TYPE_OBDIIEXTENDED, L1L2_COUNT_PID, {  0, 0, 0, 3600 }, 2, ISOTP_STD }, // L0/L1/L2 [2]
    // BUS 1
    { BMS_TXID, BMS_RXID, VEHICLE_POLL_TYPE_OBDIIGROUP, 0x01, {  0, 60, 60, 60 }, 1, ISOTP_STD },   // bat [39/41]
    { BMS_TXID, BMS_RXID, VEHICLE_POLL_TYPE_OBDIIGROUP, 0x02, {  0, 60, 60, 60 }, 1, ISOTP_STD },   // battery voltages [196]
    { BMS_TXID, BMS_RXID, VEHICLE_POLL_TYPE_OBDIIGROUP, 0x06, {  0, 0, 0, 60 }, 1, ISOTP_STD },   // battery shunts [96] Only when charging
    { BMS_TXID, BMS_RXID, VEHICLE_POLL_TYPE_OBDIIGROUP, 0x04, {  0, 180, 180, 180 }, 1, ISOTP_STD }, // battery temperatures [14]
    { BMS_TXID, BMS_RXID, VEHICLE_POLL_TYPE_OBDIIGROUP, 0x61, {  0, 900, 900, 900 }, 1, ISOTP_STD }, // SOH for ZE1
    POLL_LIST_END
  };
```

#### AZE0 Models (2013-2017)
**File:** `vehicle_nissanleaf.cpp`, Lines 87-98
```cpp
  static const OvmsPoller::poll_pid_t obdii_polls_aze0[] =
  {
    // BUS 2
    { CHARGER_TXID, CHARGER_RXID, VEHICLE_POLL_TYPE_OBDIIGROUP, VIN_PID, {  0, 3600, 0, 0 }, 2, ISOTP_STD },           // VIN [19]
    { CHARGER_TXID, CHARGER_RXID, VEHICLE_POLL_TYPE_OBDIIEXTENDED, QC_COUNT_PID, {  0, 0, 0, 3600 }, 2, ISOTP_STD },   // QC [2]
    { CHARGER_TXID, CHARGER_RXID, VEHICLE_POLL_TYPE_OBDIIEXTENDED, L1L2_COUNT_PID, {  0, 0, 0, 3600 }, 2, ISOTP_STD }, // L0/L1/L2 [2]
    // BUS 1
    { BMS_TXID, BMS_RXID, VEHICLE_POLL_TYPE_OBDIIGROUP, 0x01, {  0, 60, 60, 60 }, 1, ISOTP_STD },   // bat [39/41]
    { BMS_TXID, BMS_RXID, VEHICLE_POLL_TYPE_OBDIIGROUP, 0x02, {  0, 60, 60, 60 }, 1, ISOTP_STD },   // battery voltages [196]
    { BMS_TXID, BMS_RXID, VEHICLE_POLL_TYPE_OBDIIGROUP, 0x06, {  0, 60, 60, 60 }, 1, ISOTP_STD },   // battery shunts [96]
    { BMS_TXID, BMS_RXID, VEHICLE_POLL_TYPE_OBDIIGROUP, 0x04, {  0, 180, 180, 180 }, 1, ISOTP_STD }, // battery temperatures [14]
    POLL_LIST_END
  };
```

**Poll States** - Lines 63-67:
```cpp
enum poll_states
  {
  POLLSTATE_OFF,      //- car is off
  POLLSTATE_ON,       //- car is on
  POLLSTATE_RUNNING,  //- car is in drive/reverse
  POLLSTATE_CHARGING  //- car is charging
  };
```

Polling intervals: `{OFF, ON, RUNNING, CHARGING}` in seconds
- 0 = don't poll in this state
- Poll frequency automatically adjusts based on vehicle state

### IncomingPollReply() Handler
**File:** `vehicle_nissanleaf.cpp`, Lines 914-965

Processes responses to OBD polls and routes to specialized handlers:
- `PollReply_Battery()` - Group 0x01 responses
- `PollReply_BMS_Volt()` - Group 0x02 responses  
- `PollReply_BMS_Shunt()` - Group 0x06 responses
- `PollReply_BMS_Temp()` - Group 0x04 responses
- `PollReply_BMS_SOH()` - Group 0x61 responses (ZE1 only)
- `PollReply_VIN()` - VIN requests
- `PollReply_QC()` - Quick charge count
- `PollReply_L0L1L2()` - Level 0/1/2 charge count

---

## 4. CAN Message IDs and Functions

### CAN1 (EV Bus) - Primary Vehicle Data

| CAN ID | Purpose | Data Description | Line Ref |
|--------|---------|------------------|----------|
| 0x1d4 | Battery Current | High-precision current measurement | 974 |
| 0x1da | Battery Voltage | Pack voltage | 982 |
| 0x1db | State of Charge | SOC from instrument cluster | 1007 |
| 0x1dc | Charge Status | Charging state information | 1070 |
| 0x284 | Vehicle Speed | Raw speed value (÷98 for km/h) | 1079 |
| 0x380 | Charger Status (ZE0) | Old charger protocol (2011-2012) | 1103 |
| 0x390 | Charger Status (AZE0) | New charger protocol (2013+) | 1144 |
| 0x50a | Ambient Temperature | Outside temperature | 1236 |
| 0x54a | Interior Temperature | Cabin temperature | 1244 |
| 0x54b | HVAC Status | Climate control state | 1254 |
| 0x54c | Climate Fan Speed | Fan speed setting | 1393 |
| 0x54f | Climate Settings | Temperature setpoint, mode | 1408 |
| 0x55a | Battery Temperature | Average battery temp | 1419 |
| 0x55b | SOC Bars | Battery charge bars (0-12) | 1428 |
| 0x59e | Energy Capacity | Battery kWh capacity (optional) | 1440 |
| 0x5bc | Battery Details | Amp-hours, GIDs, SOH | 1459 |
| 0x5bf | Charge Power | AC/DC charge power | 1604 |
| 0x5c0 | Battery Stats | Various battery metrics | 1659 |
| 0x679 | Unknown | (Monitored but not processed) | 1677 |

### CAN2 (CAR Bus) - Vehicle Control & Status

| CAN ID | Purpose | Data Description | Line Ref |
|--------|---------|------------------|----------|
| 0x180 | Throttle Position | Accelerator pedal position | 1689 |
| 0x292 | Brake Pressure | Foot brake application | 1695 |
| 0x355 | Odometer Units | Miles or Kilometers flag | 1701 |
| 0x385 | TPMS | Tire pressure (all 4 wheels) | 1714 |
| 0x421 | Gear Position | P/R/N/D/B selector | 1720 |
| 0x56e | **Remote Commands** | **Lock/Unlock/Climate/Charge** | 1938 |
| 0x5a9 | Range | Instrument cluster range estimate | 1744 |
| 0x5b3 | SOH | State of Health percentage | 1763 |
| 0x5b9 | Charge Time Estimate | Time to full from 3kW | 1753 |
| 0x5c5 | Parking Brake & Odometer | Handbrake + odometer reading | 1781 |
| 0x60d | Door & Light Status | All doors + headlights + start button | 1789 |

### Remote Command CAN ID: 0x56e

**File:** `vehicle_nissanleaf.cpp`, Line 1938-1942

This is the **primary command CAN ID** for all remote operations:
- Door lock/unlock
- Climate control enable/disable  
- Start/stop charging (some models)

**Message Format:**
- **CAN ID:** 0x56e
- **Type:** Standard (11-bit ID)
- **Length:** 1 byte (pre-2016) or 4 bytes (2016+)
- **Bus:** CAN1 (pre-2016) or CAN2 (2016+)

### OBD Diagnostic CAN IDs

| Function | TX ID | RX ID | Bus | Notes |
|----------|-------|-------|-----|-------|
| BMS Diagnostics | 0x79B | 0x7BB | 1 | Battery data, cell voltages, temps |
| Charger Diagnostics | 0x797 | 0x79a | 2 | VIN, charge counts |
| Broadcast | 0x7df | varies | Both | General diagnostic queries |

**Additional Diagnostic Pairs** (Line 57):
- 743/763
- 744/764  
- 745/765
- 784/78C
- 792/793
- 79D/7BD

### CAN Message Processing Flow

```
IncomingFrameCan1() [Line 968]
  ├─ 0x1db → SOC update
  ├─ 0x284 → Speed calculation
  ├─ 0x380/0x390 → Charger status
  ├─ 0x54b → HVAC status detection
  ├─ 0x5bc → Battery GIDs, Ah, SOH
  └─ 0x5bf → Charge power
  
IncomingFrameCan2() [Line 1683]
  ├─ 0x180 → Throttle
  ├─ 0x421 → Gear position → PollState transition
  ├─ 0x5a9 → Range estimate
  ├─ 0x5c5 → Odometer
  └─ 0x60d → Doors, lights, ignition
  
IncomingPollReply() [Line 914]
  ├─ Routes to PollReply_Battery()
  ├─ Routes to PollReply_BMS_Volt()
  ├─ Routes to PollReply_BMS_Temp()
  └─ Routes to specialized handlers
```

---

## 5. Architecture & Design Patterns

### Model Year Detection
The code extensively uses model year configuration to handle hardware/protocol differences:

**Configuration Key:** `xnl.modelyear` (default: 2012)

**Major Breakpoints:**
- **< 2013** - ZE0 generation (original Leaf)
  - Uses GPIO for EV SYSTEM ACTIVATION REQUEST
  - CAN1 for TCU commands
  - 1-byte command messages
  - Old charger protocol (0x380)
  
- **2013-2015** - AZE0 generation  
  - Uses CommandWakeup() instead of GPIO
  - Still CAN1 for TCU
  - New charger protocol (0x390)
  - Different HVAC status codes
  
- **>= 2016** - Later AZE0/ZE1
  - CAN2 (CAR bus) for TCU commands
  - 4-byte command messages
  - Enhanced climate control monitoring
  - More detailed diagnostic data

### Command Repetition Pattern
All remote commands use a repetition mechanism:

**Constants** (Lines 69-70):
```cpp
#define REMOTE_COMMAND_REPEAT_COUNT 24
#define ACTIVATION_REQUEST_TIME 10
```

**Process:**
1. Command initiated → ticker set to 24
2. Timer fires every 100ms (10 Hz)
3. Command sent, ticker decremented
4. Stops early if status change detected (lock/unlock)
5. For pre-2013: GPIO released after 1 second (10 × 100ms)

### Polling State Machine
**File:** `vehicle_nissanleaf.cpp`, Various lines

State transitions based on vehicle activity:
- **POLLSTATE_OFF** - Car off, no polling
- **POLLSTATE_ON** - Car on, periodic polling
- **POLLSTATE_RUNNING** - In gear, active polling  
- **POLLSTATE_CHARGING** - Charging, charge-specific polls

Triggered by:
- Gear position (0x421) - Line 1729, 1734, 1740
- Charge status changes
- Manual state changes

---

## 6. Key Configuration Parameters

**File:** `vehicle_nissanleaf.h`, Lines 58-75

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `modelyear` | 2012 | Selects protocol variant |
| `pin.ev` | 1 | GPIO pin for EV SYSTEM ACTIVATION REQUEST |
| `cabintemp.offset` | 0.0 | Cabin temp calibration |
| `suff.range.calc` | "ideal" | Range calculation method |
| `rangedrop` | 0 | Allowed range drop after charging |
| `socdrop` | 0 | Allowed SOC drop after charging |
| `autocharge.enabled` | true | Enable automatic charge control |
| `speed.divisor` | 98.0 | Speed calculation divisor |
| `soh.newcar` | varies | SOH calculation method |
| `canwrite` | varies | Enable/disable CAN commands |
| `ze1` | varies | Enable ZE1-specific features |
| `command.wakeup` | true | Enable TCU wakeup commands |

---

## 7. Safety & Error Handling

### Command Enablement Check
All commands verify `cfg_enable_write` before executing:

**File:** `vehicle_nissanleaf.cpp`
- Line 1847: `SendCommand()` - Returns early if disabled
- Line 2527: `RemoteCommandHandler()` - Returns Fail if disabled

### CAN Write Error Handling
**File:** `vehicle_nissanleaf.cpp`, Lines 1938-1942
```cpp
      result = tcuBus->WriteStandard(0x56e, length, data);
      if (result != ESP_OK)
      {
          ESP_LOGE(TAG, "CAN TX Error: 0x56e, result=%d", result);
      }
```

### OBD Timeout Protection
**File:** `vehicle_nissanleaf.cpp`, Lines 629-631
```cpp
  // wait for response:
  bool rxok = nl_obd_rxwait.Take(pdMS_TO_TICKS(timeout_ms));
```

Default timeout: 3000ms (3 seconds)

### Mutex Protection
OBD requests use mutex to prevent concurrent access:
```cpp
OvmsMutexLock lock(&nl_obd_request);
```

---

## 8. Notable Implementation Details

### MITM (Man-In-The-Middle) Stop Charging
**File:** `vehicle_nissanleaf.cpp`, Lines 1924-1936

```cpp
      case STOP_CHARGING:
        ESP_LOGI(TAG, "Stop Charging");
        if (!m_MITM) {
          m_MITM = 10;
          xTimerStart(m_MITMstop, 0);
        }
        break;
```

Stop charging uses a different mechanism (MITM) rather than a simple CAN command. This suggests the OVMS intercepts and blocks CAN messages to stop charging.

### Climate Control Auto-Disable
**File:** `vehicle_nissanleaf.cpp`, Lines 1969-1971, 1997-1998

When climate control is enabled, a timer is started to automatically send the AUTO_DISABLE command. This prevents the HVAC from running indefinitely.

### Speed Calculation
**File:** `vehicle_nissanleaf.cpp`, Lines 1079-1102

Raw speed value from CAN ID 0x284 is divided by configurable divisor (default 98.0) to get km/h.

### GID-based Range Calculation
**File:** `vehicle_nissanleaf.cpp`, Lines 1459-1602

The Nissan Leaf uses "GIDs" (charge bars × ~80 Wh) as a primary energy metric. The code tracks:
- Current GIDs (m_gids)
- Max GIDs (m_max_gids)  
- SOC calculation: `(gids / max_gids) * 100`

---

## Summary & Recommendations

### Current Capabilities
✅ Lock/unlock doors via CAN  
✅ Enable/disable climate control  
✅ Start/stop charging (MITM method for stop)  
✅ Read battery diagnostics (voltage, temp, SOH, SOC)  
✅ Read charger status  
✅ Monitor vehicle state (speed, gear, doors, etc.)  
✅ OBD-II modes: 01, 02, 09, 10, 1A, 21, 22  

### Missing/Limited Capabilities
❌ Mode 0x19 (Read DTCs) not in allowed mode list  
⚠️ No explicit DTC clearing function  
⚠️ Stop charging uses MITM, not direct command  

### Recommendations for Enhancement

1. **Add Mode 0x19 Support**  
   Modify line 571-574 to include mode 0x19 for DTC reading:
   ```cpp
   if (mode != 0x01 && mode != 0x02 && mode != 0x09 &&
       mode != 0x10 && mode != 0x19 && mode != 0x1A && 
       mode != 0x21 && mode != 0x22) {
   ```

2. **Implement DTC Clear (Mode 0x04)**  
   Add mode 0x04 support for clearing diagnostic trouble codes

3. **Document MITM Stop Charging**  
   The MITM mechanism needs better documentation - what messages are intercepted?

4. **Add DTC-specific Commands**  
   Create dedicated shell commands:
   ```cpp
   cmd_xnl->RegisterCommand("dtc-read", ...);
   cmd_xnl->RegisterCommand("dtc-clear", ...);
   ```

5. **Enhance Error Reporting**  
   Current error handling logs but doesn't expose errors to user easily

---

## File Structure Summary

```
vehicle_nissanleaf/src/
├── vehicle_nissanleaf.cpp    (104 KB) - Main implementation
│   ├── Polling setup (lines 72-98)
│   ├── CAN1 handler (lines 968-1682)
│   ├── CAN2 handler (lines 1683-1802)
│   ├── Remote commands (lines 1839-1998, 2525-2619)
│   └── OBD interface (lines 530-652)
│
├── vehicle_nissanleaf.h      (11 KB) - Class definition
│   ├── RemoteCommand enum (lines 82-89)
│   ├── Configuration defaults (lines 58-75)
│   └── Method declarations
│
├── nl_web.cpp                (18 KB) - Web interface
└── nl_types.h                (1.5 KB) - Utility macros
```

---

**End of Analysis**
