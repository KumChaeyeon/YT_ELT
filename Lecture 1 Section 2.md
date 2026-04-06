# 01. What is an API

- 본 실습에서는 외부 데이터 수집을 위해 API(Application Programming Interface)를 활용
    - API는 클라이언트와 서버 간의 데이터 요청과 응답을 처리하는 인터페이스
    - 사용자는 직접 데이터베이스에 접근하지 않고 HTTP 요청을 통해 필요한 정보를 획득
- Python에서는 다음과 같은 방식으로 API 요청과 응답 처리를 수행
    
    ```python
    response = requests.get(url)
    data = response.json()
    ```
    
    - 이 과정을 통해 API는 단순한 기능 호출이 아니라, **네트워크 기반의 데이터 통신 구조**임을 확인
    - 특히, 반환되는 데이터가 JSON 형식이므로, 이후 데이터 추출 과정에서는 계층적 구조를 이해하고 적절한 경로를 접근하는 것이 중요

# 02. Getting the YouTube API Key

- API 요청 수행을 위해 인증 수단으로 API Key를 사용
    - 본 구현에서는 보안성을 고려하여 API Key를 코드에 직접 포함하지 않고 .env 파일을 통해 관리
    (.env 파일에 대한 자세한 내용은 본 일지 후반에 다시 다룰 예정)

```python
from dotenv import load_dotenv
import os

load_dotenv(dotenv_path="./.env")
API_KEY = os.getenv("API_KEY")
```

- 이와 같은 방식으로 구성함으로써 다음과 같은 효과를 얻을 수 있음
    - 민감 정보의 외부 노출 방지
    - 코드와 설정의 분리
    - 협업 환경에서의 보안성 확보
- 따라서 API Key 관리 또한 데이터 추출 과정에서 중요한 설계 요소로 판단

# 03. Google Cloud Shell

- 클라우드 기반 개발 환경을 활용하면 별도의 로컬 설정 없이 코드 실행이 가능함을 확인
    - 그러나 본 실습에서는 Python 라이브러리 설치, 파일 저장, Git 연동 등을 고려하여 로컬 개발 환경을 활용

# 04. Youtube API Explorer and Postman

- API 요청 구조를 사전에 분석하기 위해 다양한 테스트 도구를 활용
    - 이 과정에서 다음과 같은 핵심 요소를 확인
        - part: 요청할 데이터 범위 지정
        - id 또는 playlistId: 대상 데이터 지정
        - key: 인증 정보
- JSON 응답 구조를 분석하여, 필요한 데이터가 어떤 경로에 존재하는지 사전에 파악
    - ex) 영상 통계 데이터
    
    ```
    statistics → viewCount / likeCount / commentCount
    ```
    
    - 이러한 사전 분석을 통해 이후 Python 코드에서 데이터 접근 경로를 정확하게 설정할 수 있음

# 05. Setting Up Git Remote

- 프로젝트 진행 과정에서 Git을 활용하여 코드 버전 관리를 수행
- 주요 작업
    - 로컬 저장소 초기화
    - 원격 저장소 생성 및 연결
    - 코드 변경 시 commit 및 push
- 이를 통해 데이터 추출 코드의 수정 이력을 관리하고, 개발 과정을 체계적으로 기록
    - 데이터 수집 로직은 반복적인 수정이 발생할 수 있기 때문에 버전 관리의 중요성 또한 확인

# 06. Create Virtual Environment

- Python 가상환경을 생성하여 프로젝트 간 의존성 출동을 방지
- 이를 통해 다음과 같은 효과를 얻을 수 있음
    - 라이브러리 버전 독립성 유지
    - 프로젝트별 환경 분리
    - 재현 가능한 개발 환경 구축
- 또한, .gitignore를 활용하여 venv, ‘__pyache_, .env, 등의 파일을 제외함으로써 저장소를 효율적으로 관리

# 07. Analysis of Data Extraction Variables

- 본 실습에서는 영상 데이터를 분석하기 위해 다음 7개의 변수를 정의
    - video_id
    - title
    - publishedAt
    - duration
    - viewCount
    - likeCount
    - commentCount
- 이 변수들을 추출하기 위해 API 구조를 분석한 결과, 단일 요청으로는 모든 데이터를 획득할 수 없음을 확인
    - 따라서 다음과 같은 단계적 접근이 필요
    
    ```
    Channel → Playlist → Video IDs → Video Data
    ```
    
    - 즉, 최종 데이터에 도달하기 위해 중간 단계 데이터를 순차적으로 추출하는 구조로 설계

