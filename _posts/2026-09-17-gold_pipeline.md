---
title: "[DataPipeline] Gold Pipeline 결측치·이상치 다형성 전처리를 통한 18종 실험 환경 구축"
categories: [Engineering, AssetMind]
tags: [AssetMind, Data Pipeline, Gold, Reader, Preprocessor, Loader]
---

# 대규모 금융 시계열의 결측·이상치 다형성 정제 및 18종 실험 환경 구축: 20거래일 윈도우 역산 수집과 데이터 무결성 보존 아키텍처

**Executive Summary (핵심 요약)**

> **문제**: 결측치와 이상치에 대한 자의적 판단은 시계열 고유의 변동성 분포와 공분산 구조를 파괴하여 데이터 자산 전체의 신뢰성을 무너뜨림. 또한 20거래일 역산 수집 및 복합 결측·이상치 처리 시 18개 매트릭스의 차원 불변성(Invariant)을 보장해야 하는 높은 정합성 제약이 상존함.

> **해결**: 자의적 판단을 전면 배제하고, 결측 보간 3종(LOCF, Log Return, Kalman) × 이상 탐지 3종(IQR, Z-Score, Isolation Forest) × 이상 정제 2종(Clipping, Masking)을 결합한 18종 다형성 실험 환경 파이프라인을 구축함. 데이터 분석을 통해 최적 방식을 선정한 뒤 설정(`preprocessor.yml`) 변경만으로 단일 최적 경로로 즉시 수렴(OCP)하도록 설계함.

> **결과**: 18종의 정제 데이터셋을 누락 없이 구축하여 데이터 분석 기반을 마련하고, 엄격한 차원 불변성 검증 및 Cold Start 가드를 통해 10년 8개월 백필(134,539개 객체, 78.2 GiB) 및 데일리 운영 간 장애율 0.0%의 완전한 데이터 정합성을 달성함.

---

## 1. 문제 배경 및 목적 (Context & Problem)
골드 파이프라인은 실버 버킷(S3 Parquet)에 적재된 정형 와이드 테이블을 바탕으로 결측치와 이상치를 정밀 진단·정제하여, 분석 및 모델링을 포함한 모든 다운스트림 환경이 안전하게 소비할 수 있는 무결한 데이터셋을 사출·적재하는 책임을 갖는다.

초기 파이프라인 설계 및 운영 과정에서 직면했던 핵심 엔지니어링 과제는 다음 세 가지로 집약된다.

> **임의적 판단이 초래하는 데이터 자산의 근본적 왜곡과 붕괴**

결측치나 이상치를 마주했을 때 데이터 분석가의 자의적인 추측으로 단일 처리 방식을 강제하는 것은 원천 데이터가 지닌 통계적 실체와 인과 관계를 영구히 훼손하여 전체 데이터 자산의 신뢰성을 무너뜨린다. 임의의 외삽이나 보간을 남발하면 실제 시장에 실재하지 않았던 가짜 잔차(Artifact)가 주입된다. 더욱이 특정 시점이나 자산의 이상치를 임의로 소거하면 금융 시장의 핵심 시그널인 꼬리 위험(Tail Risk)과 자산 간 공분산(Covariance) 구조가 완전히 뒤틀린다.

> **20거래일 시계열 컨텍스트 역산 수집(Lookback Ingestion)의 무결성 요구**

수리 통계 연산을 수행하려면 당일 1일 치 데이터만으로는 불가능하며, 최소 과거 20거래일 분량의 시계열 컨텍스트가 필수적이다. 또한 수집 초기 20일 구간과 같이 물리적으로 데이터가 부족한 백필 시작점에서도 파이프라인이 중단되지 않고 완주할 수 있는 Cold Start 방어벽이 필수적이었다. 그리고 휴장일로 인한 데이터 공백으로 인해 최대 40일을 과거로 거슬러 올라가며 실제 거래가 체결된 20거래일을 동적으로 역산 수집(Lookback Ingestion)하는 탐색 가드레일이 요구되었다.

> **복합 결측·이상치 동적 라우팅 및 18종 매트릭스의 엄격한 차원 불변성(Invariant) 사수**

