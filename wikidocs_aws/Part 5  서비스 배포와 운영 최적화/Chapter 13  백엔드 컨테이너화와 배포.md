지난 Chapter 12까지 우리는 Streamlit을 이용해 Bedrock Knowledge Base와 연동된 챗봇 UI를 완성했습니다. 하지만 이 서비스가 내 노트북 안에서만 돌아간다면 그 가치는 제한적일 것입니다. 동료나 고객이 언제 어디서든 접속할 수 있게 하려면, 이 서비스를 24시간 꺼지지 않는 클라우드 서버에 올려야 합니다.

하지만 서버 환경은 내 PC 환경과 다릅니다. 운영체제(OS)가 다를 수도 있고, 설치된 파이썬 버전이나 라이브러리 버전이 다를 수도 있습니다. 이런 환경 차이 때문에 발생하는 "내 PC에서는 되는데..."라는 악명 높은 문제를 원천적으로 차단하기 위해 사용하는 기술이 바로 **컨테이너(Container)**입니다.

이번 챕터에서는 우리의 Bedrock 기반 RAG 애플리케이션을 어떤 환경에서도 똑같이 실행되도록 '컨테이너'라는 표준화된 상자에 담는 방법을 배웁니다. 이를 위해 사실상 업계 표준인 **Docker**를 사용하여 이미지를 빌드하고, AWS의 컨테이너 이미지 저장소인 **Amazon ECR(Elastic Container Registry)**에 푸시한 뒤, 로컬에서 엔드투엔드 테스트까지 완료하는 과정을 단계별로 상세히 알아보겠습니다.



---

## 13-1 FastAPI 기반 Bedrock API 서버 구조

**Streamlit**은 화면(Frontend)과 로직(Backend)이 하나로 합쳐진 도구라 프로토타이핑에는 좋지만, 대규모 서비스로 확장하거나 모바일 앱 등 다른 클라이언트와 연동하기에는 한계가 있습니다. 따라서 현업에서는 백엔드 로직을 독립적인 API 서버로 분리하는 구조를 선호합니다.



### 13-1-1 API 서버의 필요성

우리가 만든 Bedrock 기반 RAG 로직을 웹, 앱, 사내 메신저 봇 등 다양한 곳에서 가져다 쓸 수 있게 하려면, 이 기능을 HTTP 통신으로 호출할 수 있는 **API(Application Programming Interface)** 형태로 만들어야 합니다. API 서버를 분리하면 다음과 같은 이점이 있습니다.

- **다중 클라이언트 지원**: 웹, 모바일, Slack 봇 등 어떤 클라이언트든 같은 API를 호출할 수 있습니다.
- **독립적 스케일링**: 프론트엔드와 백엔드를 각각 필요에 따라 확장할 수 있습니다.
- **유지보수 용이**: 백엔드 로직을 변경해도 프론트엔드에 영향을 주지 않습니다.

파이썬 진영에서는 **FastAPI**와 **Flask**가 이 역할을 담당하는 가장 대중적인 프레임워크입니다. 우리는 비동기 처리 성능이 뛰어나고 Swagger 문서가 자동으로 생성되는 **FastAPI**를 사용하겠습니다.

> **💡 핵심 포인트:** FastAPI는 타입 힌트를 기반으로 요청/응답 데이터를 자동 검증하고, `/docs` 경로에서 대화형 API 문서(Swagger UI)를 제공합니다. 별도의 문서 작성 없이도 API를 테스트해볼 수 있어 개발 생산성이 크게 향상됩니다.

### 13-1-2 FastAPI + Bedrock Runtime 서버 코드 설계

프로젝트 폴더에 `main.py`를 생성하고 다음과 같이 작성합니다. 이 서버는 사용자의 질문을 받아 Bedrock의 Knowledge Base에서 관련 문서를 검색하고, Claude 모델을 통해 답변을 생성하는 역할을 합니다.

