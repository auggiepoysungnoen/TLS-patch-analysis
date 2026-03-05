# NB02 Handoff Document
Generated: 2026-03-04

## What Was Done

### File
`notebook/NB02_HDBSCAN_Patch_Delination.ipynb`

### Completed Changes

#### 1. Auto-param cell (`lkb1rt3fvyf`) — "big but not too big" philosophy
- `_AUTOPAR_MCS_FRAC`: 0.03 → **0.05** (5% of TLS cells = one meaningful follicle)
- `_AUTOPAR_MCS_BOUNDS`: (15, 200) → **(50, 300)** (floor 50 avoids specks)
- `EPS_FRACTION` in `_predict_hdbscan_params`:
  - sparse (norm_sp > 0.20): 0.25 → **0.35**
  - medium (norm_sp > 0.12): 0.18 → **0.25**
  - dense (default):          0.12 → **0.20**
- `p["SELECTION"]`: `"leaf"` → **`"eom"`** (largest stable clusters, not tiny fragments)
- Docstring updated with "big but not too big" philosophy

#### 2. DEFAULTS cell (`nwyayb4efz`)
- `MIN_CLUSTER_SIZE`: 60, `MIN_SAMPLES`: 30, `SELECTION`: "eom", `EPS_FRACTION`: 0.25
- All aligned with auto-param philosophy

#### 3. Test run cell (`a7bbpe2b4yb`) — switched to Mode B
- `auto_params=False` → **`auto_params=True`**
- Comment updated to show `← ACTIVE` on Mode B

#### 4. Section 8 — Sub-clustering (NEW)
Added after Section 7 visualization (Before), before Section 9 visualization (After).

**New cells added:**
- `w8kyv2496rr` — markdown header with parameter table
- `4ts6b5ypy6e` — two functions:
  - `_subcluster_patch(df_patch, ...)` — splits one cluster if `normalized_spread > spread_threshold`
  - `run_subclustering(df, params_log, ...)` — loops over all regions/clusters; skips `all_one` regions
- `72r8gu1pb33` — runner call producing `df_patched` and `split_log`

**Key logic:**
- `normalized_spread = mean_knn_dist / sqrt(bbox_area)` — scale-free spatial spread metric
- If `spread_threshold=0.15` exceeded AND cluster has >= `2 × min_sub_size` cells → run HDBSCAN within cluster
- Sub-cluster noise (cells between sub-clusters) → reassigned to nearest sub-cluster centroid
- `all_one` regions (TLS saturates tissue) → assigned as one patch, no sub-clustering
- Output: `tls_patch` column with globally unique integer IDs across all regions (-1 = noise)

#### 5. Visualization restructured
- Section 7: "Before Sub-clustering" — plots `df_out` with `cluster_col="tls_cluster"`
- Section 9 (NEW, cells `3ajr4obgz3o` + `ilnztfi4q4i`): "After Sub-clustering" — plots `df_patched` with `cluster_col="tls_patch"`
- Both use the same `plot_tls_patches()` function defined in `17b991rf5cn`

#### 6. GitHub
- Repo: https://github.com/auggiepoysungnoen/TLS-patch-analysis
- Pushed: NB01, NB02, patch_maturity_meta-analysis.ipynb, README.md, .gitignore
- Excluded: data/ (2.8 GB), resources/ (142 MB), patch_analysis.ipynb (228 MB > GitHub 100 MB limit)

---

## Notebook Cell Order (tail of notebook)

```
nzdr3ejvd3m   markdown  ## 7. Visualization — Before Sub-clustering
17b991rf5cn   code      plot_tls_patches() function definition + constants
wrs2vf2711    code      plot_tls_patches(df_out, ...) ← BEFORE subclustering
w8kyv2496rr   markdown  ## 8. Sub-clustering (parameter table)
4ts6b5ypy6e   code      _subcluster_patch() + run_subclustering() definitions
72r8gu1pb33   code      run_subclustering(df_out, params_log, ...) → df_patched, split_log
3ajr4obgz3o   markdown  ## 9. Visualization — After Sub-clustering
ilnztfi4q4i   code      plot_tls_patches(df_patched, cluster_col="tls_patch", ...)
3943bf2b      code      empty
```

---

## Remaining / Potential Next Steps

1. **Run the notebook end-to-end** — the test run (Section 6) has not been re-run since changes.
   - `df_out` and `params_log` need to be regenerated with Mode B (`auto_params=True`)
   - Then Section 8 sub-clustering can be tested

2. **Tune sub-clustering parameters** based on visual inspection of Section 7 vs 9:
   - If too few splits: lower `spread_threshold` (e.g., 0.10) or `eps_fraction` (e.g., 0.15)
   - If too many splits: raise `spread_threshold` (e.g., 0.20) or `min_sub_size` (e.g., 50)

3. **Full run** (all regions, not just testrun):
   ```python
   df_out_full, params_log_full = run_tls_hdbscan_on_df(vdf, ..., testrun=False, auto_params=True)
   df_patched_full, split_log_full = run_subclustering(df_out_full, params_log_full, ...)
   ```

4. **Save output**:
   ```python
   df_patched_full.to_csv(output_path / "vdf_tls_patches.csv", index=False)
   split_log_full.to_csv(output_path / "subclustering_log.csv", index=False)
   ```

5. **Push updated NB02 to GitHub** after running and reviewing:
   ```bash
   cd "Z:/Collaboration/Auggie_Poysungnoen/Kylee_Li/Pan_Organ/follicle_meta_analysis"
   git add notebook/NB02_HDBSCAN_Patch_Delination.ipynb
   git commit -m "Run Mode B test + subclustering results"
   git push origin main
   ```

---

## Key Variable Names (after running)
| Variable | Source | Description |
|----------|--------|-------------|
| `vdf` | Section 1 | All cells with community labels + renamed_tissue |
| `df_out` | Section 6 | vdf with `tls_cluster` column (HDBSCAN output) |
| `params_log` | Section 6 | Per-region params + decision (all_one vs cluster) |
| `df_patched` | Section 8 | df_out with `tls_patch` column (sub-clustering output) |
| `split_log` | Section 8 | Per-cluster split info: region, n_sub, spread, split bool |
