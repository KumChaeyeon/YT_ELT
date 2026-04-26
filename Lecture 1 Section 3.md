# 01. Why Docker

- Airflow DAG 실행을 위한 컨테이너 기반 환경 필요성
- 개발 환경과 실행 환경의 차이를 줄이기 위한 Docker 활용 목적
- 개발 및 소규모 운영 환경에서 Docker 사용의 적절성
- 대규모 운영 환경에서는 Kubernetes 또는 클라우드 기반 서비스 활용 가능성
- 본 강의에서 Docker를 사용하는 이유는 Airflow를 직접 실행하고, 이후 DAG를 작성 및 테스트하기 위한 기반 환경 마련 목적
- Docker의 장점
    - 동일한 환경 재현 가능성
    - 설치 및 실행 과정의 단순화
    - 로컬 시스템과 분리된 독립 실행 환경 제공
    - 프로젝트 구성 요소를 컨테이너 단위로 관리할 수 있는 구조

# 02. Dockerfile

- Dockerfile의 개념
    - 사용자 정의 Docker 이미지를 만들기 위한 명령어 스크립트
    - 어떤 베이스 이미지를 사용할지, 어떤 파일을 복사할지, 어떤 패키지를 설치할지 정의하는 파일
- 본 강의에서의 Dockerfile 구성 방식
    - Apache Airflow 공식 이미지를 기반 이미지로 사용
    - Airflow 버전과 Python 버전을 명시적으로 지정하는 방식
    - 환경 변수로 AIRFLOW_HOME 경로 설정
    - requirements.txt를 복사하여 추가 라이브러리 설치 구조 구성
- Dockerfile 작성 흐름
    - 사용할 Airflow 버전 지정
    - 사용할 Python 버전 지정
    - 공식 Airflow 이미지 지정
    - Airflow 홈 디렉토리 지정
    - requirements.txt 복사
    - pip install 실행

```docker
ARG AIRFLOW_VERSION=2.9.2
ARG PYTHON_VERSION=3.10

FROM apache/airflow:${AIRFLOW_VERSION}-python${PYTHON_VERSION}

ENV AIRFLOW_HOME=/opt/airflow

COPY requirements.txt /

RUN pip install --no-cache-dir -r /requirements.txt
```

- -no-cache-dir 옵션의 의미
    - pip 캐시를 남기지 않는 방식
    - 이미지 크기 증가 방지 목적

# 03. dockerfile versions - [IMPORTANT]

- 강의에서 지정한 버전 그대로 사용할 필요성
- 사용 권장 버전
    - ARG AIRFLOW_VERSION=2.9.2
    - ARG PYTHON_VERSION=3.10
- 버전 고정의 이유
    - 강의 전체 실습 환경과 동일한 조건 유지 목적
    - 버전 변경 시 컨테이너 시작 오류 또는 패키지 호환성 문제 발생 가능성

# 04. Build the Docker Image

- Docker 이미지 빌드 개념
    - Dockerfile을 기준으로 새로운 이미지를 생성하는 과정
    - 생성된 이미지를 Docker Hub와 같은 레지스트리에 저장 가능
- Registry 개념
    - Docker 이미지를 저장하는 저장소
    - 다른 사람이 만든 이미지 다운로드 가능
    - 자신이 만든 이미지 업로드 및 관리 가능
- 본 강의에서의 흐름
    - Docker Hub 계정 생성
    - Repository 생성
    - docker login 실행
    - docker build 실행
    - docker push 실행

```bash
docker login -u <username>
docker build -t <username>/<repository>:1.0.0 .
docker push <username>/<repository>:1.0.0
```

- -t 옵션의 의미
    - 이미지 이름과 태그 지정 목적
- 마지막 ‘.’의 의미
    - 현재 디렉토리를 build context로 사용한다는 의미
    - Dockerfile과 복사 대상 파일을 현재 디렉토리에서 찾는 방식

