# Depth-Inference Capable and Robust PhysTwin — Report Skeleton

Working outline derived from the *Digital Twin V2* weekly-update slides (weeks 3–10 + final overview).
Fill in / delete freely. Numbers in `[...]` are things only you can confirm (metrics, counts, runtime).

**Suggested chapter layout (rename to taste):**

| Ch. | Theme | Source material in slides |
| --- | --- | --- |
| 1 | Introduction | goals of DigitalTwin V2, core problems |
| 2 | Background & Related Work | prior research section (VDA, StableDPT, NVDS, DUSt3R, CoTracker, PhysTwin) |
| 3 | System Design (pipeline) | new pipeline structure diagram, misc improvements |
| 4 | Temporal Depth Smoother | core idea, architectures, all four losses, training |
| 5 | Evaluation | results, comparisons, findings |
| 6 | Discussion & Limitations | core limitations, data collection problems |
| 7 | Conclusion & Future Work | final key tasks, next steps |

---

## Abstract (draft already written in `main.tex`)

Covers: motivation (real2sim from video, no manual modelling) → gap (PhysTwin needs RGBD and is brittle)
→ what we built (DUSt3R poses + DA3 depth + learned temporal smoother + ground alignment on top of PhysTwin)
→ key technical idea (residual smoother trained with VDA as motion oracle, no ground-truth depth)
→ evaluation (PhysTwin data + ~30 self-collected sessions)
→ headline finding (working warp models from RGB only; quality gated by capture conditions)
→ limitations one-liner.

---

## Chapter 1 — Introduction

- **1.1 Motivation.** Training robots digitally requires a digital replica; digitising an interaction today is manual. Can we build the replica automatically from video?
- **1.2 Problem statement.** PhysTwin takes RGBD video → tracked motion points → spring–mass "warp" model → interactive twin. Two obstacles: (a) depth sensors are not always available, (b) the pipeline breaks randomly / is brittle.
- **1.3 Aims.** (i) accept any RGB video without depth, (ii) improve robustness/performance, (iii) improve the physics model's realism.
- **1.4 Contributions.**
  1. Depth-inference-first pipeline: automatic pose (DUSt3R + heuristic scale) and depth (DA3 fake sliding window) inference.
  2. A ground-truth-free temporal depth smoother that learns a residual over DA3, supervised by VDA as a motion oracle.
  3. Robustness engineering: resolution-agnostic data path, idempotent processing, ground/extrinsics alignment, fallbacks for shape prior and tracking.
  4. Empirical characterisation of *when* the pipeline works (object size, motion speed, controller occlusion, camera setup).
- **1.5 Report structure.** One short paragraph listing chapters.

## Chapter 2 — Background and Related Work

- **2.1 PhysTwin and real2sim digital twins.** RGBD in, tracked points + controller, spring–mass warp model trained from the points, interactive manipulation at inference.
- **2.2 Monocular / video depth estimation.** Depth Anything (v2/v3), metric depth, relative depth; the recurring tension: good spatial consistency (DUSt3R, MASt3R, DA3) vs good temporal smoothness (VDA, oVDA), and multi-camera support.
- **2.3 Pose estimation.** DA3 pose vs DUSt3R/MASt3R pairwise 3D correspondence; heuristic scale.
- **2.4 Temporal depth stabilisation (closest prior art — worth a comparison table).**
  - VDA (2024): temporal attention at multiple decoder scales, TGM loss, segment-wise sliding window.
  - StableDPT (2025): temporal attention in DPT backbone, TGM loss.
  - NVDS / NVDS+ (2023/24): post-hoc stabiliser, sliding-window cross-attention + GMFlow optical flow loss; metric, needs sensor GT.
  - "Towards Practical Consistent Video Depth Estimation" (Li et al.).
  - **Gap:** these either need labelled/sensor data, or only handle a single monocular stream. We want a multi-view, no-GT, drop-in post-processor.
- **2.5 Point tracking.** CoTracker (used), MVTracker / MVTrack, SpaTrackerV2, spatial artefacts; why tracking quality is downstream of depth quality.
- **2.6 Segmentation and reconstruction.** SAM / Grounded SAM, Trellis, 3D Gaussian splats, 4DGS as an alternative pipeline direction.
- **2.7 Summary of the gap** the project fills.

