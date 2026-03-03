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
    
### 8) Destructor (lines 882–885)

- **Module cleanup hook**
  - Defines the destructor `~HiggsDemoAnalyzerGit()`.
  - In this implementation, no explicit cleanup is performed.
  - Comment notes this is where you would close files or deallocate resources if needed (most ROOT objects are managed by `TFileService`).
  
### 9) `analyze()`: per-event entry point & event bookkeeping (lines 887–918)

- **Per-event analysis hook**
  - Defines `void HiggsDemoAnalyzerGit::analyze(const edm::Event& iEvent, const edm::EventSetup& iSetup)`, the CMSSW method that runs **once for every event**.

- **Run / event / lumisection bookkeeping**
  - Saves identifiers for debugging and data-quality filtering:
    - `nRun  = iEvent.run();`
    - `nEvt  = (iEvent.id()).event();`
    - `nLumi = iEvent.luminosityBlock();`

- **Template/example blocks (not active unless enabled)**
  - The `#ifdef THIS_IS_AN_EVENT_EXAMPLE` and `#ifdef THIS_IS_AN_EVENTSETUP_EXAMPLE` sections are **example snippets** showing how to retrieve:
    - event data products via `iEvent.getByLabel(...)`,
    - conditions / setup data via `iSetup.get<...>().get(...)`.
  - These blocks are typically **disabled** in real runs unless the macro is defined.

- **Start-of-event logging**
  - Uses `edm::LogInfo("Demo")` to print a message when starting an event:
    - includes event number, run number, and lumisection.
  - Useful for debugging / confirming the job is processing events.

### 10) `analyze()`: load event data products (AOD collections) (lines 920–969)

- **Purpose of this block**
  - Loads the reconstructed physics-object collections needed for the H→4ℓ selection from each event (AOD-level EDM products).
  - Comments provide CMS workbook / EDM references for `getByLabel` usage and AOD content tables.

- **Tracks and muons**
  - Retrieves the **general track collection**:
    - `generalTracks` → `edm::Handle<reco::TrackCollection> tracks`
  - Retrieves **global muon tracks** (note: `globalMuons` label returns a `reco::TrackCollection`):
    - `globalMuons` → `edm::Handle<reco::TrackCollection> gmuons`
  - Retrieves the **muon object collection**:
    - `muons` → `edm::Handle<reco::MuonCollection> muons`

- **Beam spot and primary vertices**
  - Retrieves beam spot (used for impact parameter / displacement variables):
    - `offlineBeamSpot` → `edm::Handle<reco::BeamSpot> beamSpotHandle`
  - Retrieves reconstructed primary vertices:
    - `offlinePrimaryVertices` → `edm::Handle<reco::VertexCollection> primvtxHandle`
  - Validity checks:
    - if the handle is valid, copies the object/collection into local variables (`beamSpot`, `primvtx`)
    - otherwise prints a log message (“No beam spot / No primary vertex available…”)

- **Electrons**
  - Retrieves the GSF electron collection:
    - `gsfElectrons` → `edm::Handle<reco::GsfElectronCollection> electrons`

### 11) `analyze()`: initialise per-event variables & fill basic multiplicities (lines 971–1061)

- **Temporary containers for selected objects**
  - Declares vectors of `(index, pT)` pairs to store candidates for later sorting/selection:
    - `vIdPt` (generic helper)
    - `vIdPtmu` (muon candidates)
    - `vIdPte` (electron candidates)

- **Per-event variable reset (sentinel initialisation)**
  - Resets all per-event / per-candidate member variables to sentinel values:
    - uses `-9999` (or `9999` where a minimum is later selected) to mark “unset / invalid”
  - This prevents values from previous events leaking into the current event if a branch of the reconstruction is not executed.

- **Reset 4-vectors**
  - Sets Z and Higgs candidate Lorentz vectors to zero:
    - `p4Za`, `p4Zb`, `p4H` initialised with `SetPxPyPzE(0,0,0,0)`

- **Physics constants used later in invariant-mass calculations**
  - Defines squared lepton masses and nominal Z mass:
    - `sqm1 = m_μ^2` with `m_μ = 0.105658 GeV`
    - `sqme = m_e^2` with `m_e = 0.0005109989 GeV`
    - `mZ = 91.1876 GeV`

- **Fill basic object multiplicity histograms**
  - Fills event-level size monitors directly from the EDM collections:
    - `h_globalmu_size->Fill(gmuons->size())`
    - `h_recomu_size->Fill(muons->size())`
    - `h_e_size->Fill(electrons->size())`

