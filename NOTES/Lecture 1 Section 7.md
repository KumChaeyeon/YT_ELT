# 01. CI/CD Introduction

- CI/CD는 지속적인 통합(Continuous Integration)과 지속적인 배포(Continuous Deployment 또는 Continuous Delivery)를 의미하는 개념
- 개발 과정에서 코드 변경 사항을 자동으로 테스트하고 빌드하며 배포하는 과정을 자동화하기 위한 목적 존재
- 기존 개발 방식에서는 기능 추가 이후 직접 테스트와 배포를 반복적으로 수행해야 했기 때문에 많은 시간과 실수가 발생 가능
    - 이러한 문제를 해결하기 위해 CI/CD 환경 사용
- 본 강의에서는 GitHub Actions Workflow를 활용하여 CI/CD 환경을 구성하는 방법 학습
    - YAML 파일을 기반으로 자동화된 작업 흐름 구성 방식 사용
    - GitHub Workflow: 하나 이상의 YAML 스크립트로 구성된 자동화 프로세스
    - Workflow 내부에는 Job과 Step이 존재하며 각각의 작업이 순차적으로 실행되는 구조 사용
- GitHub Actions를 사용하는 주요 목적
    - 코드 변경 자동 감지 기능
    - Docker 이미지 자동 빌드 기능
    - 테스트 자동 수행 기능
    - 자동 배포 환경 구성 기능
    - 개발 생산성 향상 목적
- Workflow 파일은 .github/workflows 경로 내부에 생성 필요
    
    ```
    .github/
    └── workflows/
        └── cicd_youtube_elt.yaml
    ```
    
    - Workflow 파일은 YAML 형식 사용
    - YAML은 들여쓰기 기반 설정 파일 형식이며 GitHub Actions에서 자동화 작업 정의 시 사용
        
        ```
        name: CI CD Pipeline
        ```
        
        - name 항목은 GitHub Actions 화면에서 표시되는 Workflow 이름 의미

# 02. Commit and Push

- CI/CD 작업 시작 이전 반드시 현재 코드 상태 정리 필요
    - 변경된 파일을 Commit 및 Push하여 원격 저장소와 동기화 과정 수행 필요
- 개발 과정에서는 새로운 기능 추가 또는 기존 코드 수정 지속적으로 발생 가능
    - 이러한 변경 사항은 Git을 이용하여 버전 관리 수행
- 기본적인 Git 작업 흐름
    1. 변경 파일 추가
    2. Commit 생성
    3. GitHub Push 수행
    
    ```bash
    git add .
    git commit -m "Add CI/CD workflow"
    git push
    ```
    
    - git add . : 변경 파일을 Staging Area에 추가하는 작업
    - git commit: 현재 변경 상태 저장 과정
    - git push: 로컬 저장소 내용을 GitHub 원격 저장소에 업로드하는 과정
- 버전 관리 시스템 사용 목적
    - 코드 변경 이력 관리 목적
    - 협업 환경 지원 목적
    - 이전 버전 복구 목적
    - 안정적인 개발 환경 유지 목적
- 실제 프로젝트에서는 기능 단위 Commit 진행 방식 권장
    - 예를 들어 새로운 함수 추가 또는 오류 수정 이후 즉시 Commit 수행 방식 사용 가능
- Commit Message는 작업 내용을 명확하게 표현 필요
    - 의미 없는 메시지 대신 작업 목적을 포함하는 방식 권장

```bash
git commit -m "Fix Docker compose configuration"
```

# 03. CI/CD Part 1 - Docker Image Builds

- CI/CD 구성 첫 번째 단계는 Docker 이미지 자동 빌드 과정
    - 이를 위해 GitHub Workflow 작성 필요
- Workflow 파일 내부에서는 실행 조건과 Job 정의 과정 포함
- 우선 Workflow 실행 조건 정의 필요
    - GitHub Actions에서는 on 키워드를 사용하여 이벤트 기반 실행 조건 정의 가능.

