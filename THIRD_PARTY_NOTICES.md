# Third-Party Notices (외부 프로젝트 고지)

이 프로젝트는 세 개의 외부 연구 프로젝트 — **DECA**, **PIXIE**, **SMPL-X** —
위에서 동작합니다. 이 저장소에는 세 프로젝트의 소스 코드, 모델 가중치,
보조 데이터 파일(토폴로지/템플릿 메시, landmark embedding, 매핑 테이블 등)이
전혀 포함되어 있지 않습니다. 이 문서는 그 이유와 각 프로젝트의 라이선스,
그리고 DECA의 경우 이 프로젝트를 실행하기 위해 로컬에서 정확히 무엇을
수정했는지를 설명합니다.

> **아래 내용은 이해를 돕기 위한 요약이며, 실제 사용 조건은 각 프로젝트의
> 공식 LICENSE를 확인해야 합니다.** 이 문서는 원문 LICENSE 파일을 대체하지
> 않으며, 라이선스 조항을 임의로 해석하거나 재번역한 것이 아닙니다.
> 라이선스 인용문은 공식 저장소에 실린 영문 원문을 그대로 옮긴 것이고,
> 다운로드/사용 전에는 반드시 아래 링크의 공식 LICENSE 원문을 직접
> 확인하세요.

---

## 1. DECA

- **프로젝트명**: DECA: Detailed Expression Capture and Animation (SIGGRAPH 2021)
- **공식 저장소**: https://github.com/YadiraF/DECA
- **라이선스**: Software Copyright License for non-commercial scientific
  research purposes (저작권자: Max-Planck-Gesellschaft). 원문 전체:
  https://github.com/YadiraF/DECA/blob/master/LICENSE
  - 핵심 제한 조항 (공식 저장소 LICENSE 파일 원문 인용, 번역하지 않음):
    > "The Model & Software may not be reproduced, modified and/or made available
    > in any form to any third party."
    > "The Model & Software and the license herein granted shall not be copied,
    > shared, distributed, re-sold, offered for re-sale, transferred or
    > sub-licensed in whole or in part..."
  - (요약: 비상업적 연구 목적으로만 사용할 수 있으며, Model & Software를
    제3자에게 재배포·재판매·재라이선스할 수 없습니다. 정확한 법적 의미는
    반드시 원문으로 확인하세요.)
- **이 프로젝트에서의 역할**: 단일 이미지로부터의 얼굴 복원
  (`models/DECA/decalib`, `main.ipynb`에서 `decalib.deca.DECA`로 import).
- **필요한 데이터/가중치**: `deca_model.tar`, 그리고 별도로
  https://flame.is.tue.mpg.de 에서 회원가입 후 받아야 하는 FLAME 모델 파일
  (공식 저장소의 `fetch_data.sh` 참고). 이 저장소에는 포함되어 있지 않음 —
  README의 "모델 파일 준비" 섹션 참고.
- **Citation** (공식 README 기준):
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

### DECA 로컬 수정 사항 (재현성을 위해 중요)

이 프로젝트가 사용하는 로컬 `decalib/deca.py`는 **공식 원본 그대로가
아닙니다.** `pytorch3d` 없이(CPU 전용, rasterizer 없이) DECA를 실행할 수
있도록 수정되었습니다. 수정된 부분은 로컬 파일에 한국어 주석
`[수정 1]` … `[수정 5]`로 표시되어 있습니다. 각 수정 내용을 풀어서 설명하면
다음과 같습니다.

1. `pytorch3d` 기반 렌더러 import 구문(`from .utils.renderer import SRenderY, set_rasterizer`)을 주석 처리함.
2. `_setup_renderer` 안의 `set_rasterizer(...)` 호출을 제거함.
3. `_setup_renderer` 안의 `SRenderY` 렌더러 초기화를 제거함. 대신
   텍스처/마스크 파일(`face_eye_mask_path`, `face_mask_path`,
   `fixed_displacement_path`, `mean_tex_path`, `dense_template_path`)을
   `try/except` 블록 안에서 불러오도록 바꿔서, 이 파일들 중 일부가 없어도
   클래스 초기화 자체는 계속되도록 함.