# 05. Airflow Architecture

- Airflow가 여러 구성 요소로 이루어진 시스템이라는 점
- 주요 구성 요소
    - Scheduler
    - Executor
    - Worker
    - Web Server
    - Metadata Database
- Scheduler
    - 어떤 작업을 언제 실행할지 결정하는 구성 요소
    - Airflow 전체 실행 흐름의 핵심 조정 역할
- Executor
    - 작업을 어디에서 어떤 방식으로 실행할지 결정하는 구성 요소
    - Sequential, Local, Celery, Kubernetes executor 등 다양한 방식 존재
- 강의에서 사용하는 방식
    - CeleryExecutor 기반 구조
    - Scheduler가 Worker에게 작업 전달하는 구조
- Worker
    - 실제 Task를 수행하는 실행 주체
- Web Server
    - Airflow UI 제공
    - DAG 확인, 실행, 상태 모니터링 가능
- DAG 개념
    - Directed Acyclic Graph의 약자
    - 작업의 방향성과 의존성을 가지는 비순환 그래프 구조
    - Task 순서를 시각적으로 표현 가능한 구조
- Metadata Database
    - DAG 정보, 로그, 변수, 연결 정보 등을 저장하는 데이터베이스
    - 일반적으로 Postgres 사용
- 추가 구성 요소
    - Triggerer
    - Message Broker
- Triggerer
    - 특정 이벤트를 기다리는 작업을 지원하는 구성 요소
    - 본 강의에서는 사용하지 않는 구성 요소
- Message Broker
    - Scheduler와 Worker 사이의 메시지 전달 담당
    - CeleryExecutor 사용 시 필요한 구성 요소
    - 강의에서는 Redis 사용

# 06. Airflow Directories

- Docker Volume을 통해 로컬 디렉토리와 컨테이너 내부 디렉토리를 연결하는 구조
- 로컬에서 파일 수정 시 컨테이너 내부에도 반영되는 방식
- 주요 디렉토리
    - dags
    - logs
    - config
    - plugins
- dags
    - DAG Python 코드 저장 위치
- logs
    - Task 실행 로그 및 Scheduler 로그 저장 위치
- config
    - Airflow 설정 파일 저장 위치
- plugins
    - 사용자 정의 operator, sensor, hook 등 확장 기능 저장 위치
- 본 강의에서 추가로 사용하는 디렉토리
    - tests
    - data
    - include
- tests
    - 기능 테스트 코드 저장 위치
- data
    - 추출된 JSON 데이터 저장 위치
- include
    - DAG에서 사용하는 추가 리소스 저장 위치
    - YAML 파일, SQL 스크립트, 인증서 등의 저장 가능 위치

# 07. .env file

- 민감 정보와 환경 설정 값을 분리하여 관리하기 위한 파일
- 코드 내부에 비밀번호나 API Key를 직접 작성하지 않기 위한 목적
- 개발 환경과 운영 환경에서 다른 값을 쉽게 적용하기 위한 목적
- 포함되는 주요 항목
    - Docker Hub 정보
    - Postgres 연결 정보
    - Metadata DB 정보
    - Celery backend DB 정보
    - ELT DB 정보
    - Airflow 계정 정보
    - Fernet Key
    - YouTube API 정보

```
DOCKERHUB_NAMESPACE=mattthedataengineer
DOCKERHUB_REPOSITORY=yt_api_elt
IMAGE_TAG=1.0.0

POSTGRES_CONN_USERNAME=postgres
POSTGRES_CONN_PASSWORD="..."
POSTGRES_CONN_HOST=postgres
POSTGRES_CONN_PORT=5432

METADATA_DATABASE_NAME=airflow_metadata_db
METADATA_DATABASE_USERNAME=airflow_meta_user
METADATA_DATABASE_PASSWORD="..."

CELERY_BACKEND_NAME=celery_results_db
CELERY_BACKEND_USERNAME=celery_user
CELERY_BACKEND_PASSWORD="..."

ELT_DATABASE_NAME=elt_db
ELT_DATABASE_USERNAME=yt_api_user
ELT_DATABASE_PASSWORD="..."

AIRFLOW_UID="50000"
AIRFLOW_WWW_USER_USERNAME="airflow"
AIRFLOW_WWW_USER_PASSWORD="airflow1234"
FERNET_KEY="..."

API_KEY="..."
CHANNEL_HANDLE="MrBeast"
```