```python
import os
import json
import boto3
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional

# 1. FastAPI 앱 인스턴스 생성
app = FastAPI(title="Bedrock RAG API Server")

# 2. AWS 클라이언트 초기화 (서버 시작 시 1회만 실행)
AWS_REGION = os.environ.get("AWS_REGION", "us-east-1")
KNOWLEDGE_BASE_ID = os.environ.get("KNOWLEDGE_BASE_ID")
MODEL_ID = os.environ.get("MODEL_ID", "anthropic.claude-3-5-sonnet-20241022-v2:0")

bedrock_runtime = boto3.client("bedrock-runtime", region_name=AWS_REGION)
bedrock_agent_runtime = boto3.client("bedrock-agent-runtime", region_name=AWS_REGION)

# 3. 요청/응답 데이터 구조 정의 (Pydantic 모델)
class QueryRequest(BaseModel):
    question: str
    max_tokens: Optional[int] = 1024

class QueryResponse(BaseModel):
    question: str
    answer: str
    sources: list

# 4. 헬스체크 엔드포인트
@app.get("/health")
async def health_check():
    return {"status": "healthy", "region": AWS_REGION}

# 5. RAG 질의응답 엔드포인트
@app.post("/chat", response_model=QueryResponse)
async def chat_endpoint(request: QueryRequest):
    """
    사용자의 질문을 받아 Knowledge Base 검색 + Claude 답변을 반환하는 API
    """
    try:
        # Knowledge Base에서 관련 문서 검색 후 답변 생성
        response = bedrock_agent_runtime.retrieve_and_generate(
            input={"text": request.question},
            retrieveAndGenerateConfiguration={
                "type": "KNOWLEDGE_BASE",
                "knowledgeBaseConfiguration": {
                    "knowledgeBaseId": KNOWLEDGE_BASE_ID,
                    "modelArn": f"arn:aws:bedrock:{AWS_REGION}::foundation-model/{MODEL_ID}",
                    "retrievalConfiguration": {
                        "vectorSearchConfiguration": {
                            "numberOfResults": 5
                        }
                    }
                }
            }
        )

        # 답변 텍스트 추출
        answer = response["output"]["text"]

        # 출처(Citation) 정보 추출
        sources = []
        for citation in response.get("citations", []):
            for ref in citation.get("retrievedReferences", []):
                source_uri = ref.get("location", {}).get("s3Location", {}).get("uri", "")
                sources.append(source_uri)

        return QueryResponse(
            question=request.question,
            answer=answer,
            sources=list(set(sources))  # 중복 제거
        )

    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

> 📷 **[이미지]** [Client(Web/App/Bot)] <--> [HTTP Request/Response] <--> [FastAPI Server] <--> [Bedrock Agent Runtime] <--> [Knowledge Base + Claude]로 이어지는 API 서버 아키텍처 다이어그램. 클라이언트와 백엔드가 분리된 구조를 강조

이렇게 작성하면, 외부에서 `/chat` 주소로 질문을 POST 요청으로 던졌을 때 JSON 형태로 답변과 출처를 반환하는 독립적인 API 서버가 완성됩니다. `/health` 엔드포인트는 서버 상태를 확인하는 용도로, 이후 ECS나 ALB에서 헬스체크에 활용됩니다.

다음으로 `requirements.txt` 파일도 함께 생성합니다.

```text
fastapi==0.115.0
uvicorn==0.30.0
boto3==1.35.0
pydantic==2.9.0
```

> **📌 참고:** boto3 버전은 Bedrock의 최신 기능을 지원하는 버전을 사용해야 합니다. 특히 Knowledge Bases의 `retrieve_and_generate` API는 비교적 최근에 추가된 기능이므로, boto3 버전이 너무 낮으면 해당 API를 사용할 수 없습니다.



---

## 13-2 Dockerfile 작성과 ECR 푸시

코드를 작성했으니 이제 이 코드를 실행하는 데 필요한 모든 것(파이썬, 라이브러리, 설정 파일)을 하나의 '이미지(Image)'로 구워낼 차례입니다. 이 레시피를 적어놓은 파일이 바로 **Dockerfile**이고, 구워진 이미지를 보관하는 AWS의 저장소가 바로 **Amazon ECR**입니다.



### 13-2-1 Dockerfile 핵심 명령어

프로젝트 최상위 폴더에 확장자 없는 `Dockerfile`이라는 파일을 만들고, 다음과 같이 내용을 채워 넣습니다. 각 줄이 어떤 의미인지 이해하는 것이 중요합니다.

```dockerfile
# 1. 베이스 이미지 선택: Python 3.10이 설치된 가벼운 리눅스(Slim) 사용
FROM python:3.10-slim

# 2. 작업 디렉토리 설정: 컨테이너 내부의 작업 공간을 /app으로 지정
WORKDIR /app

# 3. 의존성 파일 복사 및 설치
#    (캐시 효율을 위해 소스코드보다 먼저 복사합니다)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 4. 소스 코드 전체 복사
COPY . .

