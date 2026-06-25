I'll work through all of Weeks 5–8 comprehensively. Let me also search for the latest best practices on graph validation and object detection metrics to make sure everything is current


Here are complete solutions for Weeks 5–8, incorporating current best practices from recent research on DAG validation, low-light preprocessing pipelines, and object detection metrics.

---

## 📅 Week 5 — Blocks & Graphs

### Exercises

**1. Five Pipeline Block Designs**

| Block | Input Ports | Output Ports | Config Parameters | Description |
|-------|------------|-------------|---------------------|-------------|
| **Load Image** | `path` (string) | `image` (tensor), `metadata` (dict) | `format_hint`: "auto" \| "jpg" \| "png" | Reads image from disk, decodes to array |
| **Resize** | `image` (tensor) | `resized_image` (tensor) | `target_size`: [H, W], `interpolation`: "bilinear" \| "nearest" | Scales image while preserving aspect ratio (optional) |
| **Grayscale** | `image` (tensor) | `gray_image` (tensor) | `method`: "luminance" \| "average" | Converts 3-channel RGB to 1-channel grayscale |
| **Detect Objects** | `image` (tensor) | `detections` (list), `annotated_image` (tensor) | `model`: string, `confidence_threshold`: float, `nms_iou`: float | Runs inference, returns boxes + labels + scores |
| **Save Result** | `image` (tensor), `detections` (list) | `file_path` (string), `success` (bool) | `output_dir`: string, `format`: "json" \| "csv" | Writes annotated image and detection metadata to disk |

**Port Types Matter:** Input ports consume data; output ports produce data. A connection is only valid when the data type of the source output port matches (or is compatible with) the target input port. For example, you cannot connect `Load Image`'s `metadata` output to `Grayscale`'s `image` input because a dictionary is not a tensor.

**2. Invalid Connections & Why They Break**

```
INVALID CONNECTION #1: Detect Objects → Load Image
Reason: Load Image expects a file path, not detection tensors.
        Data type mismatch: list[dict] ≠ string

INVALID CONNECTION #2: Save Result → Grayscale
Reason: Creates a cycle: Save Result → Grayscale → Detect Objects → Save Result.
        A cycle means execution order is undefined—infinite loop.

INVALID CONNECTION #3: Grayscale (output) → Detect Objects (image) 
        AND Detect Objects (annotated_image) → Grayscale (input)
Reason: Cycle. Grayscale waits for Detect Objects, which waits for Grayscale.

INVALID CONNECTION #4: Resize (resized_image) → Load Image (path)
Reason: Semantic mismatch. Resize outputs an image tensor, 
        but Load Image expects a filesystem path string.

INVALID CONNECTION #5: Unconnected required port
Reason: Detect Objects requires an `image` input. If nothing connects to it,
        the block has no data to process and the graph cannot execute.
```

**3. Six-Block Workflow & Manual Topological Sort**

```
Graph Structure:
    A [Load Image] ──→ B [Resize] ──→ C [Grayscale] ──→ D [Detect Objects] ──→ F [Save Result]
                                          ↑
                                          │
                                    E [Enhance Contrast]
```

**Step-by-step Kahn's Algorithm (Manual Topological Sort):**

| Step | Action | In-degree Count |
|------|--------|-----------------|
| 1 | Find nodes with in-degree 0: **A** | A:0, B:1, C:1, D:2, E:0, F:1 |
| 2 | Output **A**, remove A→B edge | B:0, C:1, D:2, E:0, F:1 |
| 3 | Find nodes with in-degree 0: **B**, **E** | Choose **B** |
| 4 | Output **B**, remove B→C edge | C:0, D:2, E:0, F:1 |
| 5 | Find nodes with in-degree 0: **C**, **E** | Choose **C** |
| 6 | Output **C**, remove C→D edge | D:1, E:0, F:1 |
| 7 | Find nodes with in-degree 0: **E** | |
| 8 | Output **E**, remove E→D edge | D:0, F:1 |
| 9 | Find nodes with in-degree 0: **D** | |
| 10 | Output **D**, remove D→F edge | F:0 |
| 11 | Output **F** | |

**Final Execution Order:** `A → B → C → E → D → F` (or `A → B → E → C → D → F`)

Note: B and E are independent and can run in parallel. C must wait for B; D must wait for both C and E.

**4. Adding a Cycle & Why It Breaks Execution**

```
Cycle injection: F [Save Result] → A [Load Image]

Now: A → B → C → D → F → A (cycle!)
```

**Why execution breaks:**
- **Topological sort fails:** Kahn's algorithm never reaches in-degree 0 for any node in the cycle. Every node in the cycle has at least one incoming edge from within the cycle.
- **Infinite recursion risk:** If executed naively, A waits for F, F waits for D, D waits for C, C waits for B, B waits for A → deadlock.
- **No valid ordering:** You cannot determine which block runs first because each depends on another in the loop.
- **Data staleness:** Even if you broke the cycle forcefully, F would feed "results" back to A as "input," creating a feedback loop that may never converge.

In production DAG systems (like Apache Airflow or Kubeflow), cycle detection is a mandatory pre-deploy validation step that fails the CI/CD pipeline immediately .

**5. Graph Validation Checklist**

| # | Check | How to Verify | Failure Consequence |
|---|-------|---------------|---------------------|
| 1 | **No cycles** | DFS with color marking (white/gray/black) or Kahn's algorithm | Infinite loops, undefined execution order |
| 2 | **All required ports connected** | Every input port with `required=true` must have exactly one incoming edge | Missing data causes runtime exception |
| 3 | **No dangling outputs** (warning) | Output ports with no connections may indicate incomplete pipeline | Wasted computation |
| 4 | **Data type compatibility** | Source output type must match or be castable to target input type | TypeError at runtime |
| 5 | **Shape compatibility** (for tensors) | Output tensor shape must be compatible with input expectations (e.g., CHW vs HWC) | Dimension mismatch in downstream ops |
| 6 | **Single source per input port** | Each input port receives from exactly one output port | Ambiguity: which data to use? |
| 7 | **Graph is connected** | All nodes reachable from at least one source node | Orphaned blocks never execute |
| 8 | **At least one source and one sink** | Source: in-degree 0. Sink: out-degree 0 | No start point or no end point |
| 9 | **Config values within valid ranges** | e.g., `confidence_threshold` ∈ [0, 1] | Invalid model parameters |
| 10 | **No duplicate block IDs** | Each block instance has a unique identifier | Reference ambiguity in logs/traces |

---
