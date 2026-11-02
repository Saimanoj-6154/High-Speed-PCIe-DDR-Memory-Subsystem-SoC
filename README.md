# High-Speed-PCIe-DDR-Memory-Subsystem-SoC
RTL design and full UVM verification environment for a memory subsystem that connects a PCI Express endpoint to a DDR4 memory controller through an AXI4 interconnect.

## Overview

Host requests arrive over PCIe as memory read and write transaction-layer packets (TLPs). The PCIe endpoint converts them into AXI4 transactions, the crossbar routes them to the DDR4 controller, and the controller issues DDR4 commands while honoring bank timing and refresh. A DMA engine moves larger blocks through the same fabric without host intervention.

The project is weighted toward verification. Each interface has a UVM agent, every block is checked by a scoreboard, SVA, or both, and each feature in the verification plan maps to a named test and coverage bins.

## Target Configuration
```
Parameter	                          Target value
Host interface	              PCIe Gen3, x4 (8 GT/s per lane)
Memory	                      DDR4-3200, x64 data bus
On-chip fabric	              AXI4, 256-bit data
HDL	                          SystemVerilog (IEEE 1800)
Verification	                UVM 1.2
Simulator	                    Questa / ModelSim
```
## Architecture

```
   Host (PCIe root complex model)          DMA engine
              |                                |
              v                                |
     +------------------+                      |
     | PCIe Endpoint    |                      |
     | TL / DLL / FC    |                      |
     +--------+---------+                      |
              | AXI4 master                    | AXI4 master
              v                                v
     +---------------------------------------------+
     |              AXI4 Crossbar                  |
     |     routing, arbitration, clock crossing    |
     +----------------------+----------------------+
                            | AXI4 slave
                            v
     +---------------------------------------------+
     |            DDR4 Memory Controller           |
     |  bank FSMs, command scheduler, refresh      |
     +----------------------+----------------------+
                            | DFI-style PHY interface
                            v
                  DDR4 PHY / DRAM model

```
The design has three clock domains: PCIe core, AXI fabric, and DDR controller. Crossings use asynchronous FIFOs and synchronizers. Clock and reset details are in
## Features
PCIe endpoint: memory read and write TLPs, completions, credit-based flow control, and ACK/NAK replay
AXI4 crossbar: multi-master arbitration, burst support, and clock-domain crossing
DDR4 controller: power-up and mode register initialization, per-bank state machines, command scheduling with an open-page policy, and auto-refresh
DMA engine: descriptor-driven transfers across the fabric
UVM environment: PCIe, AXI4, and DDR4 agents, a reference memory model, an end-to-end scoreboard, and protocol SVA
Coverage: functional covergroups for TLP types, burst lengths, credit states, bank conflicts, and refresh collisions