### 12) `analyze()`: global-muon loop (control plots + “good GM” selection) (lines 1063–1118)

- **Purpose of this block**
  - Iterates over the **`globalMuons` track collection** (`reco::TrackCollection`) to fill **global-muon control plots**.
  - As stated in the comment, this is largely inherited from the **dimuon example** and is **mainly diagnostic** (not the core Higgs 4ℓ selection).

- **Per-global-muon control plots (before cuts)**
  - For each global-muon track `iMuon`, fills kinematics and fit-quality histograms:
    - kinematics: `h_p_gmu`, `h_pt_gmu_b4`, `h_eta_gmu_b4`, `h_phi_gmu`
    - fit quality: `h_chi2_gmu`, `h_ndof_gmu`, `h_normchi2_gmu`

- **Hit-pattern inspection**
  - Extracts the `reco::HitPattern` and loops over hits to count:
    - number of **valid hits** (`GM_ValidHits`)
    - number of **valid pixel hits** (`GM_PixelHits`)
  - Fills hit-count histograms:
    - `h_validhits_gmu`, `h_pixelhits_gmu`

- **“Good global muon” preselection + pT ranking**
  - Applies a simple quality cut on the global-muon track:
    - `GM_ValidHits >= 12`
    - `GM_PixelHits >= 2`
    - `normalizedChi2 < 4.0`
  - For tracks passing the cut, stores `(index, pT)` into `vIdPt` for later ranking:
    - `vIdPt.push_back( (t, iMuon.pt()) )`

- **Sort selected global muons by descending pT**
  - Sorts `vIdPt` so the highest-pT candidates come first:
    - `std::sort(..., idPt1.second > idPt2.second)`

### 13) `analyze()`: reco muon (PF muon) selection for H→4ℓ (lines 1121–1207)

- **Purpose of this block (core muon selection)**
  - Loops over the **`reco::MuonCollection`** (`muons`) and performs the **main muon selection used in the Higgs→4 lepton analysis** (as stated in the comment).
  - Uses the primary vertex position (`primvtx[0]`) as the reference point for impact parameter calculations.

- **Precondition checks (avoid invalid track refs)**
  - Only considers muons that satisfy:
    - `itMuon.isPFMuon()` (PF muon)
    - `itMuon.isPFIsolationValid()` (PF isolation available)
    - `(itMuon.globalTrack()).isNonnull()` (has a valid global track reference)

- **Control plots filled before cuts**
  - Kinematics for muons passing the preconditions:
    - `h_p_reco`, `h_pt_reco_b4`, `h_eta_reco_b4`, `h_phi_reco`
  - Track-fit quality from the global track:
    - `h_chi2_reco`, `h_ndof_reco`, `h_normchi2_reco`

- **PF relative isolation (R=0.4)**
  - Computes muon PF relative isolation:
    - `relPFIso_mu = (chargedHadronPt + neutralHadronEt + photonEt) / pT`
  - Fills isolation histogram:
    - `h_relPFIso_mu`

- **Hit and impact parameter diagnostics**
  - Extracts hit pattern from the muon global track and fills:
    - number of valid muon hits: `goodhit = numberOfValidMuonHits()` → `h_goodhit`
    - transverse impact parameter: `dxy(point)` → `h_dxy_mu`
  - Builds 3D impact parameter and its significance:
    - `IP3d_mu = sqrt(dxy^2 + dz^2)`
    - `ErrIP3d_mu = sqrt(d0Error^2 + dzError^2)`
    - `SIP3d_mu = IP3d_mu / ErrIP3d_mu`
  - Fills SIP3D histogram (before final cuts):
    - `h_SIP3d_mu_b4`

- **Track hit counters**
  - Loops over hit pattern to count:
    - valid hits (`RM_ValidHits`)
    - valid pixel hits (`RM_PixelHits`)
  - Fills:
    - `h_validhits_mu`, `h_pixelhits_mu`

- **Muon ID / kinematic cuts (select “good muons”)**
  - Applies quality + isolation + displacement cuts:
    - `|SIP3d_mu| < 4`
    - `|dxy| < 0.5`
    - `|dz| < 1`
    - `relPFIso_mu < 0.4`
  - Applies kinematic acceptance:
    - `pT > 5 GeV`
    - `|eta| < 2.4`
  - For muons passing all cuts, stores `(index, pT)` in `vIdPtmu` for later ranking.

