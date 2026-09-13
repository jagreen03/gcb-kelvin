# Paste this whole file into any AI chat to get help on this problem

You don't need repo access to use this — everything needed is inlined
below. This is one isolated problem from a larger hardware program
(GreenCoyote-Bay, a photonic/electronic AI-inference accelerator); you
are being asked to help with exactly this one piece, nothing more.

## The problem, in one sentence

An analog comparator's noise number has a measurement uncertainty band
that straddles its own pass/fail gate, and the one tool that would
resolve it (a commercial circuit simulator with a native transistor
noise model) isn't available to this program — **is there any real way
to shrink that uncertainty with open-source tools, or is this
genuinely stuck until someone gets Spectre access?**

## The real numbers (all measured or simulated, tagged where noted)

**Design point (fixed, not up for debate):**
- Photocurrent: 0.74 µA RMS, crest factor 6, full-scale 4.44 µA.
- Integration window: 256 ns. Charge: `Q_FS` = 1.137 pC.
- Integrator: a bare capacitor, `C_int` = 2.0 pF, discharge mode (no
  amplifier in the accumulate path — `V = Q/C` only). `V_FS` = 568 mV,
  0.8 V supply, ~110 mV headroom to the rail. **This capacitor value is
  pinned by a hard physical bound** — below 1.5 pF the ramp would need
  0.93 V on a 0.8 V rail (non-physical); above 2.0 pF costs ~2 dB SNDR
  per extra pF. There is no design freedom here.
- Comparator: 2-stage chopped + auto-zeroed preamp, `I_tail` = 30
  µA/stage, `C_L` = 0.8 pF, `F_chop` = 20 MHz, 20 ns gated decision
  window. 30 µA was chosen specifically because it's the *quietest*
  tail current that still resolves in time — 15 µA is quieter (~39 µV)
  but its ~40 ns settling time exceeds the ~30 ns ramp window.
- SNDR target: ≥56 dB at p05 (5th percentile across PVT/MC), evaluated
  at 9 PVT corners.

**The noise number and its problem:**
- **[Measured, BSIM4 `.noise` surrogate]** Total input-referred
  comparator noise: **53 µV**. Gate: **≤ 55 µV**.
- **Uncertainty band: ±16%, i.e. 44.5–61.5 µV** — this straddles the 55
  µV gate. The true number could already be a clean pass or an actual
  fail; nobody knows which.
- Readout SNDR crosses below the 56 dB target only once noise exceeds
  **~72 µV** — so there's about 17 µV of real headroom above the
  comparator gate before the system-level spec actually fails. The 55
  µV gate itself is conservative, not the hard physical limit.
- **[Derived, analytic roll-up from SPICE operating points]** A
  cross-check using measured op-points plus literature noise proxies
  gives **45.8 µV** total (thermal 44.2 µV + residual 1/f 12.1 µV after
  chopping) — a *lower-bound* cross-check, not the basis for the gate
  decision, which rests on the 53 µV BSIM4 number.

**Why the uncertainty exists — the real toolchain limit, already
surveyed, don't re-attempt these:**
- The real target device is an ASAP7 (7nm-class, predictive/academic)
  BSIM-CMG (FinFET) transistor model, simulated via `ngspice` with a
  custom OSDI (Open Source Device Interface) build.
- **`ngspice`'s OSDI `.noise` analysis returns literally zero** on this
  device — confirmed, not a configuration error.
- **`ngspice`'s `trnoise` (transient noise) source is not reproducible**
  on this setup: a `kT/C` reference case expected to read 64.4 µV
  instead returned 96 µV, 3.2 mV, and 3.9 mV at three different
  simulation timesteps on the *same* circuit — a ~50× spread, making
  transient-noise injection useless here.
- **Xyce could not be built without root access**: the conda-forge
  Trilinos package (a dependency) is missing `NOX_ENABLE_LOCA`,
  `EpetraExt_BUILD_BTF`, and Isorropia/Zoltan build targets. A
  from-source Trilinos build was judged out of scope (it pulls in
  hours of additional build work).
- **What does work**: `ngspice`'s `.noise` analysis against its own
  **built-in BSIM4** (a *different*, older, well-supported model) —
  gm-matched as a surrogate for the real BSIM-CMG device (`γ_eff ≈
  1.4`). This produces a real device-physics noise integral, not a
  fabricated number — but it's a proxy for the real device, not the
  real device, hence the ±16% band.
- **ASAP7 itself ships no flicker-noise or statistical/mismatch
  model** — the 1/f parameters used (`K_Fn = 5×10⁻¹¹ V²·µm²`, `A_VT =
  2.0 mV·µm`) are assumed published-FinFET literature proxies, not
  measured for this specific PDK.

**The fallback already on file, if the real number comes back bad:** a
gm–C integrating first-stage preamp — gain set by `g_m · t_gate /
C_int,pre`, independent of the 0.8 V supply headroom that causes the
noise/current tradeoff inversion in the current design (more tail
current paradoxically *increases* input-referred noise here, because
the resistor loads are pinned to a fixed voltage drop, so `R_L ∝
1/I_tail` and bandwidth grows faster than gain). Documented, never
built — only needed if the real noise number exceeds 55 µV.

## The actual question to help with

**Is there any real, honest way to narrow the ±16% BSIM4-surrogate
uncertainty band using open-source tools** (not Spectre, not any
licensed commercial simulator), given everything already tried above?
Some angles worth exploring, none pre-validated:
- Is there a way to get a usable noise estimate from a *different*
  open noise-analysis method against the real BSIM-CMG model (not
  `.noise`, not `trnoise`) — e.g., a Monte-Carlo ensemble of DC/AC
  small-signal runs combined analytically?
- Is the ~50× `trnoise` spread itself diagnosable/fixable (a known
  `ngspice`+OSDI integration bug, a seeding issue, a time-step
  artifact) rather than a fundamental limitation?
- Is there a *better* BSIM4-to-BSIM-CMG gm-matching procedure than
  whatever produced `γ_eff ≈ 1.4`, that would plausibly tighten the
  uncertainty band even without a different simulator?
- Or: is the honest answer "no, this genuinely needs a commercial
  tool, stop trying to work around it with open-source substitutes"?
  That's a completely acceptable answer — don't force a workaround
  that isn't real.

**How to work this problem (the house style for this program):** every
number needs a tag — measured, derived, or an assumed literature
proxy. Do not report a workaround as resolving the uncertainty unless
you can state exactly what new, different noise-physics it captures
that the existing BSIM4 surrogate doesn't.
