지금까지 우리는 터미널의 검은 화면(Console)에서 텍스트로만 Bedrock과 소통해 왔습니다. Chapter 11에서 Agent의 ReAct 루프, Action Group을 통한 Lambda 연동, Guardrails의 콘텐츠 필터와 PII 보호까지 다루면서, 우리의 AI 서비스는 이미 상당한 기능을 갖추었습니다. 하지만 일반 사용자나 클라이언트에게 "여기 터미널에서 boto3 코드를 실행하세요"라고 할 수는 없습니다. 훌륭한 AI 모델도 사용하기 불편하면 외면받기 마련입니다.

따라서 우리가 만든 RAG 시스템과 Bedrock API를 누구나 쉽게 쓸 수 있도록 그래픽 사용자 인터페이스(GUI), 그중에서도 접근성이 가장 좋은 웹(Web) UI로 감싸는 작업이 필요합니다.

과거에는 웹을 만들려면 HTML, CSS, JavaScript는 물론 React나 Vue 같은 프론트엔드 프레임워크까지 배워야 했습니다. 하지만 AI 시대에는 Python 개발자를 위한 혁신적인 도구들이 등장했습니다. 이번 챕터에서는 그중 가장 대표적인 **Streamlit**(스트림릿)을 사용하여, 단 몇 줄의 파이썬 코드만으로 Amazon Bedrock 기반 챗봇 웹 서비스를 구축하는 방법을 상세히 알아보겠습니다.

---

## 12-1 Streamlit/Gradio 기반 빠른 프로토타입 UI

본격적인 개발에 앞서, 파이썬 기반의 양대 산맥인 UI 라이브러리, Streamlit과 Gradio를 비교해 보고 우리가 사용할 도구를 선택해 보겠습니다.

### 12-1-1 Gradio: 모델 시연에 최적화된 도구

**Gradio**는 주로 "내 모델이 이렇게 작동해!"라고 보여주는 데모 용도로 많이 쓰입니다. 입력창(Input)과 출력창(Output)을 정의하면 자동으로 UI를 만들어줍니다. 아래는 Bedrock의 Converse API를 Gradio로 감싼 최소 예시입니다.

```python
import gradio as gr
import boto3, json

bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")

def ask_bedrock(question):
    response = bedrock.converse(
        modelId="anthropic.claude-sonnet-4-20250514",
        messages=[{"role": "user", "content": [{"text": question}]}],
        inferenceConfig={"maxTokens": 512, "temperature": 0.7},
    )
    return response["output"]["message"]["content"][0]["text"]

demo = gr.Interface(fn=ask_bedrock, inputs="text", outputs="text",
                    title="Bedrock Claude Demo")
demo.launch()
```

몇 줄만으로 웹 UI가 완성됩니다. 입력창에 질문을 넣고 Submit을 누르면 Claude의 답변이 출력창에 나타납니다.

> **💡 핵심 포인트:**
> - 설정이 매우 간편하고, Hugging Face Spaces와 같은 플랫폼에 배포하기 쉽습니다.
> - 모델의 입력-출력을 빠르게 시연해야 할 때 최적의 선택입니다.

> **⚠️ 주의:**
> - 화면 레이아웃을 마음대로 뜯어고치거나, 복잡한 멀티턴 대화 흐름을 제어하기에는 유연성이 부족합니다.
> - 사이드바, 2단 분할, 세션 상태 등 RAG 챗봇에 필요한 고급 UI 구성이 제한적입니다.

### 12-1-2 Streamlit: 대시보드와 챗봇 인터페이스

**Streamlit**은 데이터 과학자들을 위해 태어난 도구입니다. "데이터 스크립트를 정보 공유 가능한 웹 앱으로 바꾼다"는 철학을 가지고 있으며, 챗봇 전용 컴포넌트(`st.chat_message`, `st.chat_input`)까지 제공합니다.

```python
import streamlit as st

st.set_page_config(page_title="Bedrock 챗봇", page_icon="🤖")
st.title("Amazon Bedrock 문서 비서")

if prompt := st.chat_input("질문을 입력하세요"):
    with st.chat_message("user"):
        st.markdown(prompt)
    with st.chat_message("assistant"):
        st.markdown("여기에 Bedrock 응답이 표시됩니다.")
```

이것만으로도 ChatGPT와 유사한 대화형 인터페이스가 만들어집니다.

