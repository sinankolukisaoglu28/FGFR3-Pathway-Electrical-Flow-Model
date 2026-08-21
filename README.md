# FGFR3-Pathway-Electrical-Flow-Model
Adopting a Molecular Pathological Epidemiology perspective, we present a probabilistic in silico Digital Twin of the FGFR3 pathway. Bridging pathologic epidemiological data with micro-cellular multi-omics, this framework establishes a dynamic, predictive modeling foundation for uro-oncology to execute virtual experiments.

# FGFR3 Signaling Cascade Model

A mechanistic ODE (ordinary differential equation) model of FGFR3 receptor
signaling, covering the four canonical downstream branches — **MAPK/ERK**,
**PI3K/AKT**, **PLCγ/IP3/Ca²⁺**, and **JAK-STAT** — together with the major
negative-feedback loops that shape their dynamics, and a pharmacological
inhibition layer used to simulate drug-induced pathway blockade and
"bypass"/rerouting behavior.

## Contents

- `FGFR3_Model_English.py` — the model itself (parameters, ODE system,
  simulation driver, and plotting code). Fully self-contained; running it
  reproduces the baseline (unperturbed) simulation.

## Requirements

```
numpy
scipy
pandas
matplotlib
```

Install with:
```bash
pip install numpy scipy pandas matplotlib
```

## Running

```bash
python3 FGFR3_Model_English.py
```

This will:
1. Integrate the model from t = 0 to t = 1500 s (LSODA, `rtol=1e-6`, `atol=1e-9`).
2. Save the full time-course of all 28 state variables to `fgfr3_results_revised.csv`.
3. Save a 12-panel summary figure to `fgfr3_revised_model.png`.
4. Print a steady-state summary (mean of t = 1400–1500 s) for the main pathway
   readouts to the terminal.

## Model Structure

The model integrates **28 coupled ODEs**, organized into six modules:

| # | Module | Key species | Feedback included |
|---|---|---|---|
| 1 | Receptor activation & Cbl-mediated endocytosis | `R`, `LR`, `D`, `pCbl` | Dynamic, ligand-dependent internalization |
| 2 | MAPK/ERK cascade (FRS2→GRB2→SOS1→RAS→RAF→MEK→ERK) | `pFRS2`, `SHP2p`, `RAS_GTP`, `pRAF`, `ppMEK`, `ERK/pERK/ppERK` | SHP2 positive amplification; Sprouty negative feedback |
| 3 | PLCγ/IP3/Ca²⁺ (corrected Li-Rinzel formalism) | `PLCg_a`, `IP3`, `Ca_i`, `h`, `DAG`, `pPKC` | CICR (Ca-induced Ca inactivation) via the `h` gate |
| 4 | PI3K/AKT (GAB1-mediated) | `pGAB1`, `pPI3K`, `PIP3`, `pAKT` | ERK→GAB1 inhibitory phosphorylation (bypass mechanism) |
| 5 | Sprouty | `Spry` | ppERK-driven fast phosphorylation (no transcriptional delay) |
| 6 | JAK-STAT | `STAT1p`, `STAT3p`, `STAT5p`, `STATnuc`, `SOCS` | SOCS negative feedback; secondary ERK→SOCS contribution |

A shared GRB2 pool is split competitively between the MAPK arm (GRB2-SOS1,
60%) and the PI3K arm (GRB2-GAB1, 40%), coupling the two branches at the
receptor-proximal level.

## Pharmacological Inhibition

Two dimensionless inhibitor parameters are built into the model:

- **`I_MEK`** (0–1): fractional block of MEK's catalytic output (models
  allosteric, non-ATP-competitive MEK inhibitors such as trametinib). Applied
  to the ERK-phosphorylation rate terms, not to RAF→MEK phosphorylation
  itself.
