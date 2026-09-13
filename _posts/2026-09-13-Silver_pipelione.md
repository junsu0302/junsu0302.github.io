---
title: "[DataPipeline] Silver Pipeline: 189개 시계열 지표의 와이드 테이블 합성 및 고속 Parquet 파이프라인"
categories: [Engineering, AssetMind]
tags: [AssetMind, Data Pipeline, Silver, Reader, Transformer, Builder, Loader]
---

# 대규모 금융 시계열의 와이드 테이블 합성 및 고속 Parquet 파이프라인: 189개 지표 종합 1,456개 피처 정형화 아키텍처

**Executive Summary (핵심 요약)**
> **문제** : 브론즈 레이어에 적재된 189개 시계열 데이터의 파편화된 중첩 구조(Nested JSON/배열)와 동일 필드명 충돌 속에서, 루프 기반 순차 조인(Iterative Outer Join) 시 발생하는 메모리 재할당 오버헤드 및 다운스트림 분석을 위한 표준 정형화 규격 부재가 상존함.

> **해결** : 스키마 강제화 분리(Strict Data Contract), `trade_date` 인덱스 승격 후 Pandas C-엔진 기반 `pd.concat(axis=1)` 1-Shot 병합 및 `{job_id}_{col}` 네임스페이스 격리, PyArrow Zstd 하이브 파티셔닝 적재 파이프라인을 구축함.

> **결과** : 순차 조인 대비 배치 완결 시간을 약 41.7% 단축함(12.0s → 7.0s). PyArrow Zstd 컬럼형 압축을 통해 비압축 원본 대비 데이터 용량을 약 55% 감축(일별 695.7 KiB, 10년 8개월 백필 총 3.7 GiB)하고, 다운스트림 파일 조회 I/O를 99.5% 절감(189회 → 1회)하여 운영 간 100% 무장애 정합성을 달성함.
---

## 1. 문제 배경 및 목적 (Context & Problem)
데이터 파이프라인의 실버 레이어는 브론즈 레이어(S3)에 적재된 비정형 원천 데이터를 읽어와 스키마 무결성을 검증·정제하고, 골드 레이어에서 수행할 파생 피처 생성 작업을 지원하며 원본 데이터를 직관적으로 검증할 수 있도록 단일 시계열 와이드 테이블(Wide Table)로 변환·보관하는 책임을 갖는다.

수집 대상은 한국투자증권(KIS), 한국은행(ECOS), 미국 연방준비제도(FRED), 업비트(UPBIT) 등 4개 기관, 총 189개 엔드포인트다. 파이프라인은 스케줄러(Airflow DAG) 및 태스크 레벨에서 아시아 지표군(117개)과 글로벌 지표군(72개)으로 물리적으로 격리되어 구동되며, 변환 및 병합을 거쳐 최종 생성되는 정형 피처의 규모는 종합 1,456개 컬럼에 달한다.

| 제공자 (Provider) | 원천 포맷 (Bronze) | 할당 엔드포인트 수 | 스키마 변환 및 평탄화 방식 | 주요 표준화 필드 | 정제 및 결합 시 주요 기술 과제 |
| :--- | :--- | :---: | :--- | :--- | :--- |
| **KIS (국내 지수)** | Nested JSON (`output1`, `output2`) | 59개 (KOSPI 29, KOSDAQ 30) | `output2` 배열 explode 후 `pd.json_normalize` 전개 | `close`, `open`, `high`, `low`, `volume` 등 | 불필요 메타(`rt_cd`, `msg1`) 제거, 8자리 문자열 일자 파싱 |
| **KIS (글로벌/원자재)** | Nested JSON (`output1`, `output2`) | 63개 (글로벌 지수, 환율, 원자재) | 중첩 구조 전개 및 지수/원자재 하이브리드 바인딩 | `close`, `price_change_rate` 등 | 상이한 필드명(`ovrs_nmix_*`, `ovrs_prod_*`) 단일 매핑 |
| **FRED (미국 연은)** | JSON (`observations` List) | 4개 (미국 국채 2Y, 5Y, 10Y, 30Y) | `observations` explode 및 `json_normalize` 전개 | `trade_date`, `value` | 휴일 문자열 결측치(`"."`)의 안전한 `NaN` 캐스팅 |
| **ECOS (한국은행)** | Nested JSON (`StatisticSearch.row`) | 28개 (기준금리, 국고채, 시장금리) | `StatisticSearch.row` 배열 추출 후 explode 전개 | `trade_date`, `value` | 비표준 일자 키(`TIME`) 사내 표준 매핑, 단일 수치 변환 |
| **UPBIT (업비트)** | Flat JSON Array (`[Dict, ...]`) | 5개 (KRW 마켓 주요 가상자산) | Zero-Overhead 1D 패스스루 및 스키마 강제 | `trade_date`, `close`, `volume` 등 | 가상자산 캔들 데이터 타입 강제(`float64`/`float32`) |

