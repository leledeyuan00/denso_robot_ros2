# ESI_ANALYSIS.md — Detailed ESI XML Analysis for RC8 ECS MOTION

## 1. Device Identity

```
Vendor ID:     0x0000078A  (DENSO WAVE INCORPORATED)
Product Code:  0x00000001
Revision:      0x00000001
Device Name:   RC8 ECS MOTION
Profile:       CiA 402 (IEC 61800-7-204), 8 channels
Device Type:   0x00020192  (Profile 402, device type MC drive)
```

## 2. Communication Configuration

### SyncManager Assignment
| SM  | Direction | Start Addr | Size    | Purpose     |
|-----|-----------|------------|---------|-------------|
| SM0 | Output    | 0x1000     | 128 B   | MBoxOut     |
| SM1 | Input     | 0x1080     | 128 B   | MBoxIn      |
| SM2 | Output    | 0x1100     | 51 B    | RxPDO (process data out) |
| SM3 | Input     | 0x1400     | 84 B    | TxPDO (process data in)  |

### Mailbox
- CoE with SDO Info = true
- PDO Assign = false (fixed mapping)
- PDO Config = false (fixed mapping)
- Complete Access = false
- Segmented SDO = false

### Distributed Clock
- DC mode: DcOff (0x0000) — DC is **not used**
- This means Free Run or SM-Synchronous mode only

### Minimum Cycle Time
- 0x0003D090 = 250,000 ns = 250 µs = 4 kHz max update rate

## 3. RxPDO Mapping (Master → Slave, SM2)

### 0x1600: Axis 1 Receive PDO (Fixed)
| Object | SubIndex | BitLen | Name | DataType |
|--------|----------|--------|------|----------|
| 0x6040 | 0 | 16 | Controlword(J1) | UINT |
| 0x607A | 0 | 32 | Target Position(J1) | DINT |

### 0x1610: Axis 2 Receive PDO (Fixed)
| Object | SubIndex | BitLen | Name | DataType |
|--------|----------|--------|------|----------|
| 0x6840 | 0 | 16 | Controlword(J2) | UINT |
| 0x687A | 0 | 32 | Target Position(J2) | DINT |

### 0x1620: Axis 3 Receive PDO (Fixed)
| Object | SubIndex | BitLen | Name | DataType |
|--------|----------|--------|------|----------|
| 0x7040 | 0 | 16 | Controlword(J3) | UINT |
| 0x707A | 0 | 32 | Target Position(J3) | DINT |

### 0x1630: Axis 4 Receive PDO (Fixed)
| Object | SubIndex | BitLen | Name | DataType |
|--------|----------|--------|------|----------|
| 0x7840 | 0 | 16 | Controlword(J4) | UINT |
| 0x787A | 0 | 32 | Target Position(J4) | DINT |

### 0x1640: Axis 5 Receive PDO (Fixed)
| Object | SubIndex | BitLen | Name | DataType |
|--------|----------|--------|------|----------|
| 0x8040 | 0 | 16 | Controlword(J5) | UINT |
| 0x807A | 0 | 32 | Target Position(J5) | DINT |

### 0x1650: Axis 6 Receive PDO (Fixed)
| Object | SubIndex | BitLen | Name | DataType |
|--------|----------|--------|------|----------|
| 0x8840 | 0 | 16 | Controlword(J6) | UINT |
| 0x887A | 0 | 32 | Target Position(J6) | DINT |

### 0x1660: Axis 7 Receive PDO (Fixed)
| Object | SubIndex | BitLen | Name | DataType |
|--------|----------|--------|------|----------|
| 0x9040 | 0 | 16 | Controlword(J7) | UINT |
| 0x907A | 0 | 32 | Target Position(J7) | DINT |

### 0x1670: Axis 8 Receive PDO (Fixed)
| Object | SubIndex | BitLen | Name | DataType |
|--------|----------|--------|------|----------|
| 0x9840 | 0 | 16 | Controlword(J8) | UINT |
| 0x987A | 0 | 32 | Target Position(J8) | DINT |

### 0x1680: I/O Receive PDO (Fixed)
| Object | SubIndex | BitLen | Name | DataType |
|--------|----------|--------|------|----------|
| 0x3010 | 0 | 16 | Mini IO (output) | UINT |
| 0x3011 | 0 | 8  | Hand IO (output) | USINT |

**Total RxPDO: 8 × (16+32) + 16 + 8 = 408 bits = 51 bytes**

## 4. TxPDO Mapping (Slave → Master, SM3)

