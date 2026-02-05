# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a TwinCAT 3 PLC project for a VDS (Vaccine Distribution System) skid that controls valves and pumps for fluid handling operations. The system manages 10 proportional control valves and at least one pump for processes like overpressure testing, WFI (Water for Injection) distribution, and dilution operations.

## Development Environment

- **Platform**: TwinCAT 3 (Beckhoff Industrial Automation)
- **Target**: TwinCAT 3.1.4026.19
- **Target NetId**: 192.168.43.43.1.1
- **Language**: IEC 61131-3 Structured Text (ST)

### PLC Tasks

| Task | Cycle Time | Priority | Purpose |
|------|------------|----------|---------|
| PlcTask | 10ms | 20 | Main control loop (MAIN.TcPOU) |
| CyclicTask100ms | 100ms | 1 | PID operations (PRG_PID_CALL) |
| SerialTask | 2ms | 2 | Serial communication (SerialComm.TcPOU) |

## Building and Running

TwinCAT projects are built and deployed through the TwinCAT XAE (eXtended Automation Engineering) shell, which integrates with Visual Studio. The solution file is located at:
- `VDS_Skid/VDS_Skid.sln`

**Note**: This codebase does not use traditional build commands like `make` or `npm build`. Development and deployment are managed through TwinCAT XAE IDE.

## Architecture

### Core Control Loop (MAIN.TcPOU)

The MAIN program runs every PLC cycle and orchestrates all control operations:

1. **Initialization Phase**: On first cycle, initializes the `stValves` array (10 valves) with names from GVL, sets default states to closed (State=0, AO1=0.0)
2. **Input Monitoring**: `systemCheck_Valves` (FB_CheckValveInputs) continuously updates valve feedback signals (DI1, DI2, AI1) from field I/O every cycle
3. **Process Control**: Unit operation function blocks (e.g., `unitOp_OverPressure`) orchestrate multi-valve sequences and pump control
4. **Valve Control**: `valveCtrl` (FB_valveCTRL) translates valve states and setpoints into analog outputs (AO1) every cycle
5. **I/O Mapping**: Valve AO1 values are written to GVL mapped outputs which link to physical EtherCAT I/O

### Data Structures (DUTs)

**ST_Valves**: Primary valve data structure containing:
- Identification: `Name` (WSTRING), `ID` (INT index)
- Inputs: `DI1`/`DI2` (closed/open feedback), `AI1` (position 0-10V), `DO1` (init signal)
- Outputs: `AO1` (position setpoint 0-10V)
- Control: `State` (0=Off, 1=Open/Close, 2=OpenLoop, 3=PID), `isOpen` (BOOL command)
- Advanced: `Setpoint`, `PV`, `Output` (for OpenLoop/PID modes)
- Diagnostics: `Status` (0=OK, 1=Fault)

**ST_Pumps**: Pump data structure (currently minimal implementation)

### Global Variables

**GVL.TcGVL** - Constants and configuration:
- `valveCount`: Fixed at 10 valves
- `sValveNames`: Array of descriptive valve names (WSTRING)
- `stValveID`: Array of valve index values (1-10) for performance optimization

**IO.TcGVL** - Hardware I/O mapping (uses `{attribute 'qualified_only'}` - must reference as `IO.varName`):
- All physical I/O uses AT %I* (inputs) or AT %Q* (outputs) directives to link to EtherCAT terminals
- Valve naming convention: `{DI1|DI2|AI1|DO1|AO1}_{valveName}`
- Pump controls: `DO_wfiPump` (on/off), `AO_wfiPump` (speed 0-10V)
- Sensors: `AI_wfiPressureSensor` (REAL), `AI_wfiFlowMeter` (DINT)

**GVL_Serial.TcGVL** - Serial communication buffers:
- `stIn_PcCom` / `stOut_PcCom`: Serial I/O structures
- `RxBufferPcCom` / `TxBufferPcCom`: Receive/transmit buffers

**Valve Index to I/O Variable Mapping**:
| Index | Name | I/O Suffix |
|-------|------|------------|
| 1 | Dilution Selector | `_dilutionSV` |
| 2 | WFI Selector | `_wfiSV` |
| 3 | Formation Drain | `_formDrain` |
| 4 | WFI Filter Bypass Inlet | `_wfiBypassIn` |
| 5 | WFI Filter Inlet | `_wfiFilterIn` |
| 6 | WFI Filter Outlet | `_wfiFilterOut` |
| 7 | WFI Filter Bypass Outlet | `_wfiBypassOut` |
| 8 | Dilution Diverter | `_dilutionDiV` |
| 9 | Formation Manifold | `_FormManifold` |
| 10 | Suspension Manifold | `_SuspManifold` |

