# gcb-kelvin

**Want to explore this with another AI chat (ChatGPT, Gemini, etc.)
that has no access to this repo?** Paste
[`EXTERNAL_AI_BRIEFING.md`](EXTERNAL_AI_BRIEFING.md) — it's
self-contained, no repo access required.

**Progress, 2026-09-13**: see
[`FINDINGS-2026-09-13.md`](FINDINGS-2026-09-13.md). Still no Spectre
access — but a real WSL-based ngspice/OSDI toolchain was found
reachable, `.noise`'s known failure was re-confirmed (not assumed
stale), and one genuinely new, unbuilt path was found: real small-
signal parameters (`gm`/`gds`) extract cleanly from the actual target
device even though internal noise synthesis doesn't. **The core
question below is still open** — no comparator-circuit noise number
exists yet either way.

**One isolated problem: one number, from one tool this program doesn't
currently have access to, decides whether the analog readout is
`CONDITIONAL GO` or `GO`.**

## The question, stated plainly

The analog Rx front-end's comparator noise is **53 µV** (gate: **≤ 55
µV**), measured via a BSIM4 `.noise` surrogate model gm-matched to the
real ASAP7 BSIM-CMG device. That surrogate carries a **±16% uncertainty
band (44.5–61.5 µV)** that straddles the 55 µV gate — meaning the real
answer could already be a pass or could already be a fail, and nobody
knows which.

**What's needed:** a transient-noise histogram on a **commercial
simulator with a native BSIM-CMG model** — Cadence Spectre is the one
named in every evidence doc — confirming the decision-referred noise
sigma is within ±20% of 53 µV. That single run is the entire standing
item blocking `CONDITIONAL GO → GO`
(`gcb-wiki/DECISION_LEDGER.md`: `readout.comparator.noise.spectre-histogram`).

## Why this is genuinely blocked, not just undone

This program's entire SPICE toolchain is open-source by deliberate
choice (`gcb-spec/docs/spice_setup.md`): a self-built `ngspice-44` with
`--enable-osdi` + `OpenVAF`-compiled `BSIM-CMG 111`. That stack was
surveyed and found to have two real, hard limits, not just "not tried
yet":

- **`ngspice`'s OSDI `.noise` analysis returns zero on ASAP7 devices.**
  Its `trnoise` scaling is not reproducible either — a `kT/C` reference
  expected at 64.4 µV returned 96 µV / 3.2 mV / 3.9 mV at three
  different timesteps on the same setup (~50× spread).
- **Xyce could not be built no-root.** conda-forge Trilinos 16.2 lacks
  `NOX_ENABLE_LOCA`, `EpetraExt_BUILD_BTF`, and Isorropia/Zoltan — a
  from-source Trilinos build was judged out of scope.

The 53 µV number that exists today comes from a real, working
fallback: `ngspice`'s `.noise` analysis **does** work correctly against
its own **built-in BSIM4** (level 14) model, gm-matched as a surrogate
for the real BSIM-CMG device. That's a genuine device-physics noise
integral — not fabricated, not a guess — but it's a surrogate, and its
own uncertainty band is the open question.

## What this repo needs to actually produce

1. **Access to a commercial simulator with a native BSIM-CMG model**
   (Spectre is the one this program's own docs name) — this is a real,
   external dependency this repo cannot manufacture; it needs either
   institutional access or an explicit decision to accept the surrogate
   as final.
2. **If access exists:** run the transient-noise histogram at the same
   operating point already fully specified in
   `gcb-spec/ai-agents/AGENT_02_ANALOG_EVALUATOR.md`'s "hard analog
   numbers" section (2.0 pF `C_int`, 30 µA/stage comparator tail, 20 ns
   gated decision, 9 PVT corners) and report the real sigma against the
   53 µV / ±20% target.
3. **If access does not exist:** say so plainly and close this repo with
   that finding — "the toolchain genuinely cannot produce this number
   without a tool this program does not have" is itself a real,
   complete Orange-Team result, not a failure to hide. The documented
   fallback (a gm–C integrating first-stage preamp, unbuilt) is on file
   in the same evidence doc if the answer ever comes back above 55 µV.

## Not the source of truth

`gcb-papers/data/readout/spice/SPICE_RESULTS_V3.md` and
`gcb-spec/ai-agents/AGENT_02_ANALOG_EVALUATOR.md` are. This repo exists
to close one specific open item from them, not to re-derive the analog
design.

See [`intent.md`](intent.md) for scope, and
[`ai-agents/ORANGE_TEAM.md`](ai-agents/ORANGE_TEAM.md) for the dense
AI-session context — it is, by design, mostly a direct quote of the
existing Analog Evaluator persona's own hard numbers, since this repo
does not get to re-derive them.
