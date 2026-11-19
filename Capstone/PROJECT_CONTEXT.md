# PROJECT_CONTEXT: FedMPS

Basis: the unzipped `fedmps-capstone.zip` (102 files), read statically; no code was executed. No `.git`, `tests/`, CI or linter config exists, so history is [unverified].

## 1. Summary
FedMPS is a VIT-AP capstone (README.md): privacy-preserving missing-person search over CCTV. Users are a server operator (opens cases) and per-camera node operators (review alerts). Python (3.13 laptop per HOW_TO_RUN.md; AWS logs show 3.10/3.12), PyTorch, ONNX Runtime/insightface (ArcFace `buffalo_l` `w600k_r50`, SCRFD), supervision (ByteTrack), OpenCV, SQLite, Streamlit. The 43.6 M-parameter backbone is frozen. Only a residual MLP adapter is trained with FedProx across 5 simulated nodes: 66,240 params/0.26 MB per update (`adapter_final.pt`), or 263,424 params/1.05 MB in the final VGGFace2-HQ run (`adapter_vgghq_h256.pt`). Final data: 628,699 images, 4,605 identities (HOW_TO_RUN.md §2.6). Targets: accuracy parity with centralised training and real-CCTV alerting.

## 2. Architecture
- `config.py`: all hyperparameters, seed 1337. `contracts.py`: artefact shapes, `assert_partition`.
- `data/degrade.py`: blur→downscale→gamma→noise→JPEG→upscale, one fixed "camera profile" per node. `data/partition.py`: identity-disjoint split, 5 nodes, size ratio 3.0.
- `models/backbone.py`: frozen embedder, `align_faces`; `adapter.py`: 512→h→512 residual MLP, last layer zero-initialised; `head.py`: alternative (bn2 + rank-16 LoRA on fc).
- `fl/node.py`: InfoNCE local training + FedProx; `fl/server.py`: weighted delta averaging, DP and masking hooks; `fl/loop.py`: local-only, federated, centralised (= one pooled client).
- `privacy/dp.py` (clip, Gaussian noise, RDP accountant), `mask.py` (Diffie–Hellman pairwise masks), `template.py` (Fisher dimension selection, binarise, XOR key, Hamming).
- `eval/`: `metrics.py` (open-set rank-1, TPIR@FPIR), `calibrate.py` (centering, blended Procrustes, hubness), `experiments.py` (8 experiments).
- `pipeline/`: SCRFD (Haar fallback), ByteTrack (IoU fallback), `quality.py` gate, `match.py` (`Matcher`, `TrackVoter`, `VideoPipeline`); `store.py`: SQLite cases/templates/alerts/audit.
- `application/`: Streamlit server (8501), node console (8502); `training/`, `cctv_benchmark/`, `report_figures/`.

Flow: images→align→clean+degraded embeddings→node deltas→(clip/noise)→(masked sum)→global adapter→checkpoint. App: case photo embedded in memory→per-node keyed 64-byte template (+float embedding in default cosine mode)→node: detect→track→gate→embed→adapter→match→3-of-5 vote→PENDING alert→human confirm→server sees crop.

## 3. Key design decisions
1. **Freeze backbone, federate adapter.** Rejected: federating the backbone (658× larger, 174.4 MB; `Server.payload_report`). Trade-off: adapter gains ≈0 (§7).
2. **Align once on the clean image, then degrade** (`align_faces`, `align_once`). Rejected: aligning each branch separately (detector failures on tiny faces pollute the gap). Evidence: `alignment_ab.json`.
3. **Deltas + FedProx + InfoNCE masking same-identity negatives** (`node.py`). Alternative FedAvg (mu=0) swept: best mu 0.1 gains 0.0007 rank-1. Trade-off: strong mu (3.0 default) keeps updates tiny.
4. **Masks applied to weighted updates** (`mask.py` docstring: unweighted masks leave a residue). Trade-off: single-masked; dropout recovery reveals pairwise masks.
5. **Update-level DP, noise/√n when masking is on; clip C at 70th-percentile observed norm** (`calibrate_clip_norm`). Rejected: C=1.0 (docstring: ~80% clipped). Trade-off: no usable ε.
6. **512-bit keyed templates.** Rejected: 96 bits (`config.py`: −53 points). Trade-off: default `match_mode` is "cosine", which ships the float embedding to nodes.
7. **k-of-n track voting, quality gate before embedding, threshold 0.24.** Rejected: per-frame alerts. Trade-off: 0.24 suits a small watchlist (config comment: ~0.30 at 1,750 identities).
8. **Human review gate, audit log, case TTL (30 days default), revoke on close** (`store.py`). Trade-off: enforced by query convention in one shared SQLite file, not access control.

