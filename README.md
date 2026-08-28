# landgen_pipeline
# landgen — Landscape Genetics Analysis Pipeline 

By Chi-Young Choi (you0742@gmail.com)
Ph.D. Candidate, Major in Ecology and Systematics
Department of Biological Sciences, Graduate School, Daegu University



Population 좌표, pairwise FST, 그리고 road/river 같은 landscape resistance
raster를 입력으로 받아 IBD(Isolation by Distance) / IBR(Isolation by
Resistance) 분석과 MMRR(Multiple Matrix Regression with Randomization)까지
수행하는 재현 가능한(reproducible) 파이프라인입니다. 특정 종이나 연구
지역에 종속되지 않은 범용 프레임워크로, `config/`에 지역·종별 설정 파일을
추가하는 방식으로 여러 프로젝트에 재사용할 수 있습니다.

## 구조

```
main.py                        # 파이프라인 오케스트레이션 (checkpoint 기반 skip/resume)
config/
  config_ksd.py                # 지역/프로젝트별 설정 예시 (paths, population/FST 경로, 스캔 파라미터 등)
landgen/
  utils/
    config_ns.py                # config.paths.root_dir 같은 dot-access 지원
    logging_utils.py            # log_msg / log_section
    checkpoint.py                # run_step / checkpoint_path (skip-resume)
    io_utils.py                  # ensure_dir / resolve_data_root / check_raster_file 등
    raster_geom.py               # align_to / aggregate_max / write_raster / sample_points / extract_buffer_mean
    population_io.py             # load_population_csv / load_fst_named_vector
    stats_utils.py               # upper_tri_vector / run_mantel
    distance_matrix.py           # calculate_distance_matrix (cost-distance, 병렬처리)
    plotting.py                  # get_pyplot() — 한글 등 비-라틴 폰트 자동 처리
  steps/
    step01_population.py         # population 좌표 로드 + 지리적 거리 행렬
    step02_fst.py                 # pairwise FST 로드/행렬화
    step03_reference.py           # 기준 raster, IBD Mantel/회귀, 분석 범위
    step04_rasters.py             # landscape raster 로드 + geometry 정합
    step05_metrics.py             # population별 raster 지표 (거리, 버퍼 평균 등)
    step06_river_scan.py          # resistance 후보값 스캔 (facilitator/barrier)
    step07_resistance.py          # 결합 resistance distance matrix
    step08_mantel.py              # Mantel test + 단순회귀 비교
    step09_mmrr.py                # MMRR (permutation 기반 다중행렬회귀)
    step10_visualization.py       # 결과 시각화 + 최종 요약 리포트
requirements.txt
```

## 파이프라인 흐름

1. **Population/FST 준비** (01~03) — 좌표 투영, 지리적 거리, pairwise FST,
   기준 raster 기반 IBD 검정
2. **Raster 준비** (04~05) — landscape raster 로드/정합, population별 지표 추출
3. **Resistance 모델링** (06~07) — 후보 resistance 값 스캔으로 최적값 탐색,
   여러 resistance layer 결합
4. **통계 검정** (08~09) — Mantel test, MMRR로 지리적 거리 대비 landscape
   요인의 설명력 평가
5. **시각화/리포트** (10) — 모델 비교 그림, 텍스트 요약 리포트

## 다른 프로젝트/종에 적용하기

새 프로젝트는 `config/` 아래 새 설정 파일 하나만 추가하면 됩니다
(`config_ksd.py`를 템플릿으로 복사):

- `region_name`, `crs_code`: 프로젝트명과 좌표계
- `paths.*`: population/FST CSV, road/river raster 경로
- `river_scan_values`, `river_scan_max_workers`: resistance 후보값과 병렬처리 설정
- `mantel_permutations`, `mantel_seed`: 통계 검정 설정

`main.py`의 `REGION_CONFIG_NAME`을 새 설정 모듈 이름으로 바꾸면 나머지
파이프라인(01~10)은 코드 수정 없이 그대로 재사용됩니다.