금융 시계열은 단기(~10%), 중기(10~25%), 장기(25%~) 결측에 요구되는 수식이 상이하며, 일변량 극단값과 다변량 공분산 왜곡이 혼재되어 단일 진단이 불가능하다. 이에 대응해 결측 보간 3종 × 이상 탐지 3종 × 이상 정제 2종를 거쳐 18개의 데이터셋을 구축하는 과정에서, 판다스 내부 축 뒤틀림이나 형상 붕괴(Shape Invariant 파괴) 리스크가 상존한다. 따라서 복합 패턴을 동적으로 라우팅하면서도 18개 매트릭스 전수의 형상과 정렬축을 오차 없이 100% 보존하는 엔지니어링 정합성이 요구되었다.

---

## 2. 해결 대안 및 기술적 의사결정 (Alternatives & Trade-offs)
골드 파이프라인 전처리 계층의 핵심 목표는 분석가의 자의적 판단을 배제하고, 금융 시계열의 결측과 이상치를 다각도로 정제한 18종 실험 환경을 구축하는 것이다. 이를 통해 데이터 분석 과정에서 객관적인 실증을 거쳐 최적의 정제 방식을 선정한 뒤, 최종 운영 단계에서는 코드 수정 없이 단 1가지 최적 경로로 즉각 수렴할 수 있는 아키텍처를 구축하고자 했다.

이를 달성하기 위해 검토한 기술적 대안과 최종 엔지니어링 의사결정은 다음과 같다.

| 아키텍처 과제 | 기각된 대안 (Rejected) | 채택한 설계 (Adopted) | 핵심 선정 근거 및 트레이드오프 |
| :--- | :--- | :--- | :--- |
| **결측·이상치 전처리 수렴 전략** | 엔지니어 임의 판단 기반 단일 기법 고정 (단순 ffill 일괄 대치 / Z-Score 일괄 제거) | **18종 다형성 실험 환경 구축 후 설정(`preprocessor.yml`) 기반 단일 경로 수렴 (OCP)** | 자의적 판단 없이 18개의 데이터셋을 구축하고 데이터 분석을 통해 최적 방식을 선정. 운영 단계에서는 코드 수정 없이 YML 설정 변경만으로 단 1개의 최적 경로만 기동하도록 OCP를 확립함. |
| **결측 기간별 보간 (Imputation) 구조** | 단일 보간 알고리즘 전면 적용 또는 결측 자산 컬럼 일괄 삭제 | 결측 기간에 따라 요구되는 통계적 내재 연속성이 상이하므로, 단기(10% 이하), 중기(25% 이하), 장기(25% 초과)로 동적 분기하여 처리함. |
| **이상치 다차원 탐지 (Diagnosis) 체계** | 사분위수(IQR) 또는 정규화 점수(Z-Score) 등 일변량 통계 룰 단일 적용 | **일변량(IQR, Z-Score) + 자산축 전치 다변량 Isolation Forest 복합 진단** | 일변량 통계량은 자산 간 공분산 붕괴나 다차원 패턴 왜곡을 포착할 수 없음. 자산을 샘플로, 20일 시퀀스를 피처로 전치하여 비정상 자산 궤적을 공간적으로 격리함. |
| **이상치 정제 (Refinement) 메커니즘** | 극단값 단일 삭제(Drop) 또는 임의의 상수 일괄 대치 | **방향성 보존(Clipping) 및 Two-Pass 재보간(Masking) 병행 사출** | 변동성 쇼크의 방향성 시그널을 보존하는 방식(Winsorization)과, 이상치를 결측치로 치환 후 시계열 결에 맞춰 재성형하는 상반된 두 정제 방식을 데이터 분석에서 실증 비교하기 위함. |

의사결정의 핵심 축은 **'데이터 기반(Data-Driven) 실증주의'**와 **'설정 기반 개방-폐쇄 원칙(OCP)'**이었다.

> **자의적 판단을 배제하기 위해 18종 다형성 실험 환경 구축**

금융 시계열은 자산군마다 통계적 성향이 상이하므로 결측 보간 3종 × 이상 탐지 3종 × 이상 정제 2종를 결합해 18개의 실험 경로를 도출했다. 메모리와 I/O 오버헤드가 발생하지만, 자의적 가정을 배제하고 데이터 분석을 통해 최적 조합을 실증하기 위함이다. 분석을 통해 최적 전처리가 판별되면, `preprocessor.yml` 설정 변경만으로 최적 경로로 즉시 전환(OCP)된다.

