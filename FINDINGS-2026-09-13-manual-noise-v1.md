# Findings, 2026-09-13 — manual noise-injection path, first step: single-transistor analytical estimate

**Continues `FINDINGS-2026-09-13.md`'s recommended next step** (the
manual multi-transistor noise-injection model, using the real
device-extracted `gm`/`gds` instead of a BSIM4 surrogate). That file
explicitly stopped before any calculation — this pass does the first,
smallest real piece of it: **one input transistor, analytically, not
yet the full 2-stage chopped preamp + StrongARM latch.** Tagged
throughout per this repo's `[M]`/`[D]`/`[A]` convention.

**Bottom line up front: this does NOT close the core question, does
NOT constitute a GO, and does not override Hard Rule 3** (no GO except
from a real commercial-simulator run). It is a bounded analytical
sanity check on whether the manual-noise-injection *approach itself* is
even plausible before investing in the full multi-transistor build —
and it surfaces a real, unresolved ambiguity worth flagging rather than
picking silently.

## What was computed

Using the real, device-extracted small-signal parameters from
`FINDINGS-2026-09-13.md` (`[M]`, same `bsimcmg.osdi` device, same bias
point):

```
gm  = 7.792610e-4 S
gds = 3.372561e-05 S
```

and the real operating-point values from `ai-agents/ORANGE_TEAM.md` /
`AGENT_02_ANALOG_EVALUATOR.md` (`[citing existing doc]`):

```
C_L    = 0.8 pF   (comparator preamp load)
T_gate = 20 ns    (gated decision window)
F_chop = 20 MHz
```

Two classical single-transistor thermal-noise models were evaluated,
because they disagree sharply on which bandwidth actually limits the
integrated noise for this specific circuit, and nothing in this
program's evidence set states which applies:

**Model A — pole-limited.** Standard single-stage common-source result:
input-referred noise PSD `S_v = 4kTγ/gm`, output pole at
`f_3dB = gm/(2π·C_L)`, giving the classic `v_n,rms² = kTγ/C_L` (gm
cancels algebraically — this model doesn't actually need the extracted
gm at all, only C_L). `[D]erived, standard textbook result, not
device-specific]`

**Model B — gate-window-limited.** If the 20 ns gated decision window
is the tighter bandwidth constraint (a boxcar/rectangular integration
of width T has equivalent noise bandwidth ≈ 1/(2T)), the same PSD
integrates over that narrower band instead, and the real extracted
`gm` matters directly. `[D]erived]`

### Results (γ = transistor excess-noise factor; ASAP7 has no flicker
model per `AGENT_02`'s own flagged gap, so γ is `[A]ssumed` in every
case, not measured)

| Model | γ = 2/3 (long-channel) | γ = 1.0 | γ implied to hit 53 µV exactly |
|---|---|---|---|
| A (pole-limited, `f_3dB` ≈ 155 MHz) | **58.75 µV** | 71.95 µV | γ ≈ 0.543 |
| B (gate-limited, ENBW ≈ 25 MHz) | 18.82 µV | 23.05 µV | γ ≈ 5.285 |

`[M]easured` inputs, `[D]erived` arithmetic — computed and re-checked
via a standalone script before being written up here, not hand-typed
into the table.

## Reading the result honestly

**Model A lands close to the 53 µV surrogate target at a physically
ordinary γ** (58.75 µV at γ=2/3 is +10.8% — comfortably inside the
±20% band `ORANGE_TEAM.md` defines as "confirms the surrogate";
γ≈0.543 hits it exactly, and 0.543 is not an unusual value for a
short-channel device). **Model B requires γ≈5.3 to match — well outside
any physically ordinary range for γ** (typically 0.5–3 even for
aggressively short-channel devices; ASAP7 is 7nm-class, so somewhat
elevated γ is plausible, but not 5×). **This is a real signal, not
noise in the calculation**: it suggests the pole-limited model is the
more physically appropriate one for this device/load combination, and
that the 20 ns gate window is *not* the bottleneck for this particular
transistor's own noise contribution — likely because the transistor's
intrinsic bandwidth (155 MHz, τ≈1 ns) is so much faster than the gate
(20 ns) that the gate mostly just samples an already-settled, already
band-limited-by-the-pole noise process rather than acting as its own
independent bandwidth limiter.

**This is encouraging for the manual-noise-injection approach's
plausibility** — a real, device-derived single-transistor estimate
lands in the right neighborhood without any curve-fitting to the
target — but it is **explicitly not a closure**:

1. **Only one transistor.** The real circuit is a 2-stage chopped/
   auto-zeroed preamp feeding a StrongARM latch. Stage-to-stage gain,
   input-referred noise combination across stages, the chopper's own
   up/down-modulation of any 1/f content, and the latch's regeneration
   noise are all **not modeled here at all** — exactly the gap
   `FINDINGS-2026-09-13.md` already flagged as "a real, substantial
   circuit-design task, not attempted."
