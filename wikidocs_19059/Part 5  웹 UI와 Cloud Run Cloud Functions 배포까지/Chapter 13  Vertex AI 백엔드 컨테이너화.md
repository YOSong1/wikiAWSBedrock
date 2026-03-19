지난 Chapter 12까지 우리는 로컬 환경에서 Streamlit을 이용해 훌륭한 웹 서비스를 만들었습니다. 하지만 이 서비스가 내 노트북 안에서만 돌아간다면 그 가치는 제한적일 것입니다. 동료나 고객이 언제 어디서든 접속할 수 있게 하려면 이 서비스를 24시간 꺼지지 않는 클라우드 서버에 올려야 합니다.

하지만 서버 환경은 내 PC 환경과 다릅니다. 운영체제(OS)가 다를 수도 있고, 설치된 파이썬 버전이나 라이브러리 버전이 다를 수도 있습니다. 이런 환경 차이 때문에 발생하는 오류를 원천적으로 차단하기 위해 사용하는 기술이 바로 **컨테이너(Container)**입니다.

이번 챕터에서는 우리의 RAG 애플리케이션을 어떤 환경에서도 똑같이 실행되도록 '컨테이너'라는 표준화된 상자에 담는 방법을 배웁니다. 이를 위해 사실상 업계 표준인 **Docker**를 사용하여 이미지를 빌드하고, 로컬에서 실행 테스트까지 완료하는 과정을 단계별로 상세히 알아보겠습니다.



---

## 13-1 FastAPI/Flask 기반 LLM/RAG API 서버 구조

**Streamlit**은 화면(Frontend)과 로직(Backend)이 하나로 합쳐진 도구라 프로토타이핑에는 좋지만, 대규모 서비스로 확장하거나 모바일 앱 등 다른 클라이언트와 연동하기에는 한계가 있습니다. 따라서 현업에서는 백엔드 로직을 API 서버로 분리하는 구조를 선호합니다.



### 13-1-1 API 서버의 필요성

우리가 만든 **RAGBackend** 클래스(기능)를 웹, 앱, 사내 메신저 봇 등 다양한 곳에서 가져다 쓸 수 있게 하려면, 이 기능을 HTTP 통신으로 호출할 수 있는 **API(Application Programming Interface)** 형태로 만들어야 합니다. 파이썬 진영에서는 **FastAPI**나 **Flask**가 이 역할을 담당하는 가장 대중적인 프레임워크입니다.

### 13-1-2 FastAPI 서버 코드 구조 설계

우리는 성능이 뛰어나고 문서화(Swagger)가 자동으로 되는 **FastAPI**를 사용하여 간단한 서버를 만들어 보겠습니다. 프로젝트 폴더에 `main.py`를 생성하고 다음과 같이 작성합니다.



```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from rag_backend import RAGBackend # Ch8에서 만든 클래스

# 1. FastAPI 앱 인스턴스 생성
app = FastAPI(title="Vertex AI RAG API")

# 2. RAG 백엔드 로드 (서버 시작 시 1회만 실행)
# 실제 운영 환경에서는 DB 연결 등을 여기서 수행합니다.
rag_backend = RAGBackend()

# 3. 요청 데이터 구조 정의 (Pydantic 모델)
class QueryRequest(BaseModel):
    question: str

# 4. API 엔드포인트 정의
@app.post("/chat")
async def chat_endpoint(request: QueryRequest):
    """
    사용자의 질문을 받아 RAG 검색 및 답변을 반환하는 API
    """
    try:
        # 질문에 대한 관련 문서 검색
        docs = rag_backend.retrieve_documents(request.question)
        # (여기서 LLM 호출 로직이 추가될 수 있음)
        # 예시를 위해 검색된 문서 내용만 반환
        context = "\n".join([d.page_content for d in docs])
        return {
            "question": request.question,
            "answer": "생성된 답변이 여기 들어갑니다.",
            "context": context
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

이렇게 작성하면, 외부에서 `/chat` 주소로 질문을 던졌을 때 JSON 형태로 답변을 주는 독립적인 서버가 완성됩니다.

> 📷 **[이미지]** [Client(Web/App)] <--> [HTTP Request/Response] <--> [FastAPI Server] <--> [RAG Logic] <--> [Vector DB]로 이어지는 API 서버 아키텍처 다이어그램. 클라이언트와 서버가 분리된 구조임을 강조



---

## 13-2 Dockerfile 작성과 이미지 빌드

코드를 짰으니 이제 이 코드를 실행하는 데 필요한 모든 것(파이썬, 라이브러리, 설정 파일)을 하나의 '이미지(Image)'로 구워낼 차례입니다. 이 레시피를 적어놓은 파일이 바로 **Dockerfile**입니다.



### 13-2-1 Dockerfile의 핵심 명령어

프로젝트 최상위 폴더에 확장자 없는 `Dockerfile`이라는 파일을 만들고, 다음과 같이 내용을 채워 넣습니다. 각 줄이 어떤 의미인지 이해하는 것이 중요합니다.

```dockerfile
# 1. 베이스 이미지 선택: Python 3.10이 설치된 가벼운 리눅스(Slim) 사용
FROM python:3.10-slim

# 2. 작업 디렉토리 설정: 컨테이너 내부의 작업 공간을 /app으로 지정
WORKDIR /app

# 3. 의존성 파일 복사 및 설치
# (캐시 효율을 위해 소스코드보다 먼저 복사함)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 4. 소스 코드 전체 복사
COPY . .

# 5. 환경 변수 설정 (옵션)
# 파이썬 출력이 버퍼링 없이 즉시 로그에 찍히도록 설정
ENV PYTHONUNBUFFERED=1

# 6. 포트 개방 알림 (문서화 목적)
EXPOSE 8080