실버 파이프라인을 설계하는 과정에서 해결해야 했던 핵심 과제는 다음 세 가지로 집약된다.

> **이종 금융 기관 간 응답 데이터의 중첩 구조 파편화**

기관 별 각기 다른 데이터 구조를 개별 파이프라인마다 하드코딩으로 처리하면 모듈 간 결합도가 급증한다. 외부 API 변경에 유연하게 대응하면서도 다운스트림 레이어로 원치 않는 가비지 데이터가 유입되지 않도록 통제하는 엄격한 데이터 계약(Strict Data Contract) 메커니즘이 필수적이었다.

> **N개 시계열 결합 시의 메모리 재할당 및 연산 병목**

189개 개별 데이터프레임을 단일 시계열 와이드 테이블로 조립할 때, 루프 기반의 순차적 조인 방식을 취하면, 매 반복마다 전체 데이터프레임 버퍼가 메모리에 재할당·복사되는 심각한 오버헤드가 발생한다. 조인 대상 지표가 누적될수록 연산 복잡도가 $O(N^2)$로 증가하여 배치 지연을 유발하므로, 인메모리 상에서 단번에 가로 결합을 수행하는 고속 병합 구조가 요구되었다.

> **대규모 피처 결합에 따른 컬럼 네임스페이스 충돌**

아시아와 글로벌 파이프라인을 통틀어 종합 1,456개 컬럼에 달하는 피처 매트릭스를 구성하는 과정에서, 다수의 테이블이 동일한 컬럼명을 중복으로 보유하게 된다. 네임스페이스가 격리되지 않은 상태로 단순 병합될 경우 판다스 내부에서 `_x`, `_y`와 같은 접미사가 임의로 부착되어 스키마가 오염되고, 골드 레이어에서 특정 지표의 피처를 식별할 수 없는 치명적인 결함이 발생한다.

---

## 2. 해결 대안 및 기술적 의사결정 (Alternatives & Trade-offs)
실버 파이프라인의 핵심 설계 목표는 189개 시계열 데이터를 지연 없이 정형화하고, 종합 1,456개 컬럼에 달하는 대규모 피처를 메모리 단편화나 컬럼 충돌 없이 안전하게 단일 와이드 테이블로 합성·적재하는 것이었다.

이를 달성하기 위해 검토한 기술적 대안과 엔지니어링 의사결정은 다음과 같다.

| 아키텍처 과제 | 기각된 대안 (Rejected) | 채택한 설계 (Adopted) | 핵심 선정 근거 및 트레이드오프 |
| :--- | :--- | :--- | :--- |
| **다중 시계열 가로 병합 방식** | 루프 기반 순차 `pd.merge(how='outer')` 반복 | `set_index` 승격 후 C-엔진 기반 **`pd.concat(axis=1)` 1-Shot 병합** | 루프 조인은 N-1번의 전체 버퍼 메모리 복사 및 인덱스 재탐색 오버헤드를 수반함. 반면 조인 키를 인덱스로 승격한 뒤 1-Shot concat을 수행하면 단일 메모리 할당으로 O(1) 수준의 고속 병합이 가능함. 결합 직후 `copy()`를 강제하여 내부 메모리 단편화(Fragmentation)를 해소함. |
| **컬럼 네임스페이스 충돌 방어** | 판다스 기본 자동 접미사(`_x`, `_y`) 위임 또는 수동 후처리 | 사전 인덱싱 단계에서 **`{job_id}_{col}` 접두사 맵핑** 강제 | 수백 개 지표의 동일 컬럼이 병합될 때 기본 접미사에 의존하면 출처 추적이 불가능해짐. 병합 전 컬럼명에 고유 작업 식별자를 바인딩하고, 단일 수치 지표는 컬럼명을 `job_id` 자체로 치환하여 네임스페이스를 완전히 격리함. |
| **스키마 무결성 및 정제 제어** | 변환기 내부에서 조작과 캐스팅을 일괄 처리하는 묵시적 변환 | `_apply_transform`과 **`_enforce_schema` 훅 분리 (Strict Contract)** | 데이터 값 조작(Flattening/Explode)과 데이터 그릇 형태 제어(타입 캐스팅/필터링) 책임을 엄격히 분리(SRP)함. YAML에 정의되지 않은 API 잉여 필드를 원천 드롭하고, 수치 데이터를 `float32`로 강제 캐스팅하여 메모리 사용량을 절감함. |
| **실버 Parquet 스토리지 적재 I/O** | Boto3 기반 수동 S3 Key 조합 및 멀티파트 업로드 구현 | Pandas/PyArrow의 S3 내장 통신(s3fs) 기반 **자동 파티셔닝 적재** | 연/월/일 파티션 디렉터리 분할 및 업로드 버퍼링을 직접 구현하는 것은 바퀴를 재발명하는 것임. PyArrow 엔진의 `partition_cols` 옵션과 Zstd 압축을 활용하여 I/O 성능을 극대화하고 보일러플레이트 코드를 완전히 소거함. |

