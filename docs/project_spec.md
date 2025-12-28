# Project Specification: Q&A Domain Classification (This Project)

Purpose
This Project classifies user Q&A rows into canonical domains using a precomputed artifact bundle produced by QA_RAG_Classifier. The artifact specification is a hard external contract; domain definitions and embeddings must never be inferred, regenerated, or modified.

Scope
- In-scope: read an Excel Q&A table, embed Q or Q+A text, compute cosine similarity against artifact label vectors, assign domain labels, and export augmented results.
- Out-of-scope: domain discovery or merging, retraining or fine-tuning, artifact mutation, and any access to QA_RAG_Classifier databases or internals.

Inputs
1) Q&A Excel file
- One row per Q&A.
- The Excel file may contain many columns; This Project must ignore all columns except:
  - Required: question (question text).
  - Optional: answer (answer text).
- Column names are case-sensitive unless explicitly configured otherwise.
- Row order must be preserved.
- Output must retain all original columns and append new classification columns (no column removal or reordering).

2) Domain artifact bundle (QA_RAG_Classifier output)
- Must conform to docs/artifact_spec.md.
- Minimum required files: artifact_manifest.json, domains.json, label_index.json, label_vec.npy.
- Optional file: domain_repr.jsonl (used only for explainability).

3) Configuration parameters
- TEXT_MODE: Q_ONLY or Q_PLUS_A.
- QUESTION_COLUMN: default "question".
- ANSWER_COLUMN: default "answer".
- TOP_K: number of candidate domains to return.
- MIN_SCORE: minimum cosine similarity to assign a label (recommended default 0.82; range 0.0-1.0).
- AMBIGUITY_MARGIN: maximum allowed difference between top-1 and top-2 to be considered unambiguous (recommended default 0.03; range 0.0-1.0).
- UNKNOWN_LABEL: placeholder string for unclassified rows.
- OUTPUT_FIELDS: which explainability fields to include; allowed values are top_k_domain_ids, top_k_domain_names, top_k_scores, classification_status, warning_flags.
- EMBEDDING_MODEL_NAME: must match artifact_manifest.json.

Artifact Consumption Rules
- artifact_manifest.json is authoritative and must be read first.
- Validate and fail fast on any of the following:
  - Missing required files.
  - Embedding model mismatch (manifest vs configuration).
  - Vector dimension mismatch (label_vec.npy vs manifest).
  - Domain ordering mismatch (must align exactly with label_index.json).
- Checksums in the manifest must be verified when present; any mismatch is a fatal error.
- The artifact bundle is immutable; no mutation, regeneration, or reordering.

Pipeline / Flow
1) Load the Excel file into a tabular structure and read question (and answer if present) columns only; preserve all original columns and row order.
2) Load artifact bundle and validate compatibility.
3) Prepare input text per row based on TEXT_MODE:
   - Q_ONLY: use Q.
   - Q_PLUS_A: concatenate Q and A with a fixed, deterministic separator string (for example "Q: {Q}\nA: {A}"); if A is missing, fall back to Q.
4) Embed each row's text using the model declared in the artifact manifest.
5) Compute cosine similarity between each row embedding and label_vec.npy.
6) Apply classification rules and produce top-k results.

Classification Logic and Thresholds
- Score metric: cosine similarity computed using L2-normalized vectors.
- Both query vectors and label_vec rows must be L2-normalized prior to similarity; implementations may pre-normalize label_vec for efficiency but must not change values.
- Top-k selection: deterministic descending order of similarity; tie-breaking uses domain order from label_index.json.
- Assignment rules:
  - If top-1 score < MIN_SCORE, label as UNKNOWN_LABEL.
  - Else if (top-1 score - top-2 score) < AMBIGUITY_MARGIN, label as UNKNOWN_LABEL and mark ambiguous.
  - Else assign top-1 domain.
- TOP_K results may still be returned for rows classified as UNKNOWN when OUTPUT_FIELDS include top-k explainability fields.
- No stochastic components; outputs must be identical for the same input, artifact, and configuration.

Outputs
Augmented Q&A table with:
- domain_id (assigned or UNKNOWN_LABEL)
- domain_display_name
- similarity_score (top-1)
- Optional explainability fields:
  - top_k_domain_ids
  - top_k_domain_names
  - top_k_scores
  - classification_status (assigned | unknown | ambiguous | invalid_input)
    - assigned: confidently assigned to a single domain.
    - unknown: below MIN_SCORE or ambiguous.
    - ambiguous: explicitly marked when top-1/top-2 difference < AMBIGUITY_MARGIN.
    - invalid_input: empty or malformed question text.
  - warning_flags (for example missing_answer when Q_PLUS_A and answer is absent)

Determinism and Reproducibility
- Same input + same artifact + same configuration => identical output.
- No randomness; no model selection beyond the manifest-declared embedding model.
- Floating-point operations must be deterministic within the runtime environment.

Failure Handling
- Missing required artifact files: fail fast with explicit error.
- artifact_manifest.json missing or invalid: fail fast.
- Embedding model mismatch: fail fast.
- Vector dimension mismatch: fail fast.
- Malformed label_index.json or inconsistent ordering: fail fast.
- If question column is missing: fail fast with explicit error listing available columns.
- Empty or malformed rows: classify as UNKNOWN_LABEL with classification_status=invalid_input; preserve row order.
- If A is missing in Q_PLUS_A, fall back to Q and mark the row with a deterministic warning flag in output metadata (for example warning_flags includes missing_answer).

Non-Goals
- No domain discovery or merging.
- No retraining or fine-tuning.
- No artifact mutation.
- No dependency on QA_RAG_Classifier databases or internal code paths.
