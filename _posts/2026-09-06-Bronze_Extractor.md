---
title: "[DataPipeline] Bronze Pipeline: 이종 금융·거시경제 API 189개 지표의 비동기 수집 최적화"
categories: [Engineering, AssetMind]
tags: [AssetMind, Data Pipeline, Bronze, Extractor, aiohttp, Concurrency]
---

# 대규모 금융 데이터 파이프라인의 병목 격리 및 비동기 스트리밍 적재: 189개 엔드포인트 17초 완결 아키텍처

**Executive Summary (핵심 요약)**
> **문제**: 4개 금융 기관(189개 엔드포인트)의 극단적인 TPS 편차(2~20 TPS) 환경에서, 스케줄러 풀(slot=128)의 무제한 동시 호출로 인한 `HTTP 429` 차단이 발생함. 또한 루프 충돌 방지를 위한 태스크 프로세스 격리가 인메모리 인증 토큰을 증발시켜 인증 서버 429 밴을 유발하는 연쇄 병목이 상존함.

> **해결**: Airflow `external_api_pool`(slot=1)로 외부 통신을 전역 격리하고, 수집-적재 단일 체인 스트리밍 및 `asyncio.to_thread` 기반 동기 I/O 오프로딩, 파일 시스템 영속 토큰 캐싱을 결합함.

> **결과**: 아시아 117개 태스크 9.5초, 글로벌 72개 태스크 7.5초로 총 189개 수집·압축·적재를 17.0초 만에 완수하였으며, 약 10년 8개월치 백필 및 일별 배치에서 장애율 0.0%(에러 0건)를 달성함.

---

## 1. 문제 배경 및 병목 (Context & Problem)

데이터 파이프라인의 브론즈 레이어는 이종 금융 기관으로부터 수집한 비정형 원천 데이터를 무손실 상태로 데이터 레이크에 적재한다. 

수집 대상은 한국투자증권(KIS), 한국은행(ECOS), 미국 연방준비제도(FRED), 업비트(UPBIT) 등 4개 기관, 총 189개 엔드포인트로 구성된다. 
각 외부 기관은 서버 가용성 보호를 위해 초당 처리 요청 수(TPS)를 엄격히 제한하고 있었으며, 인증 메커니즘과 호출 프로토콜이 파편화되어 있다.

| 제공자 (Provider) | 주요 수집 데이터군 | 인증 및 호출 방식 | 공식 허용 TPS | 엔드포인트 수 | 주요 병목 및 제약 특성 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **KIS (한국투자증권)** | 국내외 주가지수, 환율, 원자재 선물 | OAuth2 (AppKey/Secret + Token) | **5 TPS** | 100개 | 계좌/앱키 단위의 엄격한 초당 호출 제한. 초과 시 즉시 429 차단 |
| **FRED (연준)** | 미국 국채 금리 (2Y, 5Y, 10Y, 30Y) | API Key (Query Parameter) | **2 TPS** | 4개 | 낮은 TPS 한계. 호출 한도 초과 시 일정 시간 엔드포인트 전면 차단 |
| **UPBIT (업비트)** | 가상자산 일봉 캔들 (USDT, BTC 등) | HMAC-SHA512 JWT 서명 | **10 TPS** | 5개 | 캔들 조회 시 UTC 09시 마감 시간 계산 필요, 논스 기반 서명 오버헤드 |
| **ECOS (한국은행)** | 기준금리, 국고채, 시장금리 등 | Path Variable 세그먼트 주입 | **20 TPS** | 80개 | 표준 Query String 미지원, 엄격한 URL 경로 순서 요구 |

초기 시스템의 가장 치명적인 병목은 워크플로우 스케줄러의 동시성 제어 부재에서 비롯되었다. 

Airflow의 기본 작업 풀(`default_pool`, slot=128) 환경에서 189개의 데일리 배치 및 대규모 Backfill DAG가 동시에 구동될 때, 수십 개의 태스크가 외부 API로 동시 커넥션을 생성했다. 
KIS와 같이 초당 허용량이 5 TPS에 불과한 엔드포인트에 10개 이상의 코루틴이 일시에 유입되면서 `HTTP 429 Too Many Requests` 응답이 연속 발생했고, 재시도 횟수 초과로 전체 파이프라인이 중단되는 구조적 장애로 이어졌다.