의사결정의 핵심 축은 **'메모리 지역성(Memory Locality) 극대화'**와 **'엄격한 데이터 계약(Strict Data Contract)'**이었다.

> **대규모 시계열 결합의 병목을 해소하기 위해 루프 기반 조인을 완전히 배제**

단일 루프 조인은 매 순회마다 새로운 중간 데이터프레임을 생성하므로 가비지 컬렉터 부하와 메모리 피크를 유발한다. 대신 각 데이터프레임의 조인 키(`trade_date`)를 인덱스로 올리고 `pd.concat(axis=1)`을 수행함으로써 판다스 내부 C 레벨 블록 매니저가 단 한 번의 메모리 재할당으로 가로 결합을 끝내도록 설계했다. 수평 결합으로 인해 발생하는 블록 단편화는 `wide_df.copy()` 호출을 통해 연속된 메모리 공간으로 재배정하여 후속 인덱스 정렬 및 연산 시 발생하는 성능 저하 경고(`PerformanceWarning`)를 원천 차단했다.

> **스키마 제어 관점에서는 회복탄력성보다 엄격성을 우선시**

제공 기관마다 반환하는 JSON 필드가 수시로 추가되거나 변경될 수 있다. 변환기가 이를 유연하게 수용하도록 두면 정제되지 않은 가비지 컬럼이 실버 레이어로 유입되어 종합 1,456개 피처의 스키마 정합성이 훼손된다. 따라서 `AbstractTransformer` 템플릿 메서드 패턴을 통해 비즈니스 평탄화와 스키마 강제 단계를 분리하고, `transformer.yml`에 명시되지 않은 필드는 `_enforce_schema`에서 즉각 필터링되도록 강제했다. 또한 메모리 최적화를 위해 가격 및 거래량 지표의 기본 부동소수점 타입을 `float64` 대신 `float32`로 하향 통제함으로써 다운스트림 적재 및 조회 시의 I/O 처리량을 개선했다.

---

## 3. 핵심 구현 및 아키텍처 (Implementation)
실버 파이프라인은 Reader, Transformer, Builder, Loader의 4단계 계층 분리 아키텍처(Staged Architecture)로 설계되었다. 각 계층은 단일 책임 원칙(SRP)을 준수하며 인메모리 스트리밍 방식으로 데이터를 정제·합성한다. 

전체 데이터 흐름과 레이어별 처리 구조는 다음과 같다.

