# DECA와 PIXIE/SMPL-X를 활용한 3D Human Synthesis
### (3D Human Synthesis with DECA and PIXIE/SMPL-X)

단일 이미지로부터 3D 인체 아바타를 생성하는 연구 프로젝트입니다. 얼굴의 디테일한
형상을 복원하는 **DECA**와 전신의 포즈·체형을 복원하는 **PIXIE / SMPL-X**를
결합해 하나의 메시로 만들며, 이를 위한 5가지 서로 다른 얼굴-전신 통합 방법론
(Method A~E)을 구현하고 있습니다.

## 1. 프로젝트 소개

얼굴 이미지 1장과 전신 이미지 1장을 입력으로 받아, [DECA](https://github.com/YadiraF/DECA)로
3D 얼굴 메시를, [PIXIE](https://github.com/YadiraF/PIXIE)로 3D 전신 SMPL-X 메시를
각각 복원한 뒤, 5가지 방법(Method A~E)으로 두 메시를 하나의 인체 표현으로
통합합니다. 전체 파이프라인은 하나의 Jupyter 노트북(`main.ipynb`)에서 CPU로
실행됩니다.

## 2. 연구 목적

얼굴만 복원하는 모델과 전신만 복원하는 모델은 보통 따로 학습·평가됩니다. 이
프로젝트의 목적은 DECA의 고해상도 표정 디테일과 PIXIE/SMPL-X의 전신 포즈·체형
정보를 하나의 위상학적으로 일관된 3D 인체 표현으로 통합하고, 여러 통합
방법론을 이미지 기반 복원 지표로 서로 비교하는 것입니다.

## 3. 전체 처리 흐름

`main.ipynb`는 다음 순서로 셀을 실행합니다.

```
입력 이미지
  ↓
PIXIE 전신 복원
  ↓
DECA 얼굴 복원
  ↓
Method A ~ E
  ↓
3D OBJ 생성
  ↓
성능 평가
```

조금 더 구체적으로는 다음과 같습니다.

```
입력 이미지 (얼굴 + 전신)
        │
        ▼
   PIXIE  (Step 3) ──▶ 전신 SMPL-X 메시, 관절(joints), 신체 파라미터
        │
        ▼
   DECA   (Step 4) ──▶ 디테일 얼굴 메시, 표정/턱 관절 파라미터
        │
        ▼
   Method A / B / C / D / E ──▶ 통합된 얼굴+전신 SMPL-X / 봉합 메시
        │
        ▼
   .obj 파일을 result_output_jupyter/ 에 저장
        │
        ▼
   평가 셀 ──▶ Ground-truth 이미지와 비교 렌더링, 지표 계산,
               metrics_summary.csv 저장
```

셀은 반드시 위에서 아래로 순서대로 실행해야 합니다. 뒤쪽 셀(Method A~E, 평가
셀)은 Step 1~4에서 만들어진 객체(`deca`, `pixie_model`, `testdata`,
`codedict_raw`, `codedict_mod`, `save_dir` 등)를 그대로 재사용합니다.

## 4. 통합 방법론

`main.ipynb`에 실제로 구현된 내용을 기준으로 정리했습니다.

### Method A
- **목적**: PCA 기반 적응형 슬라이싱과 라플라시안(Laplacian) 봉합부 스무딩을 통한 결합
- **주요 처리**: Neck 관절을 원점으로 좌표계를 정규화하고, 척추 벡터를 기준으로
  전신 메시를 수직 정렬한 뒤, 전신·얼굴 메시를 목 부분에서 절단한다. 경계
  루프를 리샘플링해 두 메시를 다시 연결하고, 접합부에 국소적(local)
  라플라시안 스무딩을 적용한다.
- **출력**: `Method_A_original.obj`, `Method_A_modified.obj`

### Method B
- **목적**: PCA 기반 적응형 슬라이싱과 하모닉(harmonic) 메시 페어링을 통한 결합
- **주요 처리**: Method A와 동일하게 목 부분에서 슬라이싱하되, 경계면을
  하모닉 필드로 워핑(warping)하고 다중 해상도 지퍼(zipper) 방식으로 봉합한
  뒤, 바이-라플라시안(bi-Laplacian) 곡률 최적화로 전역 스무딩을 적용한다.
- **출력**: `Method_B__original.obj`, `Method_B__modified.obj`
  *(코드에 실제로 저장되는 파일명 그대로이며, 밑줄이 2개입니다)*

### Method C
- **목적**: 이종 잠재 통합 및 파라미터 기반 합성 (절단면 없이 결합)
- **주요 처리**: DECA에서 추출한 얼굴 파라미터를 SMPL-X 파라미터/잠재 공간에
  직접 주입하여 접합면 자체가 생기지 않도록 하고, SVD 기반으로 골격을
  정렬한다.
- **출력**: `Method_C_original.obj`, `Method_C_Modified.obj`
  *(코드에 실제로 저장되는 파일명 그대로이며, "M"이 대문자입니다)*

### Method D
- **목적**: 신뢰도 가중 특징 융합을 통한 단일 SMPL-X 메시 생성
- **주요 처리**: PIXIE의 `encode()`를 한 번 실행해 전신 파라미터를 얻고, DECA의
  표정(`exp`)과 턱 관절(jaw pose)을 추출한다. 턱 관절의 axis-angle 표현을
  오일러(Euler) 각으로 변환한 뒤 `codedict["exp"]` / `codedict["jaw_pose"]`를
  DECA 값으로 덮어쓰고, `pixie.decode()`를 한 번 호출해 봉합 없는 단일 SMPL-X
  메시를 만든다.
- **출력**: `Method_D_original.obj`, `Method_D_modified.obj`

### Method E
- **목적**: 키포인트 재투영 및 디테일(주름·윤곽) 전이
- **주요 처리**: Method D의 결과 메시를 기반으로, DECA의 디테일 변위(displacement)
  맵을 추출한다. 캐시된 FLAME→SMPL-X 매핑을 이용해 FLAME UV 공간의 디테일을
  SMPL-X UV 공간으로 변환하고, 정점(vertex)의 UV 좌표에서 bilinear
  샘플링으로 변위 값을 얻은 뒤, normal 방향으로 정점을 이동시켜 토폴로지를
  유지한 채 고주파 디테일을 복원한다.
- **출력**: `Method_E_original.obj`, `Method_E_modified.obj`

각 Method는 "원본 표정(original)" 메시와, 두 번째 "driving" 얼굴 이미지의
표정을 이식한 "수정된 표정(modified)" 메시를 함께 저장합니다. 실제 파일명은
[출력 결과](#13-출력-결과)를 참고하세요.

## 5. 평가 지표

평가 셀(Step 10, 노트북 내 주석 기준 "성능지표")은 각 결과 메시를
원본(ground-truth) 이미지와 동일한 시점에서 렌더링한 뒤, 대상 영역을
마스킹/크롭하여 다음 지표를 계산합니다. 실제 프로젝트에서 사용하는 지표만
정리했으며, 사용하지 않는 지표는 추가하지 않았습니다.

| 지표 | 의미 | 해석 |
|---|---|---|
| **L1 (Photometric)** | 두 이미지 간 픽셀 값 차이의 평균 절대 오차(MAD) | 0에 가까울수록 색상·명암 차이가 적음 |
| **SSIM** (구조적 유사도) | 밝기·대비·구조 정보를 기반으로 한 구조적 유사도 | 1에 가까울수록 원본과 구조적으로 유사 |
| **PSNR** (피크 신호 대 잡음비) | 최대 신호 대 잡음비 | 값이 높을수록 화질 손실이 적음 |
| **LPIPS** (학습 기반 지각 이미지 패치 유사도) | AlexNet 기반 지각적 거리 (`lpips` 패키지) | 낮을수록 사람이 보기에 원본과 자연스럽게 유사 |
| **Cosine Similarity** | VGG16 딥 특징(deep feature) 간 코사인 유사도 | 1에 가까울수록 의미적(semantic)으로 유사 |
| **Time (s)** | 메시 1개당 렌더링 + 지표 계산에 걸린 시간 | 낮을수록 효율적 |

결과는 콘솔에 출력되는 동시에 pandas DataFrame 형태로
`result_output_jupyter/metrics_summary.csv`에 저장됩니다.

## 6. 저장소 구조

현재 이 저장소가 실제로 Git에 추적하고 있는 파일은 다음과 같습니다.

```
3D_Human_Synthesis/                 <- Repository root
├── README.md
├── THIRD_PARTY_NOTICES.md
├── environment.yaml
├── .gitignore
├── main.ipynb
└── input/
    ├── DECA/.gitkeep
    └── SMPL-X/.gitkeep
```

아래는 로컬 실행을 위해 사용자가 직접 준비해야 하는 항목입니다 (Git에는
포함되지 않음 — 9~11번 섹션 참고).

```
├── input/
│   ├── DECA/       <- 본인이 사용할 얼굴 이미지
│   └── SMPL-X/     <- 본인이 사용할 전신 이미지
└── models/
    ├── DECA/         <- https://github.com/YadiraF/DECA 에서 clone (+패치, THIRD_PARTY_NOTICES.md 참고)
    ├── PIXIE-master/ <- https://github.com/YadiraF/PIXIE 에서 clone/다운로드
    └── smplx/        <- https://github.com/vchoutas/smplx 에서 clone
```

## 7. 개발 환경

Python 3.8, CPU 전용 PyTorch 기준으로 개발되었습니다. `environment.yaml`에
개발 당시 사용한 패키지 버전이 그대로 고정되어 있습니다.

## 8. 설치 방법

아래 명령어들은 번역하지 않고 실제 실행해야 하는 그대로 표기합니다.

```bash
conda env create -f environment.yaml
conda activate deca_fusion
jupyter notebook
```

- `conda env create -f environment.yaml` — `environment.yaml`에 정의된 대로 가상환경을 생성합니다.
- `conda activate deca_fusion` — 생성한 가상환경을 활성화합니다 (`deca_fusion`은 `environment.yaml`에 선언된 환경 이름입니다).
- `jupyter notebook` — Jupyter를 실행합니다. **반드시 이 저장소의 루트 폴더(= `main.ipynb`가 있는 폴더)에서 실행해야 합니다.** `main.ipynb`의 첫 셀이 `PROJECT_ROOT = Path.cwd()`로 현재 작업 디렉터리를 그대로 프로젝트 루트로 사용하기 때문입니다. 자세한 내용은 [12. 실행 방법](#12-실행-방법)을 참고하세요.

## 9. 외부 프로젝트 및 의존성

라이선스 원문, 공식 URL, Citation, 로컬 수정 사항에 대한 전체 내용은
[THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)에 정리되어 있습니다. 아래는
요약입니다.

| 프로젝트 | 공식 저장소 | 이 프로젝트에서의 역할 | 추가로 필요한 가중치 |
|---|---|---|---|
| **DECA** | https://github.com/YadiraF/DECA | 단일 이미지로부터 디테일한 얼굴 복원 (`decalib.deca.DECA`) | `deca_model.tar`, FLAME 모델 파일 (https://flame.is.tue.mpg.de 에서 회원가입 후 다운로드) |
| **PIXIE** | https://github.com/YadiraF/PIXIE | 단일 이미지로부터 전신 SMPL-X 복원 (`pixielib.pixie.PIXIE`) | `pixie_model.tar`, `SMPLX_NEUTRAL_2020.npz`, `utilities.zip` (https://pixie.is.tue.mpg.de/ 에서 회원가입 후 다운로드) |
| **SMPL-X** | https://github.com/vchoutas/smplx | SMPL-X 바디 레이어 / 관절 회귀 (`smplx.create(...)`) | SMPL-X v1.1 모델 파일 (https://smpl-x.is.tue.mpg.de 에서 회원가입 후 다운로드) |

세 프로젝트 모두 **가중치를 다운로드하려면 각 공식 사이트에서 개별적으로
회원가입하고 라이선스에 동의해야 합니다.** 이 저장소는 해당 가중치를
재배포할 수 없습니다 ([10. 모델 파일 준비](#10-모델-파일-준비) 및
[15. 외부 라이선스 및 감사의 글](#15-외부-라이선스-및-감사의-글) 참고).

## 10. 모델 파일 준비

**이 저장소에는 DECA/PIXIE/SMPL-X의 모델 가중치, 토폴로지/템플릿 파일 등
어떠한 데이터도 포함되어 있지 않습니다** (`.gitignore`에서 `models/DECA/`,
`models/PIXIE-master/`, `models/smplx/`를 전부 제외하고 있습니다). 아래처럼
`main.ipynb`가 기대하는 구조에 맞춰 소스 코드와 데이터를 직접 준비해야
합니다.

```
models/
├── DECA/
│   ├── decalib/                      <- 공식 DECA 저장소에서 (+패치, THIRD_PARTY_NOTICES.md 참고)
│   └── data/
│       ├── deca_model.tar            <- 필수 (핵심 모델 가중치)
│       ├── generic_model.pkl         <- 필수 (FLAME 모델, FLAME2020.zip에서 추출)
│       ├── landmark_embedding.npy    <- 필수 (없으면 FLAME() 생성 시 오류 발생)
│       ├── uv_face_mask.png          <- 선택 (Method E 디테일 마스킹용, 없으면 자동으로 건너뜀)
│       └── uv_face_eye_mask.png      <- 선택 (Method E 디테일 마스킹용, 없으면 자동으로 건너뜀)
│
├── PIXIE-master/
│   ├── pixielib/                     <- 공식 PIXIE 저장소에서
│   └── data/
│       ├── pixie_model.tar           <- 필수 (핵심 모델 가중치)
│       ├── SMPLX_NEUTRAL_2020.npz    <- 필수 (PIXIE가 사용하는 SMPL-X 바디 모델)
│       ├── smplx_extra_joints.yaml   <- 필수 (없으면 SMPLX() 생성 시 오류 발생)
│       ├── SMPLX_to_J14.pkl          <- 필수 (없으면 SMPLX() 생성 시 오류 발생)
│       ├── SMPL_X_template_FLAME_uv.obj  <- Method E에 필수 (FLAME→SMPL-X UV 토폴로지)
│       └── flame2smplx_tex_1024.npy      <- Method E에 필수 (캐시된 FLAME→SMPL-X UV 매핑)
│
└── smplx/
    ├── smplx/                        <- 공식 smplx 저장소 (`pip install` 가능한 패키지)
    └── models/
        └── smplx/
            ├── SMPLX_NEUTRAL.pkl (또는 .npz)  <- 필수
            ├── SMPLX_MALE.pkl / .npz          <- 성별 지정 모델을 쓸 때만 필요
            └── SMPLX_FEMALE.pkl / .npz        <- 성별 지정 모델을 쓸 때만 필요
```

위의 "필수/선택" 구분은 `decalib/utils/config.py` + `decalib/deca.py`, 그리고
`pixielib/utils/config.py` + `pixielib/models/SMPLX.py`에서 어떤 경로가
조건 없이(unconditionally) 읽히는지를 실제 코드에서 추적하고, 이를
`main.ipynb`가 실제로 호출하는 부분과 대조해서 확인한 결과입니다. 위 목록에
없는 파일(예: PIXIE의 `smplx_hand.obj`, `smplx_tex.obj`,
`MANO_SMPLX_vertex_ids.pkl`, `SMPL-X__FLAME_vertex_ids.npy`)은 PIXIE의
`visualizer.py`에서만 쓰이는데, `main.ipynb`는 이 모듈을 import하지 않으므로
이 파이프라인 실행에는 필요하지 않습니다.

가중치를 포함하지 않는 이유는 DECA/PIXIE/SMPL-X 라이선스가 제3자 재배포를
제한하고 있기 때문이며, 동시에 용량도 매우 크기 때문입니다(합쳐서 수백 MB
이상). 자세한 내용은 [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)를
참고하세요.

## 11. 입력 이미지 준비

**이 저장소에는 샘플 이미지가 포함되어 있지 않습니다.** `input/DECA/`와
`input/SMPL-X/`에는 clone 이후에도 폴더 구조가 유지되도록 `.gitkeep`
파일만 들어 있습니다.

- 본인이 사용할 권리가 있는 얼굴 이미지를 `input/DECA/`에, 전신 이미지를
  `input/SMPL-X/`에 직접 넣어야 합니다.
- 개인 사진을 사용할 경우, 휴대폰/카메라로 찍은 사진에는 기기 모델명, 촬영
  시각, 때로는 **GPS 위치 정보**까지 EXIF 메타데이터로 포함되어 있을 수
  있습니다. 공개 전에 반드시 메타데이터를 확인하거나 제거하세요.
- 현재 `main.ipynb`는 `BODY_IMAGE_PATH`, `SOURCE_FACE_IMAGE_PATH`,
  `DRIVING_FACE_IMAGE_PATH` 변수가 예시 파일명(`body5.jpg`, `face4.jpg`,
  `face1.png`)을 가리키도록 되어 있습니다. Step 3 / Step 4에서 이 변수들을
  본인의 파일명으로 직접 수정해야 합니다.

## 12. 실행 방법

`main.ipynb`의 첫 번째 셀(Step 1)은 `PROJECT_ROOT = Path.cwd()`로 프로젝트
루트를 설정하고, 여기서 `INPUT_DIR` / `MODEL_DIR`을 파생시킵니다. 즉:

1. **Notebook 커널의 작업 디렉터리가 반드시 이 저장소의 루트**(즉
   `main.ipynb`, `input/`, `models/`가 있는 폴더)**여야 합니다.** Jupyter
   Notebook / JupyterLab에서 이 폴더의 노트북을 열면 기본적으로 이 조건이
   충족됩니다. VS Code처럼 작업 디렉터리가 노트북 파일 위치와 달라질 수 있는
   IDE를 사용한다면, Step 1을 실행하기 전에 작업 디렉터리가 저장소 루트로
   맞춰져 있는지 반드시 확인/설정하세요.
2. conda 가상환경을 생성하고 노트북 커널로 선택합니다 ([8. 설치
   방법](#8-설치-방법) 참고).
3. [10. 모델 파일 준비](#10-모델-파일-준비)에 설명한 대로 DECA, PIXIE,
   SMPL-X를 `models/` 아래에 clone/다운로드합니다.
4. [11. 입력 이미지 준비](#11-입력-이미지-준비)를 참고해 `input/DECA/`,
   `input/SMPL-X/`에 본인의 이미지를 추가하고, Step 3 / Step 4의 경로
   변수를 수정합니다.
5. 셀을 다음 순서로 실행합니다: Step 1(경로/환경 설정) → Step 2(공통 함수
   정의) → Step 3(PIXIE 전신 복원) → Step 4(DECA 얼굴 복원) → Method A →
   Method B → Method C → Method D → Method E → Step 10(성능 평가).

## 13. 출력 결과

노트북을 실행하면 `result_output_jupyter/` 폴더가 생성되며(이미 폴더가
있으면 `result_output_jupyter1`, `result_output_jupyter2`, ... 처럼 자동으로
번호가 붙습니다), 다음 파일들이 저장됩니다.

- `0_pixie_body_raw.obj`, `0_pixie_body_upright.obj` — PIXIE 전신 메시
- `1_deca_raw.obj`, `2_deca_mod.obj` — DECA 얼굴 메시 (원본 / 표정 이식본)
- `Method_A_original.obj`, `Method_A_modified.obj`
- `Method_B__original.obj`, `Method_B__modified.obj` *(밑줄 2개, 코드에 저장된 실제 파일명 그대로)*
- `Method_C_original.obj`, `Method_C_Modified.obj` *(대문자 "M", 코드에 저장된 실제 파일명 그대로)*
- `Method_D_original.obj`, `Method_D_modified.obj`
- `Method_E_original.obj`, `Method_E_modified.obj`
- `metrics_summary.csv` — 평가 결과표 ([5. 평가 지표](#5-평가-지표) 참고)

이 결과 파일들은 Git에 추적되지 않습니다 (`.gitignore`가
`result_output_jupyter*/`, `*.obj`, `metrics_summary.csv`를 제외합니다).
노트북을 실행할 때마다 새로 생성됩니다.

## 14. 알려진 한계

- **Windows 중심 개발 환경**: `environment.yaml`에 Windows 전용 패키지
  (`pywin32`, `win32-setctime`)가 포함되어 있으며, CPU 전용 PyTorch로 Windows
  환경에서 개발·테스트되었습니다.
- **모델 가중치 미배포**: DECA/PIXIE/SMPL-X의 가중치와 보조 데이터는
  각자 라이선스에 동의한 뒤 사용자가 직접 다운로드해야 합니다
  ([10. 모델 파일 준비](#10-모델-파일-준비) 참고).
- **외부 프로젝트 의존성, 일부는 수정된 상태**: 이 파이프라인은 로컬에 준비된
  DECA/PIXIE 소스 코드에 의존합니다. 로컬 DECA 코드는 (pytorch3d 렌더러
  제거를 위해) 수정되어 있어서, 공식 저장소를 그대로 clone하는 것만으로는
  이 패치를 다시 적용하지 않는 한 동일하게 재현되지 않습니다
  ([THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md) 참고). PIXIE와 SMPL-X는
  수정된 흔적이 없어 보이지만, PIXIE는 공식 저장소와 완전한 바이트 단위 비교까지
  거치지는 않았습니다.
- **`PROJECT_ROOT`는 고정 경로가 아니라 노트북의 작업 디렉터리**: 노트북을
  반드시 저장소 루트를 작업 디렉터리로 하여 실행해야 합니다
  ([12. 실행 방법](#12-실행-방법) 참고). 더 이상 하드코딩된 절대경로는
  아니지만, 노트북을 어떻게 실행하는지와 무관하지도 않습니다.
- **입력 이미지의 저작권/개인정보는 사용자 책임**: 이 저장소는 샘플 이미지를
  포함하지 않으며, `input/` 아래에 추가하는 모든 이미지의 저작권, 초상권 동의,
  EXIF 메타데이터에 대한 책임은 사용자에게 있습니다
  ([11. 입력 이미지 준비](#11-입력-이미지-준비) 참고).

## 15. 외부 라이선스 및 감사의 글

이 프로젝트는 DECA, PIXIE, SMPL-X에 의존하고 있으며, 세 프로젝트 모두 Max
Planck Institute for Intelligent Systems가 **비상업적 연구 목적(non-commercial
scientific research purposes)** 라이선스로 배포하고 있고, 제3자 재배포를
제한하는 조항을 포함하고 있습니다. 정확한 조항, 공식 LICENSE 링크, 프로젝트별
세부 사항은 [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)에 정리되어
있습니다.

이 저장소는 자체 원본 코드(Method A~E 구현, 노트북 파이프라인, 평가 셀)에
대해 아직 별도의 라이선스를 지정하지 않은 상태입니다.

## 16. 참고문헌

이 프로젝트가 의존하는 DECA, PIXIE, SMPL-X를 사용하신다면 아래 원 논문을
인용해 주세요 (각 프로젝트 공식 README에 기재된 Citation을 그대로 옮겼으며,
전체 BibTeX는 [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)에 있습니다).

- Feng et al., *Learning an Animatable Detailed 3D Face Model from In-The-Wild
  Images*, ACM TOG (Proc. SIGGRAPH) 2021 — DECA.
- Feng et al., *Collaborative Regression of Expressive Bodies using
  Moderation*, 3DV 2021 — PIXIE.
- Pavlakos et al., *Expressive Body Capture: 3D Hands, Face, and Body from a
  Single Image*, CVPR 2019 — SMPL-X.
