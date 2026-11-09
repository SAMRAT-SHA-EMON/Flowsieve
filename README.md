# FlowSieve: Encrypted Mobile Traffic Classification and Category-Based Filtering

**Paired-View Tree Learning and Hierarchical Probability Fusion for Behavioral Traffic Filtering on Consumer Routers**

**Author:** Saim Uddin Samrat Sha Emon (`@SAMRAT-SHA-EMON`)  
**Affiliation:** School of Computer Science and Technology, Southwest University of Science and Technology (SWUST)  
**Supervisor:** Prof. Huang Xiaofang  
**Programme:** M.Eng. in Software Engineering (Networking & Edge Intelligence)

## 1. Research Overview & Goal

Modern mobile applications rely almost exclusively on encrypted transport protocols (TLS 1.3 and QUIC), rendering traditional payload-based Deep Packet Inspection (DPI) tools such as OpenAppFilter ineffective when payload signatures are obfuscated or unavailable. At the same time, deep neural traffic classifiers require memory and compute budgets that exceed consumer home routers operating under tight flash and RAM limits on OpenWrt.

**FlowSieve** addresses this gap by classifying encrypted mobile network flows into three actionable behavioral categories—**Game**, **Payment**, and **Social-Media**—without inspecting packet payload bytes:

* **Behavioral Category Stability:** Individual mobile applications frequently change, whereas category-level traffic behaviors (such as small symmetric real-time packets in games, short asymmetric bursts in payment flows, and multiplexed media streams in social applications) remain structurally stable for router policy rules.
* **Unseen Application Transfer:** The classifier is designed and evaluated under strict **application-disjoint (unseen)** splits so that flows from newly encountered mobile apps can be categorized without retraining on those specific applications.
* **Resilience to Missing Name Clues (Observed vs. Absent Views):** Even when all 31 DNS and TLS Server Name Indication (SNI) metadata columns are masked or unavailable (due to encrypted DNS or shared CDN endpoints), FlowSieve remains operational using 131 pure transport-layer and flow-context features.
* **Consumer Router Enforcement:** The system pairs offline model learning with an edge router runtime in Go and a native C inference implementation benchmarked on Redmi AX6 consumer router hardware with opt-in `nftables` policy enforcement.

## 2. System Architecture

FlowSieve separates offline model research and application-disjoint benchmarking from live router execution:

| Subsystem | Input Boundary | Core Processing Pipeline | Output / Enforcement |
| :--- | :--- | :--- | :--- |
| **Paired-View Shared Forest (C1 / C2)** | 162 flow features ($x_{io}$ Observed view; $x_{ia}$ Absent view with 31 DNS/TLS name columns masked) | 100 shared class-pair trees ($\le 31$ leaves/tree) built on AOSO-LogitBoost with C1 paired loss weights and C2 cross-view curvature bounds | 3-category behavioral prediction (**Game**, **Payment**, **Social-Media**) across observed and name-absent traffic |
| **Hierarchical Probability Fusion (ISP & Lite)** | **ISP:** 162 completed-flow features<br>**Lite:** 104 early-flow features (first 20 packets + 10s/60s device context) | Independent 3-category head ($F_C$) and 16-subcategory auxiliary head ($F_S$) trained with LightGBM, combined via parent map $M$ and inner-validation weight $w_f$ | Fused category probability distribution $p_c(x) = (1 - w_f)p_C(c \mid x) + w_f H_c(x)$ |
| **Go Router Service** | Live LAN frame capture (first 20 packets + 10s/60s device state) | Ordered 104-feature extraction, saved numerical transforms, and native Go tree evaluation | Real-time category decision, separate administrative allow/block lists, web dashboard, and opt-in `nftables` filtering |
| **Native C Edge Candidate** | Prepared numerical input vectors on ARM router hardware | Static C array tree evaluation compiled for embedded Linux (OpenWrt) | Sub-millisecond prepared-input inference verified on Redmi AX6 hardware |

## 3. Core Methodological Contributions

### 3.1 Paired-View Tree Learning in a Shared Class-Pair Forest (C1 & C2)

Standard gradient boosting (LightGBM) fits three scalar trees per boosting round (300 trees at 100 rounds) and treats observed and name-masked views with uniform weight, while Group DRO reweights predefined groups by worst-group average loss rather than coupling views per individual flow. Building on the shared class-pair tree formulation of AOSO-LogitBoost (1 shared pair tree per round, 100 trees total, zero-sum leaf step $\Delta = \eta a(e_r - e_s)$), FlowSieve introduces two candidate modifications to couple the **Observed ($x_{io}$)** and **Absent ($x_{ia}$)** views of each training flow $i$:

1. **C1 — Paired Loss Weights:** Reweights per-flow paired views in split-finding and leaf statistics using view weights $\alpha_{iv}$ (with 0.5 margin scaling), assigning larger weight to whichever view ($x_{io}$ or $x_{ia}$) incurs higher loss on flow $i$.
2. **C2 — Local Covariance Curvature Bound:** Augments node and leaf second-order curvature with a cross-view interaction term $\Delta h_{iv}^{rs} = 2 w_i \alpha_{io} \alpha_{ia} (g_r - g_s)^2$ to regularize step sizes when gradients diverge between observed and name-absent views.

