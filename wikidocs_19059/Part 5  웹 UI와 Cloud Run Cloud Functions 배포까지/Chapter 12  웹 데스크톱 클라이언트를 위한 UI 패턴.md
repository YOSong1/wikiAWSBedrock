지금까지 우리는 터미널의 검은 화면(Console)에서 텍스트로만 AI와 소통해 왔습니다. 개발자인 우리에게는 익숙할지 모르지만, 일반 사용자나 클라이언트에게 "여기 터미널에 명령어를 치세요"라고 할 수는 없습니다. 훌륭한 AI 모델도 사용하기 불편하면 외면받기 마련입니다.

따라서 우리가 만든 RAG 에이전트를 누구나 쉽게 쓸 수 있도록 그래픽 사용자 인터페이스(GUI), 그중에서도 접근성이 가장 좋은 웹(Web) UI로 감싸는 작업이 필요합니다.

과거에는 웹을 만들려면 HTML, CSS, JavaScript는 물론 React나 Vue 같은 프론트엔드 프레임워크까지 배워야 했습니다. 하지만 AI 시대에는 Python 개발자를 위한 혁신적인 도구들이 등장했습니다. 이번 챕터에서는 그중 가장 대표적인 **Streamlit**(스트림릿)을 사용하여, 단 몇 줄의 파이썬 코드만으로 챗봇 웹 서비스를 구축하는 방법을 상세히 알아보겠습니다.

---

## 12-1 Streamlit/Gradio 기반 빠른 프로토타입 UI

본격적인 개발에 앞서, 파이썬 기반의 양대 산맥인 UI 라이브러리, Streamlit과 Gradio를 비교해 보고 우리가 사용할 도구를 선택해 보겠습니다.

### 12-1-1 Gradio: 모델 시연에 최적화된 도구

**Gradio**는 주로 "내 모델이 이렇게 작동해!"라고 보여주는 데모 용도로 많이 쓰입니다. 입력창(Input)과 출력창(Output)을 정의하면 자동으로 UI를 만들어줍니다.

> **💡 핵심 포인트:**
> - 설정이 매우 간편하고, Hugging Face Spaces와 같은 플랫폼에 배포하기 쉽습니다.

> **⚠️ 주의:**
> - 화면 레이아웃을 마음대로 뜯어고치거나, 복잡한 대화 흐름을 제어하기에는 유연성이 조금 부족합니다.

### 12-1-2 Streamlit: 데이터 앱과 대시보드의 강자

**Streamlit**은 데이터 과학자들을 위해 태어난 도구입니다. "데이터 스크립트를 정보 공유 가능한 웹 앱으로 바꾼다"는 철학을 가지고 있습니다.

> **💡 핵심 포인트:**
> - 위젯(버튼, 슬라이더 등) 배치가 자유롭고, **세션 상태(Session State)** 관리가 강력하여 멀티턴 챗봇 구현에 유리합니다.

> **📌 참고:**
> 우리는 RAG 챗봇을 만들 것이므로, 대화 내역(History) 관리와 레이아웃 커스터마이징이 더 강력한 **Streamlit**을 사용하여 실습을 진행하겠습니다.

> 📷 **[이미지]** 좌측에는 Gradio로 만든 단순한 입출력 화면, 우측에는 Streamlit으로 만든 사이드바가 있는 대시보드 화면을 나란히 배치하여 비교하는 이미지. Streamlit 쪽이 좀 더 '완성된 웹사이트' 느낌이 난다는 것을 시각적으로 보여줌

---

## 12-2 단일 페이지에서 "입력-결과-근거"를 보여주는 레이아웃

RAG 애플리케이션의 UI 설계에서 가장 중요한 원칙은 **투명성**입니다. 사용자는 AI의 답변뿐만 아니라, 그 답변이 "어디서 나왔는지(근거)"를 눈으로 확인하고 싶어 합니다. 따라서 화면 레이아웃을 짤 때 이 부분을 반드시 고려해야 합니다.



### 12-2-1 2단 분할 레이아웃 (Split View)

가장 권장되는 방식은 화면을 좌우로 나누는 것입니다.