> **결측치 : 시계열 연속성을 고려한 결측치 단계별 라우팅**

결측 컬럼 삭제로 인한 매트릭스 붕괴를 막기 위해 결측 기간별 동적 라우팅을 채택했다. 단기(~10%)는 3종 버킷(LOCF, Log Return, MA)으로 분기하고, 중기(10~25%)는 단변량 칼만 필터 RTS 평활화로 관측 노이즈를 제거했다. 장기(25%~)는 0.00으로 동결하되 지시자 마스크와 샘플 가중치(0.0)를 병렬 사출하여 분석 시 오염 구간을 선택적으로 배제(Loss Neutralization)하도록 설계했다.

> **이상치 : 다차원 이상치 교차 탐지와 방향성 보존 및 Two-Pass 재보간 정제**

일변량 통계량(IQR, Z-Score)과 자산축 전치 다변량 Isolation Forest를 결합해 국소 급등락과 공분산 왜곡을 동시 포착했다. 정제는 이상치를 사분위 상하한선으로 수렴시켜 방향성을 보존하는 Clipping과, NaN 치환 후 시계열 보간 체인으로 매끄럽게 재성형하는 Two-Pass Masking을 병행했다. 이는 시장의 꼬리 위험 보존과 노이즈 제거 중 어떤 방식이 데이터 가치를 덜 훼손하는지 분석 단계에서 실증 비교하기 위함이다.

---

## 3. 핵심 구현 및 아키텍처 (Implementation)
골드 파이프라인은 데이터 수집(Ingestion), 다형성 전처리(Preprocessing), 분산 적재(Loader)의 3단계 계층 분리 아키텍처로 설계되었다. 각 컴포넌트는 단일 책임 원칙(SRP)을 준수하며, 설정 파일(`preprocessor.yml`) 기반의 팩토리 패턴을 통해 하위 태스크 체인을 선언적으로 조립 구동한다.

전체 데이터 흐름과 레이어별 처리 구조는 다음과 같다.

```text
[Silver S3 Storage (Parquet Wide Table)]
│
▼
[1. INGESTION PHASE : GoldPipeline]
│ 과거 최대 40일 역산 탐색 ─── 20거래일 시계열 윈도우(20 x 190+) 확보 & Cold Start 가드
▼
[2. PREPROCESSING PHASE : PreprocessorService]
│ PreprocessorFactory 기반 yml 선언적 체인 조립 (OCP)
├─ MissingValueDiagnosis : 가짜 0.00 식별, 연속 결측장/휴장일 진단, 샘플 가중치 매트릭스 생성
├─ MissingValueImputation : 결측 기간별 동적 라우팅 (단기: 3종 버킷 / 중기: Kalman RTS / 장기: 격리)
├─ OutlierDiagnosis : 일변량(IQR, Z-Score) + 전치 다변량 Isolation Forest 교차 진단 (3 x 3)
└─ OutlierRefinement : 3중 연쇄 루프(3 x 3 x 2) 18종 버킷 사출 & Two-Pass 재보간 연쇄
│ 엄격한 차원 불변성 검증 (_validate_invariants: 18개 버킷 형상 및 인덱스 정합성 사수)
▼
[3. LOADER PHASE : GoldPipeline & S3ParquetLoader]
│ year/month/day Hive 파티션 컬럼 주입 & TransformedDTO 캡슐화
│ asyncio.to_thread 기반 비동기 스레드 오프로딩 & Zstd 압축 Parquet 적재
▼
[Gold S3 Data Lake (18종 다형성 Parquet 데이터셋)]
```

핵심 구현 메커니즘은 다음 세 가지 엔지니어링 기법으로 집약된다.

> **20거래일 시계열 컨텍스트 역산 탐색(Lookback Ingestion) 및 Cold Start 방어**