## 4. Hard problems and bugs
- **Unaligned crops.** Symptom: 3×3 calibrated rank-1 0.2064 (unaligned) vs 0.9848 (aligned), 100k images (`alignment_ab.json`). Cause: ArcFace given dataset-style crops. Fix: `models/backbone.py::align_faces`; `max_align_miss_rate` 0.05 aborts. Test: A/B run only; the backbone self-check uses the dummy embedder.
- **Adapter overfitting.** Symptom: held-out rank-1 peaks at round 1, then falls (`regularisation_sweep.json`: 0.9094→0.8396 at h=256); local-only 0.9286 < frozen 0.9474; a full-epoch-per-round HQ launch overfit and was killed (`experiments_vgghq_stopped_overfit.log`, node_0 0.9166). Cause: free map overfits training identities (`calibrate.py` docstring). Fix: hidden 64, identity-disjoint early stopping (`loop.py::split_node_identities`), FedProx, cosine LR, `--epoch-frac 0.15`. Test: `03_check_local_only.py` → local-only with early stop + prox 0.9475 (`local_only_check.json`).
- **Membership-inference bug.** First run used `val_idx` (trained-on) as non-members. Fix: `experiments.py::_sample_holdout` holds back 20% and retrains. Result AUC 0.496; no automated test.
- **Template collapse.** 96 bits gave rank-1 0.3962 vs 0.9226 float; quantisation, not dimension reduction, was the cause. Fix: 512 bits (`template_sweep_wide.json`).
- **Infra.** CUDA 13 torch vs CUDA 12 onnxruntime-gpu cuDNN clash; per-worker detectors 51–61 img/s vs 121 img/s with `--gpu-in-main`; prep hit `BrokenPipeError` near row 30,720 and was resumed twice (`prep_vgghq*.log`, `--resume-from`).

## 5. Concurrency, errors, failure behaviour
- FL is a sequential single-process loop; no network, retries or timeouts. `SecureAggregator.aggregate(dropped=...)` exists and self-checks, but no caller passes it. `clients_per_round` subsampling exists (default all nodes). All DH keypairs are generated in one process from `seed+4242`.
- Training: `01b_prepare_stream.py` resumes by row; experiment results are saved after each experiment.
- App: scans run as a `subprocess.Popen`; the UI polls a progress file every 0.5 s; Stop calls `terminate()`; failures surface from the log tail (`node_app.py::finish_scan`).
- Shared state: one SQLite file, new connection per call (`timeout=10`); audit row written after, not in the same transaction as, each mutation; `review_alert` does not check current status.
- Bad input: `ContractError` on shape/partition violations; unreadable video raises `FileNotFoundError`; `load_adapter` refuses an adapter trained on a different backbone; `load_watchlist` skips cases with key-fingerprint mismatch; expired/closed cases stop serving templates.

## 6. Testing and quality
No pytest/unittest, CI, linter or sanitizer. 21 modules have `__main__` self-checks on synthetic data (HOW_TO_RUN.md §6; 141 `assert` lines by grep), writing to a temp dir via `Paths.under` after a self-check once overwrote the real adapter (`config.py`). Covered: aggregation arithmetic, mask cancellation (<1e-5), template unlinkability/revocation, voter, store revocation/TTL. Not tested: UI, real-backbone alignment, store concurrency, dropout inside the loop, `head.py`, multiple seeds (single seed, `n_bootstrap: 0`).