추가로 런타임 환경의 실행 제약이 연쇄 병목을 일으켰다. 수집 엔진의 `asyncio.run()`과 Airflow 워커 이벤트 루프 간 충돌을 방지하기 위해 `BashOperator`로 프로세스를 격리했다.
하지만 태스크 종료 시 메모리가 초기화되어 인메모리에 캐싱된 KIS 접근 토큰(수명 12시간)이 매번 증발하는 부작용이 발생했다. 
그 결과 100여 개 태스크가 뜰 때마다 토큰 재발급 요청을 중복 전송하면서, 시세 데이터를 요청하기도 전에 인증 엔드포인트 자체가 429 차단되는 악순환으로 이어졌다.

---

## 2. 해결 대안 및 기술적 의사결정 (Alternatives & Trade-offs)

데이터 수집 파이프라인의 설계 목표는 세 가지였다. 
1. 외부 API의 엄격한 TPS 제약을 위반하지 않으면서 189개 작업을 최단 시간에 완결할 것
2. Airflow 스케줄러와 비동기 I/O 런타임 간의 프로세스 충돌을 차단할 것
3. 단일 노드 인프라 자원을 초과하는 분산 프레임워크 오버헤드 없이 시스템 복잡도를 최소화할 것. 

이를 달성하기 위해 검토한 대안과 최종 엔지니어링 의사결정은 다음과 같다.

| 아키텍처 과제 | 기각된 대안 (Rejected) | 채택한 설계 (Adopted) | 핵심 선정 근거 및 트레이드오프 |
| :--- | :--- | :--- | :--- |
| **API 동시성 제어 및 429 방어** | Celery/Redis 기반 분산 큐 및 전역 Rate Limiter 도입 | Airflow `external_api_pool` (slot=1) + 클라이언트 버킷 제한 | 단일 노드 환경에서 추가 미들웨어 운영 비용을 배제. 스케줄러 레벨에서 외부 통신 DAG를 단일 슬롯으로 직렬화, 내부 비동기 루프에서 세밀한 TPS를 통제 |
| **워커 런타임 및 프로세스 격리** | `PythonOperator` 기반 워커 프로세스 내부 비동기 실행 | `BashOperator` 기반 OS 서브프로세스 완전 격리 (`python -m src.main`) | Airflow 워커 메인 이벤트 루프와 애플리케이션의 `asyncio.run()` 간 루프 충돌 및 패키지 의존성 오염을 원천 차단. XCom 메모리 공유 제약을 감수하고 격리 안정성을 우선 |
| **프로세스 격리 환경의 인증 상태 유지** | 매 태스크 기동 시 신규 OAuth 토큰 발급 또는 별도 인증 데몬 상시 구동 | 파일 시스템 기반 토큰 영속화 (`.kis_token_cache.json`) 및 Lazy Refresh | 별도 캐시 서버 없이도 서브프로세스 종료 후 12시간 유효 토큰을 재사용. 토큰 발급 엔드포인트에 가해지는 429 차단을 방어하고 프로세스 간 결합도를 최소화 |
| **수집 및 적재 파이프라인 결합 모델** | Stop-and-Wait (전체 수집 완료 대기 후 순차/일괄 적재) | 수집 즉시 `asyncio.to_thread`로 S3 적재를 오프로딩하는 단일 체인 스트리밍 | 느린 API로 인한 Head-of-Line Blocking을 제거, I/O 지연을 백그라운드 스레드로 은닉(Latency Hiding). 메모리 버퍼 적체 방지를 위해 동시성 세마포어(10)를 강제 |
| **원천 데이터 스토리지 적재 포맷** | Raw JSON 비압축 적재 또는 브론즈 레이어 Parquet 즉시 변환 | Raw JSON 구조 보존 + Zstandard(Level 3) 스트리밍 압축 | 원형 보존 원칙 준수. Gzip 대비 3~5배 빠른 압축 속도와 낮은 CPU 점유율을 확보하여 네트워크 I/O 병목을 해소하고 스토리지 비용을 절감 |