> **💡 핵심 포인트:**
> - 위젯(버튼, 슬라이더 등) 배치가 자유롭고, **세션 상태(Session State)** 관리가 강력하여 멀티턴 챗봇 구현에 유리합니다.
> - `st.sidebar`, `st.columns`, `st.expander` 등으로 레이아웃을 유연하게 구성할 수 있습니다.

> **📌 참고:**
> 우리는 Bedrock Knowledge Base와 연동하는 RAG 챗봇을 만들 것이므로, 대화 내역(History) 관리와 레이아웃 커스터마이징이 더 강력한 **Streamlit**을 사용하여 실습을 진행하겠습니다.

아래 표로 두 도구를 간단히 비교합니다.

| 항목 | Gradio | Streamlit |
|------|--------|-----------|
| **주요 용도** | 모델 데모, 빠른 시연 | 대시보드, 챗봇, 데이터 앱 |
| **레이아웃 자유도** | 제한적 | 높음 (sidebar, columns, tabs) |
| **세션 상태 관리** | 기본적 | 강력 (`st.session_state`) |
| **채팅 전용 컴포넌트** | `gr.ChatInterface` | `st.chat_message`, `st.chat_input` |
| **배포 편의성** | HF Spaces 최적 | Streamlit Cloud, Docker, EC2 등 |
| **AWS 연동** | boto3 직접 호출 | boto3 직접 호출 + 캐싱 지원 |

> 📷 **[이미지]** 좌측에는 Gradio로 만든 단순한 입출력 화면(Input-Output 2단 구조), 우측에는 Streamlit으로 만든 사이드바가 있는 챗봇 화면을 나란히 배치하여 비교하는 이미지. Streamlit 쪽이 좀 더 '완성된 웹 서비스' 느낌이 난다는 것을 시각적으로 보여줌

---

## 12-2 입력-결과-근거를 보여주는 레이아웃 설계

RAG 애플리케이션의 UI 설계에서 가장 중요한 원칙은 **투명성**입니다. 사용자는 AI의 답변뿐만 아니라, 그 답변이 "어디서 나왔는지(근거)"를 눈으로 확인하고 싶어 합니다. 특히 기업 환경에서는 "이 답변의 출처가 뭐지?"라는 질문이 반드시 따라옵니다. Bedrock Knowledge Base의 `RetrieveAndGenerate` API는 이런 요구를 충족하기 위해 응답에 Citation(출처 인용) 정보를 포함합니다. UI에서 이를 잘 보여주는 것이 핵심입니다.

### 12-2-1 2단 분할 레이아웃 (질문 + 출처)

가장 권장되는 방식은 화면을 좌우 또는 메인-사이드바로 나누는 것입니다.

- **메인 영역 (Main):** AI와의 채팅 창입니다. 사용자가 질문하고 답변을 받는 공간입니다.
- **사이드바 (Sidebar):** '참고 문서(Context)'를 보여주는 공간입니다. AI가 답변할 때 참고한 문서 청크(Chunk)와 출처 정보를 실시간으로 띄워줍니다.

Streamlit에서 이 구조를 구현하는 것은 놀라울 정도로 간단합니다.

```python
import streamlit as st

# 사이드바 영역: 검색된 근거 문서를 표시
with st.sidebar:
    st.header("📚 참고 문서")
    if "sources" in st.session_state and st.session_state.sources:
        for i, source in enumerate(st.session_state.sources):
            with st.expander(f"출처 {i+1}: {source['title']}", expanded=False):
                st.markdown(f"**S3 위치:** `{source['uri']}`")
                st.markdown(f"**관련 내용:**\n{source['content'][:300]}...")
    else:
        st.info("질문을 입력하면 참고 문서가 여기에 표시됩니다.")

# 메인 영역: 채팅 인터페이스
st.title("🔍 사내 문서 Q&A 어시스턴트")
```

이렇게 구성하면 사용자는 AI의 답변을 읽으면서, 동시에 왼쪽 사이드바에서 원문과 비교하며 팩트 체크를 할 수 있어 서비스 신뢰도가 크게 올라갑니다.

> **💡 핵심 포인트:**
> - Bedrock의 `RetrieveAndGenerate` 응답에는 `citations` 필드가 포함되어 있어, 각 답변 문장이 어떤 문서에서 나왔는지 추적할 수 있습니다.
> - 사이드바는 화면 크기와 무관하게 항상 보이므로 근거 확인 UX에 유리합니다.

### 12-2-2 RAG 컨텍스트와 메타데이터 시각화