### 0x1A00: Axis 1 Transmit PDO (Fixed)
| Object | SubIndex | BitLen | Name | DataType |
|--------|----------|--------|------|----------|
| 0x6041 | 0 | 16 | Statusword(J1) | UINT |
| 0x6064 | 0 | 32 | Position Actual Value(J1) | DINT |
| 0x6078 | 0 | 16 | Current Actual Value(J1) | INT |
| 0x6074 | 0 | 16 | Torque Reference Value(J1) | INT |

### 0x1A10–0x1A70: Axes 2–8 (same structure, different base index)
Each axis: Statusword + Position Actual + Current Actual + Torque Reference = 80 bits

### 0x1A80: I/O Transmit PDO (Fixed)
| Object | SubIndex | BitLen | Name | DataType |
|--------|----------|--------|------|----------|
| 0x3000 | 0 | 16 | Mini IO (input) | UINT |
| 0x3001 | 0 | 8  | Hand IO (input) | USINT |
| 0x3002 | 0 | 8  | Status IO | USINT |

**Total TxPDO: 8 × (16+32+16+16) + 16 + 8 + 8 = 672 bits = 84 bytes**

## 5. RxPDO Byte Offset Map (SM2, 51 bytes)

This is the exact byte layout in the process data image:

| Byte Offset | Size | Axis | Content |
|-------------|------|------|---------|
| 0–1 | 2 | J1 | Controlword |
| 2–5 | 4 | J1 | Target Position |
| 6–7 | 2 | J2 | Controlword |
| 8–11 | 4 | J2 | Target Position |
| 12–13 | 2 | J3 | Controlword |
| 14–17 | 4 | J3 | Target Position |
| 18–19 | 2 | J4 | Controlword |
| 20–23 | 4 | J4 | Target Position |
| 24–25 | 2 | J5 | Controlword |
| 26–29 | 4 | J5 | Target Position |
| 30–31 | 2 | J6 | Controlword |
| 32–35 | 4 | J6 | Target Position |
| 36–37 | 2 | J7 | Controlword |
| 38–41 | 4 | J7 | Target Position |
| 42–43 | 2 | J8 | Controlword |
| 44–47 | 4 | J8 | Target Position |
| 48–49 | 2 | I/O | Mini IO (output) |
| 50 | 1 | I/O | Hand IO (output) |

## 6. TxPDO Byte Offset Map (SM3, 84 bytes)

| Byte Offset | Size | Axis | Content |
|-------------|------|------|---------|
| 0–1 | 2 | J1 | Statusword |
| 2–5 | 4 | J1 | Position Actual |
| 6–7 | 2 | J1 | Current Actual |
| 8–9 | 2 | J1 | Torque Reference |
| 10–11 | 2 | J2 | Statusword |
| 12–15 | 4 | J2 | Position Actual |
| 16–17 | 2 | J2 | Current Actual |
| 18–19 | 2 | J2 | Torque Reference |
| 20–21 | 2 | J3 | Statusword |
| 22–25 | 4 | J3 | Position Actual |
| 26–27 | 2 | J3 | Current Actual |
| 28–29 | 2 | J3 | Torque Reference |
| 30–31 | 2 | J4 | Statusword |
| 32–35 | 4 | J4 | Position Actual |
| 36–37 | 2 | J4 | Current Actual |
| 38–39 | 2 | J4 | Torque Reference |
| 40–41 | 2 | J5 | Statusword |
| 42–45 | 4 | J5 | Position Actual |
| 46–47 | 2 | J5 | Current Actual |
| 48–49 | 2 | J5 | Torque Reference |
| 50–51 | 2 | J6 | Statusword |
| 52–55 | 4 | J6 | Position Actual |
| 56–57 | 2 | J6 | Current Actual |
| 58–59 | 2 | J6 | Torque Reference |
| 60–61 | 2 | J7 | Statusword |
| 62–65 | 4 | J7 | Position Actual |
| 66–67 | 2 | J7 | Current Actual |
| 68–69 | 2 | J7 | Torque Reference |
| 70–71 | 2 | J8 | Statusword |
| 72–75 | 4 | J8 | Position Actual |
| 76–77 | 2 | J8 | Current Actual |
| 78–79 | 2 | J8 | Torque Reference |
| 80–81 | 2 | I/O | Mini IO (input) |
| 82 | 1 | I/O | Hand IO (input) |
| 83 | 1 | I/O | Status IO |

## 7. Supported Drive Modes

The `Supported Drive Modes` object (0x6502 and equivalents) has value **0x00000080** in the stock ESI.

Bit 7 = 1 → **Cyclic Synchronous Position (CSP, mode 0x08)** supported.