기준일(`target_date`)로부터 과거 최대 40일을 거슬러 올라가며 실제 거래가 체결된 20거래일 분량의 실버 와이드 데이터를 동적으로 역산 수집한다. 휴장일로 인한 공백을 극복하고 `lookback_days >= 20`이 충족되면 조기 탐색을 종료하여 불필요한 S3 I/O를 차단한다. 
더불어 백필(Backfill) 시작점 등 데이터 수집 초기 20일 구간에서는 과거 데이터가 물리적으로 부족하므로, 파이프라인 중단을 방지하기 위해 `INSUFFICIENT_WINDOW` 상태로 전환한 뒤 `ffill().bfill().fillna(0.0)`로 안전하게 메우는 Cold Start Fallback 방어벽을 가동했다.

> **결측치 진단 및 통계적 성향별 3종 보간 동적 라우팅**

`MissingValueDiagnosis`는 수치형 0.00 상수를 찾아 통계적 NaN과 결합한 통합 결측 매트릭스를 식별하고, 자산별 연속 결측장과 전체 자산 동시 결측 비율을 산출한다. 
`MissingValueImputation`은 이 진단서를 바탕으로 결측 기간에 따라 세 갈래로 동적 라우팅한다. 1~2일의 단기 결측은 자산의 거시적 성향을 비교하기 위해 직전값 유지(`LocfImputer`), 로그 수익률 추세 외삽(`LogReturnImputer`), 이동평균 회귀(`MovingAverageImputer`)의 3종 독립 버킷으로 분기 복제한다. 3~5일의 중기 결측은 세 버킷 모두에 단변량 칼만 필터(Kalman Filter)와 RTS(Rauch-Tung-Striebel) Smoothing을 적용하여 관측 노이즈를 제거한 매끄러운 궤적을 복원한다. 6거래일 이상의 장기 결측은 무리한 수치 대치를 배제하고 상수 0.00으로 동결하되, `missing_indicator_mask`와 `target_sample_weights=0.0`을 생성하여 분석 환경에서 결측 구간을 선택적으로 배제(Loss Neutralization)할 수 있도록 설계했다.

> **이상치 3중 연쇄 오케스트레이션과 엄격한 차원 불변성(Invariant) 사수**

`OutlierRefinement`는 3종 보간 버킷(`locf`, `log_return`, `moving_average`), 3종 탐지 엔진(`iqr`, `zscore`, `isolation_forest`), 2대 정제 엔진(`clipping`, `masking`)을 관통하는 3중 루프를 구동하여 총 18개의 정제 매트릭스를 사출한다. 특히 이상치를 NaN으로 치환하는 `MaskingRefinement` 버킷군에 대해서는, 결측치 보간(`ffill().bfill().fillna(0.0)`)을 수행하는 Two-Pass Imputation 메커니즘을 완성하여 이상치 노이즈를 소거한 시계열을 복원했다.
18개의 데이터셋이 복잡한 수리 연산을 거치는 동안 판다스 내부의 정렬축 뒤틀림이나 형상 붕괴를 원천 차단하기 위해, 사출 직전 원본 데이터와의 1:1 차원 불변성(`_validate_invariants`) 검증을 강제했다. 계약된 18개 버킷 수, 2D 행렬, 타임스탬프 인덱스, 자산코드 컬럼축이 오차 없이 원본과 일치해야만 통과된다. 검증을 통과한 데이터는 `year/month/day` Hive 파티션 메타데이터를 바인딩하고, `asyncio.to_thread`를 통해 워커 스레드로 오프로딩되어 Zstd 압축 Parquet로 비동기 분산 적재된다.

다음은 단일 휴리스틱 처리 방식과 개선된 18종 다형성 실험 환경 구축 및 차원 불변성 검증의 핵심 구현 대비다.
```python
# [Before: 단일 휴리스틱 처리 및 자의적 결측 자산 삭제]
# 엔지니어의 자의적 판단으로 단순 ffill을 강제하고, 장기 결측 자산을 일괄 dropna하여 매트릭스 형상 붕괴
def naive_preprocess(df: pd.DataFrame) -> pd.DataFrame:
    # 1. 자의적 결측 대치: 자산 고유의 통계적 성향을 무시하고 직전값 일괄 고정
    cleaned_df = df.ffill().bfill()
    
    # 2. 임의적 이상치 제거: Z-score 단일 기준으로 3-시그마 밖을 결측 처리 후 컬럼 삭제
    z_scores = (cleaned_df - cleaned_df.mean()) / cleaned_df.std()
    cleaned_df = cleaned_df.mask(z_scores.abs() > 3.0, np.nan)
    
    # 3. 행렬 구조 파괴: 결측치가 남은 컬럼을 삭제하여 다운스트림 연산 규격 붕괴
    return cleaned_df.dropna(axis=1)
```