- **Sort selected muons by descending pT**
  - Sorts `vIdPtmu` so the highest-pT muons come first:
    - `std::sort(..., idPtmu1.second > idPtmu2.second)`

### 14) `analyze()`: electron (GSF) selection for H→4ℓ (lines 1210–1286)

- **Purpose of this block (core electron selection)**
  - Loops over the **`reco::GsfElectronCollection`** (`electrons`) and selects “good electrons” for the Higgs→4 lepton analysis.
  - Uses the primary vertex position (`primvtx[0]`) as the reference point for impact parameter calculations.

- **Preselection**
  - Requires the electron passes PF-related preselection:
    - `iElectron.passingPflowPreselection()`

- **Quality / impact parameter quantities**
  - Computes electron track missing inner hits (used in electron ID):
    - `misshits = trackerExpectedHitsInner().numberOfHits()`
    - filled into `h_misshite`
  - Computes 3D impact parameter and its significance:
    - `IP3d_e = sqrt(dxy^2 + dz^2)`
    - `ErrIP3d_e = sqrt(d0Error^2 + dzError^2)`
    - `SIP3d_e = IP3d_e / ErrIP3d_e`
    - filled into `h_SIP3d_e_b4`

- **Control plots filled before final cuts**
  - Electron kinematics:
    - `h_p_e`, `h_et_e`, `h_pt_e_b4`, `h_eta_e_b4`, `h_phi_e`
  - SuperCluster quantities:
    - `h_sc_eta` (SC η), `h_sc_rawE` (SC raw energy)
  - Displacement:
    - `h_dxy_e` filled with `gsfTrack()->dxy(point)`

- **PF relative isolation**
  - Computes electron PF relative isolation:
    - `relPFIso_e = (chargedHadronIso + neutralHadronIso + photonIso) / pT`
  - Fills:
    - `h_relPFIso_e`
    - `h_relPFIso_pt_e` (2D: isolation vs pT)

- **Electron ID / kinematic cuts (select “good electrons”)**
  - Applies kinematic acceptance:
    - `pT > 7 GeV`
    - `|superCluster eta| < 2.5`
  - Applies track / IP quality:
    - `misshits <= 1`
    - `|SIP3d_e| < 4`
    - `|dxy| < 0.5`
    - `|dz| < 1`
  - Applies detector-region dependent branch (barrel vs endcap):
    - `iElectron.isEB()` (ECAL barrel) or `iElectron.isEE()` (ECAL endcap)
    - in both cases requires `relPFIso_e < 0.4`
  - For electrons passing all cuts, stores `(index, pT)` into `vIdPte` for later ranking.

- **Sort selected electrons by descending pT**
  - Sorts `vIdPte` so the highest-pT electrons come first:
    - `std::sort(..., idPte1.second > idPte2.second)`

### 15) `analyze()`: count selected (“good”) leptons and fill multiplicity histograms (lines 1288–1295)

- **Convert candidate lists into counts**
  - After building the candidate vectors:
    - `vIdPt`  (good global-muon tracks)
    - `vIdPtmu` (good PF/reco muons used in H→4ℓ)
    - `vIdPte`  (good GSF electrons used in H→4ℓ)
  - The code records the number of selected candidates:
    - `nGoodGlobalMuon = vIdPt.size();`
    - `nGoodRecoMuon   = vIdPtmu.size();`
    - `nGoodElectron   = vIdPte.size();`

- **Fill “good object” multiplicity histograms**
  - Writes the selected-object counts into the control histograms:
    - `h_nggmu->Fill(nGoodGlobalMuon);`
    - `h_ngmu->Fill(nGoodRecoMuon);`
    - `h_nge->Fill(nGoodElectron);`

### 16) `analyze()`: global-muon dimuon mass (diagnostic) (lines 1297–1330)

- **Purpose of this block**
  - Builds a **dimuon invariant mass** using the *selected global-muon tracks* (`vIdPt`) and fills the global-muon mass spectra.
  - This is mainly a **sanity-check / control** calculation (connected to the dimuon example), not the core H→4ℓ reconstruction.

- **Select the leading two global muons**
  - Requires at least two “good global muons”:
    - `if (nGoodGlobalMuon >= 2)`
  - Since `vIdPt` was sorted by descending pT, the code takes:
    - leading muons: `vIdPt.at(0)` and `vIdPt.at(1)` (highest pT)

