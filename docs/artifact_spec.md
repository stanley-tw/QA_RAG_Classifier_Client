# Artifact Output Specification (Project A -> Project B)

## Purpose
Project A must output a portable, versioned artifact bundle that encapsulates:
- The canonical domain (label) list
- One embedding vector per domain (label_vec)
- All metadata required for Project B to safely and deterministically classify user Q&A rows

Project B must be able to perform domain classification without accessing Project A's database.

## Bundle Layout
Project A produces one directory per successful run:
```
domain_bundle_v{N}/
  artifact_manifest.json
  domains.json
  label_index.json
  label_vec.npy
  domain_repr.jsonl        (optional but recommended)
```

The directory name `domain_bundle_v{N}` is a logical artifact version, not a code version.

## 1. artifact_manifest.json (REQUIRED)

### Purpose
Single source of truth for provenance, compatibility, integrity checks, and reproducibility.
Project B must read this file first.

### Schema
```
{
  "artifact_version": "v1",
  "created_at": "2025-03-01T12:34:56Z",
  "embedding_model": "text-embedding-3-small",
  "embedding_provider": "azure_openai",
  "embedding_dimension": 1536,
  "tokenization_policy": {
    "mode": "exact",
    "fallback_allowed": true
  },
  "domain_count": 42,
  "files": {
    "domains.json": "sha256:...",
    "label_index.json": "sha256:...",
    "label_vec.npy": "sha256:...",
    "domain_repr.jsonl": "sha256:..."
  },
  "generation_config": {
    "merge_threshold_name_only": 0.90,
    "merge_threshold_name_plus_summary": 0.86,
    "preferred_display_language": "auto"
  }
}
```

### Requirements
- `embedding_model` must match the model used to generate `label_vec.npy`.
- `embedding_dimension` must match the vector dimension in `label_vec.npy`.
- Checksums must be SHA-256.
- Project B should fail fast if model or dimension mismatch.

## 2. domains.json (REQUIRED)

### Purpose
Defines the canonical domain catalog for display, reporting, and debugging.

### Schema
```
{
  "domains": [
    {
      "domain_id": "domain_001",
      "display_name": "Authentication",
      "aliases": [
        "Auth",
        "Login and Authentication",
        "SSO"
      ],
      "source_pdfs": [
        "pdf_001",
        "pdf_003"
      ]
    }
  ]
}
```

### Requirements
- `domain_id` must be stable and deterministic.
- `display_name` is human-readable and may be LLM-generated.
- Ordering is not significant (ordering is defined in `label_index.json`).

## 3. label_index.json (REQUIRED)

### Purpose
Defines the row-to-domain mapping for `label_vec.npy`.

### Schema
```
{
  "embedding_model": "text-embedding-3-small",
  "embedding_dimension": 1536,
  "domain_ids": [
    "domain_001",
    "domain_002",
    "domain_003"
  ],
  "domain_display_names": [
    "Authentication",
    "Billing",
    "Performance"
  ]
}
```

### Requirements
- Length of `domain_ids` must equal rows in `label_vec.npy`.
- Ordering must be deterministic and stable.
- Project B must not reorder domains.

## 4. label_vec.npy (REQUIRED)

### Purpose
Stores the embedding matrix for canonical domains.

### Format
- NumPy `.npy`
- `dtype=float32`
- Shape `(N, D)` where:
  - `N` = number of domains
  - `D` = embedding dimension

### Semantics
`label_vec[i] == embedding(domain_ids[i])`

### Requirements
- Must be generated using the model specified in `artifact_manifest.json`.
- Must be normalized or non-normalized consistently (Project B handles cosine normalization).

## 5. domain_repr.jsonl (OPTIONAL, RECOMMENDED)

### Purpose
Provides explainability and debugging context for each domain.

### Format
JSON Lines (one domain per line)

### Schema (per line)
```
{
  "domain_id": "domain_001",
  "representation_text": "Authentication and SSO handling, including login flows, tokens, and identity providers.",
  "top_headings": [
    "Authentication",
    "Single Sign-On"
  ],
  "keywords": [
    "login",
    "sso",
    "token",
    "identity"
  ]
}
```

### Requirements
- `representation_text` should correspond to the text used to generate the domain embedding.
- Content must not invent concepts beyond Project A's source PDFs.

## Compatibility Contract (Project A -> Project B)
Project A guarantees:
- Deterministic domain IDs
- Stable domain ordering
- Embeddings compatible with declared model

Project B assumes:
- Same embedding model for query embeddings
- Cosine similarity is the scoring metric
- `artifact_manifest.json` is authoritative

## Minimal Artifact Set (Hard Requirement)
If Project A outputs only the minimum, it must include:
- `artifact_manifest.json`
- `domains.json`
- `label_index.json`
- `label_vec.npy`

## Versioning Policy
Artifact version increments when:
- Domain merging logic changes
- Thresholds materially change
- Embedding model changes

Project B should treat different artifact versions as incompatible unless explicitly allowed.