```python
# [After: OutlierRefinement - 3중 루프 18종 다형성 버킷 사출 및 차원 불변성 검증]
def execute(self, imputation_artifacts: Dict[str, pd.DataFrame], 
            diagnosis_report: Dict[str, Dict[str, pd.DataFrame]]) -> Dict[str, pd.DataFrame]:
    self._validate_inputs(imputation_artifacts, diagnosis_report)
    outlier_refinement_artifacts = {}

    # 1. 보간(3종) x 탐지(3종) x 정제(2종) 3중 연쇄 오케스트레이션 루프
    for impute_key, current_price_df in imputation_artifacts.items():
        impute_name = impute_key.replace("bucket_", "")
        for detect_key in ["iqr", "zscore", "isolation_forest"]:
            current_mask_df = diagnosis_report[impute_key][detect_key]
            for refine_key, refinement_engine in self._refinement_registry.items():
                refined_df = refinement_engine.refine(current_price_df, current_mask_df)
                artifact_key = f"bucket_impute_{impute_name}_detect_{detect_key}_refine_{refine_key}"
                outlier_refinement_artifacts[artifact_key] = refined_df

    # 2. 사출 직전 18개 버킷 전수의 셰이프 및 인덱스/컬럼축 불변성(Invariant) 검증
    self._validate_invariants(imputation_artifacts["bucket_locf"], outlier_refinement_artifacts)
    return outlier_refinement_artifacts

def _validate_invariants(self, sample_df: pd.DataFrame, final_artifacts: Dict[str, pd.DataFrame]) -> None:
    assert len(final_artifacts) == 18, f"계약 본수(18개) 불일치: {len(final_artifacts)}개 사출"
    for k, v in final_artifacts.items():
        if sample_df.shape != v.shape:
            raise OutlierRefinementExecutionError(f"차원 붕괴 감지: [{k}] {v.shape} != {sample_df.shape}")
        if not sample_df.index.equals(v.index) or not sample_df.columns.equals(v.columns):
            raise OutlierRefinementExecutionError(f"정렬축 뒤틀림 감지: [{k}] 인덱스/컬럼 불일치")
```

## 4. 검증 및 결과 (Validation & Metrics)
골드 파이프라인의 핵심 검증 목표는 엔지니어의 자의적 판단을 배제한 18개 실험 데이터셋을 누락 없이 구축하고, 약 10년 8개월(2016년 1월 ~ 2026년 8월)에 달하는 대규모 시계열 배치 과정에서 단 1셀의 축 뒤틀림이나 형상 붕괴 없이 완전한 데이터 정합성을 보존하는 데 있다.

검증을 통해 달성한 핵심 엔지니어링 성과는 다음과 같다.

> **엔지니어의 자의적 판단을 배제한 18종 다형성 실험 환경 구축 완결**

결측치와 이상치 처리에 자의적 편향이 개입되던 구조를 완전히 탈피했다. 금융 시계열의 다양한 통계적 성향을 반영한 3종 보간(LOCF, Log Return, Moving Average)과 3종 이상치 탐지(IQR, Z-Score, Isolation Forest), 그리고 상반된 두 가지 정제 방식(Clipping, Masking)을 완벽히 결합하여 총 18개의 독립적인 데이터셋을 분석 환경에 성공적으로 공급했다. 이를 통해 향후 진행될 데이터 분석 단계에서 어떤 전처리 경로가 금융 시계열의 고유한 통계적 실체를 가장 훌륭하게 보존하는지 데이터 기반(Data-Driven)으로 직접 비교·선정할 수 있는 완결된 실험 기반을 마련했다.

> **복합 결측·이상치 동적 라우팅을 통한 시계열 통계적 실체 사수**

