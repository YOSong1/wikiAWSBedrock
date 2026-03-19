Chapter 13에서 우리는 애플리케이션을 도커 이미지라는 형태로 단단하게 포장했습니다. 이제 이 포장된 박스를 배송하여 실제로 서비스를 구동할 차례입니다. 과거에는 이를 위해 가상 머신(VM)을 빌리고, OS를 설치하고, 네트워크를 설정하는 등 복잡한 서버 관리 작업이 필요했습니다.

하지만 클라우드 기술의 발전으로 이제는 **서버리스(Serverless)** 환경이 대세가 되었습니다. 서버리스란 '서버가 없다'는 뜻이 아니라, '서버 관리를 내가 할 필요가 없다'는 뜻입니다. 구글 클라우드가 서버의 사양, 보안 패치, 오토 스케일링을 모두 알아서 관리해주기 때문입니다.

이번 챕터에서는 구글 클라우드의 대표적인 서버리스 컴퓨팅 서비스인 **Cloud Run**과 **Cloud Functions**의 차이를 이해하고, 우리가 만든 RAG 챗봇을 실제 라이브 환경에 배포하여 누구나 접속할 수 있는 서비스로 런칭해 보겠습니다.



---

## 14-1 Cloud Run 개념, 트래픽·스케일링 방식 이해

**Cloud Run**은 "컨테이너를 위한 완전 관리형 컴퓨팅 플랫폼"입니다. 쉽게 말해, 우리가 만든 도커 컨테이너만 던져주면 알아서 실행해주고 인터넷 주소까지 만들어주는 서비스입니다.

### 14-1-1 스케일 투 제로 (Scale to Zero)

**Cloud Run**의 가장 강력한 특징은 바로 **'0으로의 축소(Scale to Zero)'**입니다. 일반적인 서버는 사용자가 없어도 24시간 켜져 있어 비용이 발생합니다. 하지만 **Cloud Run**은 요청이 들어올 때만 순식간에 서버를 켜서 응답하고, 사용자가 없으면 자동으로 서버를 꺼버립니다(인스턴스 개수: 0). 따라서 사용자가 없는 새벽 시간대에는 비용이 0원입니다. 이는 비용 효율성을 극대화해야 하는 스타트업이나 사내 프로젝트에 최적의 기능입니다.



### 14-1-2 오토 스케일링 (Auto-scaling)

반대로 사용자가 갑자기 폭주하면 어떻게 될까요? Cloud Run은 트래픽 양에 맞춰 컨테이너의 개수를 자동으로 늘립니다(Scale Out). 수천 명이 동시에 접속해도 개발자가 서버를 추가할 필요 없이, 구글이 알아서 수백 개의 컨테이너를 띄워 트래픽을 처리해 줍니다.

### 14-1-3 트래픽 분할 (Traffic Splitting)

새로운 버전의 AI 모델을 배포했을 때, 혹시라도 문제가 생길까 봐 걱정될 수 있습니다. **Cloud Run**은 트래픽 분할 기능을 제공합니다.

- **기존 버전(v1):** 사용자 트래픽의 90% 처리
- **신규 버전(v2):** 사용자 트래픽의 10%만 처리

이렇게 설정하면 신규 버전을 소수의 사용자에게만 먼저 테스트(Canary 배포)해보고, 안전하다고 판단되면 점진적으로 100%까지 늘려나가는 무중단 배포 전략을 손쉽게 구사할 수 있습니다.

> 📷 **[이미지]** 요청량(Request) 그래프에 따라 컨테이너 인스턴스 개수가 0개에서 N개로 늘어났다가, 다시 0개로 줄어드는 '오토 스케일링' 그래프. 특히 요청이 없을 때 인스턴스가 아예 사라지는(Zero) 구간을 강조



---

## 14-2 Cloud Run에 컨테이너 배포

로컬에 있는 도커 이미지를 **Cloud Run**이 가져가게 하려면, 먼저 구글 클라우드 상의 저장소인 **Artifact Registry**에 이미지를 업로드(Push)해야 합니다. 이를 '컨테이너 레지스트리'라고 부릅니다.

### 14-2-1 Artifact Registry 저장소 생성

콘솔에서 **Artifact Registry** 메뉴로 이동하여 도커 이미지를 보관할 '창고(Repository)'를 만듭니다.



```bash
# gcloud 명령어로 저장소 생성 예시
gcloud artifacts repositories create my-repo \
--repository-format=docker \
--location=us-central1
```

### 14-2-2 이미지 태깅 및 푸시 (Push)

이제 로컬에서 빌드한 이미지에 "이것은 구글 저장소로 갈 물건입니다"라는 꼬리표(Tag)를 붙이고 업로드합니다.

```bash
# 1. 태그 붙이기 (주소 형식: 리전-docker.pkg.dev/프로젝트ID/저장소명/이미지명:태그)
docker tag vertex-rag-server:v1 \
us-central1-docker.pkg.dev/my-project/my-repo/vertex-rag-server:v1

# 2. 업로드 (Push)
docker push us-central1-docker.pkg.dev/my-project/my-repo/vertex-rag-server:v1
```

이 과정이 끝나면 로컬에 있던 이미지가 구글 클라우드 데이터센터로 전송됩니다.

### 14-2-3 Cloud Run 서비스 배포

저장소에 이미지가 올라갔으니, 이제 **Cloud Run**에게 "이 이미지를 가져다가 서버로 띄워줘"라고 명령할 차례입니다.



```bash
gcloud run deploy vertex-rag-service \
--image us-central1-docker.pkg.dev/my-project/my-repo/vertex-rag-server:v1 \
--region us-central1 \
--allow-unauthenticated
```

