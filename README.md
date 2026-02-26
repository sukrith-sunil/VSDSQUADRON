# Digital VLSI SoC Design and Planning Course Work

![OpenLANE](https://img.shields.io/badge/OpenLANE-Flow-blue)
![SkyWater 130nm](https://img.shields.io/badge/PDK-SkyWater_130nm-green)

## 📌 About the Project

This repository documents the 6-week RTL2GDS SoC Implementation program, focusing on System & RTL Foundations to Physical Design and Sign-off using the OpenLANE flow[cite: 705, 706]. [cite_start]The objective is to achieve a clean GDSII layout without human-in-the-loop intervention, demonstrating a complete digital ASIC design lifecycle

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

```bash
# Invoke OpenLANE via Docker
cd Desktop/OpenLane
make mount

# Launch interactive flow and prepare the picorv32a design
./flow.tcl -interactive
package require openlane 1.0.2
prep -design picorv32a
```
   
  

