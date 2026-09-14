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
