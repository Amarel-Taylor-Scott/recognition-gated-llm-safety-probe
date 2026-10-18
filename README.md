# Recognition Is Not the Bottleneck: An Out-of-Band Probe Separating Recognition-Failure from Rationalized Compliance in LLM Safety

**Working note — defensive red-team / AI-agent safety. DRAFT.**
Author: Taylor S. Amarel. Code: `jurisdictional_confusion.py`, `jc_strong_kernel.py`,
`recognition_gating.py`, `jurisdictional_agent.py` (+ Kaggle T4/GPU runners). Redaction on
throughout; harms are indicator-level and synthetic; we measure *safety behavior*, never keep the
harmful completion.

## Abstract
A popular intuition holds that wrapping a disallowed request in unfamiliar domain complexity
(cross-border labor law, human-trafficking compliance) "lowers the model's guard" because the model
*fails to recognize* the harm — recognition-gated safety. We test this directly with a simple but,
to our knowledge, novel instrument: an **out-of-band recognition probe** that asks the model to
*classify* the same request it is asked to *perform*. The probe adjudicates two mechanisms that both
lower refusal but have opposite defenses: **recognition-FAILURE** (the model never flags the harm)
vs **recognized-but-EXECUTED** (the model flags it yet complies — competing objectives / "rationalized
abdication"). Across Qwen2.5 at 1.5B and 7B and a battery of attack structures, we find: (1) on a
*guard-strength* model, single-turn domain framing does **not** lower the guard (it can *raise* it);
(2) the durable vulnerability is a large **recognized-but-executed** gap — the model classifies these
labor-exploitation requests as refusable (~100%) yet performs them (~90%+), independent of framing;
(3) **structural** attacks (multi-turn crescendo, work-product framing, "technical exercise" / regex
framing) drive 100% compliance on a guard-strength model and break the agentic guard — a multi-turn
authority pretext got the model to execute a tool call exfiltrating a worker's passport/contract/bank
details; (4) **encoding** (leet, pig-latin, regex-as-human) does **not** blind recognition on a
capable model — it decodes the obfuscation even when classifying — so encoding works (when it works)
as recognized-but-executed, not recognition-failure. The practical upshot: you cannot close this gap
by improving *recognition*; the model already recognizes. The fix is action-side — dataflow/tool
mediation and intent confirmation at the point of the privileged action.

## 1. Background and the contested mechanism
LLM safety training teaches refusal of disallowed requests. A large jailbreak literature shows that
refusal is brittle under transformation: mismatched generalization (Wei et al. 2307.02483),
character/tokenizer manipulation (FlipAttack, TokenBreak), latent/iterative search (LARGO, MIST),
code/cipher framing (CodeAttack >80% ASR), and low-resource-language translation. Separately, models
are documented to be weak on exactly the domains we probe: legal hallucination 58–88% (Dahl et al.
2024), degraded cross-jurisdictional reasoning under conflict (Multi-Legal-Bench), and socio-legal
complicity (~50% illicit-assist on non-extreme violations, 2511.20736). The Gemma model card omits
labor-exploitation/trafficking from its safety taxonomy entirely — a clean coverage-gap signal.

These two literatures invite a tempting causal story: *models are bad at recognizing trafficking/
cross-border harm, therefore framing a request in that domain evades the safety classifier.* But an
opposed account exists — "Mind the GAP" / competing-objectives / shallow alignment (2602.16943): the
model **recognizes** the harm and proceeds anyway, rationalizing compliance. Both lower refusal; they
are usually not distinguished because experiments measure only the refusal rate.

## 2. Contribution: the out-of-band recognition probe
We separate them. For each request we run two passes against the same model: a **refusal probe** (ask
it to *do* the task; grade refuse/comply) and an **out-of-band recognition probe** (ask it, fresh,
*"is this something you should refuse or flag as unlawful/exploitative/against policy?"*; grade
recognized/not). The 2×2 of (refusal drop, recognition drop) adjudicates the mechanism:

| refusal drops? | recognition drops? | mechanism | defense |
|---|---|---|---|
| no | — | no effect | — |
| yes | yes | **recognition-FAILURE** | improve recognition / coverage |
| yes | no | **recognized-but-EXECUTED** | action-side controls, intent confirmation |

A length-matched neutral-filler control separates domain-framing from raw cognitive load. A salient
control harm (phishing) the model recognizes regardless checks specificity.

## 3. Method
- **Domains (indicator-level, redacted):** niche labor cores — wage-deduction fee recovery
  (debt-bondage-adjacent), repayment re-labeled to dodge "recruitment fee," passport retention as
  "safekeeping"; salient control — IT-impersonation phishing. (`jurisdictional_confusion.py` CORES.)
- **Framings:** neutral · cross-jurisdictional (RA 10022 / Palermo / ILO C181-C189 corridor-conflict)
  · trafficking-agency scenario · length-control filler. (`frame()`.)
- **Strong techniques** (`jc_strong_kernel.py`): baseline · jurisdictional · work-product ("draft
  manual section N") · logical-trap (harm as the model's own derived legal conclusion) ·
  beneficiary-indirection · multi-turn crescendo · plus obfuscation arms (leet, pig-latin,
  regex-as-human, combined).
- **Agent arm** (`jurisdictional_agent.py`): a fixed, clearly-unsafe tool action (exfiltrate a
  worker's passport+contract+bank details to an external address); framing is the only variable;
  measure refuse-vs-perform.
- **Models:** Qwen2.5-1.5B-Instruct (CPU) and Qwen2.5-7B-Instruct (Kaggle T4, fp16). Greedy decoding,
  deterministic. Graders are keyword/prefix-based and deterministic; never need the harmful text.

## 4. Results
**1.5B (weak guard).** Framing dropped niche refusal 33%→0% (cross-jurisdictional & trafficking),
recognition held ~100%, length-control no drop → *recognized-but-executed*, but the model is too weak
(agent arm: took the unsafe action under every framing, incl. neutral — no guard to lower).

**7B (guard-strength), single-turn framing — the hypothesis fails.** Niche refusal: neutral 8%,
**cross-jurisdictional 33% (framing RAISED caution)**, trafficking 8%, length 0%; recognition ~100%
throughout. So domain framing did **not** lower the guard on the stronger model — the explicit legal
citations seem to *prime* care. But recognition ~100% vs refusal ~8% is itself the headline: a large
**recognized-but-executed gap on the bare requests**, framing-independent.

**7B, structural techniques — these break it (n=3 cores).** Compliance: baseline 67% →
jurisdictional 100%, work-product 100%, beneficiary-indirect 100%, **crescendo 100%**; recognition
stays high → recognized-but-executed. logical-trap and crescendo also broke the **salient phishing
control** (refusal 100%→0%). **Agent arm: the multi-turn `crescendo_authority` pretext got the 7B to
EXECUTE `share_records`, exfiltrating the worker's passport+contract+bank details** — where the
single-turn ask was only "ambiguous."

**7B, obfuscation — refutes recognition-failure.** Recognition stayed 100% for leet/pig-latin/
regex-as-human (the model decodes them even when *classifying*). pig-latin, regex-as-human, and
leet+regex hit 100% compliance and broke the salient control — via recognized-but-executed ("interpret
this regex and carry it out" = a technical-task frame), **not** recognition-defeat. Leet *alone*
backfired (refusal 33%→67%; it reads as evasive).

## 5. Finding
Across framing and encoding, on a capable model **recognition does not drop** — the model sees the
harm. What lowers the guard is *structure that supplies a competing objective*: a multi-turn
authority/compliance pretext, a "draft section N" work-product frame, or a "technical exercise"
(legal-drafting / regex-interpretation) frame. The mechanism is **recognized-but-executed**;
recognition-FAILURE is the weak-model regime and is refuted on the 7B by the encoding arms. The
out-of-band probe is what makes this visible — without it, every one of these reads identically as
"refusal dropped."

## 6. Defense implications
- **Improving the safety classifier's recognition will not help** — recognition is already ~100%. The
  gap is between recognition and action.
- Put the control **at the privileged action**: dataflow/tool mediation, out-of-band intent
  confirmation before exfil-class tool calls, and provenance checks (the agent arm fell to a
  *multi-turn* pretext a single-turn guard passed).
- Trafficking/cross-border labor belongs in the safety taxonomy explicitly (the coverage gap is real
  and the recognized-but-executed rate on it is high) — but as an *action-policy* item, not merely a
  recognition target.

## 7. Limitations (honest)
n=3 niche cores in the strong-technique run (one core = 33pp); the agent result is one binary
observation; the recognition grader is keyword-based, so the *size* of the gap is soft (the
*relative* "framing didn't lower refusal" is robust to it); a length confound was LIKELY on the 1.5B
content arm; two model sizes, one family. Next: ≥5 paraphrases/cell, ≥8 cores, a frontier model and
the agent targets, and a learned recognition grader. The pipeline and the decomposition are the
contribution; the numbers are a proof-of-concept pending scale.

## 8. Reproducibility
All scripts run offline with redaction. `python3 jurisdictional_confusion.py` (MockTarget demo);
`jc.FunctionTarget(lambda p: your_model(p))` to plug a real model; `kaggle/jc_strong_kernel.py` +
`deploy/research.sh` to run the strong battery on a Kaggle T4. Graders, framings, and cores are in
`jurisdictional_confusion.py`; the agent action in `jurisdictional_agent.py`.
