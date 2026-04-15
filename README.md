# Verilog Skill for Claude

## 🚀 Overview

This repository provides a **Verilog HDL skill module** designed to augment Claude AI for **RTL design, code generation, debugging, and verification workflows**.

The focus of this project is not just learning — but enabling **engineer-level productivity** in digital design by integrating structured Verilog knowledge with AI-assisted workflows.

---

## 🎯 Purpose

Modern RTL design requires speed, accuracy, and strong debugging capability. This project is built to:

* Accelerate **RTL development cycles**
* Assist in writing **synthesizable and clean Verilog code**
* Improve **debugging efficiency**
* Provide **design-aware responses** instead of generic AI outputs
* Support **real-world VLSI workflows**

---

## ⚙️ Core Capabilities

### 🔧 RTL Design Assistance

* Parameterized module generation
* Combinational and sequential logic design
* FSM implementation
* Synthesizable coding practices

### 🐞 Debugging & Code Review

* Detection of:

  * Race conditions
  * Latch inference
  * Blocking vs non-blocking issues
  * Simulation vs synthesis mismatches
* Root-cause analysis with fixes

### 🧪 Verification Support

* Testbench development
* Simulation constructs and debugging
* Functional validation strategies

### 📐 Synthesis-Aware Coding

* Ensures generated code aligns with synthesis constraints
* Avoids non-synthesizable constructs
* Highlights hardware implications

---

## 🧠 Design Approach

The skill is built with a **design-first mindset**, ensuring:

* All outputs are **hardware-relevant**
* Emphasis on **RTL correctness over syntax**
* Awareness of **toolchain behavior (simulation vs synthesis)**
* Alignment with **industry coding standards**

---

## 📂 Repository Structure

```id="p8k21d"
Verilog-Skill-for-Claude/
│
├── Skill.md                # Core AI skill definition
│
├── references/             # Domain knowledge base
│   ├── 01_lexical_and_datatypes.md
│   ├── 02_modules_and_hierarchy.md
│   ├── 03_behavioral_modeling.md
│   ├── 04_rtl_coding_patterns.md
│   ├── 05_synthesis.md
│   ├── 06_testbench_and_simulation.md
│   ├── 07_gate_level_and_udp.md
│   ├── 08_timing_and_delays.md
│   ├── 09_digital_design_combinational.md
│   └── 10_digital_design_sequential.md
```

---

## 🛠️ Usage Workflow

1. Load `Skill.md` into Claude
2. Provide an RTL-related query, for example:

   * “Design a pipelined multiplier in Verilog”
   * “Debug this always block”
   * “Write synthesizable FSM for UART”
3. The system:

   * Identifies the design context
   * Pulls relevant reference knowledge
   * Generates **engineering-grade responses**

---

## 📌 Engineering Focus Areas

This skill is optimized for:

* RTL Design (ASIC/FPGA)
* Code Review & Debugging
* Testbench Development
* Interview-Oriented Problem Solving
* Design Pattern Standardization

---

## ⚠️ Design Constraints & Best Practices

* Enforces **non-blocking assignments in sequential logic**
* Avoids **latch inference**
* Promotes **parameterization and modularity**
* Ensures **complete case/default coverage**
* Highlights **timing and synthesis implications**

---

## 🔬 Practical Impact

Using this skill, an engineer can:

* Reduce RTL development time
* Improve first-time-right design quality
* Catch critical bugs early
* Standardize coding practices
* Use AI effectively in hardware design workflows

---

## 👤 Author

**Nandha Krishnan M**
📧 [nandhakrishnan004@gmail.com](mailto:nandhakrishnan004@gmail.com)

---

## 📌 Note

This repository is intended for **engineering use in RTL design workflows**, not just academic learning.
