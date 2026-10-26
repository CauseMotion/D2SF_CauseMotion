<div align="center">

[한국어](README.md) &nbsp;|&nbsp; **English**

# CauseMotion

**Root-Cause-Aware Field Recovery Engine for Service Robots**

An engine that diagnoses why a building service robot stopped and recovers it with the right action: **Environment Adaptation** or **Targeted Policy Learning**

![Status](https://img.shields.io/badge/status-early_development-0B1F3A?style=flat-square)
![ROS 2](https://img.shields.io/badge/ROS_2-Jazzy-1E3A5F?style=flat-square&logo=ros&logoColor=white)
![Nav2](https://img.shields.io/badge/Nav2-navigation-1E3A5F?style=flat-square)
![Python](https://img.shields.io/badge/Python-3.12-1E3A5F?style=flat-square&logo=python&logoColor=white)
![PyTorch](https://img.shields.io/badge/PyTorch-learning-1E3A5F?style=flat-square&logo=pytorch&logoColor=white)
![License](https://img.shields.io/badge/license-TBD-5A6B85?style=flat-square)

[Overview](#overview) &nbsp;·&nbsp; [Problem](#problem) &nbsp;·&nbsp; [System](#system) &nbsp;·&nbsp; [Diagnosis](#cause-diagnosis) &nbsp;·&nbsp; [Evaluation](#evaluation-plan) &nbsp;·&nbsp; [Roadmap](#roadmap)

</div>

---

## Overview

| | |
|:--|:--|
| **Problem** | Building service robots stop repeatedly as their environment changes, and remote-intervention logs are fed into training regardless of the cause. |
| **Approach** | Diagnose the cause of every intervention. Environment-side causes go to **Environment Adaptation**; only policy-side causes go to **Targeted Policy Learning**. |
| **Hypothesis** | A large share of real-world remote interventions stem from causes unrelated to learning (maps, calibration, facilities, transient factors). |

<div align="center">

| Recurrence Rate | Diagnosis Accuracy (High) | Diagnosis Accuracy (Overall) |
|:--:|:--:|:--:|
| **▼ 30%+** | **85%+** | **70%+** |
| vs. learning from all interventions · same budget | High-confidence cases | All diagnoses |

<sub>Targets are provisional and will be validated through controlled simulation experiments.</sub>

</div>

---

## Problem

> [!IMPORTANT]
> Of all the reasons a robot stops, **only a policy gap can be fixed by learning.**

| Cause | Typical Case | Right Action | Route |
|:--|:--|:--|:--:|
| Transient | Crowds of pedestrians, temporarily placed objects | Log only | — |
| Outdated Map | Rearranged furniture, events, temporary partitions | Partial map update | Environment Adaptation |
| Sensor Calibration | Misaligned camera or LiDAR mount | Calibration request | Environment Adaptation |
| Facility Integration | Elevator or gate not responding | Facility ticket | Environment Adaptation |
| **Policy Gap** | Repeated avoidance failures in crowded corridors | **Targeted training** | **Targeted Policy Learning** |

Training on the wrong cause does not reduce recurrence. It only raises cost, and noisy data can degrade the policy.

---

## System

### Recovery Pipeline

Each stop is handled as a closed loop: **diagnose → act → record → verify recurrence**.

```mermaid
%%{init: {'theme':'base','themeVariables':{'fontFamily':'Inter, Helvetica, Arial, sans-serif','fontSize':'15px','primaryColor':'#E8EEF7','primaryTextColor':'#0B1F3A','primaryBorderColor':'#1E3A5F','lineColor':'#2C5282','edgeLabelBackground':'#FFFFFF','tertiaryColor':'#F4F7FB'}}}%%
flowchart LR
    A[Stop · Remote Intervention] --> B{Cause Diagnosis}
    B -->|Environment-side| C[Environment Adaptation]
    B -->|Policy gap| D[Targeted Policy Learning]
    C --> E[(Recovery Ledger)]
    D --> E
    E -. Recurrence feedback .-> B

    classDef core fill:#0B1F3A,stroke:#0B1F3A,color:#FFFFFF,font-weight:bold
    classDef path fill:#1E3A5F,stroke:#1E3A5F,color:#FFFFFF
    classDef base fill:#E8EEF7,stroke:#1E3A5F,color:#0B1F3A
    class B core
    class C,D path
    class A,E base
```

<table>
<tr>
<th width="50%">Environment Adaptation</th>
<th width="50%">Targeted Policy Learning</th>
</tr>
<tr valign="top">
<td>

| Step | Description |
|:--|:--|
| Change detection | Separate persistent from transient changes |
| Partial map update | Update only the changed zones |
| Zone parameters | Tune clearance and speed locally |
| Calibration · facility tickets | Route to the responsible owner |

</td>
<td>

| Step | Description |
|:--|:--|
| Selection | Only cases diagnosed as policy gaps |
| Review | Exclude low-quality teleoperation logs |
| Targeted training | Train on that scenario type only (HG-DAgger) |
| Before/after check | Target success rate · no regression elsewhere |

</td>
</tr>
</table>

### Architecture

Swap only the input adapter and **the same code runs in simulation and on real robots**.

```mermaid
%%{init: {'theme':'base','themeVariables':{'fontFamily':'Inter, Helvetica, Arial, sans-serif','fontSize':'14px','primaryColor':'#E8EEF7','primaryTextColor':'#0B1F3A','primaryBorderColor':'#1E3A5F','lineColor':'#2C5282','clusterBkg':'#F4F7FB','clusterBorder':'#9FB3CF'}}}%%
flowchart TB
    subgraph IN[Input Adapters]
        direction LR
        S2[2D Sim] ~~~ S3[Gazebo] ~~~ LG[Log Replay] ~~~ RB[Real Robot]
    end
    IN --> T["<b>Topic Contract · ROS 2</b><br/>/scan · /odom · /amcl_pose · /tf · /cmd_vel · /map"]
    subgraph CORE[CauseMotion Core]
        direction LR
        D1[Anomaly<br/>Detector] --> D2[Cause<br/>Diagnosis] --> D3[Recovery<br/>Actions] --> D4[(Recovery<br/>Ledger)] --> D5[Diagnostic<br/>Report]
    end
    T --> CORE

    classDef contract fill:#0B1F3A,stroke:#0B1F3A,color:#FFFFFF
    classDef module fill:#1E3A5F,stroke:#1E3A5F,color:#FFFFFF
    classDef adapter fill:#FFFFFF,stroke:#2C5282,color:#0B1F3A
    class T contract
    class D1,D2,D3,D4,D5 module
    class S2,S3,LG,RB adapter
```

---

## Cause Diagnosis

Seven evidence signals are combined into ranked cause candidates with a **confidence level**, and the action taken depends on that confidence.

```mermaid
%%{init: {'theme':'base','themeVariables':{'fontFamily':'Inter, Helvetica, Arial, sans-serif','fontSize':'14px','primaryColor':'#E8EEF7','primaryTextColor':'#0B1F3A','primaryBorderColor':'#1E3A5F','lineColor':'#2C5282'}}}%%
flowchart LR
    S["<b>7 Evidence Signals</b><br/>Location recurrence · Time recurrence · Cross-robot comparison<br/>Change score · Sensor health · Facility signals · Transient obstacles"] --> C{Confidence}
    C -->|High| H[Execute after one operator approval]
    C -->|Medium| M[Low-cost, reversible actions first]
    C -->|Low| L[Defer · run diagnostic experiments]

    classDef sig fill:#E8EEF7,stroke:#1E3A5F,color:#0B1F3A
    classDef dec fill:#0B1F3A,stroke:#0B1F3A,color:#FFFFFF
    classDef hi fill:#1E3A5F,stroke:#1E3A5F,color:#FFFFFF
    classDef mid fill:#4A6A94,stroke:#4A6A94,color:#FFFFFF
    classDef lo fill:#C9D6E8,stroke:#4A6A94,color:#0B1F3A
    class S sig
    class C dec
    class H hi
    class M mid
    class L lo
```

**Design Principles**

| Principle | Description |
|:--|:--|
| Positive evidence | A policy gap is never a residual category; it requires confirmed recurrence in the same scenario type |
| Asymmetric thresholds | Try low-cost, reversible actions even at low confidence; **require the strongest evidence before training** |
| Contamination guard | Always check sensor health before updating a map |
| Sequential fallback | On recurrence, move to the next cause candidate |

---

## Evaluation Plan

Changes are injected in a controlled way in simulation to obtain **ground-truth causes**, and three approaches are compared under the same budget.

| Baseline | Description |
|:--|:--|
| A | Learn from every intervention |
| B | Scheduled inspection · full map rebuilds |
| **C** | **CauseMotion** — cause diagnosis + two-path recovery |

| Metric | Definition | Target (provisional) |
|:--|:--|:--|
| Diagnosis accuracy | Agreement with ground-truth causes | High 85%+, overall 70%+ |
| Recurrence rate | Share of same-zone, same-cause stops after action | 30%+ lower than A |
| Training contamination | Share of trained cases with non-policy causes | Minimize |
| Recovery cost | Teleoperation time · compute · map updates | Measured separately |

> [!NOTE]
> The cause distribution in simulation is set by design, so the core hypothesis cannot be proven in simulation alone. It will be validated through **diagnostics on partner operators' logs**.

---

## Roadmap

Validation is completed within six months without hardware, with a go/no-go decision at each gate.

```mermaid
%%{init: {'theme':'base','themeVariables':{'fontFamily':'Inter, Helvetica, Arial, sans-serif','fontSize':'13px','primaryColor':'#1E3A5F','primaryTextColor':'#FFFFFF','primaryBorderColor':'#0B1F3A','taskBkgColor':'#1E3A5F','taskBorderColor':'#0B1F3A','taskTextColor':'#FFFFFF','taskTextOutsideColor':'#0B1F3A','taskTextLightColor':'#FFFFFF','activeTaskBkgColor':'#2C5282','activeTaskBorderColor':'#0B1F3A','sectionBkgColor':'#F4F7FB','altSectionBkgColor':'#FFFFFF','sectionBkgColor2':'#E8EEF7','gridColor':'#C9D6E8','todayLineColor':'#2C5282','milestoneBkgColor':'#0B1F3A','milestoneBorderColor':'#0B1F3A'}}}%%
gantt
    dateFormat  YYYY-MM
    axisFormat  %Y.%m
    section Build
    Sim env · Log collector        :a1, 2026-11, 30d
    Detection · Teleop UI          :a2, after a1, 30d
    Env adaptation · Diagnosis v1  :a3, after a2, 30d
    Targeted learning · Eval       :a4, after a3, 30d
    Comparison · Log diagnostics   :a5, after a4, 30d
    Report automation              :a6, after a5, 30d
    section Gate
    M2 Problem validation   :milestone, after a2, 0d
    M4 Diagnosis accuracy   :milestone, after a4, 0d
    M5 Recurrence benchmark :milestone, after a5, 0d
    M6 Pilot secured        :milestone, after a6, 0d
```

| Phase | Environment | Goal |
|:--:|:--|:--|
| **1** | 2D Simulation | Inject 5 cause types, diagnosis v1, three-way comparison |
| **2** | Gazebo Harmonic | 3D reproduction, camera-based remote intervention |
| **3** | Real Robot | Real-sensor validation, measure the sim-to-real gap |

---

## Differentiation

| Existing Approach | Scope | CauseMotion |
|:--|:--|:--|
| Teleoperation-learning companies | Use intervention data for training | Select only interventions that need learning |
| Failure-analysis tools | Root-cause analysis for engineers | Connect diagnosis → action → recurrence check |
| Fleet-management platforms | Monitoring · remote control | Cause diagnosis and recurrence prevention after intervention |
| Manufacturer map updates | Map updates for their own robots | Vendor-agnostic; covers causes beyond maps |

> [!TIP]
> **Data Moat** — No public dataset links change, intervention, cause, action, and recurrence the way the Recovery Ledger does.

---

## Tech Stack

| Layer | Stack |
|:--|:--|
| Robotics | ROS 2 Jazzy · Nav2 · Gazebo Harmonic · TurtleBot 4 · HuNavSim |
| Data | rosbag2 (MCAP) · Foxglove |
| Backend | Python · FastAPI · SQLite / PostgreSQL |
| Learning | PyTorch |
| Infra | Ubuntu 24.04 · Docker |

## Getting Started

> Coming soon.

## Data & License

Public datasets are classified and managed by license tier (commercial use / conditional / research only). Raw data is not included in this repository.

---

<div align="center">
<sub>NAVER D2SF Campus Tech Startup Competition · Proposal v3</sub>
</div>
