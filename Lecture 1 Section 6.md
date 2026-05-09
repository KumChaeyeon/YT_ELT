# 01. Testing Introduction

- 데이터 엔지니어링에서 테스트의 중요성을 이해
- 주요 테스트 종류
    - Data Quality Test
    - Functional Test
        - Unit Test
        - Integration Test
        - End-to-End Test
- 데이터 품질 검사의 목적
    - 데이터 정확성 검증
    - 중복 데이터 확인
    - 결측값 확인
- Functional Test 목적
    - 코드 동작 검증
    - 시스템 간 상호작용 검증
    - 전체 ELT 흐름 검증

# 02. Amendment to .env

- Docker 이미지 태그 변경 과정
    - 변경 전
    
    ```python
    IMAGE_TAG=1.0.0
    ```
    
    - 변경 후
    
    ```python
    IMAGE_TAG=1.0.1
    ```
    
- Docker 이미지 버전 관리 목적
- 변경된 requirements 반영을 위한 이미지 재빌드 수행
- 새로운 이미지 태그를 통한 버전 구분 수행

# 03. Using Soda for Data Quality Tests

- Soda Core를 이용한 데이터 품질 검사 수행
- Soda 특징
    - 오픈소스 데이터 품질 프레임워크
    - YAML 기반 데이터 품질 검사 정의 가능
    - 데이터베이스별 connector 제공
- requirements.txt 패키지 추가
    
    ```python
    soda-core-postgres
    pytest
    ```
    
    - soda-core-postgres: Postgres 데이터 품질 검사 수행 패키지
    - pytest: 기능 테스트 수행 프레임워크
    - Postgres connector 사용을 통한 DB 연결 구성
- Docker 이미지 업데이트 과정
    
    ```bash
    docker compose pull
    docker compose up -d --force-recreate
    ```
    
    - 최신 Docker 이미지 다운로드 수행
    - 기존 컨테이너 강제 재생성 수행
    - 새로운 패키지 및 환경 반영 목적
- Soda 설정 파일 생성
    
    ```yaml
    data_source pg_data_source:
      type: postgres
      host: localhost
      username: user
      password: password
      database: elt_db
      schema: staging
    ```
    
    - Postgres 데이터베이스 연결 정보 정의
    - schema 값을 통해 staging/core 선택 가능 구조
    - Soda와 데이터베이스 연결 설정 목적
- 데이터 품질 검사 정의
    
    ```yaml
    checks for youtube_api:
      - missing_count(video_id) = 0
      - duplicate_count(video_id) = 0
    ```
    
    - video_id 결측값 검사 수행
    - video_id 중복 데이터 검사 수행
    - Primary Key 무결성 검증 목적
- 커스텀 SQL 기반 품질 검사
    
    ```yaml
    - failed rows:
        name: likes_count greater than views
        fail query: |
          SELECT *
          FROM core.youtube_api
          WHERE likes_count > video_views
    ```
    
    - 좋아요 수가 조회수보다 큰 비정상 데이터 검사
    - SQL 기반 사용자 정의 품질 검사 수행
    - 데이터 논리적 무결성 검증 목적
- Soda 실행 명령어
    
    ```bash
    soda scan -d pg_data_source \
    -c configuration.yml \
    checks.yml
    ```
    
    - Soda 데이터 품질 검사 실행 명령어
    - configuration.yml: DB 연결 정보
    - checks.yml: 품질 검사 규칙 정의 파일
    - staging/core 데이터 품질 검증 수행

# 04. Airflow Integration for DQ Tests

- Airflow와 Soda 연동 과정
- BatchOperator 사용
    - Batch 명령 실행 가능
    - Soda scan 자동 실행 가능
        
        ```python
        from airflow.operators.bash import BashOperator
        
        def data_quality(schema):
        
            task = BashOperator(
                task_id=f"dq_{schema}",
                bash_command=f"""
                soda scan -d pg_data_source
                -c configuration.yml
                checks.yml
                """
            )
        
            return task
        ```
        
        - BashOperator를 통한 터미널 명령 실행 구조
        - Soda scan 자동 수행 기능
        - schema별 staging/core 품질 검사 수행 가능 구조
- Airflow DAG 구성
    - staging 품질 검사 → core 품질 검사 순서 설정
    - DAG을 통한 데이터 품질 자동 검증 수행
    - 로그 기반 검사 결과 확인 가능 구조

# 05. Functional Tests Introduction