# 5. 환경 변수 설정
#    파이썬 출력이 버퍼링 없이 즉시 로그에 찍히도록 설정
ENV PYTHONUNBUFFERED=1

# 6. 포트 개방 알림 (문서화 목적, 실제 포트 매핑은 docker run에서 수행)
EXPOSE 8080

# 7. 컨테이너 실행 시 작동할 명령어 (Uvicorn으로 FastAPI 실행)
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

각 명령어의 역할을 정리하면 다음과 같습니다.

| 명령어 | 역할 | 비유 |
|--------|------|------|
| `FROM` | 기반이 되는 OS+런타임 선택 | 요리의 밑재료 |
| `WORKDIR` | 컨테이너 내부 작업 폴더 지정 | 도마 위에 재료 올리기 |
| `COPY` | 호스트 파일을 컨테이너로 복사 | 재료 담기 |
| `RUN` | 빌드 시점에 명령어 실행 | 재료 손질 |
| `ENV` | 환경 변수 설정 | 양념 준비 |
| `EXPOSE` | 외부 통신 포트 선언 | 서빙 창구 표시 |
| `CMD` | 컨테이너 시작 시 실행할 명령 | 불 켜기 |

또한, 불필요한 파일이 이미지에 포함되지 않도록 `.dockerignore` 파일도 함께 만들어 줍니다.

```text
__pycache__
*.pyc
.env
.git
.vscode
*.md
```

> **💡 핵심 포인트:** `requirements.txt`를 소스 코드보다 먼저 복사하는 이유는 **도커의 레이어 캐싱** 때문입니다. 라이브러리 목록이 바뀌지 않았다면 `pip install` 단계를 건너뛰어 빌드 속도가 크게 빨라집니다. 소스 코드만 바꿨을 때 전체 라이브러리를 다시 설치하지 않아도 되는 것이죠.

### 13-2-2 Amazon ECR 이미지 빌드 및 푸시

레시피(Dockerfile)가 준비되었으니, 이미지를 빌드하고 AWS의 컨테이너 이미지 저장소인 **Amazon ECR**에 업로드하겠습니다. ECR은 Docker Hub와 비슷한 역할을 하지만, AWS IAM과 통합되어 접근 제어가 용이하고 ECS/Fargate/Lambda와의 연동이 매끄럽습니다.

**1단계: 도커 이미지 빌드**

```bash
# 프로젝트 폴더에서 이미지 빌드
docker build -t bedrock-rag-server:v1 .
```

이 명령어를 실행하면 도커는 베이스 이미지를 다운로드하고, 라이브러리를 설치하는 과정을 거쳐 `bedrock-rag-server:v1`이라는 이름의 이미지를 생성합니다.

**2단계: ECR 리포지토리 생성**

```bash
# ECR에 리포지토리 생성
aws ecr create-repository \
    --repository-name bedrock-rag-server \
    --region us-east-1
```

실행 결과로 리포지토리 URI가 반환됩니다. 이 URI는 `123456789012.dkr.ecr.us-east-1.amazonaws.com/bedrock-rag-server` 와 같은 형태입니다.

**3단계: ECR 로그인**

```bash
# ECR에 도커 인증 (로그인)
aws ecr get-login-password --region us-east-1 | \
    docker login --username AWS --password-stdin \
    123456789012.dkr.ecr.us-east-1.amazonaws.com
```

> **⚠️ 주의:** `123456789012` 부분은 여러분의 AWS 계정 ID로 교체해야 합니다. 계정 ID는 `aws sts get-caller-identity --query Account --output text` 명령어로 확인할 수 있습니다.

**4단계: 이미지 태깅 및 푸시**

```bash
# 로컬 이미지에 ECR 리포지토리 주소 태그 부여
docker tag bedrock-rag-server:v1 \
    123456789012.dkr.ecr.us-east-1.amazonaws.com/bedrock-rag-server:v1

# ECR로 이미지 푸시
docker push \
    123456789012.dkr.ecr.us-east-1.amazonaws.com/bedrock-rag-server:v1
```

푸시가 완료되면 AWS 콘솔의 ECR 서비스에서 업로드된 이미지를 확인할 수 있습니다. 이 이미지는 이후 Chapter 14에서 ECS/Fargate나 Lambda에 배포할 때 사용됩니다.