## Chapter 3 — System Design

- **3.1 Overview.** The pipeline diagram from the slides, colour-coded (new vs modified PhysTwin components):
  `3× RGB video → DUSt3R pose estimation + heuristic scaling → DA3 depth inference → modified PhysTwin data processing → extrinsics alignment → modified PhysTwin gaussian splats → PhysTwin optimization + training → warp model`.
- **3.2 Data capture.**
  - Hardware: what you actually recorded with (phones + DJI; then the depth-camera setup).
  - ~8 initial sessions, then ~30 sessions (cloth, pad, bottles, mugs, stuffed animals).
  - Problems: different intrinsics/focal lengths/distortion, rolling shutter, auto-exposure/focus, not time-synchronised, software-aligned start times.
  - Manually measured intrinsics/extrinsics for the second dataset.
- **3.3 Pose estimation.** Why DA3 poses are bad (intrinsics *and* extrinsics off → misaligned 3D projections); DUSt3R pairwise correspondence is better; heuristic scale.
- **3.4 Depth inference.** "Fake" sliding-window / streaming DA3: repeat static views as "dynamic", process overlapping chunks, share weights over overlap. Remaining issue: large chunks break Umeyama alignment (singular matrices). Newer models tried: Pi³ (no), others.
- **3.5 Temporal depth smoother.** (Details in Ch. 4; here only where it slots in.)
- **3.6 Extrinsics / ground alignment.** Ground plane detection (RANSAC / bounding box), rotate extrinsics until ground is flat, translate above ground, propagate to tracks and meshes. Fixes over-stiff rope/mat behaviour and keeps ground = z = 0 for training.
- **3.7 Segmentation, shape prior, tracking.** Mask quality vs descriptive names, ARAP fails on non-watertight meshes → average-translation fallback, morphological occlusion to join split masks, CoTracker resolution bug, motion-filter looseness.
- **3.8 The PhysTwin stage.** Gaussian splats + warp model training; what "the broken WARP model" was and what fixed it; idempotency bug (resolution handling of RGB vs intrinsics/extrinsics).
- **3.9 Implementation notes.** Refactoring, resolution-agnostic data path, fallback mechanisms, reproducibility.

## Chapter 4 — Temporal Depth Smoother

- **4.1 Motivation.** DA3 is jittery (per-frame inference, inconsistent depth boundaries) → poor reconstruction, noisy training data; VDA is smooth but geometrically weak and monocular.
- **4.2 Core idea.** DA3 + RGB → latent space → smooth depth; predict a *residual correction*, not a whole depth map. VDA supplies temporal truth, DA3 supplies geometry. No ground truth needed.
- **4.3 Architecture.** Temporally-Dilated Residual U-Net (include the figure/diagram); ~4M–5M parameters (compute-limited); why generative/DPT-style options were considered and rejected.
- **4.4 Losses.** Four terms, give each its own subsection with equations from the slides:
  - TGM (temporal gradient matching) — frame-to-frame depth *change* matches VDA; only a motion oracle, scale-relative; `k` controls short vs long flicker.
  - TV (temporal total variation) — penalise temporal acceleration; exponential gate suppresses only *unnatural* oscillation, per-frequency via `k`.
  - Fidelity — stay close to DA3, median-scale aligned, VDA gate weighting stationary regions high and relaxing in real motion.
  - Geometric — Sobel edge mask on RGB, finite-difference gradient match at edges (unnormalised; effect uncertain — say so honestly).
- **4.5 Training.** ~5M params, 50 epochs, PhysTwin + self-collected data, batch sizes / hardware `[...]`, note that hyper-parameters are not tuned.
- **4.6 Inference.** Chunked, adapted from VDA, arbitrary resolution with padding, fast.
- **4.7 Properties.** Drop-in on any flickery depth map, no GT, resolution agnostic, fast, but blurry; discussion of per-clip overfitting as a legitimate strategy.

## Chapter 5 — Evaluation