- **왼쪽 (Main):** AI와의 채팅 창입니다. 사용자가 질문하고 답변을 받는 공간입니다.
- **오른쪽 (Sidebar or Column):** '참고 문서(Context)'를 보여주는 공간입니다. AI가 답변할 때 참고한 PDF 페이지나 텍스트 원문을 실시간으로 띄워줍니다.

이렇게 구성하면 사용자는 AI의 답변을 읽으면서, 동시에 오른쪽에서 원문과 비교하며 팩트 체크를 할 수 있어 서비스 신뢰도가 급상승합니다.

### 12-2-2 아코디언/익스팬더 (Expander) 활용

화면 공간이 좁다면, 답변 바로 아래에 "참고 문서 보기"라는 버튼(Expander)을 두는 방식도 좋습니다. 평소에는 숨겨져 있다가, 궁금한 사용자가 클릭했을 때만 펼쳐져서 출처를 보여주는 깔끔한 방식입니다. 우리는 실습에서 이 **Expander** 방식을 적용해 보겠습니다.

> 📷 **[이미지]** 와이어프레임(Wireframe) 스케치 이미지. 상단에 제목, 중앙에 채팅 버블들이 있고, AI 답변 버블 바로 아래에 'Source Documents'라는 접이식 메뉴가 달려있는 구조를 도식화

---

## 12-3 RAG Q&A·요약·멀티모달 기능 통합 UI 설계

단순한 텍스트 채팅을 넘어, 파일 업로드나 이미지 처리 같은 멀티모달 기능을 통합하려면 UI 컴포넌트들을 적재적소에 배치해야 합니다.

### 12-3-1 세션 상태(Session State) 관리의 중요성

**Streamlit**의 작동 방식은 독특합니다. 사용자가 버튼을 한 번 누를 때마다 파이썬 코드가 처음부터 끝까지 다시 실행(Rerun)됩니다. 따라서 아무런 조치를 하지 않으면, 사용자가 새 질문을 입력하는 순간 이전 대화 내용이 싹 사라져 버립니다.

이를 방지하기 위해 `st.session_state`라는 메모리 공간을 사용해야 합니다.

```python
if "messages" not in st.session_state:
    st.session_state.messages = [] # 대화 내용을 저장할 빈 리스트 생성
```

> **💡 핵심 포인트:**
> 이 코드는 앱이 다시 실행되더라도 `messages`라는 변수만큼은 지우지 말고 기억하라는 의미입니다. 챗봇 구현의 핵심입니다.

### 12-3-2 채팅 인터페이스 컴포넌트

**Streamlit**은 `st.chat_message`와 `st.chat_input`이라는 챗봇 전용 함수를 제공합니다.

- `with st.chat_message("user"):` - 사용자의 말풍선(오른쪽 정렬)을 만듭니다.
- `with st.chat_message("assistant"):` - AI의 말풍선(왼쪽 정렬, 로봇 아이콘)을 만듭니다.

이 두 가지 기능과 세션 상태를 결합하면 카카오톡이나 ChatGPT와 유사한 대화형 인터페이스를 10분 만에 구현할 수 있습니다.

> 📷 **[이미지]** st.session_state가 작동하는 원리를 보여주는 다이어그램. [User Interaction] -> [Script Rerun] -> [Load from Session State] -> [UI Update]로 이어지는 순환 구조를 그려주세요. 메모리(Session State)가 리셋을 막아준다는 개념이 표현되어야 함

---

## 12-4 RAG 백엔드를 호출하는 간단 웹 UI 구현

이제 이론을 마쳤으니, Chapter 8과 9에서 만든 **RAGBackend** 클래스를 가져와서 실제 웹 서비스로 연결해 보겠습니다. 파일 이름은 `app.py`로 생성합니다.

### 12-4-1 라이브러리 설치 및 임포트

먼저 **Streamlit**을 설치합니다.

```bash
pip install streamlit
```

그리고 코드에서 필요한 모듈을 불러옵니다.

```python
import streamlit as st
from rag_backend import RAGBackend # Ch8에서 만든 백엔드 클래스
from langchain_google_vertexai import VertexAI

# 페이지 기본 설정 (탭 이름, 아이콘 등)
st.set_page_config(page_title="Vertex AI RAG Chatbot", page_icon="🤖")
st.title("나만의 AI 문서 비서")
```

