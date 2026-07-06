# PromptShield — Production Redesign & Release Plan

**Status:** active · **Owner:** Sanskar (m4vic) · **Package:** `promptshields` (PyPI) / `promptshield` (import)
**Current:** PyPI live `3.0.1` (Mar 18) · local `3.1.0` unpublished · **Users:** ~11k downloads/5mo

> Guiding rule: **ship a clean, honest, working release fast; do the big redesign behind a frozen public API.**
> Never break `Shield.fast()/balanced()/strict()/secure()/paranoid()` + `.protect_input()/.protect_output()` — 11k users depend on them.

---

## 0. The models decision (settled)

- **Train on synthetic, validate on human.** Synthetic training data is fine; only *unverified claims* are the problem. Report F1 measured on the human-labeled **anchor set**, not on synthetic.
- **Phase the swap.** Current models work today (minus broken `gradient_boosting`). Ship a clean binary release now; swap to the new MoE models only after the anchor validates them.
- **Replace, but with validation.** The 7-config dataset is richer and should fix the false-positive lean (benign input scored 0.63 threat). Swap in Phase 2, gated on anchor metrics beating the current baseline.

---

## Phase 0 — Clean Release (`3.1.0`)  ·  target: this week

Goal: get a **working, honest, well-packaged** release out. No new features beyond the already-built OutputFilter. Product stays *binary* (block/allow).

### Bugs — DONE
- [x] **Critical:** duplicate `fast/balanced/secure` factories → `secure()` loaded **0** ML models. Removed duplicates; `secure()` now loads the ensemble; `balanced()` matches README.
- [x] `paranoid()` referenced nonexistent `xgboost.pkl` → now uses the real ensemble.
- [x] Tests corrected to documented behavior — 6/6 green.

### Release blockers — TODO
- [ ] **Console scripts:** `promptshield-keygen/-sign/-test` point at nonexistent `promptshield.scripts.*`. → **Decision: drop the 3 `[project.scripts]` entries** for now (re-add in Phase 3 when implemented). Crash-on-install otherwise.
- [ ] **`rank-bm25` undeclared:** the 3.1.0 headline OutputFilter silently disables itself. → Add `rank-bm25` to the `output` optional-extra + note in README.
- [ ] **LICENSE in the wheel:** GitHub has it, but the *wheel* needs it locally at build time. → Add `LICENSE` file + `license-files` so PyPI ships it.
- [ ] **Broken `gradient_boosting.pkl`** (numpy pickle incompat) ships and silently drops. → **Decision: drop from ensemble + exclude from wheel** this release; retrain in Phase 2.
- [ ] **Wheel bloat ~47MB → ~12MB:** loader uses only 5 of 13 pkls. → Restrict `package-data`/MANIFEST to the used models (`tfidf_core`, `rf_core`, `logistic_regression`, `linear_svc` [+`gradient_boosting` once retrained]). Drop `rf_full`(19MB), `tfidf_full`, all `_expanded`, `random_forest.pkl`, `tfidf_vectorizer.pkl`.

### Honesty pass on README
- [ ] Ensemble count: "4 models" → **3** (until GB retrained).
- [ ] Latency numbers: mark as *approximate/unbenchmarked* until Phase 3 benchmarks exist.
- [ ] F1 table: keep, but footnote "measured on held-out synthetic split; human-validated numbers in v4."

### Ship
- [ ] `python -m build` → inspect wheel (models present, LICENSE present, no dead pkls, no broken entry points) → `twine check` → **you** run `twine upload` with your token.
- **Exit criteria:** fresh `pip install promptshields` in a clean venv → `Shield.secure().protect_input(...)` blocks a known injection; `import`+CLI clean; wheel ≤ ~15MB.

---

## Phase 1 — Internal Refactor to Production Architecture (`3.2.0`)  ·  no user-facing behavior change

The library *claims* a composable component architecture (`ShieldComponent`, `_COMPONENT_REGISTRY`) but `protect_input` hardcodes checks inline as a 150-line numbered method, returns raw dicts, and takes a 40-arg constructor. Fix the foundations so the MoE can drop in cleanly.

