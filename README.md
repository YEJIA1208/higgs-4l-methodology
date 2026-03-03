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
    
### 4) Member variables: per-event bookkeeping & reconstructed kinematics (lines 267–359)

- **Counters (selected objects)**
  - Tracks how many leptons pass the selection in each category:
    - `nGoodGlobalMuon`, `nGoodRecoMuon` (muons)
    - `nGoodElectron` (electrons)

- **Generic helper scalars**
  - Temporary scalars used in intermediate calculations / sorting:
    - `s1…s4, s`, and displacement / kinematics helpers (`dx, dy, dz, rap, pz`)

- **Z-candidate masses & pairing bookkeeping**
  - Stores invariant masses for all relevant opposite-sign lepton pairings:
    - `mZ12, mZ34, mZ13, mZ24, mZ14, mZ23`
  - Stores the chosen Z candidates after applying pairing logic:
    - `mZa, mZb` (typically Z1 = “best Z” and Z2 = the other pair)

- **4μ channel reconstructed quantities**
  - 4-lepton candidate observables:
    - `mass4mu, pt_4mu, eta_4mu, phi_4mu` and components `px4mu, py4mu, pz4mu, E4mu`
  - Per-muon kinematics and charge bookkeeping:
    - `pt_mu1…pt_mu4`, `eta_mu1…eta_mu4`, `phi_mu1…phi_mu4`
    - `cas_mu1…cas_mu4` (charge sign / charge category flags)
    - momentum components `px_mu*`, `py_mu*`, `pz_mu*` and energies `E_mu*`

- **Z-candidate 4-vector components (muon/electron-independent representation)**
  - Stores energies, momenta, and derived kinematics for each Z pairing:
    - energies `eZ12…eZ23`
    - components `pxZ12…`, `pyZ12…`, `pzZ12…`
    - magnitudes `pZ12…` and transverse momenta `pTZ12…`
  - Additional displacement-like quantities:
    - `dZ12…dZ23` and helper distances `dZc1…dZc3`
  - Chosen Z candidates’ components:
    - `eZa, pxZa, pyZa, pzZa, pTZa` and `eZb, pxZb, pyZb, pzZb, pTZb`

- **Electron selection / quality bookkeeping**
  - Electron counters and quality-related variables:
    - `sqme`, `misshits` (e.g. missing hits used in electron ID)

- **4e channel reconstructed quantities**
  - 4-lepton candidate observables:
    - `mass4e, pt_4e, eta_4e, phi_4e` and components `px4e, py4e, pz4e, E4e`
  - Per-electron kinematics and charge bookkeeping:
    - `pt_e1…pt_e4`, `eta_e1…eta_e4`, `phi_e1…phi_e4`
    - `cas_e1…cas_e4`
    - components `px_e*`, `py_e*`, `pz_e*` and energies `E_e*`

- **2μ2e channel reconstructed quantities**
  - Mixed final state candidate observables:
    - `mass2mu2e, pt_2mu2e, eta_2mu2e, phi_2mu2e` and components `px2mu2e, py2mu2e, pz2mu2e, E2mu2e`
  - Per-lepton kinematics for the mixed channel:
    - muons: `pt_2mu1, pt_2mu2`, `eta_2mu1, eta_2mu2`, `phi_2mu1, phi_2mu2`, `cas_2mu1, cas_2mu2`
    - electrons: `pt_2e1, pt_2e2`, `eta_2e1, eta_2e2`, `phi_2e1, phi_2e2`, `cas_2e1, cas_2e2`
    - components `px_*`, `py_*`, `pz_*` and energies `E_*`

- **Isolation & impact-parameter variables (used in lepton ID cuts)**
  - Shared selection variables:
    - `goodhit`, `relPFIso_mu`, `relPFIso_e`
  - 3D impact parameter and its significance:
    - muons: `IP3d_mu`, `ErrIP3d_mu`, `SIP3d_mu`
    - electrons: `IP3d_e`, `ErrIP3d_e`, `SIP3d_e`

- **Run/event/lumi bookkeeping**
  - Stores identifiers for data quality filtering and debugging:
    - `nRun, nEvt, nLumi`

