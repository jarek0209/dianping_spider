# E2E Quant-Cycle Platform — Product & UX Design Spec

## 1) Product Goal
Build a visual, model-centric workflow for E2E trajectory-prediction quantization that helps engineers:
- Understand **lineage** (what model came from where).
- Spot **metric regressions** quickly.
- Jump from model selection to **bad case diagnosis** in one click.
- Compare **Float vs Quant trajectories** on BEV maps with concrete error localization.

## 2) Primary Users & Jobs
- **Quantization Engineer**: Tune qconfig/search strategy and verify quality.
- **Algorithm Engineer**: Identify whether regressions are from weights vs input/data drift.
- **Tech Lead**: Track model genealogy and release readiness.

Core jobs:
1. Find the best quantized candidate among branches/iterations.
2. Inspect where and why it regressed.
3. Decide next action (retrain, qconfig change, data issue fix).

---

## 3) Information Architecture
Three linked views:

1. **Model Genealogy (Home)**
   - Full-screen lineage canvas.
   - Right-side floating drawer opens on node click.
2. **Meta Center Drawer (Details)**
   - Snapshot of model metadata + metric comparison + auto bad cases.
3. **Trajectory Diagnosis Lab (Analysis)**
   - Deep case-level trajectory diff and consistency checks.

Navigation flow:
- Click model node → Meta Center appears.
- Click `Analyze` on any bad case row → open Diagnosis Lab for that case + selected model.
- Back action preserves current lineage canvas camera/zoom and selected node state.

---

## 4) Visual Style (Dark Technical Theme)

### Color Tokens
- Background: `#0B1020`
- Panel: `#131A2A`
- Border: `#26324D`
- Primary text: `#DDE6FF`
- Secondary text: `#8FA3C7`
- Success: `#39D98A`
- Warning: `#F6C343`
- Danger / regression: `#FF5D73`

Trajectory colors (fixed globally):
- **Float curve**: `#3B82F6` (Blue)
- **Quant curve**: `#FF4D4F` (Red, dashed)
- **Ground truth**: `#22C55E` (Green)
- **Max deviation marker**: `#FFD166`

### Typography
- Font stack: Inter / PingFang SC / Segoe UI
- Numeric metrics use tabular alignment for easy scan.

### Layout Rhythm
- 8px spacing system.
- Rounded cards (10px).
- Soft glow around selected node and active trajectory legend item.

---

## 5) Page 1 — Model Genealogy (Home & Navigation)

## 5.1 Layout
- **Canvas**: 100vw × 100vh interactive graph area.
- **Floating right sidebar trigger**: compact collapsed rail when no node is selected.
- Top-left utility controls:
  - Search model name/id
  - Filter chips (status, tag, owner, date)
  - Zoom-to-fit / reset

## 5.2 Graph Structure
- Root node: `Float_Baseline`.
- Branch type A (**Auto-Search**): one parent → 3 sibling quant nodes.
- Branch type B (**Iteration**): sequential evolution (e.g., `Quant_v1.0 -> v1.1 -> v1.2`).

Suggested edge semantics:
- Solid line: direct parent-child inheritance.
- Dotted line: search-generated sibling relation.

## 5.3 Node Card Design
Each node card includes:
- Model name (`Quant_Model_v1.2`)
- Status pill:
  - Ready = green
  - Training = amber with pulse dot
- Mini ADE sparkline (latest N evals)
- Optional tiny badge: `Top-1`, `Deployed`, `Archived`

States:
- Default
- Hover (elevated + border brightening)
- Selected (outer glow + persistent highlight)

## 5.4 Interaction Behavior
- Single click node:
  - Select node
  - Open Meta Center drawer on the right
  - Fetch summary + latest eval + bad case pool
- Double click node:
  - Center graph camera to node + immediate neighborhood
- Lasso multi-select (optional advanced mode): compare multiple nodes in metric strip

Performance targets:
- 300+ nodes with pan/zoom at 60fps on standard workstation.

---

## 6) Page 2 — Meta Center Drawer (Detail View)

Context: opens from selected node (e.g., `Quant_Model_v1.2`).

## 6.1 Drawer Structure
- Width: 420–520px adaptive.
- Sticky header + scrollable body.

