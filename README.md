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
* PCIe endpoint: memory read and write TLPs, completions, credit-based flow control, and ACK/NAK replay
* AXI4 crossbar: multi-master arbitration, burst support, and clock-domain crossing
* DDR4 controller: power-up and mode register initialization, per-bank state machines, command scheduling with an open-page policy, and auto-refresh
* DMA engine: descriptor-driven transfers across the fabric
* UVM environment: PCIe, AXI4, and DDR4 agents, a reference memory model, an end-to-end scoreboard, and protocol SVA
* Coverage: functional covergroups for TLP types, burst lengths, credit states, bank conflicts, and refresh collisions

## Verification Environment

* Agents: PCIe, AXI4, and DDR4 agents, each with a driver, monitor, sequencer, and configuration object. Agents are active or passive per interface.
* Virtual sequencer: coordinates sequences across agents for end-to-end scenarios.
* Reference model: a byte-addressable memory model that predicts the data each read should return.
* Scoreboard: compares DUT outputs against the reference model at the PCIe completion and DDR4 command levels.
* SVA: protocol checks bound into the DUT with bind. These cover AXI4 handshakes and DDR4 timing parameters (tRCD, tRP, tRAS, tRRD, tFAW, tREFI, tRFC).
* Coverage: functional covergroups with cross-coverage of read/write mix against bank activity.

## Repository Structure

```
pcie-ddr-soc/
├── README.md
├── docs/
│   ├── architecture.md          # block diagram, clock and reset domains, address map
│   ├── verification_plan.md     # features mapped to tests, checks, and coverage bins
│   └── images/                  # diagrams and waveform screenshots
├── rtl/
│   ├── top/
│   │   └── soc_top.sv           # top-level integration and parameters
│   ├── pcie/
│   │   ├── pcie_ep_top.sv       # endpoint wrapper
│   │   ├── pcie_tl.sv           # transaction layer: TLP decode and generate
│   │   ├── pcie_dll.sv          # data link layer: sequence numbers, ACK/NAK, replay
│   │   ├── pcie_fc.sv           # credit-based flow control
│   │   └── pcie_phy_if.sv       # PHY-facing interface adapter
│   ├── interconnect/
│   │   └── axi_xbar.sv          # AXI4 crossbar with arbitration and clock crossing
│   ├── dma/
│   │   └── dma_engine.sv        # descriptor-driven transfer engine
│   ├── ddr/
│   │   ├── ddr_ctrl_top.sv      # controller top
│   │   ├── ddr_init.sv          # power-up and mode register sequence
│   │   ├── ddr_bank_fsm.sv      # per-bank state machine
│   │   ├── ddr_cmd_sched.sv     # command scheduler (open-page policy)
│   │   ├── ddr_refresh.sv       # auto-refresh (tREFI / tRFC)
│   │   └── ddr_phy_if.sv        # DFI-style PHY interface
│   └── common/
│       ├── fifo_sync.sv
│       ├── fifo_async.sv
│       └── sync_ff.sv           # 2-flop synchronizer
├── dv/
│   ├── Makefile
│   ├── tb/
│   │   ├── tb_top.sv            # clocks, resets, DUT and interface instances
│   │   └── interfaces/          # pcie_if.sv, axi4_if.sv, ddr4_if.sv
│   ├── env/
│   │   ├── soc_env.sv
│   │   ├── soc_virtual_seqr.sv
│   │   ├── soc_ref_model.sv     # byte-addressable reference memory
│   │   ├── soc_scoreboard.sv
│   │   └── soc_coverage.sv
│   ├── agents/
│   │   ├── pcie_agent/          # driver, monitor, sequencer, config, seq_item
│   │   ├── axi4_agent/
│   │   └── ddr4_agent/
│   ├── sva/
│   │   ├── axi4_assert.sv
│   │   ├── ddr4_timing_assert.sv
│   │   └── pcie_assert.sv
│   ├── seq/
│   │   ├── base_seq.sv
│   │   ├── pcie_mem_seq.sv
│   │   ├── ddr_traffic_seq.sv
│   │   └── dma_seq.sv
│   ├── tests/
│   │   ├── base_test.sv
│   │   ├── pcie_link_up_test.sv
│   │   ├── mem_rd_wr_test.sv
│   │   ├── fc_stress_test.sv
│   │   ├── dll_replay_test.sv
│   │   ├── ddr_init_test.sv
│   │   ├── ddr_refresh_test.sv
│   │   ├── ddr_timing_test.sv
│   │   ├── axi_burst_test.sv
│   │   ├── dma_stress_test.sv
│   │   ├── cdc_stress_test.sv
│   │   └── e2e_integrity_test.sv
│   └── regress/
│       └── smoke.list
├── models/
│   └── README.md                # where to place the DDR4 device model
├── scripts/
│   └── cov_merge.tcl
└── reports/                     # generated coverage reports (gitignored)

```

## References
* PCI-SIG, PCI Express Base Specification
* JEDEC, JESD79-4: DDR4 SDRAM Standard
* Arm, AMBA AXI and ACE Protocol Specification (IHI0022)
* Accellera, UVM 1.2 Class Reference