`--allow-unauthenticated` 옵션은 누구나 로그인 없이 접속할 수 있게 한다는 뜻으로, 웹 서비스를 만들 때 필수적입니다. 명령어가 성공하면 `https://...run.app` 형태의 접속 URL이 출력됩니다.

> 📷 **[이미지]** [Local PC] -> (docker push) -> [Artifact Registry] -> (deploy) -> [Cloud Run]으로 이어지는 배포 파이프라인 흐름도. 이미지가 중간 저장소를 거쳐 최종 실행 환경으로 이동하는 경로



---

## 14-3 Cloud Functions로 경량 Webhook/후처리 작업 구성

**Cloud Run**이 '웹 서버'나 '메인 애플리케이션'을 위한 것이라면, **Cloud Functions**는 좀 더 가볍고 이벤트 중심(Event-driven)적인 작업을 위해 존재합니다. "어떤 사건이 터지면, 이 함수 하나만 딱 실행해!"라는 식입니다.

### 14-3-1 Cloud Functions vs Cloud Run

- **Cloud Run:** HTTP 요청을 받아 처리하는 웹 애플리케이션, 복잡한 로직, 장시간 실행되는 서버.
  - 예: Streamlit 앱, FastAPI 서버

- **Cloud Functions:** 특정 이벤트에 반응하는 단발성 코드 조각.
  - 예: 파일이 업로드되면 알림 보내기, DB가 변경되면 로그 남기기

### 14-3-2 활용 사례: RAG 파이프라인 자동화

RAG 시스템에서 **Cloud Functions**는 '데이터 파이프라인의 자동화'에 매우 유용하게 쓰입니다. 예를 들어, 관리자가 구글 클라우드 스토리지(GCS)의 특정 폴더에 새로운 PDF 문서를 업로드했다고 가정해 봅시다.

1. **이벤트 발생:** GCS에 파일 업로드 감지.
2. **트리거(Trigger):** **Cloud Functions**가 자동으로 깨어남.
3. **함수 실행:** 업로드된 PDF를 읽어서 텍스트를 추출하고(Parsing), 임베딩하여 벡터 DB(Chroma/Vertex Search)에 업데이트.
4. **종료:** 작업 완료 후 자동 소멸.

이렇게 구성하면 사람이 일일이 업데이트 코드를 돌릴 필요 없이, 파일만 올리면 자동으로 최신 지식이 반영되는 똑똑한 시스템을 만들 수 있습니다.

> 📷 **[이미지]** [User uploads PDF] -> [Cloud Storage Bucket] -> (Trigger Event) -> [Cloud Function executes code] -> [Update Vector DB]로 이어지는 이벤트 기반 아키텍처 다이어그램



---

## 14-4 웹 UI기반 RAG API 배포

이제 이론을 마쳤으니, 우리가 만든 결과물을 실제 서비스로 런칭하는 실습을 진행합니다.

### 14-4-1 Streamlit 앱 배포하기 (Cloud Run)

앞서 Chapter 12에서 만든 `app.py`와 `Dockerfile`을 사용하여 배포합니다. **Streamlit**은 웹 서버 형태로 계속 떠 있어야 하므로 **Cloud Run**이 적합합니다.

**① Google Cloud Build 사용:** 로컬에서 `docker push` 하는 대신, 구글의 빌드 서버를 이용하면 더 빠르고 간편합니다.



```bash
gcloud builds submit \
--tag us-central1-docker.pkg.dev/my-project/my-repo/streamlit-ui:v1 .
```

**② Cloud Run 배포:** 콘솔 화면(GUI)을 이용해 배포해 봅니다.

- Cloud Run 메뉴 > '서비스 만들기' 클릭
- '기존 컨테이너 이미지에서 배포' 선택 후 방금 빌드한 이미지 선택
- '인증 허용(모든 트래픽 허용)' 체크
- '만들기' 클릭

잠시 후 초록색 체크 표시와 함께 서비스 URL이 생성됩니다. 스마트폰이나 동료의 PC에서 접속해 보세요. 여러분이 만든 AI 비서가 클라우드 상에서 완벽하게 동작하는 것을 볼 수 있습니다.

### 14-4-2 간단한 후처리 함수 배포하기 (Cloud Functions)

간단한 실습으로 "사용자가 질문을 할 때마다 슬랙(Slack)으로 알림을 보내는" 함수를 만들어볼 수 있습니다. **Cloud Functions** 콘솔에서 Python 3.10 런타임을 선택하고, HTTP 트리거를 설정한 뒤 간단한 로깅 코드를 작성하여 배포합니다. 그리고 RAG 백엔드에서 이 함수 URL을 호출하도록 연결하면, 모니터링 시스템까지 갖춘 완벽한 아키텍처가 됩니다.

> 📷 **[이미지]** 배포가 완료된 Cloud Run 대시보드 화면. 서비스 이름 옆에 초록색 '정상(Healthy)' 표시가 있고, 상단에 접속 가능한 URL 링크가 활성화된 실제 콘솔 화면



---

## 마무리



이것으로 Chapter 14를 마칩니다. 여러분은 이제 로컬 개발 환경을 넘어, 구글의 강력한 인프라 위에서 서비스를 운영할 수 있는 클라우드 엔지니어가 되었습니다.

우리의 서비스는 이제 전 세계 어디서든 접속 가능합니다. 하지만 "배포했다"고 끝난 것은 아닙니다. 사용자가 몰렸을 때 에러는 없는지, 비용은 얼마나 나오고 있는지 지켜봐야 합니다. 마지막 Chapter 15에서는 안정적인 서비스를 유지하기 위한 운영 체크리스트와 프로젝트를 성공적으로 마무리하는 방법에 대해 알아보겠습니다.

