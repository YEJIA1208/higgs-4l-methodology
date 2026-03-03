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

## Code map with line references (HiggsDemoAnalyzerGit.cc)

> Line numbers refer to `HiggsDemoAnalyzerGit.cc` in the `main` branch.

### 0) High-level purpose & strategy
- **Header + analysis strategy description:** lines **1–~90**
  - Explains this is a simplified research-level H→ZZ→4ℓ example.
  - Notes the recommended dataset split to avoid trigger overlap: DoubleMu for 4μ and 2μ2e, DoubleElectron for 4e.

### 1) Module / class structure
- **Class declaration (histograms + variables):** lines **~120–~350**
  - Declares all `TH1D/TH2D` histograms used for control plots and final spectra.
- **Constructor (histogram booking):** lines **360–880**
  - `HiggsDemoAnalyzerGit::HiggsDemoAnalyzerGit(...)` books all histograms using `TFileService`.

### 2) Event loop entry point
- **Main event analysis function:** lines **892–2276**
  - `void HiggsDemoAnalyzerGit::analyze(...)` contains all selection + reconstruction logic.

### 3) AOD collections read-in (handles)
- **Load event content (`getByLabel`):** lines **920–1060**
  - Reads: `generalTracks`, `globalMuons`, `muons`, `offlineBeamSpot`, `offlinePrimaryVertices`, `gsfElectrons`.
  - Sets PV/beamspot, initializes vectors and physics variables.

### 4) Global muon "demo" selection (not central to H→4ℓ)
- **Global muon loop + quality cuts:** lines **1064–1117**
  - Fills control hists for global muons and builds `vIdPt` (sorted by pT).

### 5) PF muon selection (core for H→4ℓ)
- **Reco/PF muon selection:** lines **1122–1206**
  - Applies PF muon requirements, computes PF relative isolation (R04), SIP3D, dxy/dz, then stores passing muons in `vIdPtmu`.
  - Sorts selected muons by pT.

### 6) Electron selection (core for H→4ℓ)
- **Electron selection:** lines **1211–1285**
  - Applies PF preselection, missing hits, SIP3D, dxy/dz, PF isolation, pT and |η_SC| cuts.
  - Stores passing electrons in `vIdPte`, then sorts by pT.

### 7) Object counts (after selection)
- **Counts of good objects:** lines **1287–1330**
  - Defines `nGoodGlobalMuon`, `nGoodRecoMuon`, `nGoodElectron` and fills `h_nggmu`, `h_ngmu`, `h_nge`.

### 8) Z control regions
- **Dimuon using Global Muons (demo):** lines **~1331–~1331** (immediately before Z→μμ block)
- **Z→μμ using selected reco/PF muons:** lines **1332–1357**
  - Fills `h_mZ_2mu`.
- **Z→ee using selected electrons:** lines **1360–1385**
  - Fills `h_mZ_2e`.

### 9) 4μ channel: Z pairing → Za/Zb → 4ℓ candidate → histograms
- **ZZ/ZZ* → 4μ reconstruction:** lines **1388–1736**
  - Builds 3 OS pairing combinations: (12,34), (13,24), (14,23).
  - Chooses best pairing using |mZij − mZ| (Za is closest to mZ).
  - Applies pT requirement on Za leptons (20 GeV and 10 GeV), mass windows (mZa, mZb).
  - Builds `p4H = p4Za + p4Zb`, fills `m4μ` histograms if `m4ℓ > 70`.

### 10) 4e channel: analogous to 4μ
- **ZZ/ZZ* → 4e reconstruction:** lines **1738–2080**
  - Same pairing logic, Za/Zb choice, mass windows, `m4e` spectra.

### 11) 2μ2e channel
- **ZZ/ZZ* → 2μ2e reconstruction:** lines **2083–2272**
  - Builds Zμ and Ze, chooses Za as the one closer to mZ, applies pT + mass window cuts.
  - Builds `m2μ2e` spectra and fills isolation/kinematics “after cuts” control plots.

### 12) Optional ntuples (currently commented out)
- **beginJob() ntuple branches (commented):** lines **2279–~2460**
  - `TTree` branches for 4μ / 4e / 2μ2e candidates are present but commented out.

### 13) End of module
- **Plugin definition:** line **2467**
  - `DEFINE_FWK_MODULE(HiggsDemoAnalyzerGit);`