- **Opposite-sign requirement**
  - Requires a neutral dimuon pair:
    - `gmuon1.charge() + gmuon2.charge() == 0`

- **Fill “after cuts” kinematics for all selected global muons**
  - For each selected global muon in `vIdPt`, fills:
    - `h_pt_gmu_after` with the stored pT (`.second`)
    - `h_eta_gmu_after` with the corresponding track η

- **Invariant mass calculation**
  - Computes dimuon invariant mass `s` using track momenta and the muon mass term `sqm1 = m_μ^2`:
    - builds an energy-like product `s1`
    - builds a momentum dot-product `s2`
    - computes `s = sqrt(2 * (m_μ^2 + (s1 - s2)))`
  - Fills the dimuon mass into three histograms with different ranges:
    - `h_m1_gmu` (0–4 GeV)
    - `h_m2_gmu` (0–120 GeV)
    - `h_m3_gmu` (0–600 GeV)

### 17) `analyze()`: Z→μμ reconstruction using selected reco (PF) muons (lines 1332–1357)

- **Purpose of this block**
  - Reconstructs a **Z→μ⁺μ⁻ candidate** using the *selected “good” reco muons* stored in `vIdPtmu`.
  - Fills the Z mass spectrum `h_mZ_2mu` and “after cuts” muon kinematics for validation.

- **Select the leading two reco muons**
  - Requires at least two selected muons:
    - `if (nGoodRecoMuon >= 2)`
  - Since `vIdPtmu` is sorted by descending pT, uses:
    - `muon1 = vIdPtmu.at(0)` (highest pT)
    - `muon2 = vIdPtmu.at(1)` (second-highest pT)

- **Opposite-sign requirement**
  - Requires a neutral dimuon pair:
    - `muon1.charge() + muon2.charge() == 0`

- **Fill “after cuts” muon kinematics**
  - For all selected muons in `vIdPtmu`, fills:
    - `h_pt_after_Zto2mu` with stored pT (`.second`)
    - `h_eta_after_Zto2mu` with the muon η

- **Invariant mass calculation**
  - Computes the dimuon invariant mass `s` using the muon 3-momenta and `sqm1 = m_μ^2`:
    - `s1` energy-like term, `s2` momentum dot-product
    - `s = sqrt(2 * (m_μ^2 + (s1 - s2)))`
  - Fills the reconstructed Z mass:
    - `h_mZ_2mu->Fill(s)`

### 18) `analyze()`: Z→ee reconstruction using selected GSF electrons (lines 1360–1385)

- **Purpose of this block**
  - Reconstructs a **Z→e⁺e⁻ candidate** using the selected “good electrons” stored in `vIdPte`.
  - Fills the Z mass spectrum `h_mZ_2e` and “after cuts” electron kinematics for validation.

- **Select the leading two electrons**
  - Requires at least two selected electrons:
    - `if (nGoodElectron >= 2)`
  - Since `vIdPte` is sorted by descending pT, uses:
    - `elec1 = vIdPte.at(0)` (highest pT)
    - `elec2 = vIdPte.at(1)` (second-highest pT)

- **Opposite-sign requirement**
  - Requires a neutral dielectron pair:
    - `elec1.charge() + elec2.charge() == 0`

- **Fill “after cuts” electron kinematics**
  - For all selected electrons in `vIdPte`, fills:
    - `h_pt_e_after_Zto2e` with stored pT (`.second`)
    - `h_eta_e_after_Zto2e` using **supercluster η** (`superCluster()->eta()`), which is commonly used in electron acceptance definitions.

- **Invariant mass calculation**
  - Computes dielectron invariant mass `s` using electron 3-momenta and `sqme = m_e^2`:
    - `s1` energy-like term, `s2` momentum dot-product
    - `s = sqrt(2 * (m_e^2 + (s1 - s2)))`
  - Fills the reconstructed Z mass:
    - `h_mZ_2e->Fill(s)`

### 19) `analyze()`: ZZ/ZZ*→4μ reconstruction and 4μ mass spectrum (lines 1388–1736)

- **Purpose of this block (core 4μ channel)**
  - If at least **4 good reco/PF muons** are available, builds **ZZ/ZZ*→4μ** candidates:
    - forms opposite-sign (OS) muon pairings,
    - chooses the best Z candidate (closest to nominal mZ),
    - applies Z-mass windows + leading-lepton pT requirements,
    - constructs the 4μ candidate (H→4μ proxy) and fills the **4μ invariant-mass spectra**.

