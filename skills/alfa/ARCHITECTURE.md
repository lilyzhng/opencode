# Alfa Architecture

## What Alfa Does

Alfa is a data selection system for autonomous driving. It finds relevant video sequences from a large corpus using embedding-based search + SQL filters, then sends matched data to auto-labeling.

## Pipeline

```
prompt.yaml (config)
    ↓
Cosmo Video Embedding Model (encodes prompts + video frames)
    ↓
Embedding Search (cosine similarity against alfa table)
    ↓
SQL Filter (additional constraints on metadata)
    ↓
Matched slice_ids
    ↓
Join with ScaleX label table (on slice_id)
    ↓
Hydrate data from ScaleX format
    ↓
Send to auto-labeling
```

## Data Model

### Alfa Table (Embeddings)
- Indexed by **(slice_id, camera_name)**
- One **slice_id** = one video sequence
- Each sequence has **~6 camera views** (front, back, side, etc.)
- Contains Cosmo model embeddings for prompt-based similarity search

### ScaleX Table (Labels + Metadata)
- Indexed by **slice_id**
- Contains labeled data in ScaleX dataset format
- Requires **hydration** to access actual video/image data
- Join key with alfa table: **slice_id**

## Config: prompt.yaml

Single YAML file. Each scenario is a named data selection strategy.

### Structure
```yaml
# Scenario description comments
# * What we're looking for
# * Specific conditions
- name: "scenario_name"
  prompts:
    - prompt: "natural language description for embedding search"
      camera_names: *camera-anchor    # YAML anchor, e.g. *back-svc, *front-svc
      similarity_threshold: 0.26      # cosine similarity cutoff (higher = stricter)
      sql_filter: *filter-anchor      # YAML anchor, e.g. *m1_and_m2
```

### Camera Anchors
- `*back-svc` - rear cameras
- `*front-svc` - front cameras
- Others as defined in YAML anchor section

### SQL Filter Anchors
- `*m1_and_m2` - require both M1 and M2 data
- `*prefer_m2_over_m1` - prefer M2, fall back to M1
- Others as defined in YAML anchor section

### Fields
| Field | What it does | Typical range |
|-------|-------------|---------------|
| name | Scenario ID, used as reference | snake_case string |
| prompt | Natural language for Cosmo embedding search | Descriptive, includes camera perspective |
| camera_names | Which of the ~6 cameras to search | YAML anchor |
| similarity_threshold | Cosine similarity cutoff | 0.20 - 0.35 (lower = more results) |
| sql_filter | Additional SQL constraints on metadata | YAML anchor |

## Use Cases (Active Learning)

The config is used for **Alfa curate data selection strategies for active learning**:
1. Teams define scenarios they need data for (e.g. rear cross traffic, highway merge)
2. Embedding search finds matching video sequences
3. Matched data goes to auto-labeling pipeline
4. Labeled data feeds back into model training

## Common Adjustments

Teams periodically request changes:
- **Camera selection**: switch from all 6 cameras to front-only or back-only
- **Threshold tuning**: stricter (higher) for precision, looser (lower) for recall
- **Prompt refinement**: better natural language descriptions for the scenario
- **SQL filter changes**: different metadata constraints
- **New scenarios**: entirely new data selection strategies
