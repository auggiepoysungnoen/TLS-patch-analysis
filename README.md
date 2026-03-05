# TLS Patch Analysis

Spatial delineation of individual TLS (Tertiary Lymphoid Structure) follicle patches from single-cell spatial transcriptomics data using HDBSCAN clustering.

**Collaborators:** Auggie Poysungnoen, Kylee Li — Pan Organ Follicle Meta-Analysis

---

## Project Overview

TLS follicles appear as dense spatial clusters of TLS-labelled cells within tissue sections. This pipeline segments those clusters into individual follicle *patches*, enabling downstream analysis of follicle size, maturity, and tissue context across a pan-organ cohort.

---

## Notebooks

| Notebook | Description |
|----------|-------------|
| `NB01_Dataset_Clean.ipynb` | Data loading, QC, and community label standardisation |
| `NB02_HDBSCAN_Patch_Delination.ipynb` | **Main pipeline** — TLS follicle patch segmentation |
| `patch_maturity_meta-analysis.ipynb` | Downstream maturity scoring and meta-analysis |

---

## NB02 Pipeline Summary

### Input
- `vdf_community.csv` — all cells with `community` labels, spatial `x`/`y`, `dataset_name`, `region_imaged`
- `20250924_processed_metadata.csv` — maps `region_imaged` → `renamed_tissue` (biological tissue label)

### Output columns added to vdf
| Column | Description |
|--------|-------------|
| `tls_cluster` | HDBSCAN cluster ID per TLS cell (`-1` = noise) |
| `tls_patch` | Final follicle patch ID after sub-clustering (`-1` = noise) |

### Pipeline Sections
```
1. Setup          → imports, paths, load vdf + metadata
2. Parameter system → DEFAULTS + TISSUE_PARAMS (31 tissue types, 7 biological groups)
3. Auto-param     → spatial analysis before HDBSCAN; predicts MCS/EPS per region
4. HDBSCAN core   → epsilon estimation + HDBSCAN fit per region
5. Runner         → loops over all (dataset × region) pairs
6. Test run       → 1 region per dataset (Mode B: tissue + auto-param)
7. Viz — Before   → community vs tls_cluster scatter
8. Sub-clustering → splits dispersed clusters using KNN-distance CV signal
9. Viz — After    → tls_cluster vs tls_patch scatter (before/after comparison)
```

---

## Parameter System

Four-layer priority stack (lower layers override higher):

```
L1  DEFAULTS          global fallback
L2  TISSUE_PARAMS     biology-specific (31 tissues, 7 groups)
L3  region overrides  manual per-region fixes (optional; per_region=True)
L4  auto-param        data-driven spatial adjustment (auto_params=True)
```

### Execution Modes
| Mode | Flags | Stack |
|------|-------|-------|
| A — tissue only | `per_region=False, auto_params=False` | L1 → L2 |
| B — tissue + auto *(active)* | `per_region=False, auto_params=True` | L1 → L2 → L4 |
| C — full stack | `per_region=True, auto_params=True` | L1 → L2 → L3 → L4 |

### Key Parameters
| Parameter | Current Value | Effect |
|-----------|--------------|--------|
| `MIN_CLUSTER_SIZE` | 60 | Minimum cells for one follicle |
| `EPS_FRACTION` | 0.25 | Epsilon scale — lower = tighter clusters |
| `SELECTION` | `eom` | Largest stable clusters (not tiny fragments) |
| `_AUTOPAR_MCS_FRAC` | 0.05 | Auto MCS = 5% of TLS cells in region |
| `_AUTOPAR_MCS_BOUNDS` | (50, 300) | Floor/ceiling on auto MCS |

### Sub-clustering Parameters
| Parameter | Current Value | Effect |
|-----------|--------------|--------|
| `cv_threshold` | 0.45 | KNN-distance CV above this → attempt split |
| `min_sub_size` | 30 | Minimum cells per sub-cluster |
| `eps_fraction` | 0.20 | Tight epsilon within cluster |

---

## Task List

### ✅ Completed
- [x] HDBSCAN runner with 4-layer parameter stack
- [x] Auto-param prediction (`_analyze_tls_spatial`, `_predict_hdbscan_params`)
- [x] `all_one` short-circuit for TLS-saturated tissues
- [x] Tissue-specific TISSUE_PARAMS for all 31 `renamed_tissue` values (7 biological groups)
- [x] Metadata merge (`renamed_tissue` from metadata onto vdf)
- [x] Mode A / B / C documented in code and markdown
- [x] Sub-clustering (`run_subclustering`) with CV-based split signal
- [x] Noise reassignment to nearest sub-cluster centroid
- [x] Before/after visualization (`plot_before_after_subclustering`)
- [x] Notebook reorganised with section headers and flow table

### 🔄 Next Steps
- [ ] **Tune sub-clustering** — try `cv_threshold=0.35, min_sub_size=20, eps_fraction=0.15`; inspect Section 9 visuals; lower threshold if still few splits
- [ ] **Full run** — set `testrun=False`:
  ```python
  df_out_full, params_log_full = run_tls_hdbscan_on_df(vdf, ..., testrun=False, auto_params=True)
  df_patched_full, split_log_full = run_subclustering(df_out_full, params_log_full, ...)
  ```
- [ ] **Save output**:
  ```python
  df_patched_full.to_csv(output_path / "vdf_tls_patches.csv", index=False)
  split_log_full.to_csv(output_path / "subclustering_log.csv", index=False)
  ```
- [ ] **Validate patch counts per tissue** — check follicle numbers are biologically reasonable (lymph nodes → many, liver → few)
- [ ] **Connect to maturity analysis** — pass `tls_patch` IDs to `patch_maturity_meta-analysis.ipynb` for per-follicle maturity scoring
- [ ] **Review TISSUE_PARAMS** — validate parameters against literature per tissue group; adjust where follicle counts look off
- [ ] **Review all_one regions** — confirm `decision=all_one` regions are truly TLS-saturated, not mis-classified

---

## Repository Structure

```
follicle_meta_analysis/
├── notebook/
│   ├── NB01_Dataset_Clean.ipynb
│   ├── NB02_HDBSCAN_Patch_Delination.ipynb   ← main pipeline
│   └── patch_maturity_meta-analysis.ipynb
├── data/                  (not tracked — 2.8 GB)
├── output/                (not tracked)
├── resources/             (not tracked — 142 MB)
├── HANDOFF.md             session handoff notes
└── README.md
```

---

## Environment

```
Python 3.x | conda env: spacec
Key packages: sklearn (HDBSCAN), pandas, numpy, matplotlib, tqdm
```