- **Pick the 4 leading-pT selected muons**
  - Requires at least 4 selected muons:
    - `if (nGoodRecoMuon >= 4)`
  - Takes the top-4 pT muons from the sorted list `vIdPtmu`:
    - `muon1, muon2, muon3, muon4`

- **Total charge requirement**
  - Requires the 4-muon system to be neutral:
    - `muon1.charge() + muon2.charge() + muon3.charge() + muon4.charge() == 0`

- **Compute the three OS pairing combinations (for 4μ)**
  - The code tries three distinct pairing patterns (each with two OS dimuon pairs):
    1) **(1,2) + (3,4)** → Z12 and Z34  
    2) **(1,3) + (2,4)** → Z13 and Z24  
    3) **(1,4) + (2,3)** → Z14 and Z23  
  - For each valid OS pair, it constructs Z-candidate 4-vector components:
    - energy: `eZij = E_i + E_j` with `E = sqrt(p^2 + m_μ^2)`
    - momentum sums: `pxZij, pyZij, pzZij`
    - magnitudes: `pZij`, `pTZij`
    - invariant mass: `mZij = sqrt(eZij^2 - pZij^2)`
  - Fills pairing-mass histograms when `mZij > 0`:
    - `h_mZ12_4mu`, `h_mZ34_4mu`, …, `h_mZ14_4mu`, `h_mZ23_4mu`

- **Choose the best Z candidate (closest to nominal Z mass)**
  - Computes distance-to-Z for each dimuon mass:
    - `dZij = |mZij - mZ|`
  - For each pairing combination, keeps the better of the two Z distances:
    - `dZc1 = min(dZ12, dZ34)`
    - `dZc2 = min(dZ13, dZ24)`
    - `dZc3 = min(dZ14, dZ23)`
  - Selects the combination with smallest `dZc` (closest-to-Z pairing overall).
  - Within the chosen combination, assigns:
    - `Za` = the dimuon mass closer to mZ
    - `Zb` = the other dimuon mass
  - Also checks a **leading-pT requirement** for the chosen Za pair:
    - `ptZadaug = true` if the two muons forming Za satisfy:
      - leading muon pT > 20 GeV **and** subleading muon pT > 10 GeV  
    - (implemented by checking the relevant muon pair depending on which combination won)

- **Apply Z mass windows (analysis cuts)**
  - Only proceeds if `ptZadaug` is satisfied and Z masses lie in windows:
    - `40 < mZa < 120`  (Z1-like candidate)
    - `12 < mZb < 120`  (Z2-like candidate; allows off-shell Z*)

- **Fill chosen Z masses and build the 4μ candidate**
  - Fills the chosen-Z histograms:
    - `h_mZa_4mu->Fill(mZa)`
    - `h_mZb_4mu->Fill(mZb)`
  - Builds Lorentz vectors:
    - `p4Za.SetPxPyPzE(pxZa, pyZa, pzZa, eZa)`
    - `p4Zb.SetPxPyPzE(pxZb, pyZb, pzZb, eZb)`
    - `p4H = p4Za + p4Zb`
  - Extracts 4μ candidate observables:
    - `mass4mu = p4H.M()`, plus `pT/eta/phi` and components.

- **Record per-muon kinematics (for ntuple/diagnostics)**
  - Saves per-muon pT/η/φ, charge, momentum components, and energies.

- **Fill 4μ invariant-mass spectra**
  - Applies a final mass threshold:
    - `if (mass4mu > 70.)`
  - Fills the 4μ mass into multiple histograms (different binnings/ranges):
    - `h_m1_m4mu`, `h_m2_m4mu`, `h_m3_m4mu`, `h_m4_m4mu`

- **After-selection control plots (for events passing the 4μ candidate cuts)**
  - For each selected muon in `vIdPtmu`, recomputes PF relative isolation and fills:
    - `h_relPFIso_mu_after`
  - Also fills “after” muon kinematics:
    - `h_pt_after`, `h_eta_after`

### 20) `analyze()`: ZZ/ZZ*→4e reconstruction and 4e mass spectrum (lines 1738–2080)

- **Purpose of this block (core 4e channel)**
  - If at least **4 good electrons** are available, builds **ZZ/ZZ*→4e** candidates:
    - forms opposite-sign (OS) electron pairings,
    - chooses the best Z candidate (closest to nominal mZ),
    - applies Z-mass windows + leading-lepton pT requirements,
    - constructs the 4e candidate and fills the **4e invariant-mass spectra**.