4. 렌더러에 의존하는 헬퍼 메서드 `displacement2normal()`과 `visofp()`를
   `None`을 반환하도록 스텁(stub) 처리함.
5. `decode()` 메서드가 렌더링을 건너뛰고 geometry(형상) 정보만 반환하도록
   바꿈.

**의미**: 공식 `YadiraF/DECA` 저장소를 그대로 clone하는 것만으로는 이
파이프라인이 **재현되지 않습니다.** 공식 `deca.py`는 여전히 `pytorch3d`
렌더러를 import하고 초기화하기 때문입니다. 이 프로젝트와 동일하게
동작시키려면, 공식 저장소를 clone한 뒤 위에서 설명한 5가지 변경 사항을
`decalib/deca.py`에 직접 다시 적용해야 합니다 (렌더링 관련 코드를
제거/우회하는 것일 뿐, FLAME/encoder/decoder 내부의 연구 로직 자체는
건드리지 않습니다).

DECA 라이선스가 "Model & Software"의 제3자 재배포를 금지하고 있기 때문에,
이 저장소는 수정된 `deca.py` 파일 자체도 배포하지 않습니다 — 위 설명만으로
충분히 손으로 재현할 수 있을 정도로 변경 폭이 작기 때문에, 이 델타(delta)에
대한 설명만 문서로 남겨둡니다.

---

## 2. PIXIE

- **프로젝트명**: PIXIE: Collaborative Regression of Expressive Bodies (3DV 2021)
- **공식 저장소**: https://github.com/YadiraF/PIXIE
- **프로젝트 페이지**: https://pixie.is.tue.mpg.de/
- **라이선스**: Software Copyright License for non-commercial scientific
  research purposes (저작권자: Max-Planck-Gesellschaft). 원문 전체:
  https://github.com/YadiraF/PIXIE/blob/master/LICENSE
  (DECA와 동일하게 "may not be reproduced ... to any third party" 제한
  조항이 있습니다 — 정확한 문구는 원문을 확인하세요.)
- **이 프로젝트에서의 역할**: 단일 이미지로부터의 전신 SMPL-X 복원
  (`models/PIXIE-master/pixielib`, `main.ipynb`에서 `pixielib.pixie.PIXIE`로
  import).
- **필요한 데이터/가중치**: `pixie_model.tar`, `SMPLX_NEUTRAL_2020.npz`,
  그리고 토폴로지/UV/매핑 파일이 들어있는 `utilities.zip` 묶음. 각각
  https://pixie.is.tue.mpg.de/ 와 https://smpl-x.is.tue.mpg.de 에서 별도로
  회원가입 후 다운로드해야 합니다 (공식 저장소의 `fetch_model.sh` 참고). 이
  저장소에는 포함되어 있지 않음 — README의 "모델 파일 준비" 섹션 참고.
- **로컬 수정 여부**: 로컬 `pixielib` 코드에서는 DECA처럼 별도의 수정 표시나
  CPU 전용 패치 흔적을 찾지 못했습니다. PIXIE 코드 자체가 이미 `device`를
  전 구간에서 매개변수로 받도록 작성되어 있어서(예:
  `PIXIE.__init__(self, config=None, device='cuda:0')`), `main.ipynb`에서
  `device='cpu'`를 넘기는 것은 수정이 아니라 정상적인 생성자 인자 사용입니다.
  다만 공식 저장소와의 완전한 바이트 단위(byte-for-byte) 비교까지는
  수행하지 못했으므로, "수정되지 않았을 가능성이 높지만 공식적으로 검증되지는
  않음" 상태로 남겨둡니다.
- **Citation** (공식 README 기준):
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

