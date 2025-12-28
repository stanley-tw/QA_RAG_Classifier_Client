# Design Specification (This Project)

Design Goals
- Deterministic, reproducible Q&A domain classification.
- Strict separation between artifact consumption and classification logic.
- Zero coupling to QA_RAG_Classifier databases or internal code.
- Minimal, testable modules with clear responsibility boundaries.
- Efficient batch processing for large Excel files.

High-Level Design Principles
- Artifact is a read-only contract.
- This Project must never modify, regenerate, or reinterpret artifact contents.
- Pipeline is linear and stateless.
- No background jobs, no incremental learning, no hidden state.
- Single source of truth: artifact_manifest.json governs compatibility and validation.
- UI/CLI are orchestration layers only; no classification logic in UI or CLI.

Module Layout
src/
├── main.py                 # Entry point (CLI or UI)
├── config.py               # Configuration schema + defaults
├── cli/
│   └── app.py              # CLI commands (run, validate)
├── ui/
│   └── app.py              # Streamlit UI (rendering only)
├── pipeline/
│   ├── run.py              # End-to-end pipeline orchestration
│   ├── classify.py         # Core classification logic
│   └── thresholds.py       # Threshold evaluation helpers
├── artifact/
│   ├── loader.py           # Load artifact bundle
│   ├── validator.py        # Contract validation (fail-fast)
│   └── index.py            # Domain index & ordering utilities
├── io/
│   ├── excel_reader.py     # Excel → DataFrame (preserve order)
│   └── excel_writer.py     # Augmented DataFrame → Excel
├── embedding/
│   ├── client.py           # Azure OpenAI embedding client
│   └── normalize.py        # L2 normalization utilities
└── util/
    ├── hashing.py          # Optional checksum helpers
    └── errors.py           # Typed error definitions

Module Responsibilities
config.py
- Define configuration schema and defaults:
  - TEXT_MODE
  - QUESTION_COLUMN
  - ANSWER_COLUMN
  - TOP_K
  - MIN_SCORE
  - AMBIGUITY_MARGIN
  - UNKNOWN_LABEL
  - OUTPUT_FIELDS
  - EMBEDDING_MODEL_NAME
  - EMBEDDING_MAX_RETRIES
  - EMBEDDING_RETRY_BACKOFF_SEC
  - EMBEDDING_TIMEOUT_SEC
- Load configuration from CLI/UI overrides.
- No business logic.

artifact/loader.py
- Load files from artifact bundle directory.
- Parse:
  - artifact_manifest.json
  - domains.json
  - label_index.json
  - label_vec.npy
- Return a structured, in-memory artifact object.

artifact/validator.py
- Fail-fast contract enforcement.
- Validate:
  - Required files exist.
  - Embedding model matches config.
  - Vector dimension consistency.
  - Domain ordering integrity.
  - Checksums (if present).
- Raise typed fatal errors on any mismatch.
- No partial success allowed.

artifact/index.py
- Build immutable domain index:
  - domain_ids[]
  - display_names[]
- Provide deterministic mapping:
  - index → domain_id
  - domain_id → index
- Must preserve ordering from label_index.json.

embedding/client.py
- Wrap Azure OpenAI embedding API.
- Enforce:
  - Model name exactly matches artifact manifest.
  - Deterministic request parameters.
  - Batch embeddings for performance.
- Limited, explicit, optional retries only:
  - Must honor EMBEDDING_MAX_RETRIES, EMBEDDING_RETRY_BACKOFF_SEC, EMBEDDING_TIMEOUT_SEC.
  - Defaults must be conservative (retries off or minimal).
  - Retries, if enabled, must be deterministic and bounded. No exponential backoff or jitter is allowed.

embedding/normalize.py
- L2-normalize:
  - Query vectors.
  - Label vectors (once, on load).
- Must not change relative values beyond normalization.

io/excel_reader.py
- Load Excel file into a DataFrame-like structure.
- Read only:
  - QUESTION_COLUMN
  - ANSWER_COLUMN (if present)
- Preserve:
  - Row order.
  - All original columns.

io/excel_writer.py
- Append classification output columns:
  - domain_id
  - domain_display_name
  - similarity_score
  - Optional explainability fields
- No column reordering or deletion.

pipeline/run.py
- Single orchestration entrypoint.
- Load configuration.
- Load and validate artifact bundle.
- Load Excel input.
- Prepare input text per row.
- Generate embeddings.
- Compute similarity matrix.
- Apply classification logic.
- Write augmented output.
- No branching logic beyond configuration.

pipeline/classify.py
- Compute cosine similarity:
  - Matrix multiplication: Q × label_vecᵀ.
- Select top-k per row.
- Apply threshold rules:
  - MIN_SCORE
  - AMBIGUITY_MARGIN
- Assign:
  - assigned
  - unknown
  - ambiguous
  - invalid_input
- Return structured classification results.

pipeline/thresholds.py
- Encapsulate threshold logic.
- Ensure rules are:
  - Deterministic.
  - Unit-testable.
  - No embedding or I/O logic.

Output Schema
Output Columns (Fixed Names)

Required:
- domain_id: string
- domain_display_name: string
- similarity_score: float

Optional (controlled by OUTPUT_FIELDS):
- top_k_domain_ids: list[string]
- top_k_domain_names: list[string]
- top_k_scores: list[float]
- classification_status: enum("assigned", "unknown", "ambiguous", "invalid_input")
- warning_flags: list[string]

warning_flags may include:
- missing_answer

Data Flow (Logical)
Excel → Reader → Pipeline
                ↓
        Artifact Loader + Validator
                ↓
          Embedding Client
                ↓
        Similarity Computation
                ↓
        Threshold Classification
                ↓
           Excel Writer

Determinism Guarantees
- No randomness.
- No sampling.
- Fixed domain ordering.
- Explicit normalization.
- Same input + same artifact + same config ⇒ same output.

Error Handling Strategy
Fatal (Stop Pipeline)
- Missing artifact files.
- Model mismatch.
- Vector dimension mismatch.
- Missing question column.

Non-Fatal (Row-Level)
- invalid_input definition:
  - A row is classified as invalid_input if:
    - the question value is missing, or
    - the question value is not a string type, or
    - the question value, after trimming whitespace, is an empty string.
- Rows classified as invalid_input must:
  - receive domain_id = UNKNOWN_LABEL
  - set classification_status = "invalid_input"
  - preserve row order
- Missing answer in Q_PLUS_A → fallback + warning flag.

Non-Goals (Reaffirmed)
- No domain discovery.
- No domain merging.
- No artifact mutation.
- No database usage.

Design Rationale (Summary)
This Project is a pure artifact consumer and deterministic classifier, designed to be simple, auditable, and impossible to misuse without failing fast.