2. **γ is still assumed, not measured**, for the same reason flagged
   previously (no flicker/mismatch model in ASAP7). Model A's
   agreement is conditional on γ landing near 0.5–0.7, which is
   plausible but unverified for this exact device.
3. **The pole-vs-gate ambiguity itself needs an `.ac` sweep to
   settle properly**, not just the closed-form pole formula used
   here — `FINDINGS-2026-09-13.md`'s original recommendation was to
   propagate the injected noise source via a real `.ac` simulation,
   which this pass did not do (this pass is a pre-simulation
   analytical sanity check, deliberately smaller in scope, to decide
   whether that simulation work is worth the effort before running
   it).

## Recommended next step, not started

If a future session picks this thread up: build the actual `.ac`-based
propagation `FINDINGS-2026-09-13.md` describes (inject `4kTγ/gm` as a
current source at the input transistor in the real WSL ngspice deck,
run `.ac`, integrate the output-referred PSD numerically over the
simulated response) for **just the first preamp stage** before
attempting the full 2-stage + chopper + latch model — this pass's
Model A vs. B disagreement is exactly the kind of question a real
`.ac` sweep resolves cleanly (it shows the actual transfer function,
not an assumed single-pole shape). That is a smaller, well-scoped
increment beyond both this pass and the original recommendation's full
scope — a real next step, not a restatement of it.

## What this changes in the decision record

**Nothing in `DECISION_LEDGER.md`.** `readout.comparator.noise.spectre-
histogram` stays `Open` — this finding is encouraging for a specific
unbuilt alternative path, not new evidence toward or against the 53 µV
BSIM4-surrogate number's own status, which remains CONDITIONAL per
`AGENT_02`'s Hard Rule 1. Not promoted; logged as research progress
only.

## Related

`FINDINGS-2026-09-13.md` (the `.op`/`.ac` discovery this extends),
`ai-agents/ORANGE_TEAM.md` (operating point, Hard Rules),
`gcb-spec/ai-agents/AGENT_02_ANALOG_EVALUATOR.md` (the 53 µV surrogate
result and its CONDITIONAL status), `gcb-papers/data/readout/spice/
SPICE_RESULTS_V3.md` §7/§9/§11.

---

## Correction, 2026-09-13 (later same day) — Model A used the wrong load topology; a real, already-built rollup script exists and was missed

**What this document originally said (above, unedited):** Model A
assumed a single-pole bandwidth `f_3dB = gm/(2π·C_L)` — i.e. the
transistor's own transconductance driving a purely capacitive load —
giving 155 MHz and a 58.75 µV estimate (+11% vs. the 53 µV surrogate,
read as "encouraging for the pole-limited picture").

**What's actually true, found while tracing this further for the
`orange-team-collaborator.md` wiki refresh:** the real comparator stage
is **resistively loaded**, not capacitively/transconductance-loaded.
`gcb-spec/models/readout/spice/harness/comparator_noise.py`
`[checked, full read]` — a real, already-built, already-committed
analytic noise roll-up for exactly this circuit (2-stage chopped/AZ
preamp + StrongARM latch) that this repo's earlier sessions did not
surface — computes noise bandwidth as `NBW = 1/(4·R_L·C_L)`, a
resistor-load pole, not `gm/C_L`. At its `I_tail = 30 µA` design point
(`R_L = 26 667 Ω`, `C_L = 0.8 pF`): `NBW = 11.7 MHz` — **re-derived
independently this pass and matches the script's own printed output
exactly** (`python`, not assumed). That is close to this document's
**Model B** (gate-window-limited, 25 MHz ENBW), not Model A. **Model
A's 155 MHz pole does not describe this circuit** — it was the right
formula for the wrong topology (a bare common-source stage with no
explicit load resistor), not this design's actual resistively-loaded
stage. The single-transistor estimate above should be read as a
topology-exploration exercise, not a result that correctly models this
circuit.

**A second, more consequential finding from the same script**: its own
`MEAS` table — real SPICE-measured `g_m` at ASAP7, same device class
(`nfin=8`, `L=24n`) this session's `FINDINGS-2026-09-13.md` extracted
from — gives `g_m` = 167–395 µS across its `I_tail` = 15–60 µA sweep.
This session's own direct `.op` extraction (`FINDINGS-2026-09-13.md`)
got **`g_m` = 779 µS** — roughly **2× above even the script's highest-
current (60 µA) point.** `[new finding, unverified]` — not yet
reconciled. Candidate causes, none confirmed: (a) a different ASAP7
device flavor (the script's `MEAS` comment doesn't state LVT/RVT/SLVT;
this session's extraction used `nmos_lvt` explicitly — LVT devices run
higher `g_m` at a given bias, plausible but not quantitatively checked
against a 2× gap); (b) a different bias point / `V_gs` overdrive, since
this session's `.op` deck was not run at any of the script's six
specific `I_tail` values; (c) a unit or extraction-script discrepancy
in one of the two independent measurements. **Not resolved here** —
flagged as the real next diagnostic step, ahead of any further
noise-model building, since building on an unreconciled 2× `g_m`
discrepancy would make any downstream number unreliable regardless of
which bandwidth model is correct.