- **Final reconstructed 4-vectors**
  - Lorentz vectors for the chosen Z candidates and Higgs candidate:
    - `TLorentzVector p4Za, p4Zb, p4H`
### 5) Constructor: analysis entry point & ROOT output service setup (lines 360–375)

- **Constructor begins (`HiggsDemoAnalyzerGit::HiggsDemoAnalyzerGit(const edm::ParameterSet& iConfig)`)**
  - Marks the start of the analyzer initialisation phase.

- **High-level intent (header comment)**
  - States this module’s goal: **approximately reproduce** the Higgs→4ℓ mass spectrum from **CMS-HIG-12-028**.

- **ROOT output service initialisation**
  - Creates/uses the CMSSW `TFileService`:
    - `edm::Service<TFileService> fs;`
  - This service is the standard CMSSW mechanism to write ROOT histograms/trees into the output ROOT file.

- **Histogram booking section starts**
  - Begins the “book histograms and set axis labels” block:
    - this section is executed **once during module initialisation** (i.e. not per-event),
    - histogram objects declared earlier (`TH1D*`, `TH2D*`) will be created via `fs->make<TH1D>(...)` and configured with titles/axis labels.
    
### 6) Histogram booking: event counts + Z/4ℓ mass spectra (lines 376–602)

- **Histogram booking pattern (one-time initialisation)**
  - Uses `fs->make<TH1D>(name, title, nbins, xmin, xmax)` to create each histogram once.
  - Immediately assigns axis labels via `GetXaxis()->SetTitle(...)` and `GetYaxis()->SetTitle(...)`.

- **Basic event/object multiplicities**
  - Books histograms to monitor collection sizes and selected-object counts:
    - Global muons: `NGMuons` → `h_globalmu_size`
    - Reco muons: `NMuons` → `h_recomu_size`
    - Electrons: `Nelectrons` → `h_e_size`
    - “Good” object counts after quality cuts:
      - `NGoodGMuons` (`h_nggmu`), `NGoodRecMuons` (`h_ngmu`), `NGoodElectron` (`h_nge`)

- **Dimuon mass sanity-check spectra (Global Muons)**
  - Books three dimuon invariant-mass distributions with increasing range:
    - `GMmass` (0–4 GeV): low-mass resonances (ρ/ω, ϕ, J/ψ region)
    - `GMmass_extended` (0–120 GeV): includes Υ and the Z peak
    - `GMmass_extended_600` (0–600 GeV): wide range to spot high-mass tails/outliers

- **Baseline Z→2ℓ validation histograms**
  - Books Z candidate mass spectra used to validate lepton selection:
    - `massZto2muon` (`h_mZ_2mu`, 40–120 GeV)
    - `massZto2e` (`h_mZ_2e`, 40–120 GeV)

- **4μ reconstruction: pairing combinations + chosen Z candidates + 4μ mass**
  - Books masses for the three OS-pairing combinations in 4μ:
    - combination **1234**: `mZ12_4mu`, `mZ34_4mu`
    - combination **1324**: `mZ13_4mu`, `mZ24_4mu`
    - combination **1423**: `mZ14_4mu`, `mZ23_4mu`
  - Books the selected Z candidates (based on “closest to mZ” logic):
    - `mZa_4mu` (Z closest to nominal Z mass), `mZb_4mu` (the other Z)
  - Books 4μ invariant-mass spectra in multiple ranges / binnings:
    - `mass4mu_7TeV` (98–608 GeV)
    - `mass4mu_8TeV` (70–810 GeV)
    - `mass4mu_8TeV_low` (70–181 GeV)
    - `mass4mu_full` (0–900 GeV)

- **4e reconstruction: pairing combinations + chosen Z candidates + 4e mass**
  - Same structure as 4μ, but for dielectron pairs:
    - pairings: `mZ12_4e`, `mZ34_4e`, `mZ13_4e`, `mZ24_4e`, `mZ14_4e`, `mZ23_4e`
    - chosen Zs: `mZa_4e`, `mZb_4e`
    - 4e mass spectra: `mass4e_7TeV`, `mass4e_8TeV`, `mass4e_8TeV_low`, `mass4e_full`