- **`I_PI3K`** (0–1): fractional block of PI3K's lipid-kinase catalytic
  output (models inhibitors such as alpelisib/BKM120). Applied to PIP3
  production, not to GAB1→PI3K binding.

Both default to `0.0` (no inhibition) in this file. To simulate a drug
condition, copy `params`, set `I_MEK` and/or `I_PI3K` to a value in [0, 1],
and re-run `solve_ivp` with the modified parameter dictionary.

### Bypass/crosstalk mechanisms

Two literature-motivated crosstalk terms are included so that blocking one
arm can produce a genuine compensatory response in another:

1. **ERK → GAB1 inhibitory phosphorylation** (strong evidence: Yu et al. 2002,
   *J Biol Chem*; Lehr et al. 2004, *J Cell Biol*). When ERK is blocked, this
   suppression is relieved and PI3K/AKT activity increases compensatorily.
2. **ERK → SOCS induction**, secondary to the main STAT3-driven term
   (moderate/weak evidence; weighted by `w_ERK_SOCS`, which can be set to `0`
   to disable it for sensitivity analysis).

The PLCγ/Ca²⁺/DAG/PKC module is **deliberately left isolated** from
MAPK/PI3K inhibition — no well-documented literature mechanism linking RAS/
RAF/MEK/ERK or PI3K/AKT state to PLCγ activation was found, so this branch
does not (and should not, without further evidence) respond to `I_MEK`/`I_PI3K`.

## Key Modeling Conventions

- Most activation steps follow a **bimolecular on/off** form:
  `dX/dt = k_on · Signal · (X_tot − X) − k_off · X`.
- Saturating catalytic steps use **Michaelis-Menten** kinetics:
  `v = Vmax · S / (Km + S)`.
- The Ca²⁺/IP3R module uses the **Li-Rinzel formalism**, a Hodgkin-Huxley-like
  gating-variable model (steady-state activation functions `m_inf`, `n_inf`,
  and a dynamic inactivation gate `h`).
- Concentrations are in µM, time in seconds, all pool sizes calibrated to
  physiologically realistic ranges (low µM), not the inflated placeholder
  values used in early drafts of this model.

## Known Limitations

- Single-cell, autonomous ODE system — no paracrine/autocrine signaling
  (e.g. IL-6-driven JAK-STAT reactivation) or tumor microenvironment effects.
- The GRB2 SOS1/GAB1 split (60/40) is fixed and does not itself respond to
  pathway inhibition; only the two explicit crosstalk terms above do.
- Inhibitor doses (`I_MEK`, `I_PI3K`) are abstract fractional-block
  parameters, not calibrated to specific drug pharmacokinetics/IC50 values.

## References

Li, R., Linscott, J., Catto, J., Daneshmand, S., Faltas, B., Kamat, A., Meeks, J., Necchi, A., Pradère, B., Ross, J. S., Van Der Heijden, M. S., Van Rhijn, B. V., & Loriot, Y. (2024). FGFR Inhibition in Urothelial Carcinoma.. European urology. 
Pietzak EJ, Bagrodia A, Cha EK, Drill EN, Iyer G, Isharwal S, Ostrovnaya I, Baez P, Li Q, Berger MF, Zehir A, Schultz N, Rosenberg JE, Bajorin DF, Dalbagni G, Al-Ahmadie H, Solit DB, Bochner BH. Next-generation Sequencing of Nonmuscle Invasive Bladder Cancer Reveals Potential Biomarkers and Rational Therapeutic Targets. Eur Urol. 2017 Dec;72(6):952-959.
C., & Gurkan-Cavusoglu, E. (2024). A comprehensive review of computational cell cycle models in guiding cancer treatment strategies. NPJ Systems Biology and Applications, 10.
Hamada, T., Keum, N., Nishihara, R., & Ogino, S. (2016). Molecular pathological epidemiology: new developing frontiers of big data science to study etiologies and pathogenesis. Journal of gastroenterology, 52, 265 - 275. 

  
