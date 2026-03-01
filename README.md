# Digital VLSI SoC Design and Planning Course Work

![OpenLANE](https://img.shields.io/badge/OpenLANE-Flow-blue)
![SkyWater 130nm](https://img.shields.io/badge/PDK-SkyWater_130nm-green)

## 📌 About the Project

This repository documents the 6-week RTL2GDS SoC Implementation program, focusing on System & RTL Foundations to Physical Design and Sign-off using the OpenLANE flow.The objective is to achieve a clean GDSII layout without human-in-the-loop intervention, demonstrating a complete digital ASIC design lifecycle

## Content
- [Week 1](#week-1)
- [Week 2](#week-2)
- [Week 3](#week-3)
- [Week 4](#week-4)
- [Week 5](#week-5)
- [Week 6](#week-6)

## Week 1
 ## 📑 Table of Contents
1. [Phase 1 — OpenLANE Flow Familiarity (RTL → Synthesis literacy)](#1-fundamentals-of-digital-asic-design)

---

## Phase 1 — OpenLANE Flow Familiarity (RTL → Synthesis literacy)
**Sections Covered:** `SKY130_D1_SK2` (SoC design and OpenLANE), `SKY130_D1_SK3` (Get familiar to open-source EDA tools).

### 1.1 OpenLANE Directory Structure & Design Preparation
OpenLANE integrates several open-source EDA tools (Yosys, ABC, Magic, Ngspice, OpenSTA) to automate the ASIC flow. The design preparation step initializes the environment, merges LEF files, and creates a timestamped run directory.

**Objective:** Understand the big-picture flow, open-source EDA tools, and execute design preparation and RTL synthesis.

#### The Big Picture: ASIC Design & OpenLANE
Digital ASIC design requires three elements: RTL IP, EDA Tools, and PDK Data . The Process Design Kit (PDK) acts as the critical interface between the designer and the fabrication facility, containing process rules, device models, and standard cell libraries . OpenLANE is an automated, open-source reference flow that strings together tools like Yosys, ABC, Magic, and OpenROAD to produce a clean GDSII layout with zero human intervention .

#### Laboratory Execution Steps

**1. Environment Initialization:** Open the terminal and navigate to the OpenLANE directory.
```bash
# Invoke OpenLANE via Docker
cd Desktop/OpenLane
make mount
```

**2. Launch Interactive Flow:** Start OpenLANE in interactive mode to run step-by-step and load dependencies .
```
./flow.tcl -interactive
package require openlane 1.0.2
```
**3. Design Preparation:** Initialize the PicoRV32a core. This parses config.tcl, merges LEF files, and creates a timestamped run directory

```
prep -design picorv32a
```

<img width="977" height="237" alt="navigate" src="https://github.com/sukrith-sunil/VSDSQUADRON/blob/main/img/p1.png" />   
  
**4. Synthesis Execution & Characterization:** Logic synthesis translates the Verilog RTL into an optimized gate-level netlist using the standard cell library .

```
run_synthesis
```

<img width="977" height="237" alt="synth" src="https://github.com/sukrith-sunil/VSDSQUADRON/blob/main/img/p2_synth.png" />   

**Synthesis Characterization Output:**

<img  alt="area1" src="https://github.com/sukrith-sunil/VSDSQUADRON/blob/main/img/p3_area1.png" />  
<img  alt="area2" src="https://github.com/sukrith-sunil/VSDSQUADRON/blob/main/img/p4_area2.png" />  

<img  alt="numcell" src="https://github.com/sukrith-sunil/VSDSQUADRON/blob/main/img/p5_numcell.png" />  

## Phase 2 — Floorplan Fundamentals (Macro Awareness)
---

**1 Floorplan Configuration & Optimization**
Floorplanning establishes the chip's physical dimensions.


**Aspect Ratio:** The ratio of the core's height to its width .


**Utilization Factor:** The area occupied by standard cells relative to the total core area.

**Critical Troubleshooting (OOM Routing Crash):**
During initial testing, the utilization factor (FP_CORE_UTIL) was inadvertently set to **10**. This 10% utilization created an unnecessarily** massive die area (1267.76 x 1267.52 microns)**, causing the detailed router (TritonRoute) to build a colossal 3D routing grid. The router encountered over 11,000 violations and exhausted the ~3.28 GB memory limit of the GitHub Codespace, resulting in a child killed: software termination signal crash.

To resolve this, the utilization was increased to shrink the die area, vastly reducing the router's memory footprint. This was achieved directly in the Codespace terminal:

```
sed -i 's/set ::env(FP_CORE_UTIL) "10"/set ::env(FP_CORE_UTIL) "35"/g' ./designs/picorv32a/config.tcl
```

then run:

```
run_floorplan
```

<img  alt="floor" src="https://github.com/sukrith-sunil/VSDSQUADRON/blob/main/img/p6_corearea.png" />  


**Why Macros Behave Differently Than Standard Cells ?**

Standard cells are basic logic gates with fixed heights that can be freely placed and resized (different drive strengths) by the synthesis tool to optimize timing. Macros, such as RAM or large IPs, are massive, pre-designed blocks treated as fixed "black boxes." They are placed manually during floorplanning and cannot be resized or internally altered during synthesis because their timing and power metrics are dominated by internal memory array physics rather than simple gate-level propagation.


## Phase 3 — Timing Literacy with Ideal Clocks (OpenSTA + ECO)
---

**1. Setup & Hold Fundamentals**
To ensure the design functions at the target clock frequency without data corruption, Static Timing Analysis (STA) is performed using OpenSTA:

**Setup Time:** The minimum time data must be stable before the active clock edge.

**Hold Time:** The minimum time data must be stable after the active clock edge.


**Reports:**

<img  alt="sta1" src="https://github.com/sukrith-sunil/VSDSQUADRON/blob/main/img/p8_slack.png" />
<img  alt="sta2" src="https://github.com/sukrith-sunil/VSDSQUADRON/blob/main/img/p9_clkskew.png" />  
<img  alt="sta3" src="https://github.com/sukrith-sunil/VSDSQUADRON/blob/main/img/p10_su_hd.png" />  


## Phase 4 — CTS and Timing with Real Clocks
---

Clock Tree Synthesis (CTS) creates an optimized routing network to distribute the clock signal to all sequential elements, prioritizing minimum clock skew and low latency.
```
run_cts
```

**Pre-CTS vs. Post-CTS Timing Observations:**
During Phase 3, timing was calculated using an "ideal clock" with zero propagation delay. Post-CTS, real clock network delays (insertion delays) are introduced. The insertion of multiple clock buffers to balance the tree natively increases the time it takes for the clock to reach the flip-flops. While larger CTS buffers improve slew rates and help minimize skew, they drastically alter setup and hold margins, often requiring post-CTS optimization to fix newly introduced hold violations caused by clock skew.

**Reports :**

<img  alt="cts" src="https://github.com/sukrith-sunil/VSDSQUADRON/blob/main/img/p11_CTS.png" /> 


## Phase 5 — PDN Awareness
---

**1. Power Distribution Network Generation :**

The PDN is critical for delivering stable power ($V_{DD}$ and $V_{SS}$) to the core. It is built using the thickest, upper-level metal layers because their lower resistance mitigates dynamic IR drop .

```
gen_pdn
```

**2. Sign-off Thinking: Why PDN Precedes Routing**

The Power Distribution Network must exist before detailed signal routing begins. Power straps are massive tracks that span the entire chip. If signal routing occurred first, the layout would be too congested to drop these power lines without causing catastrophic DRC spacing and short-circuit violations. From a sign-off perspective, ensuring the PDN is robust early in the flow prevents severe voltage drops that would ultimately degrade standard cell performance and cause late-stage timing failures.

**3. to veiw in MAGIC**

```
cd ~/Desktop/OpenLane/designs/picorv32a/runs/<RUN_NAME>/results/final/
magic -T /home/vscode/.ciel/sky130A/libs.tech/magic/sky130A.tech
```
then inside MAGIC

```
lef read ../../tmp/merged.nom.lef
def read ./def/picorv32a.def
```


**Reports :**