- **Pick the 4 leading-pT selected electrons**
  - Requires at least 4 selected electrons:
    - `if (nGoodElectron >= 4)`
  - Takes the top-4 pT electrons from the sorted list `vIdPte`:
    - `elec1, elec2, elec3, elec4`

- **Total charge requirement**
  - Requires the 4-electron system to be neutral:
    - `elec1.charge() + elec2.charge() + elec3.charge() + elec4.charge() == 0`

- **Compute the three OS pairing combinations (for 4e)**
  - The code tries three distinct pairing patterns (each with two OS dielectron pairs):
    1) **(1,2) + (3,4)** → Z12 and Z34  
    2) **(1,3) + (2,4)** → Z13 and Z24  
    3) **(1,4) + (2,3)** → Z14 and Z23  
  - For each valid OS pair, it constructs Z-candidate 4-vector components:
    - energy: `eZij = E_i + E_j` with `E = sqrt(p^2 + m_e^2)`
    - momentum sums: `pxZij, pyZij, pzZij`
    - magnitudes: `pZij`, `pTZij`
    - invariant mass: `mZij = sqrt(eZij^2 - pZij^2)`
  - Fills pairing-mass histograms when `mZij > 0`:
    - `h_mZ12_4e`, `h_mZ34_4e`, …, `h_mZ14_4e`, `h_mZ23_4e`

- **Choose the best Z candidate (closest to nominal Z mass)**
  - Computes distance-to-Z for each dielectron mass:
    - `dZij = |mZij - mZ|`
  - For each pairing combination, keeps the better of the two Z distances:
    - `dZc1 = min(dZ12, dZ34)`
    - `dZc2 = min(dZ13, dZ24)`
    - `dZc3 = min(dZ14, dZ23)`
  - Selects the combination with smallest `dZc` (closest-to-Z pairing overall).
  - Within the chosen combination, assigns:
    - `Za` = the dielectron mass closer to mZ
    - `Zb` = the other dielectron mass
  - Also checks a **leading-pT requirement** for the chosen Za pair:
    - `ptZadaug = true` if the two electrons forming Za satisfy:
      - leading electron pT > 20 GeV **and** subleading electron pT > 10 GeV  
    - (implemented by checking the relevant electron pair depending on which combination won)

- **Apply Z mass windows (analysis cuts)**
  - Only proceeds if `ptZadaug` is satisfied and Z masses lie in windows:
    - `40 < mZa < 120`
    - `12 < mZb < 120` (allows off-shell Z*)

- **Fill chosen Z masses and build the 4e candidate**
  - Fills the chosen-Z histograms:
    - `h_mZa_4e->Fill(mZa)`
    - `h_mZb_4e->Fill(mZb)`
  - Builds Lorentz vectors and forms the 4e candidate:
    - `p4Za.SetPxPyPzE(pxZa, pyZa, pzZa, eZa)`
    - `p4Zb.SetPxPyPzE(pxZb, pyZb, pzZb, eZb)`
    - `p4H = p4Za + p4Zb`
  - Extracts 4e candidate observables:
    - `mass4e = p4H.M()` plus `pT/eta/phi` and components.

- **Record per-electron kinematics (for ntuple/diagnostics)**
  - Saves per-electron pT/η/φ, charge, momentum components, and energies.

- **Fill 4e invariant-mass spectra**
  - Applies a final mass threshold:
    - `if (mass4e > 70.)`
  - Fills the 4e mass into multiple histograms (different binnings/ranges):
    - `h_m1_m4e`, `h_m2_m4e`, `h_m3_m4e`, `h_m4_m4e`

- **After-selection control plots (for events passing the 4e candidate cuts)**
  - For each selected electron in `vIdPte`, recomputes PF relative isolation and fills:
    - `h_relPFIso_e_after`
  - Also fills “after” electron kinematics:
    - `h_pt_e_after`
    - `h_eta_e_after` (uses **supercluster η**)
    
### 21) `analyze()`: ZZ/ZZ*→2μ2e reconstruction and 2μ2e mass spectrum (lines 2083–2274)

