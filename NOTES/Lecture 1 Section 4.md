# 01. Airflow Introduction

- Docker를 이용한 Airflow 실행 환경 구성 이후 실제 DAG 작성 단계로 진입하는 내용
- Airflow 내부에서 Python 코드를 Task와 DAG 구조로 변환하여 실행하는 과정의 시작
- 본 단계의 핵심 목적은 기존에 작성한 Python 스크립트를 Airflow가 인식하고 실행할 수 있는 형태로 재구성하는 작업
- Airflow 학습 흐름
    - Docker 환경 준비
    - Airflow 컨테이너 실행
    - DAG 생성
    - Task 정의
    - DAG 실행 및 결과 확인
- 본 강의에서 강조되는 점
    - Airflow는 단순 Python 실행 도구가 아니라 스케줄링과 의존성 관리를 수행하는 워크플로 오케스트레이션 도구라는 점
    - DAG을 실제로 만들기 위해 기존 코드의 구조 변경이 필요하다는 점

# 02. Refactoring of scripts to use Airflow

- 기존 데이터 추출 스크립트를 Airflow DAG 구조로 변환하는 단계
- 먼저 기존 Python 스크립트를 dags 폴더 내부로 이동해야 하는 이유
    - Airflow는 dags 폴더를 감시하여 DAG 파일을 읽기 때문
    - Docker volume 마운트를 통해 컨테이너 내부 Airflow가 해당 파일을 인식하기 때문
- 추가로 API 서브폴더 생성 후 기존 스크립트 저장 구조
- dags, config, include, logs, plugins, tests디렉토리가 docker compose 실행 과정에서 생성된다는 점
- Python 함수를 Airflow Task로 바꾸는 방식
    - @task decorator 사용
    - 각 함수 위에 decorator를 추가하여 Airflow가 Task로 인식할 수 있게 만드는 구조

```python
from airflow.decorators import task

@task
def get_playlist_id():
    pass
```

- decorator의 의미
    - 기존 함수를 감싸서 기능을 확장하는 문법
    - Airflow에서는 해당 함수를 Task 단위로 등록하는 역할
- 환경 변수 처리 방식 변경 필요성
    - 기존 코드 내부에 API Key나 CHANNEL_HANDLE 값을 직접 작성하는 방식 제거 필요성
    - .env에 저장한 값을 docker-compose를 통해 Airflow 환경 변수로 전달하는 구조 사용
- Airflow Variable을 사용하는 방식
    - Variable.get(”API_KEY”)
    - Variable.get(”CHANNEL_HANDLE”)

```python
from airflow.models import Variable

api_key = Variable.get("API_KEY")
channel_handle = Variable.get("CHANNEL_HANDLE")
```

- Variable 클래스를 import해야 한다는 점
- Airflow 환경 변수로 정의된 값은 AIRFLOW_VAR_ prefix를 통해 읽힌다는 점
- 환경 변수 확인을 위한 docker exec 및 grep 활용 가능성

```bash
docker exec -it airflow-webserver bash
env | grep AIRFLOW_VAR
```

- 환경 변수 방식으로 등록된 Variable의 특징
    - airflow variables list에서 보이지 않는 점
    - Airflow UI의 Variable 화면에서도 표시되지 않는 점
    - 그러나 Variable.get()으로는 정상 접근 가능한 점
- DAG을 실제로 정의하기 위한 main.py 생성 필요성
- main.py는 dags 폴더 최상위에 위치해야 하며 API 하위 폴더 안에 두지 않는 점
- 기본 import 구성
    - from airflow import DAG
    - import pendulum
    - from datetime import datetime, timedelta
    - 기존 Python 스크립트 내부 함수 import

```python
from airflow import DAG
from datetime import datetime, timedelta
import pendulum
```

- default_args의 의미
    - 여러 DAG에서 공통으로 사용할 수 있는 기본 설정 묶음
    - owner, retry, email, start_date 등 공통 설정 관리 목적