### Header
- Model title + status
- Build/eval timestamp
- QConfig snapshot tag (e.g., `INT8 / per-channel / symmetric`)
- Link to parent model (click jumps/highlights parent on canvas)
- Quick actions: `Open Diagnosis Lab`, `Pin`, `Export Report`

## 6.2 Section A — Trajectory Metrics Comparison
Two-row comparison table:
- Row 1: Float baseline
- Row 2: Selected quant model

Columns:
- ADE
- FDE
- Collision Rate
- Comfort Score
- Delta vs Float

Rules:
- Regression cells show red text + subtle red background.
- Improvements show green.
- Tooltips explain metric definitions and eval split.

## 6.3 Section B — Automated BadCase Pool
Table columns:
- Case ID
- Scene Tag (e.g., Left Turn, Merge, Pedestrian Crossing)
- Max Deviation (m)
- Trigger Reason (optional: collision-risk / off-lane / jerk)
- Action (`Analyze`)

Interaction:
- Sort by max deviation descending by default.
- `Analyze` opens Page 3 with preloaded scene + selected frame near peak error.
- Multi-select cases supports batch export for offline review.

---

## 7) Page 3 — Trajectory Diagnosis Lab (Analysis View)

## 7.1 Layout
- Upper area (70%): BEV visualizer.
- Bottom area (30%): analysis panel with timeline + bar charts.
- Left legend dock (always visible).

## 7.2 BEV Visualizer Requirements
Render layers:
1. Lane map / drivable region (muted grayscale)
2. Ground truth trajectory (green)
3. Float trajectory (blue)
4. Quant trajectory (red dashed)
5. Deviation marker at max gap point

Deviation marker behavior:
- Dynamic line connecting float and quant nearest correspondence at current frame.
- Label format: `Max Δ = 1.20m @ t=3.4s`
- Hover reveals local heading delta and speed delta.

Controls:
- Play/pause frame animation
- Step frame ±1
- Speed 0.5x / 1x / 2x
- Toggle layer visibility
- Fit-to-trajectories

## 7.3 Bottom Analysis Panel

### A) Diff Timeline
- Scrubber with per-frame error heat strip.
- Peaks auto-annotated.
- Clicking a peak jumps BEV to that frame.

### B) Consistency Check
Two grouped bar charts:
- **Input Data Drift** (feature-level shift indicators)
- **Model Weight Drift** (post-quant weight/stat divergence)

Purpose:
- Quickly classify issue source:
  - High input drift, low weight drift -> dataset/sensor mismatch likely.
  - Low input drift, high weight drift -> quantization config/model issue likely.

---

## 8) End-to-End Interaction Scenario
1. Engineer lands on Genealogy page and sees all model branches.
2. Clicks `Quant_Model_v1.2` node.
3. Meta Center opens; ADE/FDE regression highlighted red.
4. In BadCase list, selects case with `Max Deviation = 1.2m` and clicks `Analyze`.
5. Diagnosis Lab opens directly at error peak frame.
6. Engineer checks drift bars, determines issue likely from quant weight drift.
7. Engineer returns, adjusts qconfig/search constraints, launches next iteration.

---

## 9) Data Contract Suggestions (for Engineering Alignment)

### Graph Node Schema
```json
{
  "model_id": "quant_v1_2",
  "model_name": "Quant_Model_v1.2",
  "status": "ready",
  "parent_id": "quant_v1_1",
  "relation_type": "iteration",
  "ade_series": [0.62, 0.64, 0.61, 0.66],
  "qconfig": "int8_per_channel_sym",
  "updated_at": "2026-02-18T10:20:00Z"
}
```

### BadCase Row Schema
```json
{
  "case_id": "case_20260218_0081",
  "scene_tag": "Left Turn",
  "max_deviation_m": 1.2,
  "peak_frame": 87,
  "trigger_reason": "off_lane"
}
```

---

## 10) Usability & Accessibility Notes
- Keyboard navigation for node selection and case table actions.
- Color + icon dual encoding (do not rely on red/green only).
- Minimum contrast ratio 4.5:1 for key text in dark mode.
- Persist filters and selected model in URL query for shareable debugging links.

---

## 11) KPI / Success Metrics
- Time-to-root-cause reduced by 30%.
- Mean clicks from model selection to case diagnosis <= 2.
- Bad case triage throughput (cases/hour) improved by 25%.
- Regression escape rate (bad model promoted) reduced release-over-release.
