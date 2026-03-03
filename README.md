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

### 0) High-level purpose & strategy (lines 1–69)

- **Header: what this code is**
  - States that this is a **research-level H → ZZ → 4ℓ** example, built on the CMS Open Data DEMO setup.
  - Emphasises this is a **strongly simplified reimplementation** of parts of the original CMS 4ℓ analysis (not the official CMS analysis code).
  - The target is to **approximately reproduce** the published Higgs→4ℓ mass spectrum from **CMS-HIG-12-028** (Phys. Lett. B716 (2012) 30–61, arXiv:1207.7235), with the reference spectrum plot linked.

- **Scope & limitations**
  - Clarifies that other Higgs channels (e.g. **H → γγ**) are **not included**.
  - Notes that advanced features are intentionally omitted for simplicity:
    - no extra corrections beyond those implicit in the reconstructed objects,
    - no systematic uncertainties,
    - simplified selection compared to the full CMS analysis.
  - Explains why agreement with the published spectrum is only **qualitative / approximate**:
    - partial dataset overlap,
    - legacy software/calibrations differ from those used in the original paper.

- **Outputs**
  - Mentions the output ROOT file contains:
    - the main histograms (and many auxiliary / control histograms),
    - an **ntuple** with candidate four-vectors for educational use.

- **Analysis strategy: dataset split to avoid double counting**
  - Recommends a dataset split to avoid **trigger overlap** (double counting):
    - use **DoubleMu** for **4μ** and **2μ2e** final states,
    - use **DoubleElectron** for **4e** final state.
  - Important note: the analyzer code is **agnostic** to which dataset you run on; you must select the appropriate histograms later in the **ROOT post-processing** step.

### 1) Includes & dependencies (lines 70–124)

- **System / C++ standard headers**
  - Includes core C++ utilities used throughout the analyzer:
    - memory management (`<memory>`), containers (`<vector>`), algorithms (`<algorithm>`), and helper types (`<utility>`).

- **CMSSW framework headers (EDAnalyzer skeleton)**
  - Imports the CMSSW framework interfaces required to define and run an `edm::EDAnalyzer`:
    - module base classes and event access (`EDAnalyzer`, `Event`, `Frameworkfwd`)
    - configuration via python `.cfg` (`ParameterSet`)
    - module registration (`MakerMacros`).

- **Extra CMSSW services / utilities**
  - Adds optional but commonly used framework utilities:
    - access to `EventSetup` and conditions data (`EventSetup`, `ESHandle`)
    - logging (`MessageLogger`)
    - ROOT output handling via `TFileService` (standard CMSSW way to write histograms/trees)
    - `edm::Ref` for referencing EDM objects inside collections.
   
### 2) Analyzer class interface (lines 126–137)

- **EDAnalyzer module declaration**
  - Declares the analysis module `HiggsDemoAnalyzerGit` as a subclass of **`edm::EDAnalyzer`** (standard CMSSW analyzer plugin).

- **Constructor / destructor**
  - `HiggsDemoAnalyzerGit(const edm::ParameterSet&)`: reads configuration parameters from the CMSSW python `.cfg` file.
  - `~HiggsDemoAnalyzerGit()`: handles cleanup when the job finishes.

- **CMSSW lifecycle methods**
  - `beginJob()`: called once at job start (typically used to **book ROOT histograms / trees** via `TFileService`).
  - `analyze(const edm::Event&, const edm::EventSetup&)`: called **once per event** (core analysis: object selection, Z/4ℓ reconstruction, apply cuts, fill histograms).
  - `endJob()`: called once at job end (final summaries / closing steps if needed).

- **Good-lumisection filter helper**
  - `providesGoodLumisection(const edm::Event&)`: helper function used to apply **good-run / good-lumi** quality selection (removes known-bad data-taking intervals).

### 3) ROOT outputs: histograms for physics spectra + control plots (lines 138–266)

- **ROOT objects declared**
  - Declares the ROOT outputs written to the job output file:
    - optional `TTree` (currently commented out),
    - many `TH1D` and a `TH2D` for event/control distributions and reconstructed masses.

