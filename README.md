# rl-env-makr
certified auditing of unexamined assumptions in llm-generated rl environments 

## The problem

When an LLM is asked to write an RL environment, it makes hundreds of modeling choices it does not declare: that the market has zero impact, that the brain state is Markov, that time is discrete, that reward is bounded, that the transition kernel is stationary, that the return type is a list and not an option. These unexamined assumptions are the substrate of every downstream experiment, and neither the LLM nor the human who prompted it can reliably surface them. The LLM defaults to the simplest possible structure unless explicitly pushed; the human cannot prompt for variables they do not know exist; and neither has a criterion for deciding when a model is "complex enough" for its purpose. The consequence is that RL results are built on environment models whose failure modes are invisible until someone runs them at the edge of a regime nobody thought to test.

## The insight

Three tools each solve a piece of this, and none solves the whole:

- LLMs can generate environment code and can *propose* formal statements about what that code assumes. They cannot certify their own proposals.
- Formal verification (Lean) can certify that a statement follows from assumptions, and can certify the axiom budget a proof rests on. It cannot certify that the statement corresponds to reality.
- Assumption extractors (AssumptionMiner) can surface assumptions an LLM made implicitly in code. They cannot tell you whether those assumptions are adequate for a given goal.

The architecture here separates these roles strictly: the LLM proposes, Lean disposes, and faithfulness is declared as an unverified obligation on every row rather than hidden. This is not a limitation of the implementation — it is the correct epistemic status of "a formal statement that might or might not mean what you think it means."

## The pipeline
┌──────────────────────────────────────────────────────────────────────┐
│ LLM generator │ any model: GPT, Claude, Llama, Qwen, local or hosted.│
└────────┬─────────────────────────────────────────────────────────────┘
│ (prompt, code)
▼
┌──────────────────────────────────────────────────────────────────────┐
│ AssumptionMiner │ extracts assumption records from (prompt, code).   │
└────────┬─────────────────────────────────────────────────────────────┘
│ normalized records
▼
┌──────────────────────────────────────────────────────────────────────┐
│ Proposer │ LLM proposes a Lean STATEMENT per assumption.             │
│ │ It does NOT write the proof.                                       │
└────────┬─────────────────────────────────────────────────────────────┘
│ candidate .lean files
▼
┌──────────────────────────────────────────────────────────────────────┐
│ Lean type-check │ compiles (with sorry)? → continue                  │
│ │ fails? → retry or mark not_formalizable                            │
└────────┬─────────────────────────────────────────────────────────────┘
│ type-checked statements
▼
┌──────────────────────────────────────────────────────────────────────┐
│ Tactic portfolio │ omega, linarith, nlinarith, norm_num, simp_all,   │
│ │ decide. First tactic that closes the goal wins.                    │
└────────┬─────────────────────────────────────────────────────────────┘
│ proved statements
▼
┌──────────────────────────────────────────────────────────────────────┐
│ #print axioms │ the kernel reports every axiom the proof depends.    │
│ + allowlist │ on. Clean iff ⊆ {propext, Classical.choice, Quot.sound}│
└──────────────────────────────────────────────────────────────────────┘
│
▼
┌──────────────────────────────────────────────────────────────────────┐
│ Report │ every row carries faithfulness: "declared".                 │
│ │ certified | refuted | unverified | not_formalizable                │
└──────────────────────────────────────────────────────────────────────┘

The five verdicts:

- `certified`: Statement type-checked, tactic portfolio proved it, axiom budget clean.
- `refuted`: Statement type-checked, portfolio proved its negation with a witness.
- `unverified`: Compiles, but the axiom budget contains something outside the allowlist (e.g. a `sorry` slipped through, or a user-declared axiom).
- `unproved`: Compiles, but no tactic in the portfolio closed the goal. 
- `not_formalizable`: The proposer determined that the assumption cannot be expressed in Lean 4 given available libraries.
- `failed`: Compile error, contract violation, or repeated parse failure.

Every row also carries `faithfulness: "declared"` plus a one-sentence
justification, because Lean cannot certify correspondence.

## Design principles

These are the rules that separate a real audit from a "generate and trust" system. They were arrived at by violating them and catching the failure.

1. The LLM proposes; Lean disposes. The LLM never writes a proof script. The proof is the tactic portfolio's job. A hallucinated proof is impossible by construction because the LLM has nothing to hallucinate into.

2. Certification is semantic, not textual. Early versions of this pipeline checked for the string `"declaration uses 'sorry'"` in Lean's output. That is unsound: a file containing user-declared axioms, or an `unsafe` declaration, or a `set_option` that disables the axiom linter, exits with code 0 and would be reported as verified. The fix is not a more thorough string match or a language model to read the output — it is to query Lean's semantic API. `#print axioms` reports the actual axiom set the kernel believes each proof depends on. That is what gates certification.

3. Faithfulness is always declared, never verified. Lean certifies consistency, not correspondence. A statement that compiles and is proved can still mean something other than what the natural-language assumption said. Every row reports `faithfulness: "declared"` and carries the proposer's one-sentence justification. Reviewers and downstream users are free to disagree with the proposal; that disagreement is recorded,not hidden.

4. Every assumption is a target for the next audit. The tool's own code is not exempt. The `sorry`-based string-check bug described in principle 2 was found by asking: *what does this line assume?* The `self_audit.py` module runs the pipeline on `pipeline/*.py` and reports what it finds.

5. Coverage is proved, not hoped for. A parameter sweep can miss a failure region that lies between sample points. The planned coverage theorem (Phase 4) makes the resolution requirement explicit: if the violation predicate is `L`-Lipschitz and the grid resolution is below `ε / L`, then no connected failure region of diameter greater than `ε` is missed. Until that theorem is in place, sweeps are heuristic, and the report says so.

6. The tool audits itself. `python -m pipeline.self_audit` runs the pipeline on its own modules and prints an audit table. If the pipeline ever silently becomes unsound again, the self-audit is the evidence.

## Quickstart

Requires: Lean 4 via `elan`, a Lake project with Mathlib + LeanCert, Python
3.10+, and an OpenAI-compatible LLM endpoint (Ollama or vLLM).

```bash
# 1. Toolchain
curl https://elan.lean-lang.org/elan-init.sh -sSf | sh
cd ~/lean-audit
lake update && lake exe cache get

# 2. AssumptionMiner
cd ~/research/assumptionminer/replication-package
pip install -e . && pytest -q

# 3. Toy pipeline (currently working)
cd ~/lean-audit
python -m pipeline.pipeline
# ID       STATUS      DESCRIPTION
# TOY-A1   certified   reward(theta, action=0) <= 3 for all theta in [0,1]
# TOY-A2   refuted     reward(theta, action=1) >= 0 for all theta in [0,1]

# 4. Proposer test 
python -m tests.test_proposer_a2

```
full code @ rl-env-makr-complete
𐔌՞ ܸ.ˬ.ܸ՞𐦯
