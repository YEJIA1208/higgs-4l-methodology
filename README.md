# Higgs → 4ℓ (CMS Open Data) — Methodology Notes (Role A)

**Repo:** `higgs-4l-methodology`  
Role A notes: CMS Open Data H→4ℓ analyzer methodology + data log

> Role A deliverable: explain the existing C++ analyzer logic + keep a clean “data & samples” log for the final report.

This repository contains a simplified Higgs-to-four-lepton (H→ZZ→4ℓ) analysis example based on **CMS Open Data (2011–2012 style)**.  
My responsibility (Role A) is to document **how the analyzer selects leptons, forms Z candidates, builds 4-lepton candidates, and fills histograms**, so the group can quickly understand the pipeline and write the final Data/Method section.

---

## 1) Data & Samples Log (what we used)

### Dataset chosen for our group (this repo run)
- **Physics channel:** DoubleMu (muon-triggered data)
- **Dataset name:** `/DoubleMuParked/Run2012B-22Jan2013-v1/AOD`
- **File used (example index file):**  
  `root://eospublic.cern.ch//eos/opendata/cms/Run2012B/DoubleMuParked/AOD/22Jan2013-v1/20002/00454F16-EB6C-E211-9CEB-001EC9D278CC.root`

### Scope / intention
- This DoubleMu dataset is used primarily to populate histograms for:
  - **4μ** final state
  - **2μ2e** final state (muons present; in the original strategy it avoids trigger overlap)
- (If the group later uses DoubleElectron, that is typically used for **4e** to avoid double-counting via trigger overlap.)

---

## 2) Quick pipeline overview

Pipeline (one line): **AOD → select leptons → build OS pairs → choose Za/Zb → build 4ℓ → fill histograms**

## 3) Object definitions & event selection (summary)

We reconstruct prompt, isolated leptons compatible with the primary vertex (PV).  
The analyzer builds a pool of **selected muons** and **selected electrons**, then uses them to form Z and 4ℓ candidates.

### Muons (PF muons)
- PF muon, valid PF isolation, and global track available
- Kinematics: pT > 5 GeV, |η| < 2.4
- Vertex compatibility: |SIP3D| < 4, |dxy| < 0.5, |dz| < 1
- Relative PF isolation (R=0.4): relPFIso < 0.4

### Electrons (GsfElectrons)
- Passing PF preselection
- Kinematics: pT > 7 GeV, |η_SC| < 2.5
- Missing hits: ≤ 1
- Vertex compatibility: |SIP3D| < 4, |dxy| < 0.5, |dz| < 1
- Relative PF isolation: relPFIso < 0.4

Selected leptons are sorted by pT (descending) before building candidates.