의사결정의 핵심 축은 **'인프라 단순성'**과 **'런타임 격리'**의 균형이었다. 
분산 큐(Celery)나 캐시 레이어(Redis)를 추가하면 동시성 제어는 쉬워지지만, 단일 노드(Docker Compose) 환경에서 감당해야 할 프로세스 감시와 장애 복구 비용이 증가한다.
따라서 인프라는 기본 컴포넌트(Airflow LocalExecutor + MinIO)로 고정하고, 동시성 격리는 스케줄러 풀(Pool)과 클라이언트 세마포어의 2단계 계층 제어로 해결했다.

또한 `BashOperator` 도입으로 발생한 토큰 증발 문제는 외부 인프라 추가 대신 로컬 파일 시스템 캐시를 통해 해결했다. 
KIS 토큰은 발급 후 12시간 동안 유효하므로, 파일 기반의 만료 검증(`_should_refresh`)과 동기화 락을 통해 매일 수백 회 발생하던 인증 호출을 하루 단 1~2회로 압축할 수 있었다. 
수집과 적재를 하나의 비동기 태스크로 묶어 `asyncio.to_thread`로 밀어 넣은 것 역시, CPU 바운드(Zstd 압축) 및 블로킹 네트워크 I/O(S3 멀티파트 업로드)가 비동기 이벤트 루프를 잠식하는 현상을 원천적으로 차단했다.

---

## 3. 핵심 구현 및 아키텍처 (Implementation)

브론즈 파이프라인은 Airflow 스케줄러 계층에서 시작하여 비동기 실행 런타임, 도메인 추출 계층, 그리고 스토리지 적재 계층에 이르기까지 병목을 단계별로 분해하도록 설계되었다. 전체 시스템의 데이터 흐름과 레이어별 상호작용은 다음과 같다.

```
[Airflow DAG Layer]
│ external_api_pool (slot=1) ─── 외부 통신 DAG 전역 직렬화
▼
[Runtime Isolation Layer]
│ BashOperator (python -m src.main) ─── 워커 이벤트 루프와 프로세스 완전 격리
▼
[Execution Entrypoint & Factory]
│ main.py ─── PipelineFactory.create(TARGET_TASK) (다형성 및 OCP 보장)
▼
[Bronze Pipeline Streaming Engine]
│ BronzePipeline.run_batch (asyncio.Semaphore=10)
├── 1. Extraction Sub-pipeline (Non-blocking I/O)
│   ├── ExtractorService (Facade) ─── 규격 정규화 및 에러 격리
│   ├── @rate_limit (Token Bucket) ─── 벤더별 정밀 TPS 강제 (KIS: 5/s 등)
│   ├── @retry (Equal Jitter Backoff) ─── 일시적 장애 복구 및 Thundering Herd 차단
│   ├── KISAuthStrategy ─── .kis_token_cache.json (Double-Checked Lock 영속화)
│   └── AsyncHttpAdapter ─── TCP 커넥션 풀링 (aiohttp)
│
└── 2. Loading Sub-pipeline (Latency Hiding)
    └── asyncio.to_thread(LoaderService.execute_load)
        ├── ZstdCompressor (Level 3) ─── 인메모리 바이트 직렬화 및 압축
        └── Boto3 S3 Client ─── TransferConfig 멀티파트 스트리밍 적재 (MinIO)
```

핵심 구현 메커니즘은 다음 세 가지로 요약된다.

첫째, **수집-적재 단일 체인 스트리밍과 지연 시간 은닉(Latency Hiding)**이다. 기존의 파이프라인은 189개 API 수집이 전원 완료될 때까지 대기한 후 적재를 시작하는 `Stop-and-Wait` 구조였다. 
이는 응답이 느린 단 하나의 외부 엔드포인트가 전체 배치를 가로막는 Head-of-Line Blocking을 유발했다. 개선된 아키텍처는 개별 작업(`job_id`) 단위로 '수집 즉시 적재'가 이어지는 독립 코루틴 체인을 구성했다. 
특히 CPU 바운드 연산인 Zstd 압축과 블로킹 소켓 I/O를 수반하는 S3 멀티파트 업로드를 `asyncio.to_thread`로 오프로딩하여, 메인 비동기 이벤트 루프가 멈추지 않고 다음 외부 API 요청을 지연 없이 발사하도록 최적화했다.