```text
[S3 Bronze Storage (Zstd Compressed JSONL)]
│
▼
[Reader Layer : S3ZstdStreamingReader]
│ Boto3 StreamingBody + zstandard On-the-fly 압축 해제 및 JSONL 파싱
▼
[Transformer Layer : TransformerService]
│ Prefix 라우팅 기반 구체 변환기 바인딩 (KIS, FRED, ECOS, UPBIT)
├─ 1. _validate : 중첩 전개 대상(explode_target) 무결성 사전 검증
├─ 2. _apply_transform : 배열 explode 및 pd.json_normalize 2D 평탄화
└─ 3. _enforce_schema : 사내 Data Contract 강제 (미정의 컬럼 필터링 및 타입 캐스팅)
│ Cleaned DataFrames (N개 도메인 지표)
▼
[Builder Layer : BuilderService / merger.py]
│ 사전 정합성 검증 (Fail-Fast) 및 1-Shot 벡터 병합
├─ 1. drop_duplicates(subset=['trade_date'], keep='last') 중복 인덱스 방어
├─ 2. set_index('trade_date') : 조인 키의 인덱스 승격
├─ 3. Dynamic Prefix Mapping : {job_id}_{col} 형태로 네임스페이스 격리
├─ 4. pd.concat(axis=1) : C-엔진 기반 단일 메모리 블록 1-Shot 수평 결합
└─ 5. wide_df.copy() : 대규모 가로 결합으로 발생한 메모리 단편화(Fragmentation) 해소
│ Wide DataFrame (종합 1,456개 피처)
▼
[Loader Layer : S3ParquetLoader]
│ PyArrow Engine (Zstd Compression & year/month/day Hive-style 자동 파티셔닝 적재)
▼
[S3 Silver Storage (Parquet Data Lake)]
```

핵심 구현 메커니즘은 세 가지 엔지니어링 기법으로 요약된다.

> **템플릿 메서드 패턴 기반의 스키마 강제화(_enforce_schema) 및 고속 일자 벡터화 파싱**

`AbstractTransformer`는 `transform()` 템플릿 내부에서 데이터 값 조작(`_apply_transform`)과 스키마 규격 강제(`_enforce_schema`)를 엄격히 격리한다. 이 구조를 통해 `transformer.yml`에 선언되지 않은 외부 API의 메타데이터나 잉여 필드를 원천 제거하여 데이터 레이크의 스키마 오염을 차단한다. 또한 `_cast_datetime_vectorized`를 구현하여 KIS 특유의 8자리 문자열(`YYYYMMDD`), ISO8601, 슬래시 구분자 등을 정규식 보정 후 `pd.to_datetime(errors='coerce')`로 단번에 파싱함으로써, 파이썬 for 루프 대비 수십 배 빠른 C-엔진 수준의 벡터화 일자 변환을 달성했다.

> **동적 접두사(Prefix) 네임스페이스 격리**

데이터프레임을 결합할 때 동일한 컬럼명이 공존하므로, `merger.py`는 병합 전 각 데이터프레임 컬럼에 `{job_id}_{col}` 형태의 접두사를 동적으로 맵핑한다. 단일 수치를 반환하는 컬럼은 컬럼명 자체를 해당 `job_id`로 치환하여 가독성을 높였다. 이로써 판다스가 중복 컬럼에 임의로 부여하는 `_x`, `_y` 접미사 오염을 방지하고 종합 1,456개 컬럼 전반에 걸쳐 피처의 출처를 명확히 고정했다.

> **인덱스 승격 기반 `pd.concat(axis=1)` 1-Shot 병합과 메모리 단편화 해소**

루프 기반 순차 조인 대신, 비즈니스 키(`trade_date`)를 인덱스로 설정한 뒤 `pd.concat(indexed_dfs, axis=1)`을 호출하여 단 한 번의 메모리 재할당으로 전체 피처를 결합한다. 결합 직후 `wide_df.copy()`를 호출하여 다중 컬럼 추가로 인해 분절된 인메모리 블록(Block Manager)을 연속된 메모리 공간으로 재정렬함으로써 다운스트림의 `PerformanceWarning`과 캐시 미스(Cache Miss)를 해소했다.

다음은 순차 조인 방식과 개선된 1-Shot 가로 결합 방식의 핵심 구현 대비다.

```python
# [Before: 루프 기반 순차 조인 방식]
# 매 반복마다 전체 데이터 버퍼 복사 및 O(N^2) 메모리 재할당 발생, 컬럼명 충돌(_x, _y) 유발
merged_df = pd.DataFrame()
for df in dfs:
    if merged_df.empty:
        merged_df = df
    else:
        # N개 테이블 병합 시 매번 인덱스 재탐색 및 버퍼 복사가 반복되어 심각한 병목 유발
        merged_df = pd.merge(merged_df, df, on="trade_date", how="outer")
```

