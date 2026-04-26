---
name: alfa
description: Manage Alfa data selection configs. Change prompts/cameras/thresholds, visualize current config, query matched data.
argument-hint: change <scenario> ... | visualize | query <scenario>
---

# Alfa: Data Selection Config Manager

Manage the prompt.yaml config that drives Alfa's embedding-based video search and data selection pipeline.

## Config File

`prompt.yaml` is the single source of truth. Each scenario has:

```yaml
- name: "scenario_name"
  prompts:
    - prompt: "natural language description"
      camera_names: *camera-anchor    # e.g. *back-svc, *front-svc
      similarity_threshold: 0.30
      sql_filter: *filter-anchor      # e.g. *m1_and_m2, *prefer_m2_over_m1
```

## Commands

### `/alfa change <scenario> ...`

Modify an existing scenario or add a new one. Accepts natural language.

**Examples:**
```
/alfa change rear_cross_traffic_alert to use front cameras with threshold 0.22
/alfa change rear_cross_traffic_alert add prompt "delivery truck unloading from back view"
/alfa change rear_cross_traffic_alert remove the grocery store prompt
/alfa change rear_cross_traffic_alert set sql_filter to prefer_m2_over_m1
```

**Steps:**
1. Read `prompt.yaml`
2. Find the scenario by name
3. Parse the natural language request to determine what to change (cameras, threshold, prompts, sql_filter)
4. If scenario doesn't exist, ask: "Scenario not found. Create it?" Then treat as a new scenario.
5. Show a before/after diff of the YAML change
6. Ask for confirmation before writing
7. Write the updated `prompt.yaml`

**Rules:**
- Preserve YAML anchors (e.g. `*back-svc`). Don't expand them.
- Preserve comments above scenarios (the `# *` description lines).
- When adding a prompt, copy the camera_names, similarity_threshold, and sql_filter from the first prompt in that scenario as defaults, unless the user specifies otherwise.

### `/alfa visualize`

Render the current config as a readable summary.

**Steps:**
1. Read `prompt.yaml`
2. Display a table for each scenario:

```
SCENARIO                     PROMPTS  CAMERAS     THRESHOLD  FILTER
rear_cross_traffic_alert     4        back-svc    0.26-0.30  mixed
highway_merge                3        front-svc   0.28       m1_and_m2
```

3. If a local server is available, generate an HTML preview at `preview/index.html` and open it:
   - Table view of all scenarios
   - Click a scenario to expand and see all prompts
   - Highlight threshold outliers (unusually high or low)
   - Show camera distribution (how many scenarios use which cameras)

### `/alfa query <scenario>`

Run a dry query to see how many sequences match a scenario config.

**Steps:**
1. Read `prompt.yaml`, find the scenario
2. Build the SQL query from the scenario's sql_filter + camera_names
3. Run the query against the alfa embedding table
4. Count matched slice_ids
5. Join with ScaleX label table on slice_id to find overlap
6. Report:

```
rear_cross_traffic_alert:
  Matched slice_ids:     2,400
  With ScaleX labels:    1,800 (75%)
  Camera breakdown:      back-svc: 2,400
  Threshold range:       0.26 - 0.30
```

7. If the user wants the actual data: export slice_ids to a file

## Data Model

- **Alfa table**: embeddings indexed by (slice_id, camera_name). One slice_id = one video sequence with ~6 camera views.
- **ScaleX table**: labels + metadata indexed by slice_id. Needs hydration to access actual video data.
- **Join key**: slice_id

## Preview Server

```bash
# Start local preview
bash lib/serve.sh
```

Serves `preview/index.html` on localhost. Shows all scenarios, their configs, and query results. Same pattern as warroom and career-angel.