> 📷 **[이미지]** Dockerfile의 각 단계(FROM, COPY, RUN...)가 층층이 쌓여 하나의 Docker Image가 되고, 이 이미지가 Amazon ECR 리포지토리에 푸시되는 흐름을 시각화한 다이어그램. 로컬 빌드 -> 태깅 -> ECR 푸시의 3단계 과정을 보여주세요

> **📌 참고:** ECR의 이미지를 한 곳에서 관리하면 여러 서비스(ECS, Lambda, EC2 등)에서 동일한 이미지를 가져다 쓸 수 있습니다. 이미지 버전 관리를 위해 태그를 `v1`, `v2`처럼 명시적으로 지정하거나, Git 커밋 해시를 태그로 사용하는 것이 운영 환경에서의 모범 사례입니다.



---

## 13-3 환경 변수/비밀 키 관리 (AWS Secrets Manager)

컨테이너화할 때 가장 주의해야 할 점은 **보안**입니다. AWS 리전, Knowledge Base ID, API 키와 같은 민감한 정보는 절대로 Dockerfile이나 코드 안에 직접 적어 넣어서는 안 됩니다. 이미지가 유출되면 키도 함께 유출되기 때문입니다.



### 13-3-1 환경 변수 활용

대신 우리는 `os.environ.get`을 통해, 컨테이너가 실행되는 시점에 값을 주입받도록 설계해야 합니다. 앞서 작성한 `main.py`의 상단 코드를 다시 살펴봅시다.

```python
import os

# 코드에 직접 쓰는 대신 환경 변수에서 가져옴
AWS_REGION = os.environ.get("AWS_REGION", "us-east-1")
KNOWLEDGE_BASE_ID = os.environ.get("KNOWLEDGE_BASE_ID")
MODEL_ID = os.environ.get("MODEL_ID", "anthropic.claude-3-5-sonnet-20241022-v2:0")
```

이렇게 하면 코드 자체에는 어떤 비밀 정보도 포함되지 않으며, 실행 환경에 따라 다른 값을 주입할 수 있습니다.

**로컬 개발 시 (.env 파일)**

개발 편의를 위해 `.env` 파일에 설정값을 저장해두고 `python-dotenv` 라이브러리로 읽어올 수 있습니다.

```text
# .env (로컬 개발용 - 절대 Git에 커밋하지 마세요!)
AWS_REGION=us-east-1
KNOWLEDGE_BASE_ID=ABCDEFGHIJ
MODEL_ID=anthropic.claude-3-5-sonnet-20241022-v2:0
```

```python
# main.py 상단에 추가 (로컬 개발 시에만 동작)
from dotenv import load_dotenv
load_dotenv()  # .env 파일이 있으면 환경 변수로 로드
```

> **⚠️ 주의:** `.env` 파일은 반드시 `.gitignore`와 `.dockerignore`에 추가하여 Git 저장소와 Docker 이미지에 포함되지 않도록 해야 합니다. 이것은 아무리 강조해도 지나치지 않는 보안 원칙입니다.

**AWS 자격 증명과 IAM 역할**

로컬 개발 환경에서는 `AWS_ACCESS_KEY_ID`와 `AWS_SECRET_ACCESS_KEY` 환경 변수나 `~/.aws/credentials` 파일을 통해 인증합니다. 하지만 ECS/Fargate에 배포할 때는 **IAM 역할(Task Role)**을 컨테이너에 부여하는 것이 모범 사례입니다. IAM 역할을 사용하면 자격 증명 키를 코드나 환경 변수에 넣을 필요가 전혀 없습니다.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "bedrock:InvokeModel",
                "bedrock:InvokeModelWithResponseStream",
                "bedrock:Retrieve",
                "bedrock:RetrieveAndGenerate"
            ],
            "Resource": "*"
        }
    ]
}
```

위 IAM 정책을 ECS Task Role에 부여하면, 컨테이너 내부의 boto3가 자동으로 이 역할의 자격 증명을 사용합니다.

> **💡 핵심 포인트:** AWS 환경에서 컨테이너를 운영할 때는 키를 직접 주입하는 것보다 **IAM 역할**을 활용하는 것이 훨씬 안전합니다. boto3는 자동으로 ECS Task Role의 임시 자격 증명을 가져와 사용하므로, 코드를 수정할 필요가 없습니다.

### 13-3-2 AWS Secrets Manager 연계

API 키, 데이터베이스 비밀번호 등 IAM 역할로 대체할 수 없는 민감한 정보는 **AWS Secrets Manager**에 저장하는 것이 정석입니다. Secrets Manager는 비밀값을 암호화하여 저장하고, 자동 교체(Rotation)까지 지원하는 보안 금고 서비스입니다.

**1단계: 시크릿 생성 (AWS CLI)**

```bash
aws secretsmanager create-secret \
    --name bedrock-rag/api-config \
    --description "Bedrock RAG API configuration" \
    --secret-string '{"KNOWLEDGE_BASE_ID":"ABCDEFGHIJ","EXTERNAL_API_KEY":"sk-xxxx"}' \
    --region us-east-1