- **2μ2e reconstruction: Z(μμ) + Z(ee) + chosen Z ordering + 2μ2e mass**
  - Books separate Z masses built from the muon pair and electron pair:
    - `massZmu_2mu2e` (`h_mZmu_2mu2e`), `massZe_2mu2e` (`h_mZe_2mu2e`)
  - Books “chosen Z” ordering for the mixed final state:
    - `mZa_2mu2e` (Z closest to nominal Z mass), `mZb_2mu2e` (the other Z)
  - Books 2μ2e invariant-mass spectra (multiple ranges / binnings):
    - `mass2mu2e_7TeV`, `mass2mu2e_8TeV`, `mass2mu2e_8TeV_low`, `mass2mu2e_full`

### 7) Histogram booking: control plots for lepton ID/quality (lines 605–880)

- **Control plots section**
  - Books diagnostic histograms (“control plots”) used to validate lepton reconstruction and selection cuts.
  - These plots are not the final Higgs mass spectra; they are for checking **kinematics, track quality, isolation, and impact parameters** before/after cuts.

- **Global Muon (GM) control plots**
  - Books kinematics and track-fit quality for **global muons**:
    - momentum and kinematics: `GM_momentum`, `b4_GM_pT`, `b4_GM_eta`, `GM_phi`
    - fit quality: `GM_chi2`, `GM_ndof`, `GM_normchi2`
    - hit information: `GM_validhits`, `GM_pixelhits`
  - Books “after cuts” distributions to show the effect of the selection:
    - `after_GM_pT`, `after_GM_eta`

- **Reco Muon (RM) / PF-muon related control plots**
  - Books kinematics and quality metrics for **reco muons**:
    - momentum and kinematics: `RM_momentum`, `b4_RM_pt`, `b4_RM_eta`, `RM_phi`
    - fit quality: `RM_chi2`, `RM_Ndof`, `RM_NormChi2`
  - Books tracking / muon-system quality and displacement:
    - muon chamber hits: `RM_goodMuonChamberHit`
    - transverse impact parameter: `RM_dxy`
    - hit counters: `RM_validhits`, `RM_pixelhits`
  - Books PF relative isolation **before/after** muon isolation cuts:
    - `RM_RelPFIso`, `after_RM_RelPFIso`
  - Books “after cuts” kinematics for different selection contexts:
    - after Z→μμ selection: `after_RM_pt_Z2mu`, `after_RM_eta_Z2mu`
    - after general muon selection: `after_RM_pt`, `after_RM_eta`
    - after 2μ2e channel selection: `after_pt_2mu2e`, `after_eta_2mu2e`
  - Books channel-specific isolation checks for the 2μ2e selection:
    - muon isolation after cuts: `after_relPFIso_2mu`
    - electron isolation after cuts: `after_relPFIso_2e`

- **Electron control plots**
  - Books kinematics and ECAL supercluster information:
    - momentum and kinematics: `e_momentum`, `e_eT`, `b4_e_pT`, `b4_e_eta`, `e_phi`
    - supercluster variables: `e_SC_eta`, `e_SC_rawE`
  - Books PF relative isolation **before/after** electron isolation cuts:
    - `e_RelPFIso`, `after_e_RelPFIso`
  - Books a 2D diagnostic histogram to study isolation vs pT:
    - `e_RelPFIso_pT` (uses custom bin edges in isolation)
  - Books transverse impact parameter distribution:
    - `e_dxy`
  - Books “after cuts” electron kinematics for different contexts:
    - after Z→ee selection: `after_e_pT_Zto2e`, `after_e_eta_Zto2e`
    - after general electron selection: `after_e_pT`, `after_e_eta`
    - after 2μ2e channel selection: `after_e_pT_2mu2e`, `after_e_eta_2mu2e`

- **Impact parameter significance & electron missing hits**
  - Books SIP3D (3D impact parameter significance) for muons and electrons:
    - `SIP3d_mu`, `SIP3d_e`
  - Books electron track missing hits histogram (used in electron ID quality):
    - `e_misshit`