### 12-4-2 백엔드 초기화 (캐싱 활용)

RAG 백엔드(벡터 DB 로딩)는 무거운 작업이므로, 매번 다시 실행하면 느립니다. `@st.cache_resource` 데코레이터를 사용하여 한 번만 로딩하고 재사용하게 만듭니다.

```python
@st.cache_resource
def get_rag_backend():
    return RAGBackend() # 벡터 DB 로드

rag = get_rag_backend()
llm = VertexAI(model_name="gemini-1.5-pro-001") # Gemini 모델 설정
```



### 12-4-3 채팅 로직 구현

이제 화면을 그리는 메인 로직입니다.

1. **히스토리 출력:** 세션에 저장된 이전 대화를 먼저 화면에 그려줍니다.
2. **입력 대기:** `st.chat_input`으로 사용자의 입력을 기다립니다.
3. **답변 생성:** 입력이 들어오면 RAG 검색을 수행하고, 결과를 보여줍니다.



```python
# 1. 세션 스테이트 초기화
if "messages" not in st.session_state:
    st.session_state.messages = []

# 2. 이전 대화 내용 화면에 출력 (History)
for msg in st.session_state.messages:
    with st.chat_message(msg["role"]):
        st.markdown(msg["content"])

# 3. 사용자 입력 처리
if prompt := st.chat_input("궁금한 점을 물어보세요!"):
    # 사용자 메시지 표시 및 저장
    with st.chat_message("user"):
        st.markdown(prompt)
    st.session_state.messages.append({"role": "user", "content": prompt})

    # AI 답변 생성 과정
    with st.chat_message("assistant"):
        message_placeholder = st.empty() # 답변이 들어갈 빈 공간

        # RAG 검색 수행 (Ch8 기능)
        docs = rag.retrieve_documents(prompt)
        context_text = "\n".join([doc.page_content for doc in docs])

        # 프롬프트 구성 및 Gemini 호출 (Ch9 기능 간소화)
        full_prompt = f"Context:\n{context_text}\n\nQuestion:\n{prompt}\nAnswer:"
        response = llm.invoke(full_prompt)

        # 답변 출력
        message_placeholder.markdown(response)

        # ★ 근거 문서(Expander) 보여주기
        with st.expander("참고 문서 확인하기"):
            for i, doc in enumerate(docs):
                st.info(f"문서 {i+1} (p.{doc.metadata.get('page', '?')})\n{doc.page_content[:100]}...")

    # AI 메시지 저장
    st.session_state.messages.append({"role": "assistant", "content": response})
```

### 12-4-4 실행 및 테스트

코드를 저장한 후 터미널에서 다음 명령어를 입력합니다.

```bash
streamlit run app.py
```

잠시 후 브라우저가 자동으로 열리면서 여러분이 만든 챗봇이 나타날 것입니다. 질문을 던져보세요. AI가 답변하고, 그 아래 "참고 문서 확인하기"를 누르면 실제 PDF의 내용이 보이는 것을 확인할 수 있습니다. 이것이 바로 상용 수준의 RAG 서비스 프로토타입입니다.

> 📷 **[이미지]** 웹 브라우저에서 실행된 app.py의 실제 스크린샷. 상단에는 "🤖 나만의 AI 문서 비서" 제목이 있고, 채팅창에는 대화가 오고 간 모습, 그리고 가장 중요하게는 답변 아래에 st.expander가 펼쳐져서 참고 문서의 일부가 파란색 박스(st.info)로 표시된 화면

---

## 마무리



이것으로 Chapter 12를 마칩니다. 우리는 복잡한 웹 기술을 배우지 않고도, Python과 Streamlit만으로 사용자가 편리하게 쓸 수 있는 웹 기반 RAG 애플리케이션을 완성했습니다.

이제 로컬 PC에서는 완벽하게 동작합니다. 하지만 이 서비스를 내 동료나 고객이 쓰게 하려면 내 PC를 계속 켜둘 수는 없습니다.

다음 Chapter 13에서는 이 애플리케이션을 '컨테이너(Container)'라는 배송 상자에 담아 포장하는 기술인 Docker에 대해 알아보고, 클라우드 배포를 준비하겠습니다.