```python
# [After: merger.py - 인덱스 승격, 네임스페이스 격리 및 1-Shot Vectorized Concat]
indexed_dfs = []

for df, job_id in zip(dfs, job_ids):
    if df.empty or merge_key not in df.columns:
        continue
        
    # 1. 중복 수집 데이터 선별 제거 및 조인 키의 인덱스 승격
    clean_df = df.drop_duplicates(subset=[merge_key], keep='last')
    df_indexed = clean_df.set_index(merge_key)
    
    # 2. 동적 접두사 맵핑으로 네임스페이스 충돌 원천 방어
    rename_map = {
        col: job_id if col.lower() == "value" else f"{job_id}_{col}"
        for col in df_indexed.columns
    }
    indexed_dfs.append(df_indexed.rename(columns=rename_map))

# 3. C-엔진 기반 단일 메모리 블록 1-Shot 수평 결합 (Outer Concat)
wide_df = pd.concat(indexed_dfs, axis=1)

# 4. 메모리 단편화(Fragmentation) 해소 및 인덱스 복원
wide_df = wide_df.copy()
wide_df = wide_df.sort_index().reset_index().rename(columns={"index": merge_key})
```

---

## 4. 검증 및 결과 (Validation & Metrics)
개선된 실버 파이프라인의 성능과 정합성 검증은 약 10년 8개월치 시계열 데이터에 대한 대규모 백필(Backfill) 및 일별 배치 운영 환경에서 실측되었다.
평가 축은 다중 시계열 결합 레이턴시(Latency), 스키마 정형화 규모, 다운스트림 조회 I/O, S3 Parquet 스토리지 효율, 그리고 인덱스 무결성 및 장애율의 다섯 가지로 설정했다.

| 평가 항목 (Metric) | 개선 전 (Before: 루프 조인 / 분절 적재) | 개선 후 (After: 1-Shot Concat / 와이드 Parquet) | 개선 성과 및 정량적 검증 결과 |
| :--- | :--- | :--- | :--- |
| **데일리 완결 시간** (Latency) | **약 12.0s** (아시아 ~7s, 글로벌 ~5s / 루프 조인 오버헤드) | **총 7.0s** (아시아 4.0s, 글로벌 3.0s / 1-Shot 병합) | **약 41.7% 지연 시간 단축** (189개 지표 전 과정 7초 완결) |
| **피처 매트릭스 형상** (Feature Dimension) | 개별 189개 시계열 분절 (동일 컬럼명 충돌 리스크 상존) | **종합 1,456개 컬럼** 단일 와이드 테이블 | **전 지표 단일화** ({job_id}_{col} 네임스페이스 격리 완결) |
| **다운스트림 조회 I/O** (S3 Get Object) | **189회** (분석 시 189개 브론즈 파일 개별 요청) | **1회** (단일 일자 와이드 Parquet 파티션 로드) | **I/O 요청 횟수 99.5% 절감** (네트워크 오버헤드 및 홉 제거) |
| **스토리지 적재 효율** (S3 Parquet / Zstd) | **약 1.5 MiB** (전체 약 8.2 GiB / 비압축 원본 약 70.8만 개) | **695.7 KiB** (전체 3.7 GiB / 단일 와이드 Parquet 7,487개) | **약 55% 스토리지 용량 감축** |
| **파이프라인 안정성** (무장애율) | 중복 일자 유입 시 InvalidIndexError 크래시 위험 상존 | **100% 무장애 (에러 0건)** | **10년 8개월 백필 및 데일리 운영 간 무결격 완주** |

정량적 개선 외에도 아키텍처 측면에서 다음과 같은 정성적 성과를 확보했다.

> **다운스트림(골드 레이어) 분석 생산성 및 원본 데이터 검증 편의성 극대화**

기존에는 분석가나 골드 파이프라인이 특정 일자의 시장 상황을 파악하기 위해 수십 개의 개별 브론즈 파일을 각각 열어 수동 조인해야 했다. 실버 파이프라인 구축 후에는 비즈니스 키(trade_date)를 기준으로 정렬된 단일 와이드 테이블(1,456개 피처)이 일자별 하이브 파티션(year/month/day)으로 균일하게 제공된다. 이로써 골드 레이어는 복잡한 I/O 병목 없이 즉시 시계열 롤링 지표나 Lag Feature 등 고차원 파생변수 생성 작업에 착수할 수 있게 되었다.

> **엄격한 데이터 계약(Strict Data Contract)을 통한 데이터 품질 보장**

