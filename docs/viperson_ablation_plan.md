# VIPerson PV / SV / SR ablation — Phase 1 audit

**Status: Phase 2 is blocked.** The four paper configurations cannot be reconstructed faithfully from the current release. Do not train, do not subset the released images, and do not fake PV/SV/SR with color jitter or random deletions.

Date: 2026-10-15  
Official repo HEAD checked: `f7f1a367` (`iSEE-Laboratory/VIPerson`, still `ID_generator/` + README only)

---

## Verdict

| Question | Answer |
|---|---|
| 1. Where are PV, SV, SR implemented? | **Described in the paper only. No implementation in the official repo or this workspace.** |
| 2. Can each component be toggled independently? | **No.** There are no generation switches. |
| 3. Do pre-generated ablation datasets exist? | **No**, not in this workspace and not in the public GitHub tree. |
| 4. Does the released dataset have provenance metadata? | **No.** The tarball is 114,738 flat `{pid}_{n}.jpg` files only. |
| 5. Reconstruct from existing assets, or regenerate? | **Must regenerate** (and first re-implement the missing stages), or obtain the authors’ ablation dumps. |
| 6. Regeneration cost? | See §6. Large: LUPerson + SD/ControlNet/StyleID/BLIP, multi-day A100, **>20 GB** downloads. |
| 7. Can FastReID register separate variants? | **Yes, later**, via extra classes/env roots. Not useful until real variant folders exist. |
| 8. Is MSMT17 available? | **No. Blocker for the MSMT17 column.** FastReID has a loader; the images are not here. |

**Do not launch the ~30–32 GPU-hour 4×3-seed training plan.** It would only re-train the already-finished full (PV+SV+SR) dataset under four names.

---

## Paper configurations (Figure 6)

Direct transfer, BoT / ResNet-50. Rank-1 only (paper does not give mAP for this figure).

| Configuration | Meaning | Market-1501 Rank-1 | MSMT17 Rank-1 |
|---|---|---:|---:|
| PV | pose variation only | 11.7 | 0.9 |
| PV + SV | pose + scene | 33.6 | 7.5 |
| SV + SR | scene + style (pose variation minimised) | 39.8 | 20.5 |
| PV + SV + SR | full pipeline = released VIPerson | 66.5 | 32.5 |

Your baseline (full released set, FastReID BoT/R50, Market only): **68.32 / 40.61 / 7.46** (R1 / mAP / mINP). That is one seed of **PV+SV+SR**, not an ablation suite.

These four rows are **four separately generated training sets**, not four filters of the same 114,738 images. Style and scene are baked into pixels. Dropping files, relabeling, or applying ColorJitter is **not** this experiment.

---

## 1. Where PV / SV / SR are implemented

### Official code (current GitHub + local clone)

Local clone: `Untitled/VIPerson/`  
Remote tree: `ID_generator/`, `LICENSE`, `README.md` (68 files). **No pose / scene / style / ControlNet / StyleID scripts.**

What exists is **identity generation only**:

| Stage | Path | Role |
|---|---|---|
| Train generator | `Untitled/VIPerson/ID_generator/personmain.py` | Latent DDPM on pose-normalised images |
| Random IDs | `Untitled/VIPerson/ID_generator/persongener_uncondi_new.py` | `c_r ~ normalize(randn)` |
| Hard IDs | `Untitled/VIPerson/ID_generator/persongener_part_dropout.py` | dropout 0.25 on CLIP text emb |
| Captions / CLIP | `Untitled/VIPerson/ID_generator/extract_text_embeddings_from_dir_clip.py` | needs a caption JSON you must supply |
| Docs | `Untitled/VIPerson/ID_generator/README.md` | Steps 1–4 stop after hard IDs |

README acknowledgements name ControlNet and StyleID but **do not ship them**.

### Paper-only (no code)

From ICCV 2025 §§3.3–3.4 (not in repo):

| Component | Paper method | Code here |
|---|---|---|
| **PV** | OpenPose on LUPerson; ControlNet IP-Adapter + BLIP identity text | **Missing** |
| **SV** | Inpaint pedestrian out of LUPerson scenes; segment generated person; paste | **Missing** |
| **SR** | StyleID [9] style from a random LUPerson (or target-set) image | **Missing** |

Identity generator training itself assumes **already pose-normalised** MALS images (`geners_plus_fixbone_sort` in `ID_generator/configs/dataset/person_folder.yaml`). That pose-normalisation script is also **missing**.

---

## 2. Independent enable / disable

**No.** There is no config flag, CLI, or stage script for PV, SV, or SR.