화면 공간이 좁거나 모바일 환경을 고려한다면, 답변 바로 아래에 "참고 문서 보기"라는 **Expander**(접이식 패널)를 두는 방식도 좋습니다. Bedrock이 반환하는 메타데이터를 파싱하여 출처 URI, 관련 점수(Score), 문서 내용을 구조화해서 보여줍니다.

```python
def display_citations(citations):
    """Bedrock RetrieveAndGenerate 응답의 citations을 파싱하여 표시"""
    if not citations:
        return

    with st.expander("📄 참고 문서 확인하기", expanded=False):
        for i, citation in enumerate(citations):
            # 각 citation에서 retrievedReferences 추출
            refs = citation.get("retrievedReferences", [])
            for j, ref in enumerate(refs):
                content = ref.get("content", {}).get("text", "내용 없음")
                s3_uri = ref.get("location", {}).get("s3Location", {}).get("uri", "알 수 없음")

                st.markdown(f"**출처 {i+1}-{j+1}**")
                st.info(f"📁 **위치:** {s3_uri}\n\n{content[:200]}...")
                st.divider()
```

> **📌 참고:**
> `RetrieveAndGenerate` API는 답변 텍스트와 함께 `citations` 배열을 반환합니다. 각 citation 안에는 `retrievedReferences`가 포함되어 있고, 여기에 원본 문서의 S3 URI, 텍스트 내용 등이 담겨 있습니다. 이 구조를 UI에서 그대로 활용하면 됩니다.

> 📷 **[이미지]** 와이어프레임(Wireframe) 스케치 이미지. 왼쪽 사이드바에 "참고 문서" 목록이 있고, 메인 영역에 채팅 버블들이 있으며, AI 답변 아래에 접이식 "참고 문서 확인하기" 패널이 펼쳐져 S3 URI와 문서 내용이 보이는 구조

---

## 12-3 고객 지원 챗봇 또는 사내 문서 Q&A 어시스턴트 UI 구현

이제 이론을 마쳤으니, Chapter 8~9에서 구성한 **Bedrock Knowledge Base**를 Streamlit UI와 연결하여 실제 동작하는 웹 서비스를 만들어 보겠습니다. 두 가지 접근 방식을 다룹니다: (1) 간편한 `RetrieveAndGenerate` API 원콜 방식, (2) `Retrieve` + `Converse` API를 분리하여 더 세밀하게 제어하는 방식입니다.

### 12-3-1 세션 상태 관리

**Streamlit**의 작동 방식은 독특합니다. 사용자가 버튼을 한 번 누르거나, 텍스트를 입력할 때마다 파이썬 코드가 **처음부터 끝까지 다시 실행(Rerun)** 됩니다. 따라서 아무런 조치를 하지 않으면, 사용자가 새 질문을 입력하는 순간 이전 대화 내용이 싹 사라져 버립니다.

이를 방지하기 위해 `st.session_state`라는 메모리 공간을 사용해야 합니다.

```python
# 대화 내역 초기화 (앱이 다시 실행되어도 유지)
if "messages" not in st.session_state:
    st.session_state.messages = []

# 참고 문서 목록 초기화
if "sources" not in st.session_state:
    st.session_state.sources = []

# Bedrock 세션 ID 초기화 (RetrieveAndGenerate의 멀티턴 대화용)
if "session_id" not in st.session_state:
    st.session_state.session_id = None
```

> **💡 핵심 포인트:**
> - `st.session_state`는 Streamlit이 코드를 다시 실행하더라도 "이 변수만큼은 지우지 말고 기억하라"는 의미입니다. 챗봇 구현의 핵심입니다.
> - Bedrock의 `RetrieveAndGenerate` API는 `sessionId`를 반환합니다. 이를 세션 상태에 저장해두면 다음 호출 때 전달하여 **멀티턴 대화**가 가능합니다. 이전 대화 맥락을 Bedrock이 기억하는 것입니다.

> 📷 **[이미지]** st.session_state가 작동하는 원리를 보여주는 다이어그램. [User Interaction] -> [Script Rerun] -> [Load from Session State] -> [UI Update]로 이어지는 순환 구조. 메모리(Session State)가 리셋을 막아준다는 개념, 그리고 session_id가 Bedrock 측 대화 맥락을 유지한다는 것이 표현됨

### 12-3-2 채팅 인터페이스와 Knowledge Base 연동

이제 전체 코드를 작성합니다. 파일 이름은 `app.py`로 생성합니다.

#### 라이브러리 설치

