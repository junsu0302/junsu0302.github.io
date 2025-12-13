---
title: "[Airflow] 02-Airflow DAG"
categories: [MLOps, Orchestration]
tags: [MLOps, Orchestration, Airflow, Data Pipeline, DAG]
---

# Airflow DAG

Airflow의 핵심 작업 단위인 Task와 이를 연결하는 DAG의 작성 문법을 정의하고, Python 및 Docker 환경에서의 실행 방법과 웹 UI를 통한 파이프라인 상태 모니터링 및 트러블슈팅 기법을 다룬다.

## Airflow DAG 생성

Airflow는 복잡한 데이터 파이프라인을 여러 개의 **태스크**로 분할하고, 이를 순서와 의존성에 따라 **DAG**로 구성하여 관리한다. 이를 통해 서로 독립적인 태스크들을 병렬로 실행할 수 있다.

Airflow에서 DAG 인스턴스는 다음과 같이 생성할 수 있다.
```python
from airflow import DAG
from datetime import datetime

dag = DAG(
    dag_id="DAG_ID",             # Airflow UI에 표시되는 고유 ID
    start_date=datetime(2023, 1, 1),     # 워크플로우가 시작되는 기준 날짜
    schedule_interval="@daily",          # 실행 주기 (Cron 표현식 또는 프리셋 사용)
    catchup=False                        # 과거 데이터 소급 적용 여부
)
```

생성된 `dag` 객체에 실제 수행할 태스크들을 연결한다. 가장 많이 사용되는 `BashOperator`와 `PythonOperator`의 예시는 다음과 같다.
```python
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator

# Bash 명령어를 실행하는 태스크
task1 = BashOperator(
    task_id="print_date",                # UI에 표시될 태스크 이름
    bash_command="date",                 # 실행할 쉘 명령어
    dag=dag
)

def my_python_func():
    print("Hello Airflow")

# Python 함수를 실행하는 태스크
task2 = PythonOperator(
    task_id="run_python_func",
    python_callable=my_python_func,      # 실행할 파이썬 함수 객체
    dag=dag
)
```

정의된 태스크들의 실행 순서는 비트 시프트 연산자`(>>, <<)`를 사용하여 직관적으로 설정할 수 있다.

```python
Task1 >> Task2
```

## Airflow 실행

Airflow는 파이썬 환경과 Docker 환경에서 주로 사용된다. 각 환경별로 다음과 같이 실행된다.

```bash
// Airflow 설치
pip install apache-airflow

// 메타스토어 초기화
airflow db init
airflow users create --username [USERNAME] --password [PASSWORD] --firstname [FIRSTNAME] --lastname [LASTNAME] --role Admin --email [EMAIL]

// 웹서버 및 스케줄러 실행
airflow webserver
airflow scheduler

// http://localhost:8080
```

```bash
docker run -it -p 8080:8080 \ // 포트 설정
               -v /[로컬의 파일경로]/airflow/dags/파일명.py //컨테이너에 DAG 파일 마운트
               --entrypoint=/bin/bash --name airflow \
               apache/airflow:2.0.0-python3.8 \ // Airflow 도커 이미지
               -c (airflow users create --username [USERNAME] --password [PASSWORD] --firstname [FIRSTNAME] --lastname [LASTNAME] --role Admin --email [EMAIL]) //메타스토어 초기화
airflow webserver & airflow scheduler
```

## Airflow UI

http://localhost:8080에 접속하여 로그인하면 메인 대시보드에서 등록된 DAG 목록을 확인할 수 있다. 메인 대시보드에서는 다음과 같은 작업을 수행할 수 있다.
- DAG 활성화: DAG 이름 왼쪽의 Toggle 스위치를 On으로 변경해 스케줄링이 시작
- 수동 실행: 우측의 재생 버튼을 눌러 즉시 실행

DAG ID를 클릭하면 상세 화면으로 이동하여 태스크의 상태를 모니터링 할 수 있다. 태스크의 상태는 색상별로 확인할 수 있다.
- 초록 : 실행 완료
- 빨강 : 실행 실패
- 주황 : 이전 태스크의 실패로 실행 못함

DAG를 구성하는 전체 태스크들의 관계는 그래프 뷰와 트리 뷰로 확인할 수 있다.
- 그래프 뷰 : 그래프 형식으로 각 태스크 간의 연결 관계와 실행 흐름을 시각적으로 표현
- 그리디 뷰 : 과거부터 현재까지의 실행 이력과 성공/실패 여부를 바둑판 형태로 표현

파이프라인 실행 중 특정 태스크가 실패하여 파이프라인이 멈춘 경우 다음과 같이 조치한다.
1. 실패한 태스크(빨간색 사각형)를 클릭하고 Log 탭을 확인하여 에러 원인을 분석
2. 코드를 수정한 후, 다시 태스크를 클릭하고 Clear 버튼을 누름
3. Clear는 해당 태스크의 상태를 초기화하고 재실행을 유도하며, Downstream 옵션이 켜져 있을 경우 이후의 태스크들도 순차적으로 재실행