```yaml
on:
  push:
    branches:
      - main

  pull_request:
    branches:
      - main

  workflow_dispatch:
```

- 위 설정은 다음 상황에서 Workflow 실행 의미
    - main 브랜치 Push 발생 시
    - Pull Request 생성 시
    - 수동 실행 발생 시
- Feature Branch까지 포함하는 경우 다음과 같은 방식 사용 가능
    
    ```yaml
    on:
      push:
        branches:
          - main
          - feature/*
    ```
    
    - feature/*는  feature로 시작하는 모든 브랜치 의미
        - 새로운 기능 개발 과정에서 사용 가능한 구조
- 다음 단계는 Job 정의 과정
    - GitHub Actions에서 Job은 실제 작업 단위 의미
    - 기본 구조
        - 
        
        ```yaml
        jobs:
        ```
        
    - Ubuntu 최신 환경 사용 예시
        
        ```yaml
        runs-on: ubuntu-latest
        ```
        
        - ubuntu-latest는 GitHub에서 제공하는 최신 Ubuntu 가상 환경 의미.
- Workflow 실행 시 가장 먼저 Repository 코드 Checkout 과정 수행 필요
    
    ```yaml
    - name: Checkout Repository
      uses: actions/checkout@v4
    ```
    
    - 위 과정은 GitHub 저장소 코드 접근 목적
        - 코드를 가져와야 이후 Docker Build 및 테스트 가능.
- 다음 과정은 특정 파일 변경 여부 확인 단계
    - Docker 이미지 재빌드 필요 여부 판단 목적
    - 사용 Action
        
        ```yaml
        - name: Check Changed Files
          id: changed-files
          uses: tj-actions/changed-files@v45
        ```
        
        - 해당 Action은 특정 파일 변경 여부 확인 가능
        - Dockerfile 또는 requirements.txt 변경 시에만 이미지 재빌드 수행 구조 사용
- Docker Build 환경 설정 과정 필요
    
    ```yaml
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    ```
    
    - Buildx는 Docker 이미지 빌드 기능 제공 도구
- Docker Hub 로그인 과정 필요
    - 이미지 Push를 위해 인증 과정 수행 필요
    
    ```yaml
    - name: Login to DockerHub
      uses: docker/login-action@v3
      with:
        username: ${{ vars.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_PASSWORD }}
    ```
    
    - GitHub Actions에서는 민감 정보 저장을 위해 Secrets 기능 제공
        - Secrets : 비밀번호 등 민감 정보 저장 목적
        - Variables : 일반 변수 저장 목적
- Docker Image Build 및 Push 과정
    - 
    
    ```yaml
    - name: Build and Push Docker Image
      uses: docker/build-push-action@v6
      with:
        push: true
        tags:|
          ${{ vars.DOCKERHUB_NAMESPACE }}/${{ vars.DOCKERHUB_REPOSITORY }}:latest
          ${{ vars.DOCKERHUB_NAMESPACE }}/${{ vars.DOCKERHUB_REPOSITORY }}:${{ github.sha }}
    ```
    
    - latest태그는 최신 이미지 의미.
    - github.sha는 Commit Hash 기반 고유 태그 의미.
    - Commit Hash를 사용하는 이유는 특정 코드 버전과 Docker 이미지를 연결하기 위한 목적 존재.
        - 문제 발생 시 어떤 Commit에서 생성된 이미지인지 추적 가능 구조.
    - Docker Compose에서는 기존 .env기반 이미지 태그 사용 방식 제거 필요.
        - GitHub Actions 환경에서는 .env파일 접근 불가능하기 때문.
        - 따라서 이미지 태그를 latest로 직접 지정하는 방식 사용

# 04. CI/CD Part 2 - Testing

- 두 번째 Job은 테스트 자동화 목적
    - Unit Test, Integration Test, End-to-End Test 실행 과정 포함
- 해당 Job은 첫 번째 Job 이후 실행 필요
    - GitHub Actions에서는 needs 키워드를 사용하여 Job Dependency 설정 가능
    
    ```yaml
    needs: build-and-push-image
    ```
    
    - 위 설정은 Docker 이미지 Build 이후 테스트 실행 의미
- GitHub Actions 환경에서는 .env 파일 사용 불가능
    - 따라서 Workflow 내부에서 환경 변수 직접 지정 필요
    
    ```yaml
    env:
      POSTGRES_USER: ${{ vars.POSTGRES_USER }}
      POSTGRES_PASSWORD: ${{ secrets.POSTGRES_PASSWORD }}
    ```
    
    - Docker Compose 내부에서도 .env 참조 부분 제거 필요
        - GitHub Actions에서 전달된 환경 변수 사용 구조 변경 필요
- 테스트 실행 전 Docker Compose 실행 과정 필요
    
    ```yaml
    - name: Docker Compose Up
      run: docker compose up -d
    ```
    
    - 위 과정은 Airflow, Postgres 등 컨테이너 실행 목적
- 다음 단계는 Pytest 기반 Unit 및 Integration Test 수행 과정
    
    ```yaml
    - name: Run Tests
      run: pytest
    ```
    
    - Pytest는 Python 테스트 프레임워크 의미
        - 자동으로 테스트 코드 실행 가능
- End-to-End 테스트는 Airflow DAG 실행 방식 사용
    
    ```yaml
    airflow dags test
    ```
    
    - 강의에서는 세 개 DAG 사용
        - produce_json
        - update_db
        - data_quality
- 반복 실행 예시
    
    ```yaml
    for dag_idin produce_json update_db data_quality
    do
        airflow dags test$dag_id
    done
    ```
    
- 모든 테스트 종료 이후 Docker Compose 종료 과정 수행
    
    ```yaml
    - name: Docker Compose Down
      run: docker compose down
    ```
    
    - 테스트 자동화 목적 다음과 같음
        - 코드 오류 조기 발견 목적
        - 서비스 안정성 향상 목적
        - 배포 전 검증 자동화 목적
- GitHub Actions 실행 결과는 웹 화면에서 확인 가능
    - 성공 시 초록색 체크 표시 출력 구조 사용

# 05. Github Actions Workflow Dispatch

- GitHub Actions는 자동 실행 외 수동 실행 기능 제공
    - Workflow Dispatch라고 표현
- Workflow Dispatch는 GitHub Actions UI에서 직접 Workflow 실행 가능한 기능을 의미
    - 하지만 기존 Workflow 조건은 파일 변경 여부 기반 실행 구조 사용
    - 따라서 수동 실행 시 Job이 Skip 되는 문제 발생 가능
    - 이를 해결하기 위해 조건문 수정 필요
- 기존 방식
    
    ```yaml
    if: steps.changed-files.outputs.any_changed == 'true'
    ```
    
- 수정 방식
    
    ```yaml
    if: steps.changed-files.outputs.any_changed == 'true' || github.event_name == 'workflow_dispatch'
    ```
    
    - 위 설정은 다음 두 경우 실행 의미
        - 파일 변경 발생 시
        - Workflow Dispatch 수동 실행 시
- GitHub Actions 화면에서는 Run workflow 버튼 사용 가능
    - 수동 실행 이후 Build, Test, Push 과정 전체 실행 가능
- Workflow 실행 결과 확인 가능 항목
    - Docker Build 성공 여부
    - 테스트 성공 여부
    - Docker Hub Push 성공 여부
    - 로그 출력 결과
- Docker Hub에서는 최신 이미지 확인 가능
    - Commit Hash 기반 Tag도 함께 생성 구조 사용
    - Git Commit Hash 확인 명령
        
        ```bash
        git log
        ```
        
        - Docker Image Tag와 Commit Hash 연결을 통해 특정 버전 추적 가능 구조 사용