### Function Blocks

**General FBs** (VDS_Skid/PLC_Skid/POUs/General FBs/):
- `FB_valveCTRL`: Core valve control logic, executes state machine (Open/Close, OpenLoop, PID modes) for all valves
- `FB_ValvePOS`: Sequential valve positioning with state machine. Supports separate open/close sequences with edge detection on DI2 feedback and configurable delays (750ms between valves, 5s timeout)
- `FB_PulseValve`: Generates timed pulses for valve initialization
- `FB_MovingAverage`: Circular buffer moving average filter (configurable 1-100 samples) for signal smoothing
- `FB_ScaleAnalog`: Linear scaling for analog inputs (e.g., 4-20mA to engineering units) with range validation

**Status FBs** (VDS_Skid/PLC_Skid/POUs/Status FBs/):
- `FB_CheckValveInputs`: Maps all 10 valve field inputs (DI1, DI2, AI1) from IO GVL to valve structures every cycle

**Unit Operation FBs** (VDS_Skid/PLC_Skid/POUs/UnitOp FBs/):
- `FBUO_OverPressure`: Orchestrates overpressure test sequence - positions valves via FB_ValvePOS, controls pump, maps all valve outputs to IO GVL
- `FBUO_OverPressure_EX`: Extended variant opening 9 valves [2,3,4,5,6,7,8,9,10]
- These FBs coordinate multiple valves and equipment for specific process operations

### Serial Communication (SerialComm.TcPOU)

Communicates with Atlas peristaltic pump via USB serial (COM8, 57600 baud, 8N1):
- **State 0 (IDLE)**: Wait for start command
- **State 10 (SEND)**: Transmit command string
- **State 20 (WAIT)**: Receive response with 1s timeout
- Uses `fbSerialLineControlADS` for port management

### I/O Hardware Configuration

The system uses two EtherCAT networks with IOLite distributed I/O:
- **Device 1**: Basic EtherCAT with EK1200 coupler
- **Device 2**: IOLite GATE LATCH modules with multiple I/O types:
  - 32xDI (Digital Input modules)
  - 16xLV (Low Voltage Digital Output)
  - 16xAO (Analog Output for valve control)
  - 32xDO (Digital Output)
  - 6xSTG (Strain Gauge inputs)
  - 4xCNT (Counter modules)
  - 8xRTD-HS (RTD Temperature inputs)

Configuration files are in `VDS_Skid/_Config/IO/` as .xti XML files.

## Coding Patterns

### Valve Index vs Name Lookup

Use index-based valve lookup for performance:
- Valve names are stored as WSTRING in GVL.sValveNames for HMI/debugging only
- `stValveID` array provides INT indices (1-10) assigned in GVL
- `ST_Valves.ID` stores the index for each valve instance
- All function blocks should use ID-based lookup (see FB_ValvePOS pattern)

### Valve Control State Machine

Valve control follows this pattern (implemented in FB_valveCTRL):
```
State 0: Off/Closed - AO1 = 0.0V
State 1: Open/Close - AO1 = 10.0V if isOpen=TRUE, else 0.0V
State 2: OpenLoop - AO1 = Setpoint (manual position control)
State 3: PID - Reserved for future closed-loop control
```

### VAR_IN_OUT for Valve Arrays

All FBs that modify valve states use `VAR_IN_OUT` for the valve array to ensure changes persist:
```
VAR_IN_OUT
    fbValves : ARRAY[1..GVL.valveCount] OF ST_Valves;
END_VAR
```

### I/O Mapping Flow

1. Field inputs → IO GVL mapped variables (AT %I* declarations in IO.TcGVL)
2. IO GVL inputs → Valve structure inputs (in FB_CheckValveInputs, every cycle)
3. Valve structure outputs → IO GVL mapped outputs (in Unit Operation FBs)
4. IO GVL outputs → Field outputs (AT %Q* declarations)

**Important**: IO GVL uses `{attribute 'qualified_only'}`, so all references must use `IO.` prefix (e.g., `IO.AO1_wfiSV := fbValves[2].AO1;`)

## Important Conventions

- Prefix `fb` for function block input/output parameters (e.g., `fbValves`, `fbProcessState`)
- Prefix `h` for HMI/high-level control variables (e.g., `hOP_ProcessState`, `hOP_PumpSpeed`)
- Use `GVL.` for constants, `IO.` for I/O variables (IO requires qualified access)
- Array indices in TwinCAT/IEC 61131-3 are 1-based, not 0-based
- Time literals use format `T#1500MS` for TIME type
- Comments use `//` for single line
- Always initialize valves to safe state (closed, State=0) on startup
- Prefix `st` for structure instances (e.g., `stValves`, `stSerialConfig`)