# 7. 컨테이너 실행 시 작동할 명령어 (Uvicorn으로 FastAPI 실행)
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

### 13-2-2 도커 이미지 빌드 (Build)

레시피(Dockerfile)가 준비되었으니 요리를 시작합니다. 터미널에서 다음 명령어를 입력합니다.

```bash
# docker build -t [이미지이름]:[태그] [Dockerfile위치]
docker build -t vertex-rag-server:v1 .
```

이 명령어를 실행하면 도커는 베이스 이미지를 다운로드하고, 라이브러리를 설치하는 과정을 거쳐 `vertex-rag-server:v1`이라는 이름의 이미지를 생성합니다. 이 이미지는 이제 내 컴퓨터뿐만 아니라, 도커가 설치된 세상의 모든 컴퓨터에서 똑같이 실행될 수 있습니다.

> 📷 **[이미지]** Dockerfile의 각 단계(FROM, COPY, RUN...)가 층층이 쌓여 하나의 'Docker Image'가 되는 레이어(Layer) 구조를 시각화. 소스 코드와 라이브러리가 합쳐져서 하나의 단단한 블록이 되는 모습



---

## 13-3 환경 변수/비밀 키 관리(.env, Secret Manager 연계)

컨테이너화할 때 가장 주의해야 할 점은 **보안**입니다. Google Cloud 프로젝트 ID나, API Key와 같은 민감한 정보는 절대로 **Dockerfile**이나 코드 안에 직접 적어넣어서는 안 됩니다. 이미지가 유출되면 키도 함께 유출되기 때문입니다.

### 13-3-1 환경 변수(Environment Variable) 활용

대신 우리는 변수 처리(`os.environ.get`)를 통해, 컨테이너가 실행되는 시점에 값을 주입받도록 설계해야 합니다.

**코드 수정 (rag_backend.py 등):**

```python
import os

# 코드에 직접 쓰는 대신 환경 변수에서 가져옴
PROJECT_ID = os.environ.get("GCP_PROJECT_ID")
```

**로컬 개발 시 (.env 파일)**

개발 편의를 위해 `.env` 파일에 키를 저장해두고 `python-dotenv` 라이브러리로 읽어옵니다. 단, 이 `.env` 파일은 `.dockerignore`에 추가하여 도커 이미지에 포함되지 않도록 막아야 합니다.



### 13-3-2 Google Secret Manager 연계

클라우드(Cloud Run)에 배포할 때는 **Secret Manager**라는 보안 금고 서비스를 사용하는 것이 정석입니다.

1. GCP 콘솔의 Secret Manager에 비밀값을 저장합니다.
2. Cloud Run 배포 시 "이 Secret을 환경 변수로 넣어줘"라고 설정합니다.

이렇게 하면 코드는 수정하지 않고도 안전하게 비밀 정보를 관리할 수 있습니다.

> 📷 **[이미지]** 개발자 PC(.env), 도커 컨테이너(Env Vars), 클라우드(Secret Manager) 세 가지 환경에서 비밀 키가 어떻게 주입(Injection)되는지 보여주는 흐름도. 코드는 하나지만, 환경에 따라 키를 받는 출처가 달라짐을 보여주세요



---

## 13-4 로컬 Docker 실행으로 엔드투엔드 테스트

이미지를 빌드했다면, 클라우드에 올리기 전에 내 PC에서 먼저 실행해 봐야 합니다. 이를 통해 패키지 누락이나 경로 설정 오류 등을 미리 잡아낼 수 있습니다.

### 13-4-1 도커 컨테이너 실행 (Run)

다음 명령어로 방금 만든 이미지를 실행합니다. 이때 `-e` 옵션을 사용하여 필요한 환경 변수를 주입합니다.

```bash
docker run -p 8080:8080 \
-e GCP_PROJECT_ID="your-project-id" \
vertex-rag-server:v1
```

**옵션 설명:**

- `-p 8080:8080` - 내 PC의 8080번 포트로 들어오는 요청을 컨테이너의 8080번 포트로 연결해라.
- `-e ...` - 환경 변수 설정.



### 13-4-2 API 테스트 (curl 또는 Postman)

컨테이너가 정상적으로 실행되었다면, 새로운 터미널을 열어 요청을 보내봅니다.

```bash
curl -X POST "http://localhost:8080/chat" \
-H "Content-Type: application/json" \
-d '{"question": "Vertex AI가 뭐야?"}'
```

성공적으로 JSON 응답이 온다면, 여러분의 백엔드 서버는 이제 배포 준비를 완벽하게 마친 것입니다. 만약 에러가 난다면 `docker logs [컨테이너ID]` 명령어로 에러 로그를 확인하며 디버깅해야 합니다.

> 📷 **[이미지]** 터미널 두 개를 띄워놓은 화면 캡처. 왼쪽 터미널은 docker run 명령어로 서버가 실행되어 로그가 올라가는 모습, 오른쪽 터미널은 curl 명령어로 요청을 보내고 응답을 받은 모습을 보여주세요



---

## 마무리



이것으로 Chapter 13을 마칩니다. 우리는 파이썬 코드를 FastAPI로 감싸서 서버로 만들고, 이를 Docker로 포장하여 어디서든 실행 가능한 상태로 만들었습니다. 이제 여러분의 애플리케이션은 "내 컴퓨터에서만 되는" 코드가 아니라, "전 세계 어디서나 실행 가능한" 서비스가 되었습니다.

다음 Chapter 14에서는 이 도커 이미지를 구글 클라우드의 저장소(Artifact Registry)에 업로드하고, 서버리스 컴퓨팅 서비스인 Cloud Run에 배포하여 실제 인터넷 주소로 접속할 수 있는 라이브 서비스를 런칭해 보겠습니다.