```

**2단계: 코드에서 시크릿 조회**

```python
import json
import boto3

def get_secret(secret_name: str, region: str = "us-east-1") -> dict:
    """AWS Secrets Manager에서 비밀값을 가져오는 함수"""
    client = boto3.client("secretsmanager", region_name=region)
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response["SecretString"])

# 서버 시작 시 시크릿 로드
secrets = get_secret("bedrock-rag/api-config")
KNOWLEDGE_BASE_ID = secrets["KNOWLEDGE_BASE_ID"]
```

**3단계: IAM 정책에 Secrets Manager 접근 권한 추가**

```json
{
    "Effect": "Allow",
    "Action": [
        "secretsmanager:GetSecretValue"
    ],
    "Resource": "arn:aws:secretsmanager:us-east-1:123456789012:secret:bedrock-rag/*"
}
```

> 📷 **[이미지]** 개발자 PC(.env 파일), Docker 컨테이너(환경 변수 + IAM 역할), AWS 클라우드(Secrets Manager + IAM Task Role) 세 가지 환경에서 비밀 정보가 어떻게 주입되는지 보여주는 흐름도. 코드는 하나지만 환경에 따라 자격 증명의 출처가 달라지는 모습을 보여주세요

> **📌 참고:** ECS 서비스 정의에서 Secrets Manager의 값을 컨테이너 환경 변수로 직접 매핑할 수도 있습니다. 이 방법을 사용하면 코드에서 Secrets Manager를 직접 호출하지 않아도 되어 코드가 더 단순해집니다. 이에 대해서는 Chapter 14에서 ECS 배포 시 자세히 다루겠습니다.



---

## 13-4 로컬 Docker 실행으로 엔드투엔드 테스트

이미지를 빌드했다면, ECR에 푸시하거나 클라우드에 올리기 전에 내 PC에서 먼저 실행해 봐야 합니다. 이를 통해 패키지 누락이나 경로 설정 오류, 환경 변수 미설정 등의 문제를 미리 잡아낼 수 있습니다.



### 13-4-1 도커 컨테이너 실행

다음 명령어로 방금 만든 이미지를 실행합니다. 로컬 환경에서는 `-e` 옵션으로 환경 변수를 직접 주입하고, AWS 자격 증명은 호스트의 `~/.aws` 디렉토리를 마운트하여 전달합니다.

```bash
docker run -p 8080:8080 \
    -e AWS_REGION="us-east-1" \
    -e KNOWLEDGE_BASE_ID="여러분의_KB_ID" \
    -e MODEL_ID="anthropic.claude-3-5-sonnet-20241022-v2:0" \
    -v ~/.aws:/root/.aws:ro \
    bedrock-rag-server:v1
```

**옵션 설명:**

| 옵션 | 설명 |
|------|------|
| `-p 8080:8080` | 내 PC의 8080번 포트를 컨테이너의 8080번 포트에 연결 |
| `-e KEY=VALUE` | 컨테이너에 환경 변수 주입 |
| `-v ~/.aws:/root/.aws:ro` | 호스트의 AWS 자격 증명을 읽기 전용으로 마운트 |

서버가 정상적으로 실행되면 터미널에 다음과 같은 로그가 출력됩니다.

```
INFO:     Started server process [1]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8080 (Press CTRL+C to quit)
```

> **⚠️ 주의:** `-v ~/.aws:/root/.aws:ro` 옵션은 로컬 테스트 전용입니다. 실제 배포 환경에서는 절대 자격 증명 파일을 마운트하지 않고, IAM 역할을 사용해야 합니다. `:ro` 플래그는 읽기 전용(read-only)을 의미하여 컨테이너가 자격 증명을 수정하는 것을 방지합니다.

### 13-4-2 API 테스트 (curl, Postman)

컨테이너가 정상적으로 실행되었다면, 새로운 터미널을 열어 API 요청을 보내봅니다.

**헬스체크 테스트:**

```bash
curl http://localhost:8080/health
```

정상 응답:
```json
{"status": "healthy", "region": "us-east-1"}
```

**채팅 API 테스트:**

```bash
curl -X POST "http://localhost:8080/chat" \
    -H "Content-Type: application/json" \
    -d '{"question": "Amazon Bedrock의 Knowledge Base란 무엇인가요?"}'
