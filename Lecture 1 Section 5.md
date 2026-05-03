# 01. Postgres Data Warehouse Introduction

- 이전 강의에서 API로부터 데이터를 추출하고 데이터 디렉토리에 저장하는 과정 학습
- 다음 단계로 데이터 웨어하우스에 데이터를 적재하고 변환하는 과정 진행
- 데이터 처리 흐름
    - 데이터 추출(Extract) 완료 상태
    - 데이터 적재(Load) 및 변환(Transform) 단계 진행 예정

# 02. Loading to Data Warehouse & Transformations

- 데이터 웨어하우스 구성 요소 정의
    - 테이블: 데이터 저장 단위
    - 스키마: 테이블 및 객체를 논리적으로 그룹화하는 구조
    - 객체 종류: 테이블, 뷰, 저장 프로시저 등
- 데이터 적재 과정
    - 초기 테이블 생성 시 데이터 없음
    - 데이터 삽입/수정/삭제 기능 필요
    - Python 함수로 데이터 조작 구현
- 데이터 변환 과정
    - Raw Layer → Refined Layer 변환 수행
    - Staging Schema: Raw 데이터 저장
    - Core Schema: 정제 데이터 저장
- 데이터 흐름 구조
    - API → Raw Layer(Staging) → Refined Layer(Core)

# 03. Setting up Connection to Data Warehouse using Airflow

- 데이터베이스 연결 구성
    - Hook 사용하여 서비스 연결
    - PostgresHook 활용
- 핵심 개념
    - Connection: DB 연결 객체
    - Cursor: SQL 실행 객체

```python
from airflow.providers.postgres.hooks.postgres import PostgresHook
import psycopg2
from psycopg2.extras import RealDictCursor

def get_conn_cursor():
    hook = PostgresHook(postgres_conn_id='postgres_db_youtube', database='elt_db')
    conn = hook.get_conn()
    cursor = conn.cursor(cursor_factory=RealDictCursor)
    return conn, cursor

def close_conn_cursor(conn, cursor):
    cursor.close()
    conn.close()
```

- PostgresHook을 이용한 Airflow 기반 데이터베이스 연결 생성 기능
- conn 객체: 실제 데이터베이스 연결 수행 객체
- cursor 객체: SQL 실행 및 결과 반환 담당 객체
- RealDictCursor 사용을 통한 결과를 딕셔너리 형태로 반환하는 구조
- close 함수에서 cursor와 connection 종료를 통한 리소스 관리

# 04. Creationg the Schemas and Tables

- 스키마 및 테이블 생성
    - staging → raw 데이터
    - core → 정제 데이터

```python
def create_schema(schema):
    conn, cursor = get_conn_cursor()
    schema_sql = f"CREATE SCHEMA IF NOT EXISTS {schema}"
    cursor.execute(schema_sql)
    conn.commit()
    close_conn_cursor(conn, cursor)
```

- 스키마 생성 함수 정의
- IF NOT EXISTS 조건을 통한 중복 생성 방지
- cursor.execute를 통한 SQL 실행
- commit을 통한 변경 사항 반영

```python
def create_table(schema):
    conn, cursor = get_conn_cursor()

    if schema == "staging":
        ...
    else:
        ...

    cursor.execute(table_sql)
    conn.commit()
    close_conn_cursor(conn, cursor)
```

- schema 값에 따른 staging / core 테이블 분기 처리
- staging: 원본 데이터 저장 목적
- core: 정제 데이터 저장 및 video_type 컬럼 추가 구조
- 데이터 타입에 따른 컬럼 설계 (varchar, timestamp, integer 등)

# 05. Loading the JSON data

- JSON 데이터 로딩 과정
    - 파일 읽기
    - Python 객체로 변환
    - logging 사용

```python
import json
import logging

def load_data(path):
    try:
        with open(path, "r", encoding="utf-8") as f:
            data = json.load(f)
        return data
    except Exception as e:
        raise e
```

- JSON 파일을 Python dict 형태로 변환하는 함수
- with open을 통한 파일 자동 종료 구조
- json.load를 통한 데이터 파싱 수행
- 예외 발생 시 오류 전달 구조

# 06. Inserts, Updates & Deletes

- 데이터 조작 기능
    - Insert: 신규 데이터 삽입
    - Update: 기존 데이터 수정
    - Delete: 데이터 삭제

```python
def insert_row(cursor, conn, schema, row):
    if schema == "staging":
        ...
    else:
        ...

    cursor.execute(sql, row)
    conn.commit()
```

- schema에 따라 staging / core 분기 처리
- INSERT INTO SQL을 통한 데이터 삽입 수행
- %(컬럼명)s 형태의 named placeholder 사용
- row 딕셔너리와 컬럼 매핑 구조
- commit을 통한 데이터 반영

# 07. Transformations

- 데이터 변환 과정
    - ISO 8601 → 시간 형식 변환
    - 영상 유형 분류
- 변환 기준
    - 1분 이하 → shorts
    - 1분 초과 → normal

```python
from datetime import timedelta, datetime

def parse_duration(duration):
    duration = duration.replace("P", "").replace("T", "")
    values = {"H":0,"M":0,"S":0}

    for key in values:
        if key in duration:
            num, duration = duration.split(key)
            values[key] = int(num)

    return timedelta(hours=values["H"], minutes=values["M"], seconds=values["S"])
```

- ISO 8601 형식 문자열에서 불필요 문자 제거 처리
- H, M, S 기준으로 시간 요소 분리
- timedelta 객체로 변환하여 시간 계산 가능 구조

```python
def transform_row(row):
    td = parse_duration(row["duration"])
    row["duration"] = (datetime.min + td).time()

    if td.total_seconds() <= 60:
        row["video_type"] = "shorts"
    else:
        row["video_type"] = "normal"

    return row
```

- duration 문자열을 실제 시간 객체로 변환
- total_seconds를 이용한 영상 길이 계산
- 조건문을 통한 video_type 컬럼 생성
- 최종적으로 변환된 row 반환 구조

# 08. Populating Staging and Core Tables

- 데이터 적재 통합 처리
    - JSON 데이터 로딩
    - 스키마 및 테이블 생성
    - Insert / Update 수행
    - 삭제 데이터 반영
- 데이터 흐름
    - JSON → Staging → Core
- staging: 원본 데이터 저장 단계
- transform_row 적용 후 core로 데이터 이동
- 기존 데이터 존재 시 update 수행
- 존재하지 않는 데이터 insert 수행
- 삭제된 데이터 처리 로직 포함

# 09. Defining the Data Warehouse DAG & Debugging

- Airflow DAG 정의
    - DAG 생성
    - Task 정의
    - 실행 순서 설정
- 구성 요소
    - staging task
    - core task
    - dependency 설정
- 디버깅 방법
    - 로그 확인
    - 오류 메시지 분석
    - 코드 수정
- DAG을 통해 전체 파이프라인 자동 실행 구조
- staging → core 순서로 작업 의존성 설정
- 로그 기반 오류 추적 및 디버깅 수행

# 10. Interaction with the Data Warehouse using Dbeaver

- 데이터 조회 방법
    1. CLI 방식
    2. GUI 방식 (DBeaver)
- 주요 SQL 명령
    - SELECT
    - \l : 데이터베이스 목록
    - \dn : 스키마 목록
    - \dt : 테이블 목록
- 데이터 확인 및 분석 가능
