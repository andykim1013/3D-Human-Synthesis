# Third-Party Notices

This project builds on top of three external research projects — **DECA**, **PIXIE**,
and **SMPL-X**. None of their source code, model weights, or auxiliary data files
(topology/template meshes, landmark embeddings, mapping tables, etc.) are
distributed in this repository. This document explains why, what license each
project carries, and — for DECA — exactly what was changed locally to make this
project's pipeline run.

This file summarizes license terms for reference; it is **not** a substitute for
the original LICENSE files. Always read the official LICENSE at the links below
before downloading or using each project.

---

## 1. DECA

- **Project**: DECA: Detailed Expression Capture and Animation (SIGGRAPH 2021)
- **Official repository**: https://github.com/YadiraF/DECA
- **License**: Software Copyright License for non-commercial scientific research
  purposes (Max-Planck-Gesellschaft). Full text:
  https://github.com/YadiraF/DECA/blob/master/LICENSE
  - Key restriction (quoted from the LICENSE file shipped with the official repo):
    > "The Model & Software may not be reproduced, modified and/or made available
    > in any form to any third party."
    > "The Model & Software and the license herein granted shall not be copied,
    > shared, distributed, re-sold, offered for re-sale, transferred or
    > sub-licensed in whole or in part..."
- **Role in this project**: face reconstruction from a single image (`models/DECA/decalib`,
  imported as `decalib.deca.DECA` in `main.ipynb`).
- **Data / weights required**: `deca_model.tar`, and the FLAME model files obtained
  by registering separately at https://flame.is.tue.mpg.de (see `fetch_data.sh`
  in the official repo). Not included here — see README "Model Files".
- **Citation** (from the official README):
  ```
  @inproceedings{DECA:Siggraph2021,
    title={Learning an Animatable Detailed {3D} Face Model from In-The-Wild Images},
    author={Feng, Yao and Feng, Haiwen and Black, Michael J. and Bolkart, Timo},
    journal = {ACM Transactions on Graphics, (Proc. SIGGRAPH)},
    volume = {40},
    number = {8},
    year = {2021},
    url = {https://doi.org/10.1145/3450626.3459936}
  }
  ```

### Local modifications to DECA (important for reproducibility)

The local copy of `decalib/deca.py` used by this project **is not the unmodified
official file**. It was edited so that DECA can run without `pytorch3d`
(CPU-only, no rasterizer). The edits are marked in the local file with Korean
comments `[수정 1]` … `[수정 5]`. In prose, the changes are:

1. The `pytorch3d`-based renderer import (`from .utils.renderer import SRenderY, set_rasterizer`) is commented out.
2. The `set_rasterizer(...)` call in `_setup_renderer` is removed.
3. The `SRenderY` renderer initialization in `_setup_renderer` is removed; texture/mask
   files (`face_eye_mask_path`, `face_mask_path`, `fixed_displacement_path`,
   `mean_tex_path`, `dense_template_path`) are instead loaded inside a `try/except`
   block so the class still initializes even if some of those files are missing.
4. `displacement2normal()` and `visofp()` (renderer-dependent helper methods) are
   stubbed to return `None`.
5. The `decode()` method skips rendering and returns geometry-only outputs.

**Implication**: cloning the official `YadiraF/DECA` repository as-is will *not*
reproduce this pipeline, because the official `deca.py` still imports and
initializes the `pytorch3d` renderer. To reproduce this project's behavior, clone
the official repository and re-apply the five changes described above to
`decalib/deca.py` (they only remove/guard rendering code — no research logic
inside FLAME/encoder/decoder is touched).

Because DECA's license prohibits redistributing the "Model & Software" to third
parties, this repository does not ship a modified copy of `deca.py` either — only
this description of the delta, which is small enough to reproduce by hand from
the instructions above.

---

## 2. PIXIE

- **Project**: PIXIE: Collaborative Regression of Expressive Bodies (3DV 2021)
- **Official repository**: https://github.com/YadiraF/PIXIE
- **Project page**: https://pixie.is.tue.mpg.de/
- **License**: Software Copyright License for non-commercial scientific research
  purposes (Max-Planck-Gesellschaft). Full text:
  https://github.com/YadiraF/PIXIE/blob/master/LICENSE
  (same "may not be reproduced ... to any third party" restriction as DECA).