- **프로젝트명**: SMPL-X / SMPLify-X (CVPR 2019)
- **공식 저장소**: https://github.com/vchoutas/smplx
- **라이선스**: Software Copyright License for non-commercial scientific
  research purposes (저작권자: Max-Planck-Gesellschaft). 원문 전체:
  https://github.com/vchoutas/smplx/blob/master/LICENSE
  (DECA/PIXIE와 동일하게 "may not be reproduced/modified/made available ...
  to any third party" 및 "may not be used ... for commercial ... purposes"
  제한 조항이 있습니다 — 정확한 문구는 원문을 확인하세요.)
- **이 프로젝트에서의 역할**: `smplx` Python 패키지(`smplx.create(...)`)를
  `main.ipynb`(Step 3, Method C)에서 직접 사용해 SMPL-X 레이어를 만들고
  PIXIE 전신 메시로부터 관절(joint)을 계산합니다.
- **필요한 데이터/가중치**: SMPL-X v1.1 모델 파일(`SMPLX_NEUTRAL.pkl`/`.npz`
  등), https://smpl-x.is.tue.mpg.de 에서 회원가입 후 다운로드. 이 저장소에는
  포함되어 있지 않음 — README의 "모델 파일 준비" 섹션 참고.
- **로컬 수정 여부**: 로컬 사본은 공식 저장소를 git clone한 것입니다.
  `origin/main`(커밋 `1265df7`)을 기준으로 `git status` / `git diff`를
  실행한 결과, 추적되는 소스 파일 중 **로컬에서 수정된 것은 없었습니다** —
  로컬에 추가된 것은 (라이선스가 필요한, 추적되지 않는) 모델 가중치와
  `SMPL-X_to_FLAME.npy`뿐입니다. 즉 공식 저장소를 clone하면 코드가 그대로
  재현됩니다.
- **Citation** (공식 README 기준. 이 프로젝트가 실제로 사용하는 모델은
  SMPL-X이므로 SMPL-X 인용만 옮겼으며, SMPL/SMPL+H/MANO 인용은 범위 밖이라
  생략했습니다):
  ```
  @inproceedings{SMPL-X:2019,
      title = {Expressive Body Capture: 3D Hands, Face, and Body from a Single Image},
      author = {Pavlakos, Georgios and Choutas, Vasileios and Ghorbani, Nima and Bolkart, Timo and Osman, Ahmed A. A. and Tzionas, Dimitrios and Black, Michael J.},
      booktitle = {Proceedings IEEE Conf. on Computer Vision and Pattern Recognition (CVPR)},
      year = {2019}
  }
  ```

---

## 세 프로젝트를 저장소에 포함하지 않는 이유

세 프로젝트의 LICENSE는 모두 "비상업적 연구 목적으로만 사용 가능하며, Model
& Software를 제3자에게 재현·복사·공유·배포·재판매·재라이선스할 수 없다"는
취지의 문구를 실질적으로 동일하게 담고 있습니다 (정확한 법적 표현은 각
LICENSE 원문 기준). 이 제한을 확실하게 지키기 위해, 이 저장소는
DECA/PIXIE/SMPL-X의 소스 코드나 데이터 파일을 어떤 형태로도 commit하지
않습니다. 대신:

- `README.md`에 어떤 공식 저장소를 clone해야 하는지, 어떤 파일을
  다운로드해야 하는지를 (각 프로젝트의 `fetch_*.sh` 스크립트와 대조해)
  링크와 함께 정확히 문서화했습니다.
- 이 문서에는 로컬 설정이 공식 버전과 실제로 달라지는 유일한 지점(DECA의
  `deca.py`)을 기록해 두어서, 필요할 때 손으로 그 차이를 재현할 수 있도록
  했습니다.

이 프로젝트 자체의 코드(`main.ipynb`, Method A~E, 평가 셀)는 독자적으로
작성한 것이며 위 외부 라이선스의 적용을 받지 않습니다.