The only generation knobs in-repo are identity-level (`gener_path`, `hard_per_num`, dropout). They do not produce the Figure 6 variants.

---

## 3. Pre-generated ablation sets

| Asset | Present? | Notes |
|---|---|---|
| Full VIPerson (PV+SV+SR) | **Yes** | `../DATA/VIPerson/` and `VIPerson.tar.gz` (114,738 jpg, 5,000 IDs) |
| PV-only dump | **No** | |
| PV+SV dump | **No** | |
| SV+SR dump | **No** | |
| Intermediate (pre-style, pre-paste) | **No** | |
| Official GitHub extra archives | **No** | |

Do not treat the full set as four ablations.

---

## 4. Provenance metadata

Inspected released archive (earlier audit; still true):

- Layout: `VIPerson/{pid}_{n}.jpg` only (`pid` 0–4999, `n` 1–50).
- Tarball non-jpg entries: the directory itself. **No JSON, CSV, or sidecar.**
- Filename `{pid}_{n}`: identity + intra-ID index. **Not** camera, pose, scene, or style.

README advertises [VIPerson.json](https://pan.baidu.com/s/1RrbwimdasYzhQPD_4BWSlw) (code `6cxf`). **Not in this workspace.** Contents **UNKNOWN / NEEDS VERIFICATION**. Even if it is a caption/attribute list, that would not recover pre-style or pre-scene pixels.

FastReID loader: `repro/fast-reid/fastreid/data/datasets/viperson.py` — parses `{pid}_{n}.jpg`, dummy `camid=0`, no component tags.

---

## 5. Reconstruct vs regenerate

**Cannot reconstruct faithfully from current assets.**

| Approach | Faithful? |
|---|---|
| Train four times on the same 114,738 images | No — all four would be PV+SV+SR |
| Keep 1 image / ID (fake “no variation”) | No — that image already has pose+scene+style |
| Color jitter / blur as “no SR” | No — not StyleID; not the paper’s SV+SR vs PV+SV contrast |
| Randomly drop intra-ID images | That is the **intra-ID scale** study (Figure 5), not Figure 6 |
| Drop hard IDs (`*_??` after a cutoff) | That is the **hard-identity** study (Figure 4), not Figure 6 |

**Legitimate options**

1. Ask the authors (`zhangxw75@mail2.sysu.edu.cn`) for the four generated sets or the missing PV/SV/SR code.  
2. Re-implement the paper pipeline and generate four sets (high cost, not an exact official reproduction).  
3. Download `VIPerson.json` from Baidu and inspect it. If it has no per-image pipeline flags and no alternate image dumps, Phase 2 stays blocked.

---

## 6. Regeneration requirements (if you later choose option 2)

Estimates. Paper/repo do not report runtimes. Flag **>20 GB** and **>6–8 h GPU** before starting.

### Missing code you would have to write

- Pose: OpenPose + ControlNet-openpose + IP-Adapter + BLIP text  
- Scene: Inpaint-Anything / LaMa + human matting + paste  
- Style: StyleID (SD 1.4)  
- Orchestration + four output manifests  
- Trained identity-generator U-Net weights (still **not released**; issue #6)

### Data / weights

| Item | Why | Size | Required? |
|---|---|---|---|
| LUPerson | pose / scene / style references | **>100 GB** | Yes for paper-like PV/SV/SR |
| MALS + pose-normalise + BLIP | train generator if no U-Net ckpt | **>20 GB** | If authors never send generator weights |
| Autoencoder `.pt` (Drive) | latent encode/decode | ~200 MB | Identity stage |
| CLIP ViT-B/16 | text embeddings | ~335 MB | Identity / hard IDs |
| SD 1.5 + ControlNet-openpose + IP-Adapter | PV | ~10 GB | PV, PV+SV, full |
| SAM / LaMa / matting | SV | several GB | SV rows |
| StyleID + SD 1.4 | SR | several GB | SV+SR, full |
| Market-1501 | eval | local | Yes |
| MSMT17 | eval | ~2.5 GB, licence | MSMT column only |

### Compute (order of magnitude, A100; T4 much slower)

| Stage | Note |
|---|---|
| Train generator | ~15–25 h if no official U-Net |
| 5,000 IDs + hard IDs | hours |
| Cross-camera (PV/SV) on ~115k images | tens of hours |
| StyleID | **dominant**; paper-scale can be hundreds of A100-hours |
| Four configs × 3 seeds Re-ID | ~30–32 h at 2.5–2.7 h/run (your measured baseline) |

Colab T4 cannot host LUPerson or paper-scale StyleID.

---

## 7. FastReID registration (only after real folders exist)

Current loader (`repro/fast-reid/fastreid/data/datasets/viperson.py`) is registered in `fastreid/data/datasets/__init__.py`. It binds **one** tree via `VIPERSON_ROOT`.

A later, non-baseline-breaking design:

- Keep `VIPerson` + `VIPERSON_ROOT` unchanged (your 68.32 / 40.61 run).  
- Add `VIPersonAblation` (or four tiny subclasses) reading `VIPERSON_ABLATION_ROOT`.  
- YAMLs under `repro/fast-reid/configs/VIPerson/ablations/` with the **same** BoT settings as `configs/VIPerson/bot_r50.yml` / `viperson_direct_transfer_R50.yml`: 256×128, batch 128, Adam 3.5e-4, steps `[10, 40]`, 60 epochs, `NECK_FEAT: after`.  
- Keep `zero_grad(set_to_none=False)` in `repro/fast-reid/fastreid/engine/train_loop.py` (lines 248 and 346). Do not revert it.

**Do not create those YAMLs or loaders now** — there is no honest data to point them at.

---

## 8. MSMT17

| Item | Status |
|---|---|
| FastReID class | `repro/fast-reid/fastreid/data/datasets/msmt17.py` (expects `MSMT17_V1` or `MSMT17_V2`) |
| Images in this workspace | **Absent** (search under `privacy_person_reid/` found none) |
| Licence | Must request from MSMT17 authors |

**Blocker** for the paper’s MSMT17 Rank-1 column. Market-1501 is already available (`repro/datasets/Market-1501-v15.09.15/` / Windows `D:\shared_datasets\Market-1501-v15.09.15`).

---

## Mapping table (paper → what you can actually do)

| Paper config | Code switch | Data directory | Generation step | Feasible now? |
|---|---|---|---|---|
| PV | none | none | Re-implement OpenPose+IP-Adapter; skip SV/SR | **No** |
| PV + SV | none | none | PV + inpaint/paste; skip StyleID | **No** |
| SV + SR | none | none | Paper: “minimise pose variation”; then SV+StyleID | **No** |
| PV + SV + SR | n/a | `DATA/VIPerson/` or `D:\sharons\projects\VIPerson` | Already released | **Yes — this is your baseline** |

---

## Recommended plan (do not run full jobs)

1. **Stop** ablation training/generation until authors send code or the four dumps.  
2. Email authors: Figure 6 PV / PV+SV / SV+SR image archives (or scripts + seeds).  
3. Optional, low cost: download Baidu `VIPerson.json` and confirm it has no per-image pipeline flags.  
4. If you later get real folders, then (and only then):
   - register a separate dataset class / `VIPERSON_ABLATION_ROOT`;
   - add `configs/VIPerson/ablations/{pv,pv_sv,sv_sr,full}_s{0,1,2}.yml`;
   - identical train protocol; independent `OUTPUT_DIR`;
   - Market eval always; MSMT only after the licence set is on disk;
   - summary CSV: config, seed, Rank-1/5/10, mAP, mINP; report mean ± std.
5. Budget if data appears: **4 configs × 3 seeds × ~2.6 h ≈ 31 GPU-hours**, plus eval. Predeclare seeds; do not pick checkpoints by target score.

### Commands you must **not** run yet

```bat
REM These would only re-train the full PV+SV+SR set under fake names.
python tools\train_net.py --config-file configs\VIPerson\ablations\pv.yml
```

### Commands that are valid now

```bat
REM Baseline already done; do not overwrite it.
REM logs\viperson\bot_r50_bs128_e60_fixed\model_final.pth
REM Market R1 68.32 / mAP 40.61 / mINP 7.46
```

---

## Phase 3 checklist (only after real variants exist)

- [ ] Each folder generated with the stated component set (visual + generation log), not a filter of the full set  
- [ ] Identity / image / images-per-ID counts recorded  
- [ ] No Market or MSMT image in any training folder  
- [ ] Same eval protocol as baseline (`NECK_FEAT: after`, cosine, no re-rank)  
- [ ] Compare to Figure 6 with mean ± std; expect gap vs 11.7 / 33.6 / 39.8 / 66.5  
- [ ] If any config cannot be generated faithfully, **omit it** rather than substitute  

---

## Identity / hard-ID / intra-ID-scale studies (out of scope for now)

Figures 4–5 vary identity count, hard IDs, and intra-ID group size. Those **might** be approximated from the released filenames (`{pid}_{n}.jpg`) by keeping fewer IDs or fewer `n` per ID. That is a **different** paper experiment. Do not use it as a stand-in for Figure 6.