먼저 필요한 패키지를 설치합니다.

```bash
pip install streamlit boto3
```

#### 방법 1: RetrieveAndGenerate API (원콜 방식)

가장 간단한 접근입니다. Bedrock이 검색과 응답 생성을 한 번에 처리합니다.

```python
import streamlit as st
import boto3

# ── 페이지 설정 ──────────────────────────────────────────
st.set_page_config(page_title="사내 문서 Q&A", page_icon="📚", layout="wide")
st.title("📚 사내 문서 Q&A 어시스턴트")
st.caption("Amazon Bedrock Knowledge Base 기반 RAG 챗봇")

# ── Bedrock 클라이언트 초기화 (캐싱으로 재사용) ──────────
@st.cache_resource
def get_bedrock_client():
    return boto3.client("bedrock-agent-runtime", region_name="us-east-1")

client = get_bedrock_client()

# Knowledge Base ID (Ch8에서 생성한 값으로 교체)
KNOWLEDGE_BASE_ID = "YOUR_KNOWLEDGE_BASE_ID"
MODEL_ARN = "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-sonnet-4-20250514"

# ── 세션 상태 초기화 ─────────────────────────────────────
if "messages" not in st.session_state:
    st.session_state.messages = []
if "sources" not in st.session_state:
    st.session_state.sources = []
if "session_id" not in st.session_state:
    st.session_state.session_id = None

# ── 사이드바: 참고 문서 표시 ─────────────────────────────
with st.sidebar:
    st.header("📄 참고 문서")
    if st.button("대화 초기화"):
        st.session_state.messages = []
        st.session_state.sources = []
        st.session_state.session_id = None
        st.rerun()

    if st.session_state.sources:
        for i, src in enumerate(st.session_state.sources):
            with st.expander(f"출처 {i+1}", expanded=(i == 0)):
                st.markdown(f"**S3:** `{src['uri']}`")
                st.markdown(src["content"][:300] + "...")
    else:
        st.info("질문을 입력하면 참고 문서가 여기에 표시됩니다.")

# ── 이전 대화 내용 표시 ──────────────────────────────────
for msg in st.session_state.messages:
    with st.chat_message(msg["role"]):
        st.markdown(msg["content"])

# ── 사용자 입력 처리 ─────────────────────────────────────
if prompt := st.chat_input("궁금한 점을 물어보세요!"):
    # 사용자 메시지 표시 및 저장
    with st.chat_message("user"):
        st.markdown(prompt)
    st.session_state.messages.append({"role": "user", "content": prompt})

    # AI 답변 생성
    with st.chat_message("assistant"):
        with st.spinner("문서를 검색하고 답변을 생성하는 중..."):
            # RetrieveAndGenerate API 호출
            rag_params = {
                "input": {"text": prompt},
                "retrieveAndGenerateConfiguration": {
                    "type": "KNOWLEDGE_BASE",
                    "knowledgeBaseConfiguration": {
                        "knowledgeBaseId": KNOWLEDGE_BASE_ID,
                        "modelArn": MODEL_ARN,
                        "retrievalConfiguration": {
                            "vectorSearchConfiguration": {
                                "numberOfResults": 5
                            }
                        }
                    }
                }
            }

            # 멀티턴 대화를 위한 세션 ID 전달
            if st.session_state.session_id:
                rag_params["sessionId"] = st.session_state.session_id

            response = client.retrieve_and_generate(**rag_params)

            # 응답 텍스트 추출
            answer = response["output"]["text"]
            st.markdown(answer)

            # 세션 ID 저장 (다음 대화에서 맥락 유지)
            st.session_state.session_id = response.get("sessionId")

            # Citation(출처) 정보 파싱
            sources = []
            for citation in response.get("citations", []):
                for ref in citation.get("retrievedReferences", []):
                    content = ref.get("content", {}).get("text", "")
                    uri = ref.get("location", {}).get("s3Location", {}).get("uri", "알 수 없음")
                    sources.append({"uri": uri, "content": content})

            st.session_state.sources = sources

            # 답변 아래에 Expander로 출처 표시
            if sources:
                with st.expander("📄 참고 문서 확인하기"):
                    for i, src in enumerate(sources):
                        st.info(f"**출처 {i+1}**\n📁 {src['uri']}\n\n{src['content'][:200]}...")

    # AI 메시지 저장
    st.session_state.messages.append({"role": "assistant", "content": answer})
    st.rerun()  # 사이드바 업데이트를 위해 리런
```