```

정상 응답 예시:
```json
{
    "question": "Amazon Bedrock의 Knowledge Base란 무엇인가요?",
    "answer": "Amazon Bedrock Knowledge Base는 사용자의 데이터 소스를 연결하여 RAG(검색 증강 생성) 파이프라인을 구성할 수 있는 관리형 서비스입니다...",
    "sources": [
        "s3://my-bucket/docs/bedrock-guide.pdf"
    ]
}
```

**Swagger UI로 테스트:**

브라우저에서 `http://localhost:8080/docs`에 접속하면 FastAPI가 자동 생성한 대화형 API 문서를 확인할 수 있습니다. 여기서 직접 요청을 보내고 응답을 확인할 수 있어 curl보다 편리합니다.

> 📷 **[이미지]** 터미널 두 개를 띄워놓은 화면 캡처. 왼쪽 터미널은 docker run 명령어로 서버가 실행되어 Uvicorn 로그가 올라가는 모습, 오른쪽 터미널은 curl 명령어로 /chat 요청을 보내고 JSON 응답을 받은 모습을 보여주세요

**에러가 발생했다면?**

컨테이너 실행 중 에러가 발생하면 다음 명령어로 디버깅합니다.

```bash
# 실행 중인 컨테이너 로그 확인
docker logs $(docker ps -q --filter ancestor=bedrock-rag-server:v1)

# 컨테이너 내부에 직접 접속하여 디버깅
docker exec -it $(docker ps -q --filter ancestor=bedrock-rag-server:v1) /bin/bash
```

자주 발생하는 에러와 해결 방법을 정리하면 다음과 같습니다.

| 에러 메시지 | 원인 | 해결 방법 |
|-------------|------|-----------|
| `ModuleNotFoundError` | requirements.txt에 패키지 누락 | 패키지 추가 후 이미지 재빌드 |
| `NoCredentialsError` | AWS 자격 증명 미설정 | `-v ~/.aws:/root/.aws:ro` 마운트 확인 |
| `AccessDeniedException` | IAM 권한 부족 | Bedrock 관련 권한 정책 추가 |
| `ResourceNotFoundException` | Knowledge Base ID 오류 | 환경 변수의 KB ID 값 확인 |

성공적으로 JSON 응답이 온다면, 여러분의 백엔드 서버는 이제 배포 준비를 완벽하게 마친 것입니다.

> **💡 핵심 포인트:** 로컬에서 Docker로 테스트하는 단계를 절대 건너뛰지 마세요. 클라우드에 배포한 뒤 에러를 찾는 것보다 로컬에서 미리 잡는 것이 시간과 비용 모두에서 훨씬 효율적입니다. 특히 환경 변수 누락이나 패키지 의존성 문제는 로컬 테스트에서 대부분 발견됩니다.



---

## 마치며

이번 Chapter 13에서 우리는 다음과 같은 과정을 거쳤습니다.

1. **FastAPI + Bedrock Runtime**으로 독립적인 API 서버를 설계했습니다.
2. **Dockerfile**을 작성하고, **Amazon ECR**에 이미지를 빌드하여 푸시했습니다.
3. **환경 변수**, **IAM 역할**, **AWS Secrets Manager**를 활용한 안전한 비밀 관리 전략을 수립했습니다.
4. **로컬 Docker 실행**으로 엔드투엔드 테스트를 완료했습니다.

이제 여러분의 애플리케이션은 "내 컴퓨터에서만 되는" 코드가 아니라, 어떤 환경에서든 동일하게 실행 가능한 컨테이너 이미지가 되었습니다. ECR에 안전하게 저장된 이 이미지는 AWS의 다양한 컴퓨팅 서비스에 배포할 준비를 마쳤습니다.

다음 Chapter 14에서는 이 Docker 이미지를 **AWS Lambda**로 서버리스 배포하는 방법과, **ECS/Fargate**로 컨테이너를 본격적으로 운영하는 방법을 다룹니다. API Gateway와 ALB를 연동하여 실제 인터넷 주소로 접속할 수 있는 라이브 서비스를 런칭해 보겠습니다.