- [ ] **`ShieldConfig` dataclass** replaces the ~40-arg `__init__`. Presets become declarative config objects → kills the entire *class* of duplicate-preset bug at the root.
- [ ] **Standardize on `ShieldResult`** (the dataclass already exists, unused). Every stage returns it; `protect_input/output` serialize at the boundary → consistent keys (fixes today's branch-dependent dict shapes).
- [ ] **Real component pipeline:** convert Pattern / ML / Session / PII / RateLimit / Canary into registered `ShieldComponent`s run through an ordered pipeline. Adding a stage becomes "register a component," not "edit a method."
- [ ] **Formalize lazy-import boundary:** one clear "optional heavy deps" seam (transformers, sentence-transformers, bm25) vs core.
- [ ] **Test suite: 6 → real coverage.** Unit per component, integration per preset, a golden-file **regression** suite (frozen inputs → expected verdicts) so future model swaps can't silently regress. Target ≥85% on core.
- [ ] **CI (GitHub Actions):** matrix py3.8–3.12, run tests + `twine check` + wheel-content assertion (models present, size budget) on every PR.
- **Backward-compat gate:** a compatibility test asserting the exact public API signatures + result keys are unchanged. **Exit criteria:** identical outputs to 3.1.0 on the regression suite.

---

## Phase 2 — The MoE Classification Upgrade (`4.0.0`)  ·  the product leap

Turn PromptShield from a binary filter into a **threat-classification engine** using the 7-config dataset. This is where the new models and the anchor set land.

### 2a. The keystone — human-labeled anchor set (blocks everything downstream)
- [ ] Stratified sample ~1,000 items across every class of every dimension (seed with human-authored attacks from public sets for the `source=human` slice).
- [ ] **LLM-assisted, human-verified** labeling: model emits gloss + technique + 6-dim proposal + rationale; you accept/correct from the *gloss*, not the manipulative raw text (mitigates the language-gap risk). Dual-model disagreement → your review queue; genuine unresolvables → `ambiguity=true`.
- [ ] Report inter-annotator **κ** per dimension. Freeze as `anchor_v1`; **never train on it.** (Same set validates the ASRT judge — one asset, two products.)

### 2b. Train the new models (gated MoE, not 7-in-parallel)
- [ ] **Binary gate** (always-on, must be fast → **classical** TF-IDF + LR/RF) trained on the `binary` config. Replaces the old twitchy binary models.
- [ ] **Specialist heads** fired *only when the gate flags*: `intent`, `severity` first (then `technique`, `surface`, `source`). **Mix backends:** classical where cheap; small transformer (DistilBERT-class) where semantics matter. Optional DeBERTa gate for a high-security mode.
- [ ] **Latency rule (hard):** classical only on the always-on hot path; transformers only on the conditionally-fired specialist path. Preserves "thousands of attacks, fast."

### 2c. Integrate + validate
- [ ] `BinaryGateComponent` + `ExpertRouterComponent` as pipeline components (Phase-1 architecture pays off here).
- [ ] Additive result field `classification: {binary, intent, technique, severity, surface, source}` — backward-compatible.
- [ ] **New `clarify` action** driven by `ambiguity` (the differentiator: guardrail that asks instead of blindly refusing).
- [ ] **Validate every model against `anchor_v1`** → publish honest F1 + κ. **Promotion gate:** new binary gate must beat the current baseline on the anchor before it replaces it (frozen-anchor safeguard, mirroring ASRT's judge policy).
- **Exit criteria:** anchor-measured F1 ≥ baseline on `binary`; `clarify` demonstrably fires on borderline inputs; hot-path latency within budget on benign traffic.

---

## Phase 3 — Hardening & Developer Experience (`4.x`)

- [ ] **Real latency benchmarks** (per preset, per component) → replace aspirational README numbers with measured ones.
- [ ] **Model versioning + signing:** implement the `verify_models` path and the `keygen`/`sign` CLIs for real; re-add the console scripts.
- [ ] **On-demand model download** option (host heavy models on HF, fetch on first use like DeBERTa) → keeps the base wheel tiny; heavy ensemble optional.
- [ ] Docs site, worked examples (FastAPI/LangChain/MCP tool-guard), migration guide 3.x→4.0.
- [ ] Telemetry opt-in for real-world FP/FN signal to feed the next training round.

---

## Cross-cutting standards (apply every phase)
- **SemVer + CHANGELOG.md** per release; git tag each version.
- **No breaking changes** except at a major bump, documented with a migration note.
- **Every bug fix ships with a regression test** so it can't silently return.
- **Reproducible model training:** pin numpy/sklearn; store training script + data hash + env with each `.pkl` (the GB break was an unpinned-env pickle failure — never again).
- **Security posture:** this is the *defensive* product → stays fully open-source. Offensive/ASRT internals stay private.

## Immediate next actions
1. Confirm the two Phase-0 decisions: **drop console-script entry points** (y/n) and **drop `gradient_boosting`** (y/n).
2. I execute Phase 0 (packaging fixes + trim + README honesty pass + changelog), rebuild, verify a clean-venv install, hand you the `twine upload` command.
3. Kick off Phase 2a (anchor set) in parallel — it gates the whole MoE arc.