- **Role in this project**: full-body SMPL-X reconstruction from a single image
  (`models/PIXIE-master/pixielib`, imported as `pixielib.pixie.PIXIE` in
  `main.ipynb`).
- **Data / weights required**: `pixie_model.tar`, `SMPLX_NEUTRAL_2020.npz`, and the
  `utilities.zip` bundle (topology/UV/mapping files), obtained by registering
  separately at https://pixie.is.tue.mpg.de/ and https://smpl-x.is.tue.mpg.de
  (see `fetch_model.sh` in the official repo). Not included here — see README
  "Model Files".
- **Local modifications**: no modification markers or CPU-only patches were found
  in the local copy of `pixielib`. PIXIE's own code already parameterizes
  `device` throughout (e.g. `PIXIE.__init__(self, config=None, device='cuda:0')`),
  so `device='cpu'` is used as a normal, unmodified constructor argument in
  `main.ipynb`. A full byte-for-byte diff against the official repository was
  **not** performed, so this should be treated as "likely unmodified, not
  formally verified."
- **Citation** (from the official README):
  ```
  @inproceedings{PIXIE:2021,
        title={Collaborative Regression of Expressive Bodies using Moderation},
        author={Yao Feng and Vasileios Choutas and Timo Bolkart and Dimitrios Tzionas and Michael J. Black},
        booktitle={International Conference on 3D Vision (3DV)},
        year={2021}
  }
  ```

---

## 3. SMPL-X

- **Project**: SMPL-X / SMPLify-X (CVPR 2019)
- **Official repository**: https://github.com/vchoutas/smplx
- **License**: Software Copyright License for non-commercial scientific research
  purposes (Max-Planck-Gesellschaft). Full text:
  https://github.com/vchoutas/smplx/blob/master/LICENSE
  (same "may not be reproduced/modified/made available ... to any third party"
  and "may not be used ... for commercial ... purposes" restrictions).
- **Role in this project**: the `smplx` Python package (`smplx.create(...)`) is
  used directly in `main.ipynb` (Step 3 and Method C) to build an SMPL-X layer
  and compute joints from the PIXIE body mesh.
- **Data / weights required**: SMPL-X v1.1 model files (`SMPLX_NEUTRAL.pkl`/`.npz`,
  etc.), obtained by registering at https://smpl-x.is.tue.mpg.de. Not included
  here — see README "Model Files".
- **Local modifications**: the local copy is a git clone of the official
  repository. `git status` / `git diff` against `origin/main` at commit
  `1265df7` show **no local modifications** to any tracked source file — only
  the (untracked, license-gated) model weights and `SMPL-X_to_FLAME.npy` were
  added locally. Cloning the official repository reproduces the code exactly.
- **Citation** (from the official README; SMPL-X is the model actually used in
  this project — SMPL/SMPL+H/MANO citations are omitted as out of scope):
  ```
  @inproceedings{SMPL-X:2019,
      title = {Expressive Body Capture: 3D Hands, Face, and Body from a Single Image},
      author = {Pavlakos, Georgios and Choutas, Vasileios and Ghorbani, Nima and Bolkart, Timo and Osman, Ahmed A. A. and Tzionas, Dimitrios and Black, Michael J.},
      booktitle = {Proceedings IEEE Conf. on Computer Vision and Pattern Recognition (CVPR)},
      year = {2019}
  }
  ```

---

## Why none of the three are vendored in this repository

All three LICENSE files use materially the same "non-commercial research use
only; do not reproduce, copy, share, distribute, re-sell or sub-license the
Model & Software to any third party" language. To stay clearly on the safe side
of that restriction, this repository does not commit any DECA/PIXIE/SMPL-X
source file or data file. Instead:

- `README.md` documents exactly which official repository to clone and which
  files to download (with links, matching each project's own `fetch_*.sh`
  script).
- This file documents the one place (DECA's `deca.py`) where the local setup
  actually diverges from upstream, so the diff can be reproduced by hand.

This project's own code (`main.ipynb`, Method A–E, the evaluation cell) is
original work and is not covered by the above third-party licenses.
