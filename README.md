# Detection of Quantum Phase Transitions in the ANNNI (Frustrated TFIM) Model

A reproduction and extension, in Qiskit, of the VQE methodology from:

> K. Lively, T. Bode, J. Szangolies, J.-X. Zhu, B. Fauseweh,
> "Noise robust detection of quantum phase transitions,"
> *Phys. Rev. Research* **6**, 043254 (2024).
> https://doi.org/10.1103/PhysRevResearch.6.043254

**This is a reproduction/extension study, not novel research.** The physical
model, the phase-transition detection method (Hellmann-Feynman derivative
of the energy), and the overall experimental logic (classically-optimized
parameters, executed once on noisy hardware) all belong to the paper above.
Attempt to implement on real hardware failed due to failing IBM account verification (their end)  and no funds available to buy access plan.

## What's here

| File | What it does | Depends on |
|---|---|---|
| `ANNNI_Ideal.py` | Main entry point. Sweeps J2/J1 = 0.40-0.60, optimizing a VQE ansatz against a noiseless statevector simulator for each point. Produces the "ideal simulation" reference data. | Nothing (run first) |
| `ANNNI_Noisy.py` | Takes the *already-optimized* parameters from the file above, binds them as fixed circuits, and measures them under a realistic IBM device noise model (no re-optimization see "Why no optimization on hardware" below). | Output of `ANNNI_Ideal.py` |

Scripts must be run **in that order** each one after the first reads a
`.npz` file the previous one writes. This isn't automatic just because the
files are in one repository; see "Running on your own machine" below.

### Data files included

- `annni_N12_h0.1_periodic_reps2_ideal.npz`  ideal (noiseless)
  simulation results: energy, dE/dJ2, and the full optimized parameter set
  for all 21 points, against both VQE and exact diagonalization.
- `annni_N12_h0.1_periodic_reps2_noisy.npz`  the same 21
  points executed under simulated IBM device noise (FakeSherbrooke).
- `annni_three_way_comparison.png`
- ` annni_scan_ideal.png`

These are included so the results can be inspected or replotted without
re-running.

## Findings


1. **Warm-starting across a phase transition can silently converge to the
   wrong state.** Continuing a J2-sweep's optimizer from the previous
   point's parameters works well *within* a phase, but can get trapped
   describing the old phase's local minimum after crossing a transition.

2. **Optimizer tolerances can cause silent zero-iteration "convergence."**
   With loose tolerances, L-BFGS-B can accept a warm-started point whose
   gradient already looks small for the *new* Hamiltonian and terminate
   in `nit=0` real steps silently reusing the previous point's answer
   rather than re-optimizing. Confirmed by instrumenting the optimizer
   directly; fixed by tightening convergence tolerances.

## Why no optimization on real/noisy hardware

`ANNNI_Noisy.py` and `annni_hardware_execution.py`(N/A) never
re-optimize circuit parameters  they only measure fixed, already-optimal
ones.  reason:

- The exact-gradient method used for ideal-simulation optimization
  (`ReverseEstimatorGradient`) works by inspecting the classical
  simulator's internal statevector  a real, noisy-simulated backend
  never exposes this, so it cannot be used outside noiseless simulation.

This mirrors the parent paper's own methodology: all parameter
optimization is classical and offline; hardware is used only to execute
and measure.

## Setup

```bash
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
```
Tested against the exact versions pinned in `requirements.txt`; Qiskit's
API has changed across versions before (e.g. `EfficientSU2` is deprecated
as of Qiskit 2.1 in favor of the `efficient_su2` function), so an
unpinned `pip install qiskit` some months from now may not run this
code unmodified.

## Running on your own machine

```bash
python ANNNI_Ideal.py      # ~20-25 min; produces the _exactgrad.npz file
python ANNNI_Noisy.py       # ~25-30 min; noisy simulation is slow per point.
                                    #   Checkpointed  if interrupted, run again to resume.
python plot_three_way.py           # seconds; produces annni_three_way_comparison.png (this is the last cell of ANNNI_Noisy.py)
```
Run all three from the same directory — they use relative filenames, no
path configuration needed.


**Status:** real-hardware execution is implemented and calibration-tested
in simulation, but not yet run against real hardware  pending IBM Open
Plan account verification.

## Repository does not include
Note: This reproduction covers ground-state energy and its derivative (dE/dJ2) only.

`annni_hardware_execution.py` (failed IBM account verififcation )
Earlier, superseded versions of the sweep script (single-blind-reset and
targeted-reset-point variants, both replaced by the multi-start strategy
in `ANNNI_Ideal.py` for the reasons in "Findings" above) and an
earlier plotting script (`plot_annni_scan.py` v1, superseded by
`plot_three_way.py`) were intentionally left out to keep the repository
to its final, working state rather than its debugging history. Also It does not include TFIM 1D VQE nor the similar study of frustrated TFIM for N=4, 8, 12 ,16  using QuSpin. 

These frustrated Ising-type models have been more successfully studied on trapped-ion systems, given their all-to-all connectivity better suits the model's interactions, one such example ([Kirmani et al., 2025](https://arxiv.org/abs/2505.22932)). Anyone looking to work in this direction would benefit from exploring those modalities. Open to discussion and comments.