**Vendor-unlocked extension:** DENSO WAVE has provided custom firmware that also
enables **Cyclic Synchronous Velocity (CSV, mode 0x09)**. The actual value of
Supported Drive Modes on the unlocked firmware may be **0x00000180** (bits 7+8).

### Mode Selection via SDO
To switch modes, write to `Modes of Operation` (0x6060 / 0x6860 / ... per axis):
- 0x08 = CSP (Cyclic Synchronous Position) — DEFAULT
- 0x09 = CSV (Cyclic Synchronous Velocity) — VENDOR-UNLOCKED

Mode must be set **before** transitioning to OPERATION_ENABLED.
Confirm by reading `Modes of Operation Display` (0x6061 / 0x6861 / ...).

### CSP Mode Behavior
- Master sends Target Position every cycle
- Slave interpolates internally
- Master is responsible for trajectory planning and smooth position commands

### CSV Mode Behavior
- Master sends Target Velocity every cycle
- Slave integrates velocity to update position internally
- Master must ensure velocity ramps to 0 on stop
- **OPEN QUESTION:** How is Target Velocity transmitted? See CLAUDE.md for details.
  Possible: reuse Target Position PDO field, alternate PDO mapping, or SDO-only.

### Target Velocity Objects (per axis, all PDO-mappable)
| Axis | Target Velocity | Velocity Offset | Velocity Actual |
|------|----------------|-----------------|-----------------|
| J1 | 0x60FF | 0x60B1 | 0x606C |
| J2 | 0x68FF | 0x68B1 | 0x686C |
| J3 | 0x70FF | 0x70B1 | 0x706C |
| J4 | 0x78FF | 0x78B1 | 0x786C |
| J5 | 0x80FF | 0x80B1 | 0x806C |
| J6 | 0x88FF | 0x88B1 | 0x886C |
| J7 | 0x90FF | 0x90B1 | 0x906C |
| J8 | 0x98FF | 0x98B1 | 0x986C |

## 8. Additional SDO Objects (Not in PDO, accessible via CoE/SDO)

### Per-axis objects (using J1 base 0x60xx as example):
| Index | Name | Type | Access | Notes |
|-------|------|------|--------|-------|
| 0x6060 | Modes of Operation | SINT | rw | Set to 0x08 for CSP |
| 0x6061 | Modes of Operation Display | SINT | ro | Current mode |
| 0x6065 | Following Error Window | UDINT | rw | Error threshold |
| 0x6066 | Following Error Time Out | UINT | rw | Error timeout |
| 0x606C | Velocity Actual Value | DINT | ro | Actual velocity |
| 0x6077 | Torque Actual Value | INT | ro | Actual torque |
| 0x607B | Position Range Limit | struct | rw | Min/Max limits |
| 0x607D | Software Position Limit | struct | rw | Software limits |
| 0x6080 | Max Motor Speed | REAL | ro | Max speed |
| 0x60B1 | Velocity Offset | DINT | rw | PDO-mappable |
| 0x60B2 | Torque Offset | INT | rw | PDO-mappable |
| 0x60C2 | Interpolation Time Period | struct | rw | Cycle time config |
| 0x60C5 | Max Acceleration | REAL | ro | Max accel |
| 0x60D9 | Supported Sync Functions | UDINT | ro | = 0x04 |
| 0x60DA | Supported Sync Settings | UDINT | rw | |
| 0x60F4 | Following Error Actual | DINT | ro | Current error |
| 0x60FF | Target Velocity | DINT | rw | PDO-mappable |

### Manufacturer-specific objects:
| Index | Name | Type | Access |
|-------|------|------|--------|
| 0x2100 | Error Count | UDINT | ro |
| 0x2101 | Error Record | array[64] of UDINT | ro |
| 0x2200 | Mass of Payload | UDINT | ro |
| 0x2201–0x2208 | Load Rate (J1–J8) | REAL | ro |

## 9. I/O Objects
| Index | Name | Type | Direction | PDO |
|-------|------|------|-----------|-----|
| 0x3000 | Mini IO (input) | UINT16 | Slave→Master | TxPDO |
| 0x3001 | Hand IO (input) | USINT8 | Slave→Master | TxPDO |
| 0x3002 | Status IO | USINT8 | Slave→Master | TxPDO |
| 0x3010 | Mini IO (output) | UINT16 | Master→Slave | RxPDO |
| 0x3011 | Hand IO (output) | USINT8 | Master→Slave | RxPDO |

## 10. EEPROM Configuration
```
ByteSize: 65536 (64 KB)
ConfigData: 06 00 00 00 e8 03
  → SM watchdog divider: 0x0006
  → SM watchdog: 0x03E8 (1000 = 100ms default watchdog)
```