### 3.2 Hierarchical Probability Fusion & Grouped Training Controls

In the dual-head architecture, FlowSieve trains two LightGBM ensembles on shared feature representations:

* **Direct Category Head ($F_C$):** Predicts the 3 parent categories (**Game**, **Payment**, **Social-Media**) directly.
* **Auxiliary Subcategory Head ($F_S$):** Predicts 16 fine-grained subcategories (9 Game, 3 Payment, and 4 Social-Media subcategories).
* **Parent Pooling & Validation Blend:** Child probabilities are summed by parent mapping $\pi(s) = c$ to form $H_c(x) = \sum_{s:\pi(s)=c} p_S(s \mid x)$ and fused with the direct head via $p_c(x) = (1 - w_f)p_C(c \mid x) + w_f H_c(x)$, where $w_f \in [0, 1]$ is selected on inner validation per outer fold.
* **Leakage-Free Grouped Preprocessing:** Target encoding excludes each fitting row's entire capture/application group from its counts and priors, application-balanced row weights equalize application influence within categories, and training-time metadata dropout masks DNS/TLS name features on selected rows.

## 4. Evaluation Protocol & Experimental Summary

FlowSieve separates two orthogonal evaluation axes: **Application Overlap** (*Seen* capture-disjoint vs. *Unseen* application-disjoint) and **Input Visibility** (*Observed* name clues retained vs. *Absent* 31 DNS/TLS columns masked).

### 4.1 Datasets & Evaluation Cohorts

| Cohort / Study | Applications | Total Flows | Category Composition | Protocol & Partitioning |
| :--- | :--- | :--- | :--- | :--- |
| **Full Frozen Benchmark** | 348 mobile apps (87 Game, 181 Payment, 80 Social-Media) | 1,040,782 training pool<br>2,054,048 evaluation pool | Natural evaluation share: **84.06% Payment**, Game & Social-Media minority | 5 frozen outer folds for both Seen (capture-disjoint) and Unseen (application-disjoint) protocols |
| **89-App Unseen Paired-View Study** | 89 held-out unseen apps (zero overlap with fitting fold) | 258,948 unseen flows per view ($x_{io}$ and $x_{ia}$) | **84.1% Payment**, **8.8% Game**, **7.2% Social-Media** | Evaluated across Unseen + Observed ($x_{io}$) and Unseen + Absent ($x_{ia}$, 31 name columns masked) |

### 4.2 Key Experimental Findings (89-App Unseen Transfer Study)

| Learner / Configuration | Forest Structure | Unseen + Absent Macro-F1 | Key Behavioral Observation |
| :--- | :--- | :--- | :--- |
| **Standard LightGBM Baseline** | 300 scalar trees (3 trees/round, $\le 31$ leaves) | 57.60% | Equal view weights without per-flow paired coupling |
| **Uniform Shared Class-Pair Forest (AOSO-LogitBoost)** | 100 shared pair trees (1 tree/round, $\le 31$ leaves) | 59.59% | Equal view weights ($0.5 \ell_{io} + 0.5 \ell_{ia}$) inside a 3,100-leaf shared forest |
| **FlowSieve C1 (Paired Loss Weights)** | 100 shared pair trees ($\le 31$ leaves) | **59.95%** | **+0.36 pp** over Uniform shared forest and **+2.35 pp** over Standard LightGBM on name-absent unseen apps (95% bootstrap CIs overlap zero) |
| **FlowSieve C2 (C1 + Local Covariance Bound)** | 100 shared pair trees ($\le 31$ leaves) | Under refinement | Improves **Social-Media recall from 49.59% to 51.66%** under the Absent view |

## 5. Planned Repository Layout

```text
FlowSieve/
├── README.md                  # Project overview, methodology, and benchmark summary
├── docs/                      # Architecture diagrams, mathematical derivations, and evaluation notes
├── benchmark/                 # Offline grouped training, C1/C2 paired-view forest, and hierarchical fusion
├── router-service/            # Go LAN flow capture (20 packets + 10s/60s context), tree evaluator & web UI
├── native-c/                  # Pure C inference engine and Redmi AX6 / OpenWrt benchmark harness
└── configs/                   # Category/subcategory taxonomy maps and opt-in nftables policy templates
```

## 6. Key References

1. Guolin Ke et al., *LightGBM: A Highly Efficient Gradient Boosting Decision Tree*, Advances in Neural Information Processing Systems (NeurIPS), 2017.
2. Peng Sun, Mark D. Reid, and Jie Zhou, *AOSO-LogitBoost: Adaptive One-Vs-One LogitBoost for Multi-Class Problem*, International Conference on Machine Learning (ICML), 2012.
3. Shiori Sagawa et al., *Distributionally Robust Neural Networks for Group Shifts: On the Importance of Regularization for Worst-Case Generalization*, International Conference on Learning Representations (ICLR), 2020.
4. Zied Aouini and Adrian Pekar, *NFStream: A Flexible Network Data Analysis Framework*, Computer Networks, 2022.

## 7. Contact

* **Author:** Saim Uddin Samrat Sha Emon (`@SAMRAT-SHA-EMON`)
* **Institution:** School of Computer Science and Technology, Southwest University of Science and Technology (SWUST)
* **Email:** `samratshaemon13@gmail.com`