둘째, **3단계 다중 방어 Throttling 체계**다. 외부 API의 `HTTP 429` 밴을 원천 차단하기 위해 계층화된 방어벽을 구축했다. 
1. **스케줄러 레벨**: Airflow `external_api_pool`(slot=1)을 통해 복수의 백필이나 일별 배치 DAG가 동시에 외부망을 타격하지 못하도록 전역 직렬화했다.
2. **클라이언트 레벨**: `@rate_limit` 데코레이터를 적용하여 메모리 큐(`deque`) 기반의 토큰 버킷 알고리즘으로 KIS(5/s), FRED(2/s), UPBIT(10/s), ECOS(20/s)의 허용 TPS를 런타임에 엄격히 준수했다.
3. **회복 탄력성 레벨**: 외부 서버의 일시적 순단 시 `@retry` 데코레이터를 통해 `Equal Jitter` 기반 지수 백오프를 수행하여, 수십 개의 재시도 요청이 동일 시점에 외부 서버를 다시 타격하는 Thundering Herd 현상을 방지했다.

셋째, **프로세스 격리 환경의 인증 상태 영속화**다. 
`BashOperator`로 매 태스크 기동 시 메모리가 초기화되는 환경에서, `KISAuthStrategy`는 인메모리 뮤텍스(`asyncio.Lock`)와 함께 로컬 파일 시스템 캐시(`.kis_token_cache.json`)를 도입했다. 
프로세스가 시작되면 먼저 파일 시스템에서 유효 토큰과 만료 일시를 로드하여 불필요한 네트워크 I/O를 건너뛴다. 
만료 60분 전 임계치에 도달했을 때만 `Double-Checked Locking`을 통해 단 1회의 갱신 요청을 전송하고 디스크에 즉시 동기화함으로써, 100여 개 태스크가 실행되는 동안 발생하는 인증 API 호출을 하루 1~2회로 완벽히 통제했다.

다음은 파이프라인 지연을 근본적으로 해소한 핵심 로직의 Before/After 구현 대비다.

```python
# [Before: 단계별 단절(Stop-and-Wait) 방식]
# 100여 개 수집이 끝날 때까지 적재가 멈추며, 느린 API 1개로 인해 전체 블로킹 발생
extracted_results = await self._extractor_service.extract_batch(job_ids)

load_results = []
for item in extracted_results:
    if isinstance(item, ExtractedDTO):
        # 동기 S3 I/O 및 Zstd 압축이 비동기 이벤트 루프를 점유하여 전체 지연 누적
        is_loaded = self._loader_service.execute_load(item)
        load_results.append(is_loaded)
```

```python
# [After: 수집-적재 단일 체인 스트리밍 및 asyncio.to_thread 오프로딩]
concurrency_semaphore = asyncio.Semaphore(10)

async def process_pipeline_streaming_task(job_id: str) -> Dict[str, Any]:
    async with concurrency_semaphore:
        # 1. 단일 엔드포인트 비동기 수집 (내부 @rate_limit으로 벤더별 TPS 준수)
        extracted = await self._extractor_service.extract_job(job_id, runtime_params)
        
        # 2. 수집 완료 즉시 백그라운드 워커 스레드로 적재 위임 (Latency Hiding)
        # Zstd 압축(CPU 바운드) 및 S3 I/O 블로킹으로부터 메인 이벤트 루프 완벽 분리
        return await asyncio.to_thread(self._loader_service.execute_load, extracted)

# 189개 작업을 개별 스트리밍 태스크로 생성 후 비동기 동시 오케스트레이션
streaming_tasks = [process_pipeline_streaming_task(jid) for jid in job_ids]
loaded = await asyncio.gather(*streaming_tasks, return_exceptions=True)
```

---

## 4. 검증 및 결과 (Validation & Metrics)

개선된 브론즈 파이프라인의 성능 및 안정성 평가는 2016년 1월 1일부터 2026년 8월 31일까지 약 10년 8개월치 데이터에 대한 대규모 백필(Backfill)과 일별 배치 운영 환경에서 실측 검증되었다. 
평가 축은 전체 배치 처리 레이턴시(Latency), 외부 API 통신 장애율(HTTP 429 및 타임아웃), 그리고 스토리지 압축 효율성의 세 가지로 설정했다.