- **Purpose of this block (core 2μ2e channel)**
  - If at least **2 good muons** and **2 good electrons** are available, builds a **ZZ/ZZ*→2μ2e** candidate:
    - constructs Z(μμ) and Z(ee),
    - decides which one is `Za` (closest to nominal mZ),
    - applies Z-mass windows + leading-lepton pT requirements,
    - forms the 4ℓ candidate and fills the **2μ2e invariant-mass spectra**.

- **Select the leading leptons**
  - Requires:
    - `nGoodRecoMuon >= 2` and `nGoodElectron >= 2`
  - Uses the leading-pT muons and electrons (vectors are pre-sorted by pT):
    - `muon1, muon2` from `vIdPtmu`
    - `elec1, elec2` from `vIdPte`

- **Total charge requirement**
  - Requires the 2μ2e system to be neutral:
    - `muon1.charge() + muon2.charge() + elec1.charge() + elec2.charge() == 0`

- **Build Z(μμ) and Z(ee) (only one pairing structure)**
  - For 2μ2e there is only one natural pairing:
    - Z12 from the two muons, and Z34 from the two electrons
  - Requires each pair to be opposite-sign:
    - `muon1.charge() + muon2.charge() == 0`
    - `elec1.charge() + elec2.charge() == 0`
  - Computes energies, summed momenta, transverse momenta, and invariant masses:
    - `mZ12` (μμ) and `mZ34` (ee)
  - Fills per-pair Z histograms:
    - `h_mZmu_2mu2e` (μμ Z mass)
    - `h_mZe_2mu2e` (ee Z mass)

- **Choose Za/Zb by closeness to mZ**
  - Computes:
    - `dZ12 = |mZ12 - mZ|` (μμ)
    - `dZ34 = |mZ34 - mZ|` (ee)
  - Assigns:
    - if `dZ12 < dZ34`: `Za = Z(μμ)`, `Zb = Z(ee)`
    - else: `Za = Z(ee)`, `Zb = Z(μμ)`

- **Leading-pT requirement for Za leptons**
  - Applies the same “pT(leading)>20, pT(subleading)>10” requirement, but on whichever pair became `Za`:
    - if `Za` is μμ: check `muon1.pt()>20 && muon2.pt()>10`
    - if `Za` is ee: check `elec1.pt()>20 && elec2.pt()>10`

- **Apply Z mass windows (analysis cuts)**
  - Only proceeds if `ptZadaug` is satisfied and Z masses lie in windows:
    - `40 < mZa < 120`
    - `12 < mZb < 120` (allows off-shell Z*)

- **Fill chosen Z masses and build the 2μ2e candidate**
  - Fills the chosen-Z histograms:
    - `h_mZa_2mu2e->Fill(mZa)`
    - `h_mZb_2mu2e->Fill(mZb)`
  - Builds Lorentz vectors and forms the 4ℓ candidate:
    - `p4Za.SetPxPyPzE(pxZa, pyZa, pzZa, eZa)`
    - `p4Zb.SetPxPyPzE(pxZb, pyZb, pzZb, eZb)`
    - `p4H = p4Za + p4Zb`
  - Extracts candidate observables:
    - `mass2mu2e = p4H.M()`, plus `pT/eta/phi` and components.

- **Record per-lepton kinematics (for ntuple/diagnostics)**
  - Saves pT/η/φ, charge, momentum components, and energies for:
    - the two muons and the two electrons.

- **Fill 2μ2e invariant-mass spectra**
  - Applies a final mass threshold:
    - `if (mass2mu2e > 70.)`
  - Fills the 2μ2e mass into multiple histograms:
    - `h_m1_m2mu2e`, `h_m2_m2mu2e`, `h_m3_m2mu2e`, `h_m4_m2mu2e`

- **After-selection control plots (for events passing the 2μ2e candidate cuts)**
  - Recomputes and fills isolation + kinematics after the final selection:
    - for muons:
      - `h_relPFIso_2mu_after`, `h_pt_after_2mu2e`, `h_eta_after_2mu2e`
    - for electrons:
      - `h_relPFIso_2e_after`, `h_pt_e_after_2mu2e`, `h_eta_e_after_2mu2e`

### 22) `beginJob()`: job-start hook (ntuple booking placeholder) (lines 2277–2286)

- **One-time job initialisation hook**
  - Defines `void HiggsDemoAnalyzerGit::beginJob()`, which is called **once per job** before the event loop starts.

- **Intended purpose (as commented)**
  - The comment states the intention to **book an ntuple** for “surviving” 4-lepton candidates in the mass window:
    - **70 < m4ℓ < 181 GeV**
  - This corresponds to a typical “Higgs signal region” style window used for educational/analysis output.

