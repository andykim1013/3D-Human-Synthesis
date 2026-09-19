# 1. Project Title

**3D Human Synthesis: DECA–SMPL-X Integration** — single-image 3D human avatar
reconstruction that combines **DECA** (detailed facial geometry) with
**PIXIE / SMPL-X** (full-body pose and shape) into one mesh, using five
different face–body integration strategies (Method A–E).

## 2. Overview

Given one face image and one body image, this project reconstructs a 3D face
mesh with [DECA](https://github.com/YadiraF/DECA) and a 3D full-body SMPL-X
mesh with [PIXIE](https://github.com/YadiraF/PIXIE), then integrates the two
into a single human representation using five alternative methods (Method
A–E). The entire pipeline runs in one Jupyter notebook, `main.ipynb`, on CPU.

## 3. Research Objective

Face-only and body-only 3D reconstruction models are usually trained and
evaluated separately. The objective of this project is to combine DECA's
high-resolution facial expression detail with PIXIE/SMPL-X's full-body pose
and shape into a single, topologically consistent 3D human representation, and
to compare several integration strategies against each other using
image-based reconstruction metrics.

## 4. Pipeline

`main.ipynb` executes as a sequence of cells, in this order:

```
Input images (face + body)
        │
        ▼
   PIXIE  (Step 3) ──▶ full-body SMPL-X mesh, joints, body parameters
        │
        ▼
   DECA   (Step 4) ──▶ detailed face mesh, expression/jaw parameters
        │
        ▼
   Method A / B / C / D / E ──▶ integrated face+body SMPL-X / stitched mesh
        │
        ▼
   .obj files saved to result_output_jupyter/
        │
        ▼
   Evaluation cell ──▶ render vs. ground-truth image, compute metrics,
                        write metrics_summary.csv
```

Cells must be run top to bottom; later cells (Method A–E, evaluation) reuse
objects created in Step 1–4 (e.g. `deca`, `pixie_model`, `testdata`,
`codedict_raw`, `codedict_mod`, `save_dir`).

## 5. Integration Methods

As implemented in `main.ipynb`:

- **Method A — PCA-based adaptive slicing + Laplacian seam smoothing**:
  normalizes the body around the neck joint, aligns it upright using the spine
  vector, slices both meshes at the neck, resamples the boundary loop to
  reconnect them, and applies local Laplacian smoothing at the seam.
- **Method B — PCA-based adaptive slicing + harmonic mesh pairing**: same
  neck-slicing setup as Method A, but the boundary is warped with a harmonic
  field, stitched with a multi-resolution "zipper" pass, and smoothed with
  bi-Laplacian curvature optimization.
- **Method C — Heterogeneous latent/parameter fusion**: extracts DECA's face
  parameters and injects them into the SMPL-X parameter/latent space directly
  (no geometric cut/seam), aligning the skeletons via SVD.
- **Method D — Confidence-weighted feature fusion**: runs a single PIXIE
  `encode()` pass for the body, extracts DECA's expression (`exp`) and jaw
  pose, converts the jaw rotation to Euler angles, overwrites
  `codedict["exp"]` / `codedict["jaw_pose"]` with the DECA values, and calls
  `pixie.decode()` once to produce a single seamless SMPL-X mesh.
- **Method E — Keypoint reprojection / detail transfer**: starts from Method
  D's mesh, extracts a DECA detail-displacement map, converts it from FLAME UV
  space to SMPL-X UV space via a cached FLAME→SMPL-X mapping, bilinearly
  samples it per vertex, and applies it along vertex normals to add
  high-frequency detail (wrinkles/contours) while keeping the topology fixed.

Each method saves an "original expression" mesh and a "modified expression"
mesh (expression transplanted from a second, "driving" face image) — see
[Output](#13-output) for the exact filenames actually written by the code.

## 6. Evaluation Metrics

The evaluation cell (Step 10 / "성능지표") renders each output mesh from the
same viewpoint as the ground-truth body image, crops/masks to the subject, and
computes:

- **L1 (Photometric)** — mean absolute pixel-intensity difference.
- **SSIM** — structural similarity between rendered and ground-truth image.
- **PSNR** — peak signal-to-noise ratio.
- **LPIPS** — learned perceptual patch similarity (AlexNet backbone,
  `lpips` package).
- **Cosine Similarity** — cosine similarity of VGG16 deep features.
- **Time (s)** — wall-clock time for rendering + metric computation per mesh.

Results are printed to the console and written to
`result_output_jupyter/metrics_summary.csv` as a pandas DataFrame.

## 7. Repository Structure

What this repository actually tracks in Git:

```
3D_Human_Synthesis/                 <- repository root
├── README.md
├── THIRD_PARTY_NOTICES.md
├── environment.yaml
├── .gitignore
├── main.ipynb
└── input/
    ├── DECA/.gitkeep
    └── SMPL-X/.gitkeep
```

What you need to add locally (not tracked in Git — see sections 9–11):

```
├── input/
│   ├── DECA/       <- your own face image(s)
│   └── SMPL-X/     <- your own body image(s)
└── models/
    ├── DECA/         <- cloned from https://github.com/YadiraF/DECA (+ patch, see THIRD_PARTY_NOTICES.md)
    ├── PIXIE-master/ <- cloned/downloaded from https://github.com/YadiraF/PIXIE
    └── smplx/        <- cloned from https://github.com/vchoutas/smplx
```

## 8. Environment

Python 3.8, CPU-only PyTorch. `environment.yaml` pins the exact package
versions used during development.

```bash
conda env create -f environment.yaml
conda activate deca_fusion
```

(`deca_fusion` is the environment name declared in `environment.yaml`.)

## 9. Third-party Dependencies

Full details, license text links, citations and local-modification notes are
in [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md). Summary:

| Project | Official repo | Role in this project | Extra weights needed |
|---|---|---|---|
| **DECA** | https://github.com/YadiraF/DECA | Single-image detailed face reconstruction (`decalib.deca.DECA`) | `deca_model.tar`, FLAME model files (registration at https://flame.is.tue.mpg.de) |
| **PIXIE** | https://github.com/YadiraF/PIXIE | Single-image full-body SMPL-X reconstruction (`pixielib.pixie.PIXIE`) | `pixie_model.tar`, `SMPLX_NEUTRAL_2020.npz`, `utilities.zip` (registration at https://pixie.is.tue.mpg.de/) |
| **SMPL-X** | https://github.com/vchoutas/smplx | SMPL-X body layer / joint regression (`smplx.create(...)`) | SMPL-X v1.1 model files (registration at https://smpl-x.is.tue.mpg.de) |

All three require you to individually register and agree to their own license
before downloading their weights — this project cannot redistribute them (see
[Model Files](#10-model-files) and [Third-party Licenses](#15-third-party-licenses--acknowledgements)).

## 10. Model Files

**No model weights, topology/template files, or other DECA/PIXIE/SMPL-X data
are included in this repository** (see `.gitignore`: `models/DECA/`,
`models/PIXIE-master/`, `models/smplx/` are all excluded). You must obtain the
source code and data yourself and place them so that the layout matches what
`main.ipynb` expects:

```
models/
├── DECA/
│   ├── decalib/                      <- from the official DECA repo (+ patch, see THIRD_PARTY_NOTICES.md)
│   └── data/
│       ├── deca_model.tar            <- required (main model weights)
│       ├── generic_model.pkl         <- required (FLAME model, from FLAME2020.zip)
│       ├── landmark_embedding.npy    <- required (FLAME() crashes without it)
│       ├── uv_face_mask.png          <- optional (Method E detail masking; silently skipped if missing)
│       └── uv_face_eye_mask.png      <- optional (Method E detail masking; silently skipped if missing)
│
├── PIXIE-master/
│   ├── pixielib/                     <- from the official PIXIE repo
│   └── data/
│       ├── pixie_model.tar           <- required (main model weights)
│       ├── SMPLX_NEUTRAL_2020.npz    <- required (SMPL-X body model used by PIXIE)
│       ├── smplx_extra_joints.yaml   <- required (SMPLX() crashes without it)
│       ├── SMPLX_to_J14.pkl          <- required (SMPLX() crashes without it)
│       ├── SMPL_X_template_FLAME_uv.obj  <- required for Method E (FLAME→SMPL-X UV topology)
│       └── flame2smplx_tex_1024.npy      <- required for Method E (cached FLAME→SMPL-X UV mapping)
│
└── smplx/
    ├── smplx/                        <- from the official smplx repo (`pip install`-able package)
    └── models/
        └── smplx/
            ├── SMPLX_NEUTRAL.pkl (or .npz)  <- required
            ├── SMPLX_MALE.pkl / .npz        <- only if you use gender-specific models
            └── SMPLX_FEMALE.pkl / .npz      <- only if you use gender-specific models
```

The "required" vs. "optional" status above was determined by tracing which
config paths are read unconditionally in `decalib/utils/config.py` +
`decalib/deca.py`, and `pixielib/utils/config.py` + `pixielib/models/SMPLX.py`,
against what `main.ipynb` actually calls. Files not listed here (e.g. PIXIE's
`smplx_hand.obj`, `smplx_tex.obj`, `MANO_SMPLX_vertex_ids.pkl`,
`SMPL-X__FLAME_vertex_ids.npy`) belong to PIXIE's `visualizer.py`, which
`main.ipynb` never imports, so they are not needed to run this pipeline.

Weights are not included because DECA/PIXIE/SMPL-X's licenses restrict
redistribution to third parties, and because they are large binary files
(hundreds of MB combined) — see [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).

## 11. Input Preparation

**No sample images are included in this repository.** `input/DECA/` and
`input/SMPL-X/` only contain a `.gitkeep` placeholder so the folder structure
exists after cloning.

- You must supply your own face image(s) in `input/DECA/` and full-body
  image(s) in `input/SMPL-X/` — images you have the rights to use.
- If you use a personal photo, be aware that phone/camera photos frequently
  embed EXIF metadata (device model, timestamp, and sometimes **GPS
  coordinates**). Strip or check this metadata before sharing images publicly.
- `main.ipynb` currently points `BODY_IMAGE_PATH`, `SOURCE_FACE_IMAGE_PATH`
  and `DRIVING_FACE_IMAGE_PATH` at specific example filenames
  (`body5.jpg`, `face4.jpg`, `face1.png`) — edit these variables in Step 3 /
  Step 4 to point at your own filenames.

## 12. Running the Project

`main.ipynb` sets `PROJECT_ROOT = Path.cwd()` in its first cell (Step 1) and
derives `INPUT_DIR` / `MODEL_DIR` from it. This means:

1. **The notebook's kernel working directory must be this repository's root**
   (the folder containing `main.ipynb`, `input/`, and `models/`). In
   Jupyter Notebook / JupyterLab this is the default when you open the
   notebook from this folder. If you use an IDE (e.g. VS Code) whose working
   directory can differ from the notebook's own folder, verify/set the
   working directory to the repository root before running Step 1.
2. Create the conda environment and select it as the notebook kernel (see
   [Environment](#8-environment)).
3. Clone/download DECA, PIXIE and SMPL-X into `models/` as described in
   [Model Files](#10-model-files).
4. Add your own images to `input/DECA/` and `input/SMPL-X/` (see
   [Input Preparation](#11-input-preparation)) and edit the corresponding path
   variables in Step 3 / Step 4.
5. Run the cells in order: Step 1 (paths/env) → Step 2 (helper functions) →
   Step 3 (PIXIE body) → Step 4 (DECA face) → Method A → Method B → Method C →
   Method D → Method E → Step 10 (evaluation).

## 13. Output

Running the notebook creates a `result_output_jupyter/` folder (auto-numbered
to `result_output_jupyter1`, `result_output_jupyter2`, ... if the base name
already exists) containing:

- `0_pixie_body_raw.obj`, `0_pixie_body_upright.obj` — PIXIE body mesh.
- `1_deca_raw.obj`, `2_deca_mod.obj` — DECA face mesh (raw / expression-transplanted).
- `Method_A_original.obj`, `Method_A_modified.obj`
- `Method_B__original.obj`, `Method_B__modified.obj` *(double underscore, as written by the code)*
- `Method_C_original.obj`, `Method_C_Modified.obj` *(capital "M", as written by the code)*
- `Method_D_original.obj`, `Method_D_modified.obj`
- `Method_E_original.obj`, `Method_E_modified.obj`
- `metrics_summary.csv` — the evaluation table (see [Evaluation Metrics](#6-evaluation-metrics)).

None of these output files are tracked in Git (`.gitignore` excludes
`result_output_jupyter*/`, `*.obj`, and `metrics_summary.csv`) — they are
regenerated every time you run the notebook.

## 14. Known Limitations

- **Windows-oriented development environment**: `environment.yaml` includes
  Windows-only packages (`pywin32`, `win32-setctime`), and the pipeline was
  developed/tested on Windows with CPU-only PyTorch.
- **Model weights are not distributed**: DECA/PIXIE/SMPL-X weights and
  auxiliary data must be downloaded separately by each user under their own
  license agreement (see [Model Files](#10-model-files)).
- **External project dependency, partially patched**: the pipeline depends on
  local copies of DECA and PIXIE source code; the local DECA copy has been
  modified (pytorch3d renderer removed) and is not reproduced by a plain clone
  of the official repository without re-applying that patch (see
  [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)). PIXIE and SMPL-X appear
  unmodified, but PIXIE was not verified with a full byte-for-byte diff
  against upstream.
- **`PROJECT_ROOT` is the notebook's working directory, not a fixed path**:
  the notebook must be launched with its working directory set to the
  repository root (see [Running the Project](#12-running-the-project)); it is
  no longer a hardcoded absolute path, but it is also not independent of how
  the notebook is launched.
- **Input image copyright/privacy is the user's responsibility**: this
  repository ships no sample images; any image you add under `input/` is your
  own responsibility with respect to copyright, consent, and EXIF metadata
  (see [Input Preparation](#11-input-preparation)).

## 15. Third-party Licenses / Acknowledgements

This project depends on DECA, PIXIE, and SMPL-X, each released by the Max
Planck Institute for Intelligent Systems under a **non-commercial scientific
research purposes** license that restricts redistribution to third parties.
See [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md) for the exact clauses,
official LICENSE links, and per-project notes.

This repository does not itself declare a license for its own original code
(Method A–E implementation, the notebook pipeline, the evaluation cell) at
this time.

## 16. Citation / References

If you use the DECA, PIXIE, or SMPL-X components this project depends on,
please cite the original works (citations copied from each project's own
README — see [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md) for the full
BibTeX entries):

- Feng et al., *Learning an Animatable Detailed 3D Face Model from In-The-Wild
  Images*, ACM TOG (Proc. SIGGRAPH) 2021 — DECA.
- Feng et al., *Collaborative Regression of Expressive Bodies using
  Moderation*, 3DV 2021 — PIXIE.
- Pavlakos et al., *Expressive Body Capture: 3D Hands, Face, and Body from a
  Single Image*, CVPR 2019 — SMPL-X.