## 7. Measured results
- VGGFace2 run (`experiments_vgg_big_threeway.json`; 172,620 images, 3,021-identity gallery, 15,105 mate / 18,126 non-mate probes, 3 photos × 3 frames, scoring layer on, 30 rounds, mu 3.0, round 1 kept). Rank-1 / TPIR@FPIR 1% / TAR@FAR 0.1% (6,000 pairs): frozen 0.9474/0.7968/0.6287; federated 0.9473/0.8020/0.6193; centralised 0.9476/0.8043/0.6087; local-only mean 0.9286.
- HQ final (`experiments_vgghq.json`; 200 rounds, h256, mu 0.3; 691-identity gallery, 200,000 pairs): rank-1 frozen 0.9733, federated 0.9734, centralised 0.9733, local-only 0.9620; TPIR@1% 0.9054→0.9107.
- Degradation, 1×1: 0.9179 clean→0.5240; 3×3: 0.9962→0.9294 raw cosine, 0.9474 with scoring layer (`experiments_vgg_big.json`).
- Masking: identical rank-1; overhead 1.008× (10 rounds) / 1.031× (30 rounds, HQ).
- DP (vgg_big): ε=251.9 at σ=0.3, rank-1 0.9391→0.8751; HQ: 0.9707→0.0566.
- Templates (HQ): 512 bits 0.9244 vs float 0.9715. MIA AUC 0.496 (HQ 0.501).
- ChokePoint video (`cctv_chokepoint_video.json`; 24 sequences, 29 people, g4dn T4, threshold 0.24, vote 3/5): 1,988/2,088 enrolled walk-throughs alerted (95.2%); crowd false alarms 88/2,232 (3.9%) frozen, 83 federated; 0 wrong-identity; median 1.3 s; gate passed 4,000/7,433 faces.

## 8. Known limitations / next
Simulated nodes in one process; hard-coded `template_master_key` and node-side key re-derivation; scoring layer absent from live matcher; threshold tuned on demo footage; single seed; ChokePoint enrolment from the same cameras; WebFace600K/VGGFace2 overlap unchecked (thesis); DP has no usable point; fix dropout (Shamir self-mask); per-site thresholds.

## 9. Likely interview questions
1. Why freeze ArcFace? `Server.payload_report`.
2. Why does the adapter add ≈0? `loop.py` warnings; `experiments_vgg_big_threeway.json`.
3. How can federated ≥ centralised? `loop.py::run_centralised`.
4. How do masks cancel with weights? `mask.py` docstring.
5. Why does DP fail? `dp.py::noise_std`; `experiments_vgghq.json`.
6. What does the 0.24 threshold depend on? `config.py` PipelineConfig.
7. Is the data leak-free? `contracts.py::assert_partition`; `partition.py`.
8. What do templates protect? `template.py` docstring; `match.py::WatchlistEntry`.
9. How was fairness of local-only ensured? `03_check_local_only.py`.
10. Why is 95.2% optimistic? `chokepoint_video.py`; thesis §4.10.

## 10. Discrepancies and unverified claims
1. README headline numbers are the first run (70% held out, round 1); HOW_TO_RUN.md and thesis v3 call the HQ run final; `config.py` defaults (30/70 split, hidden 64, mu 3.0, 200 rounds) match neither run exactly.
2. README says the demo uses 66,240-parameter `adapter_final.pt`; `match.py::DEPLOY_ADAPTERS` loads `adapter_vgghq_h256.pt` first.
3. README links a thesis PDF and `docs/thesis/`; only `FedMPS_Capstone_Thesis_v3.docx` exists. README says 10 figures; `figures/` has 11 PNGs. `artifacts_vgg_big/`, `media/`, `fedmps.sqlite` are absent.
4. README "512-bit templates cost about 9 points" comes from `template_sweep_wide.json` (older, smaller run); HQ shows 4.7; vgg_big templates stop at 256 bits (−28.7).
5. `quality.py` says ~70% of faces are discarded; measured 46.2% (3,429 `too_small`, 4 `too_blurry`).
6. README omits `unattributed_alerts` (4 frozen, 7 federated); `cctv_chokepoint_video_vgghq.json` shows federated 91 crowd alerts vs 88 frozen.
7. README's DP "tolerable" point costs 6.4 rank-1 points and fails in HQ.
8. Stale text: `partition.py` says 70% held out (default 30%); `config.py` unaligned/aligned cosine 0.51/0.70 vs `alignment_ab.json` 0.292/0.514.
9. Thesis says secure aggregation has no dropout recovery; `mask.py` implements it (unused).
10. "Consecutive frames" in the probe protocol are randomly grouped images (`build_eval_sets`), not video frames. FedProx mu 3.0 lies outside the swept {0, 0.01, 0.1, 1.0}; its justification is [unverified]. `onnx` (needed by `head.py`) is missing from `requirements.txt`.