결측이 발생한 자산 행이나 컬럼을 물리적으로 삭제하여 데이터 매트릭스의 구조를 파괴하던 문제를 근본적으로 해결했다. 결측 기간별 동적 라우팅을 통해 단기 결측은 자산의 거시적 성향에 맞춰 다형성 분기를 수행하고, 중기 결측은 단변량 칼만 필터(Kalman Filter)와 RTS 평활화를 통해 관측 노이즈를 제거한 최적 궤적으로 복원했다. 6거래일 이상의 장기 결측 구간은 무리한 외삽을 배제하고 상수(0.00)로 동결하되, 지시자 마스크와 샘플 가중치를 병렬 사출하여 다운스트림 분석 환경에서 오염 구간을 선택적으로 배제(Loss Neutralization)할 수 있는 완벽한 통계적 방어벽을 확립했다.

> **엄격한 차원 불변성(Invariant) 검증 및 10년 8개월 무결격 적재 완주**

18개의 데이터셋이 복잡한 수리 연산 루프를 거치는 동안 발생할 수 있는 판다스 내부의 축 뒤틀림이나 인덱스 불일치를 차단하기 위해 `_validate_invariants` 가드레일을 운용했다. 매 배치마다 18개 버킷 수, 2D 행렬, 타임스탬프 인덱스, 자산코드 컬럼 순서가 원본과 1:1로 완벽히 일치하는지 검증했다. 아울러 데이터 수집 초기 20거래일 미만 구간의 수렴 실패를 Cold Start Fallback 로직으로 방어함으로써, 10년 8개월치 백필 및 데일리 운영 간 누적 134,539개의 Parquet 객체를 단 1건의 장애나 데이터 누락 없이(장애율 0.0%) 100% 무결하게 적재 완결했다.

---

## 5. 프로덕션 관점의 한계와 과제 (Production Readiness & Next Step)
현재 골드 파이프라인은 자의적 편향을 배제하고 결측·이상치를 다각도로 정제한 18종 다형성 데이터셋을 100% 무결하게 사출하는 구조적 안정성을 입증했으나, 프로덕션 상용화 및 대규모 확장 관점에서 다음과 같은 한계와 차기 과제를 내포하고 있다.

> **파생 피처 생성의 데이터 분석 이관에 따른 구조적 한계**

최초 기획 시 파생 피처 생성을 일괄 처리하고자 했으나, 전체 파이프라인 정합성을 사수하는 과정에서 초과 작업이 발생했다. 이에 따라 파생 피처 생성을 분석 단계로 이관하고, 골드 파이프라인은 '18개 데이터셋의 정밀 사출 및 무결 적재'라는 단일 책임(SRP)에 집중했다. 파이프라인 정합성이 완비된 만큼, 분석 결과를 기반으로 파생 피처 생성 고도화가 요구된다.

> **정적 임계치(Static Threshold)의 한계와 비지도 기반 시장 국면(Regime) 적응형 탐지 과제**

이상치 엔진은 고정된 정적 임계치(IQR 1.5, Z-Score 3.0 등)를 적용하므로, 국면별 변동성 차이를 반영하지 못해 위기 국면의 꼬리 위험(Tail Risk)을 오탐지할 위험이 있다. 차기 단계에서는 프로토타입 단계에서 검증한 바 있는 **오토인코더(Autoencoder) 기반 잠재 공간(Latent Space) 클러스터링 기법**을 연계해 시장 국면별 탄력적 임계 체계로 진화시킬 필요가 있다. 다만 딥러닝 모델 도입 시 발생하는 GPU 비용, 지연 시간, 모델 드리프트(Drift) 관리에 대한 실무적 트레이드오프가 함께 고려되어야 한다.

> **장기 결측 자산의 상수(0.00) 동결 한계와 모델 기반 예측 대치(Model-based Imputation) 도입 과제**

6거래일 이상 결측 자산을 0.00으로 동결하고 손실 중립화 마스크를 부여한 구조는 모델 격리에는 유효하나, 단순 통계 분석 환경에서는 단면 통계량을 왜곡할 위험이 있다. 향후 18개 실험을 거쳐 최적의 예측 모델이 선정되면, 해당 모델을 활용해 장기 결측을 정밀 예측 대치(Model-based Imputation)하는 확장이 가능하다. 이때 모델이 예측한 값으로 데이터를 다시 채우며 발생하는 순환 데이터 누수(Circular Leakage)를 차단하기 위해, 미래 시점을 보지 않는 엄격한 자기회귀(Autoregressive) 구조의 2단계 분리 파이프라인 구축이 요구된다.
