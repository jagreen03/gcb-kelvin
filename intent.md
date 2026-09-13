# Intent: gcb-kelvin

## Scope

**One measurement, one tool-access question, one binary outcome:**

1. Determine whether this program (or the person running this repo) has
   real access to a commercial transient-noise simulator with a native
   BSIM-CMG model (Spectre is the one named in every evidence doc).
2. If yes: run the histogram, report the real sigma against 53 µV ± 20%,
   and update `gcb-wiki/DECISION_LEDGER.md`'s
   `readout.comparator.noise.spectre-histogram` entry from `Open` to
   `Accepted` (GO) or to whatever the fallback decision becomes (NO-GO →
   gm–C first-stage preamp, already documented, unbuilt).
3. If no: say so, close the repo's active task with that finding, and
   leave the `Open` status exactly as-is — this repo does not get to
   manufacture tool access it doesn't have.

## Out of scope

- Re-deriving the 53 µV surrogate number, the ±16% uncertainty band, or
  any of the other already-measured analog numbers in
  `gcb-spec/ai-agents/AGENT_02_ANALOG_EVALUATOR.md`. Quote them; don't
  recompute them.
- Designing the gm–C fallback preamp. It's documented, unbuilt, and
  explicitly out of scope unless the Spectre run (if it happens) lands
  above 55 µV — building it pre-emptively would be exactly the kind of
  "change the architecture before measuring" move Orange-Team doctrine
  rejects.
- The MMI combiner-loss extraction (`gcb-lumen`) and the laser/modulator
  bank reconciliation (`gcb-lambda`) — distinct problems, sibling repos.
- Touching RTL, the digital TDC/calibration back-end, or macro-
  architecture. Those belong to `gcb-spec/ai-agents/AGENT_01`/`AGENT_03`.

## Governing discipline

Same Orange-Team "Physical Reality First" discipline as `gcb-lumen` and
`gcb-lambda` (`gcb-workspaces/backlog/TKT-000`) — but this is the
sharpest-scoped of the three: there is exactly one number missing, one
tool that would produce it, and a clean binary outcome either way. The
discipline here is mostly about **not pretending the surrogate number is
final when it isn't, and not inventing a workaround the toolchain survey
already ruled out** (ngspice's OSDI `.noise` returning zero on ASAP7;
Xyce's no-root Trilinos build failing).

## Relationship to the rest of the program

This repo's only real output is a single decision-ledger status change
in `gcb-wiki/DECISION_LEDGER.md`, backed by either a real Spectre run or
an honest "access doesn't exist" finding. It does not fork or duplicate
`gcb-spec/ai-agents/AGENT_02_ANALOG_EVALUATOR.md`'s content — it
consumes it.