# 08. Amending the .env

- 기존 .env 파일에 추가 변수들을 보완하는 단계
- 단순한 민감 정보 저장뿐 아니라 환경별 설정값도 함께 저장하는 구조
- 변수 이름 변경 금지 이유
    - docker-compose.yaml 등 다른 파일에서 해당 이름을 그대로 참조하기 때문
- 값 변경 가능 항목
    - Docker Hub namespace
    - Docker Hub repository
    - 계정 비밀번호
    - Airflow UI 계정 정보
    - API Key
- 유지 권장 항목
    - Postgres host 값 postgres
    - 각 데이터베이스 이름 구조
    - 기본 포트 5432
- 비밀번호 설정 시 주의사항
    - $, @등 특수문자 사용 시 parsing 오류 발생 가능성
    - 단순한 영문자/숫자 조합 사용 권장
- AIRFLOW_UID의 의미
    - 컨테이너 내부에서 Airflow가 사용할 사용자 ID
    - 기본값 50000 사용
- Fernet Key의 역할
    - 메타데이터 DB 내부 민감 정보 암호화 목적
    - Airflow 보안 기능과 관련된 핵심 값

# 09. docker-compose.yaml file to use - [VERY IMPORTANT]

- 강의에서 제공한 docker-compose.yaml을 그대로 사용할 필요성
- Airflow 공식 docker-compose.yaml이 아니라 강의 제공 버전 사용 지시 사항
- docker-compose.yaml의 역할
    - 여러 개의 컨테이너를 하나의 설정 파일로 관리하는 방식
    - 한 번의 명령으로 전체 Airflow 환경 실행 가능 구조
- 주요 설정 내용
    - 공통 Airflow 설정 정의
    - Postgres, Redis, Webserver, Scheduler, Worker, Init 등 서비스 정의
    - .env 파일과 연결된 환경 변수 적용
    - volume 마운트
    - healthcheck 설정
    - restart 정책 적용
- 공통 Airflow 설정 내용
    - CeleryExecutor 사용
    - Metadata DB 연결
    - Celery result backend 연결
    - Redis broker 연결
    - Fernet Key 적용
    - 기본 DAG pause 설정
    - example DAG 비활성화
    - Airflow Connection 및 Variable 환경 변수 등록
- Volume 마운트 구조
    - ./config:/opt/airflow/config
    - ./dags:/opt/airflow/dags
    - ./data:/opt/airflow/data
    - ./include:/opt/airflow/include
    - ./logs:/opt/airflow/logs
    - ./pulgins:/opt/airflow/plugins
    - ./tests:/opt/airflow/tests
- 주요 서비스
    - postgres
    - redis
    - airflow-webserver
    - airflow-scheduler
    - airflow-worker
    - airflow-init
    - airflow-cli

# 10. init-multiple-databases.sh script - [VERY IMPORTANT]

- Postgres 내부에 3개의 데이터베이스를 초기화하기 위한 셸 스크립트 필요성
- 저장 위치
    - docker/postgres/init-multiple-databases.sh
- 디렉토리 구조 생성 필요성
    - 루트에 docker 폴더 생성
    - 그 아래 postgres 하위 폴더 생성
    - 해당 위치에 초기화 스크립트 저장
- macOS / Linux 사용자의 추가 작업
    - 실행 권한 부여 필요성
    - 실행 권한이 없으면 postgres 또는 airflow-init 컨테이너 시작 실패 가능성

