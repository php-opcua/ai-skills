# Catalogue — the 51 companion specifications

One-line description per spec. Use this to pick the right registrar for a given domain. Folder names under `src/` match the registrar class prefix.

## Foundational / cross-cutting

| Spec | Folder | What it covers | Direct dependencies |
| --- | --- | --- | --- |
| **DI** (Device Integration) | `DI/` | Generic device modelling — `DeviceType`, `IVendorNameplateType`, health states. The base nearly every other spec depends on. | — |
| **Machinery** | `Machinery/` | Machine-level state machines (Operation, Mode, Production), `MachineryItemIdentificationType`, identification + life-cycle. | DI |
| **PADIM** | `PADIM/` | Process-automation device information model. Generic process-control devices. | DI |
| **IA** (Industrial Automation) | `IA/` | Foundational types reused by domain-specific specs (IO-Link, sensors). | DI |
| **FDI / FDT** | `FDI/`, `FDT/` | Field Device Integration / Field Device Tool wrappers (device proxies). | DI |
| **AML** | `AML/` | AutomationML mapping (engineering data exchange). | DI |
| **DEXPI** | `DEXPI/` | Data Exchange in the Process Industry — P&ID schematics. | DI |

## Manufacturing / machine tools

| Spec | Folder | What it covers | Direct dependencies |
| --- | --- | --- | --- |
| **MachineTool** | `MachineTool/` | CNC machine tools, work pieces, axes, tools. | Machinery, DI |
| **UA-Companion-MachineTool** | `UA-Companion-MachineTool/` | Higher-level machine-tool orchestration. | MachineTool |
| **CNC** | `CNC/` | NC-program execution, axis groups. | Machinery, DI |
| **NCG** | `NCG/` | Numerical-control gateway. | DI |
| **CuttingTool** | `CuttingTool/` | Cutting-tool geometry, identification, lifecycle. | MachineTool |
| **TMC** (Tooling Management Communications) | `TMC/` | Tool inventory + lifecycle management. | CuttingTool |
| **Tightening** | `Tightening/` | Tightening/fastening processes (screwdrivers, torque tools). | Machinery, DI |
| **Robotics** | `Robotics/` | Industrial robots — axes, motion, operational modes. | Machinery, DI |
| **OPCRobotics** | `OPCRobotics/` | Robot-specific extensions. | Robotics |
| **CommercialKitchenEquipment** | `CommercialKitchenEquipment/` | Commercial-kitchen appliances. | Machinery, DI |
| **CranesHoists** | `CranesHoists/` | Cranes and hoist controllers. | Machinery, DI |
| **CSPPlusForMachine** | `CSPPlusForMachine/` | CSP+ for machinery (Japanese vendor consortium). | Machinery |
| **PERFM** | `PERFM/` | Performance / OEE reporting. | Machinery |
| **Scales** | `Scales/` | Weighing scales. | Machinery |
| **Weighing** | `Weighing/` | Industrial weighing systems. | Scales, Machinery |
| **Woodworking** | `Woodworking/` | Woodworking-machine companion spec. | Machinery, DI |
| **MES** | `MES/` | Manufacturing execution system integration. | DI |
| **ISA-95** | (under `MES`-related folders) | ERP↔MES↔shopfloor data exchange. | DI |

## Process / control

| Spec | Folder | What it covers | Direct dependencies |
| --- | --- | --- | --- |
| **PackML** | `PackML/` | Packaging-machine state model (PackML/IEC). | Machinery |
| **PLCopen** | `PLCopen/` | IEC 61131-3 PLC types. | DI |
| **IEC61131-5** | (folder name varies) | IEC 61131-5 communication function blocks. | DI |
| **Safety** | `Safety/` | Safety controllers, safety state machines. | Machinery |
| **POWERLINK** | `POWERLINK/` | Ethernet POWERLINK transport. | DI |
| **PROFINET** | `PROFINET/` | PROFINET device + I/O modules. | DI |
| **PROFIBUS** | `PROFIBUS/` | PROFIBUS slave model. | DI |
| **GPOS** | `GPOS/` | General Purpose Object Spec (deprecated; legacy). | DI |
| **TightSpecVOC** | `TightSpecVOC/` | Tight-spec VOC monitoring. | DI |