# 08. Building the Videos Statistics script - Part 1 Playlist ID

- 첫 번째 단계에서는 채널 정보를 기반으로 업로드 플레이리스트 ID를 추출

```python
def get_playlist_id():
    url = f"...channels?part=contentDetails&forHandle={CHANNEL_HANDLE}&key={API_KEY}"
    response = requests.get(url)
    data = response.json()

    channel_items = data['items'][0]
    channel_playlistId = channel_items['contentDetails']['relatedPlaylists']['uploads']
```

- 이 과정에서 확인한 점
    - 채널 API는 영상 목록을 직접 제공하지 않음
    - 업로드 영상은 별도의 playlist로 관리됨
    - JSON 구조 분석이 필수적
- 따라서 데이터 추출의 첫 단계는 **playlist ID** 확보로 설정

# 09. Introducing the .env

- API Key를 .env 파일로 분리하여 관리함으로써 코드의 보안성과 유지보수성을 향상
    - 이는 단순한 설정이 아니라 실제 개발 환경에서 반드시 고려해야 할 보안 설계 요소임을 확인

# 10. Building the Videos Statistics script - Part 2 Unique Video IDs

- 두 번째 단계에서는 playlist를 기반으로 전체 video ID를 수집

```python
while True:
    if pageToken:
        url += f"&pageToken={pageToken}"

    data = response.json()

    for item in data.get('items', []):
        video_ids.append(item['contentDetails']['videoId'])

    pageToken = data.get('nextPageToken')

    if not pageToken:
        break
```

- 이 과정에서 다음과 같은 특징을 확인
    - API는 한 번에 최대 50개 데이터만 반환
    - nextPageToken을 이용한 pagination 필요
    - 반복문을 통해 전체 데이터 수집 가능
- 즉, 대량 데이터 수집에서는 단일 요청이 아닌 **페이지 기반 반복 요청 구조가 필수적**임을 확인

# 11. Building the Videos Statistics script - Part 3 Video Data

- 세 번째 단계에서는 video ID를 기반으로 실제 영상 데이터를 추출

```python
video_data = {
    "video_id": video_id,
    "title": snippet['title'],
    "publishedAt": snippet['publishedAt'],
    "duration": contentDetails['duration'],
    "viewCount": statistics.get('viewCount', None),
    "likeCount": statistics.get('likeCount', None),
    "commentCount": statistics.get('commentCount', None),
}
```

- 이 과정에서 확인한 주요 사항
    - 데이터는 snippet, contentDetails, statistics 등 여러 영역에 분산됨
    - 일부 데이터는 존재하지 않을 수 있음
    - .get()을 통해 예외 상황 처리 필요
- 또한, video ID는 최대 50개 단위로 요청해야 하므로, batch 처리 로직을 구현
    
    ```python
    video_ids_str = ",".join(batch)
    ```
    
    - 이를 통해 API 요청 횟수를 줄이면서 효율적으로 데이터를 수집할 수 있음

# 12. Building the Videos Statistics script - Part 4 Save to JSON

- 마지막 단계에서는 추출한 데이터를 JSON 파일로 저장

```python
def save_to_json(extracted_data):
    file_path = f"./data/YT_data_{date.today()}.json"

    with open(file_path, "w", encoding="utf-8") as json_outfile:
        json.dump(extracted_data, json_outfile, indent=4, ensure_ascii=False)
```

- 이 과정의 의미는 다음과 같음
    - 데이터 영구 저장
    - 분석 및 활용 가능성 확보
    - 데이터 파이프라인의 “Load” 단계 준비
- 파일명에 날짜를 포함시킴으로써 실행 시점별 데이터 관리가 가능하도록 설계

# 13. 최종 결론

- 본 실습에서는 YouTube API를 활용하여 특정 채널의 영상 데이터를 단계적으로 수집하는 시스템을 구현
    - 채널 정보에서 플레이리스트 ID를 추출, 해당 플레이리스트를 기반으로 전체 video ID를 수집한 뒤,
    각 영상의 상세 데이터를 조회하는 구조로 설계
- API 호출 제한을 고려하여 pagination과 batch 처리를 적용
    - 일부 데이터 누락 가능성을 반영하여 예외 처리를 포함
- 수집된 데이터를 JSON 파일로 저장함으로써 데이터 분석 및 데이터 웨어하우스 적재가 가능한 형태로 변환
- 이를 통해 단순한 API 사용을 넘어, 실제 데이터 엔지니어링 관점에서의 데이터 수집 및 처리 과정을 구현