- 기능 테스트 개념 학습
- 기능 테스트 종류
    - Unit Test
    - Integration Test
    - End-to-End Test
- 사용 라이브러리
    - unittest
    - pytest
- 테스트 목적
    - 코드 단위 검증
    - 시스템 통합 검증
    - 전체 ELT 프로세스 검증

# 06. Unit Tests

- Unit Test 목적
    - 개별 함수 단위 검증
    - 외부 의존성과 분리된 테스트 수행
- Ficture 개념
    - 테스트용 공통 데이터 제공 기능
    - pytest fixture decorator 사용
- API Key Mock 테스트
    
    ```python
    @pytest.fixture
    def api_key():
        with mock.patch.dict(os.environ, {
            "AIRFLOW__API_KEY": "mock_key_1234"
        }):
            yield Variable.get("API_KEY")
    ```
    
    - 환경 변수 Mock 처리 수행
    - 실제 API Key 대신 테스트용 값 사용
    - 외부 의존성 제거 목적
- API Key 검증 테스트
    
    ```python
    def test_api_key(api_key):
        assert api_key == "mock_key_1234"
    ```
    
    - Mock 값 정상 반환 여부 검증
    - assert를 통한 테스트 결과 판별 수행
- Postgres Connection Mock 테스트
    - Mock Connection 생성 수행
    - 실제 DB 연결 없이 테스트 가능 구조
    - Connection URI 검증 수행
- DAG Integrity Test
    - 검사 항목
        - import error 존재 여부
        - DAG 정상 로딩 여부
        - DAG 개수 검증
        - Task 개수 검증
    - Airflow DAG 구조 무결성 검증 목적

# 07. Integration Tests

- Integration Test 목적
    - 여러 시스템 간 상호작용 검증
    - 실제 API 및 DB 연결 검증 수행
- 실제 API 연결 테스트
    
    ```python
    def test_api_connection(airflow_variable):
    
        response = requests.get(url)
    
        assert response.status_code == 200
    ```
    
    - 실제 API 요청 수행
    - HTTP 200 응답 여부 확인
    - API 연결 정상 동작 검증 목적
- 실제 Postgres 연결 테스트
    
    ```python
    cursor.execute("SELECT 1")
    result = cursor.fetchone()
    
    assert result[0] == 1
    ```
    
    - 실제 Postgres SQL 실행 수행
    - SELECT 1 결과 검증
    - DB 연결 및 SQL 실행 정상 여부 확인 목적

# 08. End to End (E2E) Test

- End-to-End Test 목적
    - 전체 ELT 파이프라인 검증
    - API → JSON → DB → DQ 전체 흐름 확인
- Airflow Dag Test 사용
    
    ```bash
    airflow dags test produce_json
    ```
    
    - 특정 DAG 단독 실행 수행
    - 전체 파이프라인 정상 동작 여부 검증
    - JSON 생성 확인 가능 구조
- 추가 DAG 테스트
    
    ```bash
    airflow dags test update_db
    airflow dags test data_quality
    ```
    
    - DB 업데이트 DAG 실행
    - 데이터 품질 검사 DAG 실행
    - 전체 ELT 흐름 정상 여부 검증 수행

# 09. DAGs Re-Structure

- 기존 DAG 구조 문제점
    - 시간 기반 실행 의존 구조
    - 이전 DAG 실패 시 이후 DAG도 실패 가능성 존재
    - 비효율적 대기 시간 발생 가능성
- 해결 방법
    - TriggerDagRunOperator 사용
    - DAG 간 의존성 연결 수행
- TriggerDagRunOperator 사용
    
    ```python
    from airflow.operators.trigger_dagrun import TriggerDagRunOperator
    ```
    
    - 다른 DAG 자동 실행 기능 제공
    - 이전 DAG 성공 시 다음 DAG 실행 구조
    - DAG 간 순차 실행 보장 목적
- DAG Trigger Task 추가
    
    ```python
    trigger_update_db = TriggerDagRunOperator(
        task_id="trigger_update_db",
        trigger_dag_id="update_db"
    )
    ```
    
    - produce_json 완료 후 update_db 자동 실행
    - DAG 간 연결 구조 형성
    - 전체 파이프라인 자동화 강화 목적
- Task 개수 변경 사항 반영
    - DAG 구조 변경에 따른 Unit Test 수정 필요
    - expected task count 값 수정 수행
    - DAG Integrity Test 유지 목적