| 평가 지표 (Metric) | 개선 전 (Before: 단일 풀 & Stop-and-Wait) | 개선 후 (After: Pool 격리 & 스트리밍 체인) | 개선 성과 및 정량적 검증 결과 |
| :--- | :--- | :--- | :--- |
| **전체 189개 엔드포인트 완결 시간** | **약 120s** (80s+45s) | **17.0s** (9.5s + 7.5s) | **약 85% 지연 시간 단축** (189개 수집·압축·S3 적재 완결) |
| **외부망 API 장애율** (429/타임아웃) | 빈발 (slot=128 무제한 동시 격발로 빈번한 429 밴 발생) | **0.0% (에러 0건)** | **10년 8개월 백필 및 운영 간 결측·장애 0건 완주** |
| **원천 데이터 스토리지 용량** (Zstd Lv.3) | **약 1.8 KiB** | **297 Bytes** (`kis_kospi_daily` 실측 기준) | **약 83.8% 스토리지 공간 절감** (약 6.2배 압축률 달성) |

정량적 개선 외에도 아키텍처 측면에서 다음과 같은 정성적 성과를 확보했다.

첫째, **완벽한 데이터 멱등성(Idempotency) 확립**이다. 
S3 키 생성 시 실행 시점의 타임스탬프나 난수(UUID)를 배제하고, `EXECUTION_DATE` 기반의 대상 데이터 기준일(`file_date`)을 명시하여 `provider=.../job=.../year=.../job_date.json.zst` 형태의 결정론적 경로를 생성하도록 강제했다.
이로 인해 동일 날짜 배치가 네트워크 오류나 재시도로 재실행되더라도 파일 중복 증식 없이 덮어쓰기(Overwrite)가 안전하게 수행된다.

둘째, **도메인 결합도 완화 및 장애 격리(Fault Isolation)**이다. 
YAML 설정 파일(`extractor.yml`, `pipeline.yml`) 기반의 팩토리 패턴을 통해 수집 엔진과 레이어를 완전히 분리했다.
이를 통해 KIS API에 장애가 발생하더라도 ECOS나 FRED 등 다른 수집기에 영향이 전파되지 않으며, 장애 복구 시 특정 도메인(예: KIS)만 선별적으로 재실행하거나 브론즈·실버·골드 레이어를 분절 구동할 수 있는 운영 유연성을 확보했다.

---

## 5. 프로덕션 관점의 한계와 과제 (Production Readiness & Next Step)

현재 아키텍처는 단일 노드 컨테이너 환경(`LocalExecutor`)에서 추가적인 미들웨어(Redis, Celery 등)의 운영 비용 없이 189개 엔드포인트를 17초 만에 완결하도록 최적화된 구조다. 그러나 시스템 확장성 관점에서 다음과 같은 기술적 한계와 프로덕션 과제를 내포하고 있다.

첫째, **로컬 프로세스 기반 동시성 제어의 분산 환경 한계**다. 
`BronzePipeline`의 `asyncio.Semaphore(10)` 및 `@rate_limit`의 메모리 버킷(`_buckets`), 그리고 `.kis_token_cache.json` 파일 캐시는 단일 호스트 인프라를 전제로 동작한다. 
향후 데이터 볼륨이 수천 개 자산군으로 확장되어 Airflow `CeleryExecutor`나 `KubernetesExecutor` 등 멀티 노드 워커 환경으로 스케일아웃될 경우, 각 워커 노드의 로컬 세마포어가 파편화된다. 
이로 인해 전역(Global) TPS 합산치가 외부 벤더의 허용 임계치를 초과하여 다시 `HTTP 429` 밴을 유발할 수 있다.

둘째, **분산 캐시 및 중앙 집중형 Rate Limiter로의 고도화 과제**다. 
멀티 노드 환경에서도 안정적인 429 방어벽을 유지하려면, 로컬 파일 캐시와 인메모리 버킷을 **Redis 기반의 중앙 집중형 토큰 관리 및 분산 레이트 리미터(Distributed Token Bucket)** 구조로 전환해야 한다. 
또한 워커 노드 간 토큰 갱신 경합을 방지하기 위해 `Redlock` 알고리즘 기반의 분산 락을 도입하여, 분산 노드 환경에서도 KIS OAuth 토큰 발급 횟수를 하루 1~2회로 일관되게 제어하는 아키텍처 확장이 차기 과제다.
