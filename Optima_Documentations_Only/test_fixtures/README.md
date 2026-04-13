# Frozen Test Datasets

These JSON files contain canonical input/output pairs for testing Optima's core algorithms.
They are the contract — tests MUST assert against these exact values.
Do not modify these files unless the underlying algorithm specification changes.

| Dataset | Tests | Minimum Pass Rate |
|---|---|---|
| error_normalization_dataset.json | sanitizeError + normalizeError pipeline | 100% (deterministic) |
| entity_extraction_dataset.json | extractEntities() Tree-sitter parsing | 100% (deterministic) |
| gotcha_retrieval_dataset.json | Hierarchical directory prefix match + file array match | 100% (deterministic) |
| directory_scoping_dataset.json | Rules directory scoping precedence | 100% (deterministic) |