```bash
chmod +x ./docker/postgres/init-multiple-databases.sh
```

# 11. Docker Compose

- Docker Compose의 개념
    - 여러 컨테이너를 YAML 파일 하나로 정의하고 실행하는 방식
    - 복잡한 서비스 구성을 단순화하는 도구
- 강의에서 설명한 핵심 포인트
    - 공통 변수는 airflow-common 영역에 정의
    - Docker 이미지 이름은 .env 변수로 참조
    - database connection URI를 환경 변수로 구성
    - Airflow Connections / Variables를 환경 변수 방식으로 정의
    - local 디렉토리를 volume으로 마운트
    - postgres 데이터를 named volume으로 영속 저장
- Postgres 서비스 설명
    - container_name 지정
    - postgres:13 이미지 사용
    - .env 파일 연결
    - 포트 5432 매핑
    - named volume 사용
    - init shell script 마운트
    - healthcheck 적용
    - restart always 설정
- Named volume의 의미
    - 컨테이너를 내려도 데이터 유지 가능 구조
    - Postgres DB 데이터 보존 목적
- init shell script 마운트 목적
    - 컨테이너 시작 시 3개 DB와 사용자 자동 생성 목적
- Redis 서비스 설명
    - CeleryExecutor용 메시지 브로커 역할
    - Scheduler와 Worker 사이 메시지 전달 기능
- Webserver 설명
    - 포트 8080에서 Airflow UI 제공
- Scheduler 설명
    - DAG 스케줄 감시 및 작업 실행 트리거 역할
- Worker 설명
    - 실제 Task 수행 역할
    - 필요 시 여러 개 확장 가능 구조
- Triggerer와 Flower
    - 강의에서는 비활성화된 상태
    - 현재 실습 범위에서 사용하지 않는 구성 요소
- 중요한 주의사항
    - .env 안의 자격 증명을 바꾼 뒤 기존 named volume을 유지하면 충돌 가능성
    - 이미 생성된 volume에는 이전 설정이 남아 있기 때문
- 충돌 해결 방식
    - 이전 비밀번호로 되돌리기
    - 또는 volume 삭제 후 다시 실행하기

```bash
docker volume ls
docker volume rm <volume_name>
```

# 12. docker commands

- Docker Compose 실행 명령어
    - docker compose up -d
    - detached mode로 백그라운드 실행하는 방식
- 컨테이너 실행 상태 확인 명령어
    - docker ps
    - 실행 중인 컨테이너 목록 확인 목적
- 컨테이너 내부 접근 명령어
    - docker exec -it <container_name> bash
    - 컨테이너 내부 쉘 접근 목적

```bash
docker compose up -d
docker ps
docker exec -it airflow-scheduler bash
```

- Airflow 관련 확인 명령어
    - airflow - -help
    - airflow info
    - airflow cheat-sheet
- 변경 사항 반영을 위한 재빌드 명령어
    - docker compose up -d - -build
    - Dockerfile 수정 후 이미지 재생성 및 재실행 목적
- 전체 종료 명령어
    - docker compose down
    - 컨테이너, 네트워크 정리 목적

```bash
docker compose up -d --build
docker compose down
```

- 종료 전 docker compose down 권장 이유
    - 포트 충돌 방지
    - 네트워크 오류 방지
    - 다음 실행 시 안정성 확보 목적

# 13. Stopping Docker containters before shutting down laptop - [IMPORTANT]

- 노트북 종료 전 docker compose down 실행 권장 사항
- 단순히 시스템을 종료하는 것보다 더 안전한 방식
- 이유
    - 실행 중 컨테이너 완전 종료
    - 포트 및 네트워크 즉시 해제
    - 다음 실행 시 충돌 방지
- 종료 후 확인 방법
    - docker ps 실행
    - 실행 중인 컨테이너가 없는지 확인