AbstractTransformer 템플릿의 _enforce_schema 단계를 통해, 외부 API 응답의 잉여 메타데이터(rt_cd, msg1 등) 및 스키마에 정의되지 않은 비표준 필드가 데이터 레이크로 유입되는 현상을 100% 차단했다. 또한 가격 및 거래량 컬럼의 타입을 float32로 하향 통제하고 결측치를 안전하게 NaN으로 치환(Safe Coercion)함으로써, 인메모리 연산 효율과 데이터 일관성을 동시에 확보했다.

> **인덱스 방어 로직을 통한 결정론적 멱등성 확립**

상류 API 재시도나 중복 수집으로 인해 동일한 trade_date 레코드가 중복 유입되더라도, 병합 직전 drop_duplicates(subset=['trade_date'], keep='last')를 수행하여 판다스의 인덱스 고유성을 보장했다. 이를 통해 인덱스 충돌로 인한 배치 중단을 원천 차단하고, 언제 배치를 재실행하더라도 항상 최신의 동일한 단일 와이드 매트릭스가 산출되는 멱등적 파이프라인을 확립했다.

---

## 5. 프로덕션 관점의 한계와 과제 (Production Readiness & Next Step)
현재 구축된 실버 파이프라인은 189개 시계열 지표를 단일 와이드 테이블로 7초 만에 합성하며 높은 처리량과 무장애 정합성을 입증했으나, 대규모 확장 및 장기 운영 관점에서 다음과 같은 구조적 한계와 차기 과제를 내포하고 있다.

> **침묵하는 결측치(Silent Bypass) 방어 및 데이터 품질 모니터링(Data Observability) 체계 부재**

현재 파이프라인은 수집되지 못한 데이터를 `errors='coerce'`를 통해 `NaN`으로 안전 치환(Safe Coercion)하여 적재를 완결한다. 이는 파이프라인 전체 크래시를 방어하는 런타임 회복탄력성을 보장하지만, 지표의 장기 결측이나 이상치 유입을 실시간으로 감지하기 어렵다. 차기 단계에서는 실버 적재 직전에 데이터 품질 검증을 연동하여 지표별 결측률 및 값의 유효 범위를 자동 검증하고, 임계치 초과 시 즉각 경보를 발송하는 관측 체계 구축이 필수적이다.

> **종합 1,456개 컬럼 단일 와이드 테이블의 물리적 확장 한계와 피처 그룹(Feature Group) 분할**

현재는 1,456개 컬럼을 일자별 단일 Parquet 파일로 패킹하여 관리하고 있으나, 향후 개별 주식 종목이나 세부 매크로 지표가 수천 개 규모로 확장될 경우 단일 Parquet 파일의 Thrift 메타데이터 오버헤드가 급증하게 된다. 또한 다운스트림 모델이 단 몇 개의 거시지표만을 필요로 하는 경우에도 광역 스키마 메타데이터를 파싱해야 하는 비효율이 발생한다. 따라서 향후에는 단일 가로 병합 구조를 넘어, 도메인 특성에 맞는 단위로 논리적·물리적 **피처 그룹(Feature Group)**을 분할 적재하고 메타데이터 카탈로그로 유연하게 조합하는 아키텍처로의 전환이 요구된다.

> **대규모 백필 및 고빈도(분봉/틱) 시계열 전환 시의 Out-of-Core 스트리밍 병합 한계**

현재의 `1-Shot pd.concat` 인메모리 결합 방식은 1일치 배치(1행 × 1,456열) 연산에서 뛰어난 속도를 제공하지만, 10년 이상의 다년치 시계열 전체를 단일 배치에서 재처리(Reprocessing)하거나 분봉·초봉 단위의 고빈도 데이터로 확장할 경우 워커 컨테이너의 메모리 상한을 초과하여 OOM(Out-Of-Memory)을 유발할 수 있다. 이에 대응하여 전체 데이터를 메모리에 올리지 않고도 파티션 단위 디스크 버퍼링과 청크 스트리밍 조인을 수행하는 **Out-of-Core 연산 파이프라인**을 구축하거나, Apache Arrow 기반의 분산 처리 프레임워크를 도입하여 데이터 볼륨 증가에도 일정한 메모리 공간 복잡도 $O(1)$를 보장하는 아키텍처 고도화가 차기 과제다.