## 설계 메모 (유지보수 시 참고)

- **raster 반환 규칙**: 파일 핸들(`rasterio.DatasetReader`)은 pickle이 안
  되므로 checkpoint에 저장할 값으로 직접 반환하지 말 것. 배열이 필요 없고
  grid(transform/crs/shape) 정보만 필요하면 `GridMeta`, 배열이 필요하면
  읽어서 `RasterGrid`에 담고 원본 데이터셋은 닫을 것.
- **pandas 배열 in-place 연산 주의**: `.to_numpy()`가 종종 read-only view를
  반환하므로, `np.fill_diagonal()` 등 in-place 연산 전에는 `np.array(...,
  copy=True)`로 명시적 복사할 것.
- **(수정됨) `calculate_distance_matrix`의 워커 간 배열 전송 문제**: 초기
  구현은 population(태스크)마다 resistance 배열 전체를 매번 pickle해서
  워커로 전송했다. 큰 raster(158M셀, ~1.27GB)에 population이 많으면 같은
  배열을 수십 번 복제 전송하는 셈이 되어, **Windows에서 실제로 파이프
  리소스 고갈(WinError 1450)로 프로세스가 죽는 사례가 발생**했다. 이제
  `multiprocessing.shared_memory`로 배열을 공유메모리에 한 번만 올리고,
  각 태스크에는 공유메모리 이름(문자열)만 전달하도록 고쳤다 — 반복 pickle
  전송이 완전히 사라진다. `executor` 파라미터(06단계의 persistent 클러스터
  재사용)와도 함께 동작하며, 매 `calculate_distance_matrix` 호출마다 새
  공유메모리 블록을 만들고 끝나면 확실히 `unlink()`한다.
- **(신규) 워커 수 자동 산정** (`resource_utils.estimate_safe_worker_count`):
  예전엔 워커 수가 "CPU 코어 - 2, 상한 2" 같은 고정 로직이었는데, 이러면
  raster가 크면(워커당 메모리가 raster 크기에 비례) 여전히 OOM 위험이 있고,
  raster가 작으면 불필요하게 병렬성을 낮게 잡는다. 이제 `psutil`로 가용
  메모리를 확인하고, raster 셀 수 × `memory_overhead_factor`(기본 8.0 —
  158M셀 raster에서 실측된 워커당 ~10GB를 반영)로 워커당 예상 메모리를
  추정해서, `가용 메모리 × safety_fraction(기본 0.7) ÷ 워커당 예상 메모리`
  만큼만 워커를 띄운다. `calculate_distance_matrix()`의
  `n_workers_override`를 안 주면(기본 경로) 이 자동 산정이 적용되므로,
  **06단계뿐 아니라 07단계(Road/River 결합 resistance, `calculate_distance_matrix`
  6회 호출)에도 동일하게 적용된다.** raster 특성에 따라 실제 배수가 다르면
  `config.river_scan_memory_overhead_factor` / `river_scan_max_workers` 등으로
  조정 가능 (`config_ksd.py` 참고). `psutil`이 없으면 CPU 코어 수 기준으로만
  폴백하고 경고를 남긴다.
- **(수정됨) `calculate_distance_matrix`가 raster 픽셀 해상도를 무시하던
  버그**: `skimage.graph.MCP_Geometric`은 `sampling` 파라미터를 안 주면
  픽셀 한 칸을 "1 단위"로 취급한다. 이 파라미터를 빼먹고 있어서, River/Road/
  결합 resistance distance(06, 07단계에서 쓰는 `calculate_distance_matrix`
  전부)의 결과가 실제 미터가 아니라 **"픽셀 개수 기준" 값**이었다. 이제
  `transform`에서 실제 픽셀 크기를 뽑아 `sampling=(abs(transform.e),
  abs(transform.a))`로 넘기도록 고쳤다 — 20칸 이동 시 해상도 1m면 20m,
  28.5m면 570m로 정확히 스케일링되는 것을 실측 검증함.
  **다만 이 버그가 통계적 결론(Mantel r/p-value, MMRR 회귀계수/R²)에는
  영향이 없었다** — Mantel test는 상관계수 기반이라 두 행렬 중 하나에
  곱해진 상수배(raster 해상도)는 상관관계 자체를 바꾸지 않고, MMRR의
  예측변수도 z-score 표준화(`(x-mean)/std`)를 거치므로 상수배가 그 과정에서
  상쇄된다. **영향을 받는 것은 CSV/GeoTIFF로 저장되는 raw distance 값의
  절대 크기(단위가 "m"라고 표기돼 있었지만 실제로는 픽셀 개수였음)** —
  이미 돌려서 저장해둔 06/07단계 결과가 있다면, 통계 결과(Mantel/MMRR)는
  다시 볼 필요 없지만, raw distance 숫자를 논문 등에 절대값(m)으로 직접
  인용했다면 재계산이 필요하다.