```python
default_args = {
    "owner": "airflow",
    "retries": 1,
    "retry_delay": timedelta(minutes=5)
}
```

- start_date에 대한 중요 개념
    - DAG가 시작되는 기준 시점
    - 실제 첫 실행은 그 시점 직후가 아니라 다음 interval 종료 시점에 이루어지는 구조
    - 예를 들어 daily DAG에서 1월 1일 00:00를 start_date로 두면 실제 첫 실행은 1월 2일 00:00라는 점
- DAG 선언 방식 세 가지 존재
    - with statement 방식
    - constructor 방식
    - decorator 방식
- 강의에서 선호하는 방식
    - with context manager 방식
    - 가독성과 구조 파악 측면에서 직관적인 방식

```python
with DAG(
    dag_id="produce_json",
    default_args=default_args,
    description="DAG to produce JSON file with the raw data",
    schedule_interval="0 14 * * *",
    catchup=False
) as dag:
    pass
```

- dag_id 의미
    - DAG의 고유 식별자
- description 의미
    - DAG 설명
    - DAG 수가 많아질수록 관리 편의성 향상 목적
- schedule_interval 의미
    - cron 표현식 기반 실행 주기 지정
    - “0 14 * * *”는 매일 오후 2시 실행 의미
- catchup=False 의미
    - 과거에 실행되지 못한 스케줄을 한꺼번에 다시 실행하지 않도록 하는 설정
- Task 정의 방식
    - decorator가 붙은 함수 호출 결과를 Task 객체처럼 연결하는 구조

```python
playlist_id_task = get_playlist_id()
video_ids_task = get_video_ids(playlist_id_task)
extract_data_task = extract_video_data(video_ids_task)
save_json_task = save_json(extract_data_task)
```

Task dependency 정의 방식

- >> 연산자를 이용해 실행 순서 명시
- 왼쪽 Task 완료 후 오른쪽 Task 실행 의미

```python
playlist_id_task >> video_ids_task >> extract_data_task >> save_json_task
```

- 실행 순서의 의미
    - playlist ID 추출
    - playlist를 기반으로 video ID 추출
    - video 데이터를 추출
    - 최종 JSON 파일 저장
- Airflow UI에서 DAG 확인 방식
    - webserver 실행 후 localhost:8080 접속
    - 기본 계정으로 로그인
    - DAG 목록에서 생성한 DAG 확인 가능
- 최초 DAG 오류 사례
    - 변수 이름과 실제 값 혼동으로 인한 오류 발생 가능성
    - 환경 변수 이름을 정확히 사용해야 한다는 점
- DAG 상태와 실행 방법
    - 기본적으로 paused 상태로 생성
    - 필요 시 UI에서 수동 trigger 가능
    - DAG 내부 상세 화면에서 task 진행 상태 확인 가능
- UI 색상 의미
    - 진한 초록색: 성공
    - 연한 초록색: 실행 중
    - 빨간색: 실패
- 각 Task 클릭 후 logs에서 실행 로그 확인 가능
- print 결과 등도 로그에서 확인 가능
- DAG 실행 성공 여부 최종 확인 방법
    - Airflow UI에서 task 성공 여부 확인
    - 로컬 data 디렉토리에 해당 날짜 JSON 파일 생성 여부 확인
- 로컬 환경의 한계
    - 스케줄 시간이 되어도 Docker 컨테이너가 실행 중이어야만 DAG이 자동 실행된다는 점
    - 노트북이 꺼져 있거나 Docker가 꺼져 있으면 실행 불가능하다는 점
- 운영 환경과의 차이
    - 운영 환경은 24시간 실행 서버 기반
    - self-managed 서버 또는 cloud/Kubernetes 환경에서 지속 실행 가능
    - 본 강의는 실무 적용을 위한 기초 도구 습득 목적이라는 점