- **Note**
  - In the shown snippet, the ntuple booking is only indicated by comments (the actual `TTree` booking/filling appears commented out elsewhere as `t1/t2/t3`).

### 23) `beginJob()`: ntuple (TTree) definitions for surviving 4ℓ candidates (lines 2287–2460)

- **Create three TTrees (one per final state)**
  - `t1 = new TTree("tree4mu", "tree4mu");`  (4μ candidates)
  - `t2 = new TTree("tree4e", "tree4e");`    (4e candidates)
  - `t3 = new TTree("tree2mu2e", "tree2mu2e");` (2μ2e candidates)
  - Each tree stores event-level identifiers plus reconstructed kinematics for the selected 4ℓ candidate.

- **Common event identifiers (stored in all trees)**
  - `nRun`, `nEvt`, `nLumi`  
  Used to trace candidates back to the original event/run/lumisection.

- **Tree `tree4mu` (t1): 4μ candidate content**
  - **4μ candidate kinematics**
    - `mass4mu`, `pt_4mu`, `eta_4mu`, `phi_4mu`
    - 4-vector components: `px4mu`, `py4mu`, `pz4mu`, `E4mu`
  - **Per-muon kinematics (μ1…μ4)**
    - `pt_mu1…pt_mu4`, `eta_mu1…eta_mu4`, `phi_mu1…phi_mu4`
    - charges: `cas_mu1…cas_mu4`
    - momentum components: `px_mu1…px_mu4`, `py_mu1…py_mu4`, `pz_mu1…pz_mu4`
    - energies: `E_mu1…E_mu4`
  - **Chosen Z candidates**
    - `mZa`, `mZb` (the selected Z1/Z2 masses used to form the 4μ candidate)

- **Tree `tree4e` (t2): 4e candidate content**
  - **4e candidate kinematics**
    - `mass4e`, `pt_4e`, `eta_4e`, `phi_4e`
    - 4-vector components: `px4e`, `py4e`, `pz4e`, `E4e`
  - **Per-electron kinematics (e1…e4)**
    - `pt_e1…pt_e4`, `eta_e1…eta_e4`, `phi_e1…phi_e4`
    - charges: `cas_e1…cas_e4`
    - momentum components: `px_e1…px_e4`, `py_e1…py_e4`, `pz_e1…pz_e4`
    - energies: `E_e1…E_e4`
  - **Chosen Z candidates**
    - `mZa`, `mZb`

- **Tree `tree2mu2e` (t3): 2μ2e candidate content**
  - **2μ2e candidate kinematics**
    - `mass2mu2e`, `pt_2mu2e`, `eta_2mu2e`, `phi_2mu2e`
    - 4-vector components: `px2mu2e`, `py2mu2e`, `pz2mu2e`, `E2mu2e`
  - **Per-lepton kinematics**
    - muons: `pt_2mu1`, `pt_2mu2`, `eta_2mu1`, `eta_2mu2`, `phi_2mu1`, `phi_2mu2`, `cas_2mu1`, `cas_2mu2`
    - electrons: `pt_2e1`, `pt_2e2`, `eta_2e1`, `eta_2e2`, `phi_2e1`, `phi_2e2`, `cas_2e1`, `cas_2e2`
    - momentum components: `px_*`, `py_*`, `pz_*`
    - energies: `E_2mu1`, `E_2mu2`, `E_2e1`, `E_2e2`
  - **Chosen Z candidates**
    - `mZa`, `mZb`

- **Important note (current status in your file)**
  - In your snippet, the entire TTree booking block is inside a `/* ... */` comment, meaning it is currently **disabled**.
  - In `analyze()`, the corresponding `t1->Fill() / t2->Fill() / t3->Fill()` calls are also commented out, consistent with the ntuple being optional.

### 24) `endJob()` and module registration (lines 2462–2467)

- **`endJob()` (job-end hook)**
  - Defines `void HiggsDemoAnalyzerGit::endJob()`, which is called **once after the event loop finishes**.
  - In this implementation it is empty (no post-processing is done here).  
    Typical uses would include printing summaries, final counters, or cleanup (if needed).

- **CMSSW plugin registration**
  - `DEFINE_FWK_MODULE(HiggsDemoAnalyzerGit);` registers this analyzer as a CMSSW plugin module so it can be loaded and run via a CMSSW configuration file.