**Why this matters beyond just fixing one number**: `comparator_noise.
py` already IS the "manual multi-transistor noise-injection model"
`FINDINGS-2026-09-13.md` recommended building — thermal + residual 1/f
(with real chopper fold-back) + latch contribution, two real stages,
already coded and already producing a number (45.8 µV at `γ=1.0`,
30 µA) close to the 53 µV BSIM4-surrogate total (the gap explained in
`SPICE_RESULTS_V3.md` §9 by `γ_eff≈1.4` read from BSIM4, vs. the
script's assumed `γ=1.0`). **The real next step is not building a new
model from scratch — it's reconciling the `g_m` discrepancy above, then
re-running this existing script with the real BSIM-CMG-extracted `g_m`/
`R_L` in place of its current proxy `MEAS` values**, which is a much
smaller, better-grounded increment than anything this document or
`FINDINGS-2026-09-13.md` proposed building new.

---

## Correction 2, 2026-09-14 — the `g_m` discrepancy is very likely a non-converged-solve artifact, not a real device/bias difference

**Checked directly, not assumed**: re-ran the exact `.op`-extraction
deck methodology (`Itail` as an ideal DC current source into
`nmos_lvt nfin=8 l=24n`, `Vg=0.42 V`, resistor load) across the same
six `I_tail` points `comparator_noise.py`'s own `MEAS` table uses
(15/22/30/40/50/60 µA), both as the original swept `alter`+`op` loop
(`~/gcb_spice/tb/_gm.cir`) and as six freshly generated, fully
independent single-point decks with `.options gminsteps=60` added.

**Real result: roughly half of all six points, in both methodologies,
converge to a physically impossible operating point** — `v(op)` in the
**millions of volts** (e.g. `9.13e6 V`) or negative, with correspondingly
nonsensical `g_m` (often negative, `~1e-13 S`) — while the other half
land at plausible values (`v(op)` between 0 and 0.8 V, `g_m` in the
**320–375 µS** range) **broadly consistent with `comparator_noise.py`'s
own `MEAS` table** (167–395 µS across the same sweep), not with
`FINDINGS-2026-09-13.md`'s reported 779 µS. **Re-ran the 30 µA failing
point three independent times — identical garbage output every time**,
confirming this is a deterministic convergence failure of this exact
netlist/bias combination, not solver-state noise.

**The straightforward reading**: `FINDINGS-2026-09-13.md`'s original
779 µS figure was tagged `[M]easured-in-sim, confirmed` without
checking `v(op)` for physical plausibility (it should read between 0 V
and the 0.8 V rail) — exactly the failure mode this pass now finds
affects roughly half of all attempts at this bias with this exact OSDI
netlist shape. **The ~2× "discrepancy" flagged in Correction 1 above
most likely never existed as a real device- or bias-point difference —
it was very likely one unchecked non-converged solve being compared
against a table of values that happened to be real.** Not stated as
certain (the original failing deck/run that produced 779 µS was not
preserved to re-run verbatim), but the pattern — half of all
independent attempts at this exact bias fail the same deterministic
way, and every successful attempt lands near `comparator_noise.py`'s
own real range — makes "real ~2× device/bias difference" the less
likely explanation of the two, not the more likely one as Correction 1
treated it.

**Practical rule this adds, going forward**: any future `.op`/`.ac`
extraction on this OSDI/BSIM-CMG path must check `0 < v(op) < V_dd`
(or the equivalent physical-plausibility bound for the node being
read) before trusting `gm`/`gds` from it — this program did not have
that check anywhere before this pass, and it would have caught the
779 µS figure immediately rather than after a second, separate
investigation.

**Direct relevance to `GCB-045`** (filed separately, low priority):
this is a second, concrete instance of internal medium-effort checking
resolving a question that was about to be handed to an external
frontier AI as a "diagnose this real discrepancy" task (see the
rewritten `gcb-kelvin` prompt in `gcb-wiki/appendices/
orange-team-collaborator.md`) — the actual cause was a missing
sanity check, found by re-running the original deck, not a deep
device-physics puzzle requiring frontier-level reasoning at all.