> **💡 핵심 포인트:**
> - `@st.cache_resource`는 Bedrock 클라이언트처럼 무거운 리소스를 한 번만 초기화하고 재사용하게 만드는 데코레이터입니다. 매 Rerun마다 새 클라이언트를 만드는 낭비를 방지합니다.
> - `sessionId`를 저장하고 다음 호출에 전달하면, Bedrock이 이전 대화 맥락을 기억합니다. "아까 말한 문서에서 3번 항목만 자세히 설명해줘" 같은 후속 질문이 가능해집니다.

#### 방법 2: Retrieve + Converse API (분리 방식)

검색과 응답 생성을 분리하면 프롬프트를 직접 커스터마이징할 수 있습니다. 예를 들어 System Prompt를 세밀하게 조정하거나, 검색 결과를 필터링한 후 모델에 전달하는 등의 고급 제어가 가능합니다.

```python
import streamlit as st
import boto3

# ── 두 개의 클라이언트를 각각 초기화 ──────────────────────
@st.cache_resource
def get_clients():
    agent_client = boto3.client("bedrock-agent-runtime", region_name="us-east-1")
    runtime_client = boto3.client("bedrock-runtime", region_name="us-east-1")
    return agent_client, runtime_client

agent_client, runtime_client = get_clients()

KNOWLEDGE_BASE_ID = "YOUR_KNOWLEDGE_BASE_ID"
MODEL_ID = "anthropic.claude-sonnet-4-20250514"

# ── 검색 함수: Knowledge Base에서 관련 문서 가져오기 ─────
def retrieve_context(query, top_k=5):
    """Retrieve API로 관련 문서 청크를 검색"""
    response = agent_client.retrieve(
        knowledgeBaseId=KNOWLEDGE_BASE_ID,
        retrievalQuery={"text": query},
        retrievalConfiguration={
            "vectorSearchConfiguration": {
                "numberOfResults": top_k
            }
        }
    )

    contexts = []
    sources = []
    for result in response.get("retrievalResults", []):
        text = result.get("content", {}).get("text", "")
        uri = result.get("location", {}).get("s3Location", {}).get("uri", "")
        score = result.get("score", 0)

        contexts.append(text)
        sources.append({"uri": uri, "content": text, "score": round(score, 3)})

    return "\n\n---\n\n".join(contexts), sources

# ── 응답 생성 함수: Converse API로 답변 생성 ────────────
def generate_answer(query, context, chat_history):
    """검색된 컨텍스트와 함께 Converse API로 답변 생성"""
    system_prompt = """당신은 사내 문서를 기반으로 질문에 답변하는 AI 어시스턴트입니다.
아래 규칙을 반드시 따르세요:
1. 제공된 참고 문서(Context)에 기반하여 답변하세요.
2. 참고 문서에 없는 내용은 "제공된 문서에서 해당 정보를 찾을 수 없습니다"라고 말하세요.
3. 답변의 근거가 되는 문서 내용을 간단히 언급하세요.
4. 한국어로 답변하세요."""

    # 대화 이력에 새 질문 추가
    messages = list(chat_history)  # 복사본 생성
    user_message = f"""[참고 문서]
{context}

[질문]
{query}"""

    messages.append({
        "role": "user",
        "content": [{"text": user_message}]
    })

    response = runtime_client.converse(
        modelId=MODEL_ID,
        system=[{"text": system_prompt}],
        messages=messages,
        inferenceConfig={"maxTokens": 1024, "temperature": 0.3},
    )

    return response["output"]["message"]["content"][0]["text"]
```

> **📌 참고:**
> 방법 1(RetrieveAndGenerate)은 간편하지만 프롬프트를 직접 제어할 수 없습니다. 방법 2(Retrieve + Converse)는 코드가 길어지지만, System Prompt 커스터마이징, 검색 결과 후처리, 대화 이력 직접 관리 등이 가능합니다. 프로토타입에는 방법 1, 프로덕션에는 방법 2를 권장합니다.

> **⚠️ 주의:**
> - `bedrock-agent-runtime` 클라이언트는 Knowledge Base 관련 API(Retrieve, RetrieveAndGenerate)를 호출할 때 사용합니다.
> - `bedrock-runtime` 클라이언트는 모델 추론 API(Converse, InvokeModel)를 호출할 때 사용합니다.
> - 두 클라이언트를 혼동하지 마세요. 역할이 다릅니다.

#### 실행 및 테스트