- **5.1 Setup.** Datasets (PhysTwin released vs self-collected), evaluation protocol, and metrics — state clearly what is measured: depth temporal error, spatial consistency, tracking accuracy, and end-to-end warp-model quality (this is the weakest-specified part in the slides; decide and justify).
- **5.2 Depth quality.** Raw DA3 vs smoothed vs VDA, qualitative video frames + a quantitative table `[...]`.
- **5.3 Ablations.** Loss-term ablation, `k` values, fidelity gating on/off, resolution sensitivity.
- **5.4 End-to-end results.** Pipeline on PhysTwin data vs on own data; a couple of the "cool videos" as figures/links; tracking artefacts noted.
- **5.5 Findings (quantify or at least tabulate).**
  - Large, slow-moving objects work well.
  - Minimal controller/hand occlusion works well; significant occlusion corrupts shape prior, alignment and CoTracker.
  - Small, fast objects fail: 3D consistency dies, depth variance too high.
  - Controller must stay in contact with the object throughout.
- **5.6 Failure cases and negative results.** Pi³ for depth (no), MVTracker / MVTrack for 3D tracking (worse), Trellis + CoTracker versions already current (no gain), ARAP on non-watertight meshes, Umeyama failure on large chunks.

## Chapter 6 — Discussion and Limitations

- **6.1 The central diagnosis.** PhysTwin's architecture is 2D-first (2D masks, 2D CoTracker) and lifts to 3D using depth; so depth quality dominates everything. Needed: a model that is *both* spatially consistent and temporally smooth — no existing model does both with multi-camera support.
- **6.2 Remaining limitations.**
  - No true streaming: the current workaround breaks for large chunk sizes.
  - Data capture: three different cameras, no time sync (frame-synced ≠ time-synced), rolling shutter, auto-exposure/focus → flicker and progressive spatial drift.
  - Environment interaction: static-environment assumption, plane colliders only; rope needs unrealistic stiffness.
  - Smoothing can blur depth.
- **6.3 Alternative designs considered.** (a) adapt streaming + MVTrack inside the PhysTwin pipeline; (b) full rewrite on 4DGS + 3D Grounded SAM + 3D tracking with no explicit poses. Trade-offs and why neither was taken.
- **6.4 Threats to validity.** Small dataset, unmeasured intrinsic/extrinsic error, hyper-parameters untuned.

## Chapter 7 — Conclusion and Future Work

- **7.1 Summary.** A pipeline that turns multi-view RGB video into a working warp digital twin, and a general depth smoother that needs no ground truth.
- **7.2 Future work.**
  - Real multi-view streaming inference; adapt the actual `da3_streaming` model to multi-pose input.
  - Better data collection: synchronisation cues (clap/flash), fixed exposure/focus, measured intrinsics, watch for fisheye distortion.
  - Robust 3D point tracking (SpaTrackerV2 and newer), or move tracking into 3D.
  - Improve depth 3D coherence; better prior alignment; environment-aware colliders (arbitrary rigid bodies).
  - Clean up and publish the code.

## Appendices

- A. Hyper-parameters, loss weights, training details.
- B. Extended qualitative results / links to result videos.
- C. Code listings for the smoother and the alignment stages.
- D. Full pipeline diagram, larger version.

---

## Figures and tables checklist

- Fig: motivation / manual vs automatic digitisation.
- Fig: full pipeline diagram (colour-coded new vs modified) — Ch. 3.
- Fig: pipeline version history / problems-fixed timeline — Ch. 1 or 3.
- Fig: DA3 jitter vs VDA smoothness illustration — Ch. 4.
- Fig: residual U-Net architecture — Ch. 4.
- Fig: loss term diagram (VDA gate, Sobel mask, k) — Ch. 4.
- Table: prior temporal-depth-stabilisation methods (inputs, supervision, GT needed, multi-view?) — Ch. 2.
- Table: depth metrics raw DA3 / smoothed / VDA — Ch. 5.
- Table: per-object outcome summary (object, size, speed, occlusion → good/bad) — Ch. 5.
- Table: ablation of losses and `k` — Ch. 5.

## Open questions to resolve while writing

- What are the actual quantitative metrics, and is end-to-end warp quality measurable at all?
- Which of the ~30 new sessions are reported? Report the sampling policy either way.
- Is DUSt3R still the pose source, or DA3 with DUSt3R fallback — pick one story.
- Parameter count: slides say ~4M in one place and ~5M in another.
