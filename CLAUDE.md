# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a TwinCAT 3 PLC project for a VDS (Vaccine Distribution System) skid that controls valves and pumps for fluid handling operations. The system manages 10 proportional control valves and at least one pump for processes like overpressure testing, WFI (Water for Injection) distribution, and dilution operations.

## Development Environment

- **Platform**: TwinCAT 3 (Beckhoff Industrial Automation)
- **Target**: TwinCAT 3.1.4026.19
- **Target NetId**: 192.168.43.43.1.1
- **PLC Task**: PlcTask runs at 100ms cycle time (Priority 20, Port 350)
- **Language**: IEC 61131-3 Structured Text (ST)

## Building and Running

TwinCAT projects are built and deployed through the TwinCAT XAE (eXtended Automation Engineering) shell, which integrates with Visual Studio. The solution file is located at:
- `VDS_Skid/VDS_Skid.sln`

The PLC project configuration is in `VDS_Skid/PLC_Skid.xti`.

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

### Global Variables (GVL.TcGVL)

**Constants**:
- `valveCount`: Fixed at 10 valves
- `sValveNames`: Array of descriptive valve names (e.g., "Dilution Selector", "WFI Filter Inlet")
- `stValveID`: Array of valve index values (1-10) for performance optimization

**I/O Mapping**: All physical I/O uses AT %I* (inputs) or AT %Q* (outputs) directives to link to EtherCAT terminals:
- Valve naming convention: `{DI1|DI2|AI1|DO1|AO1}_{valveName}`
- Example: `DI1_dilutionSV`, `AO1_wfiFilterIn`
- Pump controls: `DO_wfiPump` (on/off), `AO_wfiPump` (speed 0-10V)

### Function Blocks

**General FBs** (VDS_Skid/PLC_Skid/POUs/General FBs/):
- `FB_valveCTRL`: Core valve control logic, executes state machine (Open/Close, OpenLoop, PID modes) for all valves
- `FB_ValvePOS`: Positions multiple valves to open state based on valve index array, includes delay logic
- `FB_ValvePOSClose`: Closes multiple valves based on valve name array
- `FB_SetValvesOpen`: Sets multiple valves to open by matching names and setting State=1, isOpen=TRUE
- `FB_SetValvesClose`: Sets multiple valves to closed by matching names and setting State=0, isOpen=FALSE
- `FB_PulseValve`: Generates timed pulses for valve initialization

**Status FBs** (VDS_Skid/PLC_Skid/POUs/Status FBs/):
- `FB_CheckValveInputs`: Maps all 10 valve field inputs (DI1, DI2, AI1) from GVL to valve structures every cycle

**Unit Operation FBs** (VDS_Skid/PLC_Skid/POUs/UnitOp FBs/):
- `FBUO_OverPressure_1`: Orchestrates overpressure test sequence - positions valves via FB_ValvePos, controls pump, maps all valve outputs to GVL
- These FBs coordinate multiple valves and equipment for specific process operations

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

Recent optimization (commit 5a791b6) changed from string-based lookup to index-based for performance:
- Valve names are still stored as WSTRING in GVL.sValveNames for HMI/debugging
- New `stValveID` array provides INT indices (1-10) assigned in GVL
- `ST_Valves.ID` stores the index for each valve instance
- Function blocks should use ID-based lookup (see FB_ValvePOS) instead of name-based when possible
- Legacy FBs still use name-based lookup (FB_SetValvesOpen, FB_SetValvesClose) but should be migrated to index-based

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

1. Field inputs → GVL mapped variables (AT %I* declarations)
2. GVL inputs → Valve structure inputs (in FB_CheckValveInputs, every cycle)
3. Valve structure outputs → GVL mapped outputs (in Unit Operation FBs or MAIN)
4. GVL outputs → Field outputs (AT %Q* declarations)

## Important Conventions

- Prefix `fb` for function block input/output parameters (e.g., `fbValves`, `fbProcessState`)
- Prefix `h` for HMI/high-level control variables (e.g., `hOP_ProcessState`, `hOP_PumpSpeed`)
- Use `gvl.` to reference global variables
- Array indices in TwinCAT/IEC 61131-3 are 1-based, not 0-based
- Time literals use format `T#1500MS` for TIME type
- Comments use `//` for single line
- Always initialize valves to safe state (closed, State=0) on startup

## Recent Development Focus

Based on git history:
- Migration from name-based to index-based valve lookup for performance
- Implementing Unit Operation function blocks for process sequences (overpressure testing)
- Expanding from initial valve testing (few valves) to full 10-valve operational system
- Integration of pump control with valve sequencing
- OPC UA integration for SCADA/Ignition connectivity (mentioned in commit messages)