코드를 `app.py`로 저장한 후 터미널에서 다음 명령어를 실행합니다.

```bash
streamlit run app.py
```

잠시 후 브라우저가 자동으로 열리면서(`http://localhost:8501`) 여러분이 만든 챗봇이 나타날 것입니다. 질문을 던져보세요. AI가 답변하고, 사이드바와 답변 아래 Expander에서 실제 참고 문서의 내용을 확인할 수 있습니다.

> 📷 **[이미지]** 웹 브라우저에서 실행된 app.py의 실제 스크린샷. 왼쪽 사이드바에 "참고 문서" 목록이 있고, 메인 영역에는 "사내 문서 Q&A 어시스턴트" 제목 아래 채팅 버블이 오고 간 모습. AI 답변 아래에 st.expander가 펼쳐져 S3 URI와 문서 내용이 파란색 박스(st.info)로 표시된 화면

#### Converse API 스트리밍 응답 적용

사용자 경험을 한 단계 더 높이려면 **스트리밍(Streaming)** 응답을 적용할 수 있습니다. 답변이 한 글자씩 타이핑되듯 나타나므로 체감 대기 시간이 크게 줄어듭니다.

```python
def generate_answer_streaming(query, context, chat_history):
    """Converse API의 스트리밍 모드로 답변 생성"""
    system_prompt = "당신은 사내 문서 기반 AI 어시스턴트입니다. 참고 문서에 기반하여 한국어로 답변하세요."

    messages = list(chat_history)
    messages.append({
        "role": "user",
        "content": [{"text": f"[참고 문서]\n{context}\n\n[질문]\n{query}"}]
    })

    response = runtime_client.converse_stream(
        modelId=MODEL_ID,
        system=[{"text": system_prompt}],
        messages=messages,
        inferenceConfig={"maxTokens": 1024, "temperature": 0.3},
    )

    # 스트리밍 응답을 yield로 반환
    for event in response["stream"]:
        if "contentBlockDelta" in event:
            delta = event["contentBlockDelta"].get("delta", {})
            if "text" in delta:
                yield delta["text"]
```

이 함수를 Streamlit의 `st.write_stream`과 결합하면 됩니다.

```python
# 스트리밍 답변 표시
with st.chat_message("assistant"):
    context, sources = retrieve_context(prompt)
    answer = st.write_stream(generate_answer_streaming(prompt, context, chat_history))
```

> **💡 핵심 포인트:**
> - `converse_stream`은 `converse`의 스트리밍 버전입니다. 응답을 이벤트 스트림으로 받아 실시간 표시할 수 있습니다.
> - `st.write_stream`은 제너레이터(generator)를 받아 텍스트를 점진적으로 화면에 렌더링합니다. ChatGPT처럼 글자가 하나씩 나타나는 효과를 줍니다.

---

## 마치며

이것으로 Chapter 12를 마칩니다. 우리는 복잡한 웹 기술을 배우지 않고도, Python과 Streamlit만으로 Amazon Bedrock Knowledge Base와 연동하는 사용자 친화적인 웹 기반 RAG 애플리케이션을 완성했습니다.

핵심 내용을 정리하면 다음과 같습니다.

- **Gradio**는 빠른 모델 시연에, **Streamlit**은 세션 관리와 레이아웃이 필요한 챗봇 서비스에 적합합니다.
- **2단 분할 레이아웃**(사이드바 + 메인)과 **Expander**를 활용하면, 답변의 근거를 투명하게 보여주는 신뢰할 수 있는 UI를 구성할 수 있습니다.
- Bedrock의 **RetrieveAndGenerate** API는 검색-생성을 한 번에 처리하고 `sessionId`로 멀티턴 대화를 지원합니다.
- **Retrieve** + **Converse** API 조합은 프롬프트 커스터마이징과 검색 결과 후처리가 필요할 때 유연한 선택입니다.
- `converse_stream`과 `st.write_stream`을 조합하면 스트리밍 응답으로 사용자 경험을 크게 향상시킬 수 있습니다.

이제 로컬 PC에서는 완벽하게 동작합니다. 하지만 이 서비스를 내 동료나 고객이 쓰게 하려면 내 PC를 계속 켜둘 수는 없겠죠.

다음 Chapter 13에서는 이 애플리케이션을 **컨테이너(Container)**라는 배송 상자에 담아 포장하는 기술인 Docker에 대해 알아보고, FastAPI 기반 백엔드 서버를 구성하여 Amazon ECR에 배포하는 과정을 다루겠습니다.
