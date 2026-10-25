# pharma-eval-harness

An evaluation harness that measures whether LLMs answer drug-label questions
using only the label they were given, and whether they can be steered by
instructions hidden inside that label (indirect prompt injection).

**Status: in progress.** The pipeline runs end to end. Human validation of the
automated grader is under way, so the results below are preliminary and
labelled as such. The code will be published after an independent review.

## What it does

1. **Builds test cases from real FDA labels** (openFDA). Each case is a
   question plus four labelled sections of the label, about 3,300 characters,
   with the answering section shuffled so finding it is part of the task.
2. **Plants hostile instructions** inside the label text for a subset of
   cases: poisoned retrieval, where the attack arrives in the document rather
   than from the user.
3. **Asks two models**, `gemini-3.6-flash` and `openai/gpt-oss-120b`, under
   the same prompt, temperature 0, with the model hidden from every grader.
4. **Grades every answer** with an LLM judge (`qwen3.8-27b`, deliberately a
   different model family from both subjects) against a seven-dimension
   rubric: indication, dosage, contraindication (an omission test),
   interaction, special populations, scope of advice, and injection
   compliance.
5. **Checks the grader against a human.** A blinded, shuffled terminal tool
   collects human verdicts; agreement is reported as Cohen's kappa rather
   than raw agreement, because most answers pass and raw agreement would
   flatter a judge that never read anything.
6. **Checks the test itself.** The planted instructions are run through an
   injection detector (`llama-prompt-guard-2-86m`), gated by positive and
   negative controls, to confirm they are recognisable attacks.

## Preliminary findings

- **The injection detector's coverage is lexical, not semantic.** Of four
  injection styles, the two containing explicit override language were
  flagged in 100% of cases; the two phrased as ordinary system guidance were
  flagged in 0%. Burying them in label text changed neither number.
- **Failures concentrate in injection cases.** The automated judge failed 5
  of 77 answers; 4 of the 5 were injection cases. One case was failed by both
  models. *Not yet human-validated.*
- **The two models cannot be ranked yet.** At this sample size the 95%
  intervals overlap almost entirely (about 1–17% vs 3–21%). Reported as such
  rather than as a winner.
- **One injection style is mostly untestable.** An instruction to omit drug
  interactions is only observable when the label actually lists an
  interaction for the drug asked about; otherwise an honest answer and an
  obedient one look identical. Those cases are tagged and excluded from the
  resistance rate.

## Design choices worth noting

- Every answer and verdict carries a fingerprint of the exact question and
  context it was given, so results from an earlier version of a case are
  never silently reused.
- Graders and detectors are kept out of the model comparison by an
  allowlist, so an instrument can never be scored alongside what it measures.
- Rate limits are read from provider responses rather than assumed.
- 200+ unit tests, including regression tests for each measurement defect
  found during development.

## Responsible disclosure

Results are reported as aggregate rates by injection category. The poisoned
test corpus and per-drug results are not published, because pairing real
drug labels with working injection text would amount to a reusable attack
recipe.