- **(수정됨) `step06_river_scan`의 best value 중복 계산 버그**: R 원본
  구조를 그대로 따라 스캔 완료 후 best value를 "원본 해상도로 재계산"하는
  섹션이 있었는데, `river_scan_aggregate_factor`가 기본값 1(저해상도 스캔
  미사용)일 때는 스캔 루프 자체가 이미 원본 해상도로 돌기 때문에 이 재계산이
  완전히 중복이었다. 이제 `scan_aggregate_factor <= 1`이면
  `river_scan_list[best_idx]["distance_matrix"]`를 그대로 재사용하고,
  `scan_aggregate_factor > 1`(저해상도 스캔을 실제로 쓴 경우)일 때만
  best value 하나에 대해 원본 해상도로 재계산한다.
- **한글/비-라틴 폰트**: matplotlib 기본 폰트는 한글 글리프가 없어 그림의
  한글 텍스트가 깨진다. 모든 플로팅 코드는 `matplotlib.pyplot`을 직접
  import하지 않고 `landgen.utils.get_pyplot()`을 통해서만 가져와야 한다.
  폰트 우선순위는 **맑은 고딕(한글) + Tahoma(라틴 문자) — Windows 기본
  내장 폰트를 최우선으로 사용**하고, 맑은 고딕이 없는 환경(Mac/Linux)에서만
  `koreanize-matplotlib`(NanumGothic 내장)로 자동 폴백한다. 새 플롯을
  추가할 때도 반드시 `get_pyplot()`을 쓸 것 — 안 그러면 한글이 다시 깨짐.
  부수적으로, 로그축에 mathtext 지수 표기("10⁻¹")를 쓰면 마이너스 글리프가
  깨지는 문제도 있어서, `06_river_scan`의 스캔 플롯은 로그축 자동 눈금
  대신 실제 후보값을 직접 눈금으로 표시하도록 바꿨습니다 (오히려 가독성도
  더 나음).
- **`extract_buffer_mean()`**: `rasterio.features.geometry_mask`로 버퍼
  폴리곤 내부 셀을 마스킹하는 방식. 경계 셀 포함 규칙(centroid-in vs
  any-overlap)이 GIS 소프트웨어마다 다를 수 있으니, 정밀도가 중요한 최종
  결과라면 다른 도구의 결과와 교차검증 권장.
- **환경변수**: 데이터 루트 경로는 `LANDGEN_DATA_DIR` 환경변수로 지정
  가능 (`resolve_data_root()` 참고). 지정하지 않으면 OS별 기본 경로를 따름.
- **MMRR permutation**: `sample()` 반복 대신 `argsort(random)`을 열 단위로
  적용해 벡터화되어 있음. 유효한 무작위 순열이지만 다른 소프트웨어와 동일한
  permutation 순서가 나오는 것은 아니므로, 다른 도구 결과와 permutation
  p-value를 소수점 단위까지 비교하려면 별도 검증이 필요함 (표본이 충분히
  크면 유의성 판단 자체는 일치해야 함).

## 설치

```bash
pip install -r requirements.txt
```

## 실행

```bash
python3 main.py
```