- **Object counting / preselection bookkeeping**
  - Multiplicity and “good object” counters used to sanity-check selections:
    - muons/electrons sizes and counts (e.g. `h_globalmu_size`, `h_recomu_size`, `h_e_size`, `h_ngmu`, `h_nge`, …)
    - intermediate mass distributions for early muon filtering (`h_m1_gmu`, `h_m2_gmu`, `h_m3_gmu`).

- **Z-candidate mass histograms (baseline 2ℓ)**
  - Reconstructed Z→2ℓ masses used to validate lepton selection and Z building:
    - `h_mZ_2mu` (Z→μ+μ−), `h_mZ_2e` (Z→e+e−).

- **Final-state specific mass bookkeeping**
  - **4μ channel:** stores all possible opposite-sign pairing combinations and the chosen Z candidates:
    - pairing masses (`h_mZ12_4mu`, `h_mZ34_4mu`, `h_mZ13_4mu`, `h_mZ24_4mu`, `h_mZ14_4mu`, `h_mZ23_4mu`)
    - chosen Z candidates (`h_mZa_4mu`, `h_mZb_4mu`)
    - per-lepton mass / candidate-level bookkeeping (`h_m1_m4mu … h_m4_m4mu`)
  - **4e channel:** same structure as 4μ:
    - pairing masses (`h_mZ12_4e`, `h_mZ34_4e`, …)
    - chosen Z candidates (`h_mZa_4e`, `h_mZb_4e`)
    - bookkeeping (`h_m1_m4e … h_m4_m4e`)
  - **2μ2e channel:** stores the Z built from muons and the Z built from electrons, plus the chosen Z ordering:
    - `h_mZmu_2mu2e`, `h_mZe_2mu2e`, `h_mZa_2mu2e`, `h_mZb_2mu2e`
    - bookkeeping (`h_m1_m2mu2e … h_m4_m2mu2e`)

- **Control plots (selection diagnostics)**
  - **Global muon quality (before/after cuts):**
    - kinematics and fit quality (`h_p_gmu`, `h_pt_gmu_b4`, `h_eta_gmu_b4`, `h_phi_gmu`, `h_chi2_gmu`, `h_ndof_gmu`, `h_normchi2_gmu`)
    - tracking hits (`h_validhits_gmu`, `h_pixelhits_gmu`)
    - after-selection kinematics (`h_pt_gmu_after`, `h_eta_gmu_after`)
  - **Tracker muon quality:**
    - `h_p_reco`, `h_pt_reco_b4`, `h_eta_reco_b4`, `h_phi_reco`, `h_chi2_reco`, `h_ndof_reco`, `h_normchi2_reco`
  - **PF muon isolation / impact parameters:**
    - hit quality & displacement (`h_goodhit`, `h_dxy_mu`)
    - PF relative isolation before/after (`h_relPFIso_mu`, `h_relPFIso_mu_after`)
    - after-selection kinematics (inclusive + per-channel) (`h_pt_after`, `h_eta_after`, `h_pt_after_Zto2mu`, `h_eta_after_Zto2mu`, `h_pt_after_2mu2e`, `h_eta_after_2mu2e`)
  - **Electron ID / isolation diagnostics:**
    - kinematics (`h_p_e`, `h_et_e`, `h_pt_e_b4`, `h_eta_e_b4`, `h_phi_e`)
    - supercluster info (`h_sc_eta`, `h_sc_rawE`)
    - PF relative isolation before/after (`h_relPFIso_e`, `h_relPFIso_e_after`)
    - isolation vs pT (`h_relPFIso_pt_e` as `TH2D`)
    - displacement (`h_dxy_e`)
    - after-selection kinematics (Z→ee, inclusive, 2μ2e) (`h_pt_e_after_Zto2e`, `h_eta_e_after_Zto2e`, `h_pt_e_after`, `h_eta_e_after`, `h_pt_e_after_2mu2e`, `h_eta_e_after_2mu2e`)
  - **Final control variables used in lepton quality cuts**
    - per-channel isolation summaries (`h_relPFIso_2mu_after`, `h_relPFIso_2e_after`)
    - impact parameter significance and missing hits (`h_SIP3d_mu_b4`, `h_SIP3d_e_b4`, `h_misshite`)