## Vision / quality

| Spec | Folder | What it covers | Direct dependencies |
| --- | --- | --- | --- |
| **MachineVision** | `MachineVision/` | Vision systems — cameras, recipes, results. | Machinery |
| **AutoID** | `AutoID/` | RFID, barcode, OCR readers — identification capture. | DI |
| **IEEE Auto-ID** | (folder name varies) | IEEE-standardised auto-ID protocols. | AutoID |

## Lab / analytical

| Spec | Folder | What it covers | Direct dependencies |
| --- | --- | --- | --- |
| **LADS** | `LADS/` | Laboratory & analytical device standard. | Machinery, DI |
| **ADI** (Adaptive Diagnostics) | `ADI/` | Adaptive process-monitoring / analytical instruments. | DI |
| **AMB** | `AMB/` | Analytical methodology blocks. | ADI |

## Building / infrastructure

| Spec | Folder | What it covers | Direct dependencies |
| --- | --- | --- | --- |
| **BACnet** | `BACnet/` | BACnet-to-OPC-UA gateway types — HVAC, lighting, BMS. | DI |
| **ECM** | `ECM/` | Energy / consumption management. | DI |
| **GDS** | `GDS/` | Global Discovery Server. | DI |
| **CAS** | `CAS/` | Certificate Authority Service. | DI |
| **ServerCapabilities** | `ServerCapabilities/` | Server-capability profile types. | — |
| **RPI** | `RPI/` | RFID-system Plug-in Interface. | AutoID |

## Architecture / meta

| Spec | Folder | What it covers | Direct dependencies |
| --- | --- | --- | --- |
| **I4AAS** | `I4AAS/` | Industry 4.0 Asset Administration Shell mapping. | DI |
| **NodeSet-Sercos** | `NodeSet-Sercos/` | Sercos III drive / I/O bus mapping. | DI |
| **MTConnect** | `MTConnect/` | MTConnect data model bridge. | Machinery |
| **PubSub** | `PubSub/` | OPC UA Part 14 PubSub configuration types (DataSet, WriterGroup, ReaderGroup). | — |

## How to find the spec you need

```bash
# List every spec folder under src/
ls src/

# Find the registrar for a known spec name
find src -name '*Registrar.php' | head -20

# See which specs ship a particular DataType (e.g. AxisInformation)
find src -name 'AxisInformation.php'
```

## Spec families that go together

For applications that integrate with one industrial domain, load this set:

- **Robotics application**: `RoboticsRegistrar` (auto-pulls DI + Machinery)
- **Machine tool / CNC**: `MachineToolRegistrar` (auto-pulls Machinery + DI + IA)
- **Lab automation**: `LADSRegistrar` (auto-pulls Machinery + DI)
- **Building automation (HVAC, lighting)**: `BACnetRegistrar` (auto-pulls DI)
- **Packaging line**: `PackMLRegistrar` (auto-pulls Machinery)
- **Process plant**: `PADIMRegistrar` + `DEXPIRegistrar` (both pull DI)
- **Asset tracking / RFID**: `AutoIDRegistrar` (pulls DI)
- **Sensor / IO-Link gateway**: `IARegistrar` (pulls DI)

For multi-domain applications (a factory with robots + CNC + vision + safety), load all relevant leaf registrars — `loadGeneratedTypes()` de-duplicates the shared bases (DI, Machinery) automatically.

## How to verify a server's specs

```php
// Read the server's namespace table — i=2255 = Server_NamespaceArray
$namespaces = $client->read('i=2255')->getValue();
foreach ($namespaces as $idx => $uri) {
    echo "[$idx] $uri\n";
}
```

Spec namespace URIs follow the pattern `http://opcfoundation.org/UA/<SpecName>/`. Match the URIs against this catalogue to pick the registrars.
