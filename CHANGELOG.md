# Changelog

All notable changes to `promptshields` are documented here. This project follows
[Semantic Versioning](https://semver.org/).

## [3.2.0]

Stability, correctness, and packaging release. No new user-facing features; the
public API (`Shield.fast/balanced/strict/secure/paranoid`, `protect_input`,
`protect_output`, `protect_stream`, `protect_tool_call`) is unchanged.

### Fixed
- **Critical — `Shield.secure()` ran with zero ML models.** Duplicate factory-method
  definitions shadowed the correct presets, so the "maximum security" preset silently
  loaded no ML ensemble at all. Removed the shadowing duplicates; `secure()` (and every
  preset) now loads the intended models and matches the documentation.
- **`Shield.paranoid()` loaded no models** — it referenced a non-existent `xgboost.pkl`.
  It now uses the trained ensemble.
- **`Shield.balanced()`** now matches the documented behavior (patterns + session tracking).
- **Dropped `gradient_boosting` from the shipped ensemble.** Its pickle is incompatible
  with current NumPy and failed to load. The ensemble is now three models — Random Forest,
  Logistic Regression, LinearSVC. Gradient boosting will return, retrained, in a future release.

### Packaging
- **Removed the broken `promptshield-keygen` / `-sign` / `-test` console scripts** — they
  pointed at a module that does not exist and crashed on invocation. They will return once
  implemented.
- **Declared `rank-bm25`** in the `output` optional extra so the OutputFilter feature works
  when installed via `pip install promptshields[output]`.
- **Trimmed the wheel (~47 MB → ~12 MB)** by packaging only the model files the loader
  actually uses.
- **Ensured the `LICENSE` file ships** in the built distribution.

### Tests
- Corrected canary tests that asserted the previous (buggy) preset behavior.

## [3.0.1]
- Previous published release (bidirectional Output Filter, v3.0.0 line).
