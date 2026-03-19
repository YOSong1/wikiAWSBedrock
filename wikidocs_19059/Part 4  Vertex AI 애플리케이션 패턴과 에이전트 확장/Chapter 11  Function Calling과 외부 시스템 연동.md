지금까지 우리는 검색된 문서를 바탕으로 똑똑하게 답변하는 **RAG 시스템**을 구축했습니다. 하지만 이 시스템에는 여전히 치명적인 한계가 하나 있습니다. 바로 **실시간성**과 **행동력**의 부재입니다.

예를 들어, 사용자에게 "지금 서울 날씨 어때?"라고 물어본다면, 학습된 데이터나 저장된 문서만으로는 답할 수 없습니다. 또한 "내일 오후 2시에 회의 일정 잡아줘"라고 해도, AI는 "잡았습니다"라고 말만 할 뿐 실제 구글 캘린더에 일정을 등록하지는 못합니다.

이러한 한계를 극복하고 AI가 외부 세상과 소통하며 실제 데이터를 조회하고 행동하게 만드는 기술이 바로 **Function Calling(함수 호출)**입니다.

이번 챕터에서는 **Gemini**가 스스로 필요한 도구를 선택하고 사용하는 방법을 배우고, 날씨 정보와 캘린더를 연동하는 실전 봇을 구현해 보겠습니다.

> **💡 핵심 포인트:** Function Calling을 통해 AI는 정적인 데이터 처리를 넘어 실시간 행동이 가능한 진정한 에이전트로 진화합니다.

---

## 11-1 Function Calling 개념과 설계 시 주의점

**Function Calling**은 이름 때문에 오해하기 쉽습니다. AI 모델이 직접 Python 코드를 실행하거나 API를 찌르는 것이 아닙니다.

정확히 말하면, **"AI가 사람(개발자)에게 '이 함수를 이 파라미터로 실행해 주세요'라고 JSON 형식으로 요청을 보내는 기능"**입니다.

> **⚠️ 주의:** Function Calling은 AI가 직접 행동하는 것이 아니라, 행동을 위한 지시를 요청하는 메커니즘입니다.

### 11-1-1 LLM과 애플리케이션의 핑퐁 게임

작동 원리를 이해하기 위해 **LLM**과 **애플리케이션**의 핑퐁 게임을 진행하는 과정을 소개합니다.

**Function Calling**은 다음과 같은 4단계의 **대화(Turn)**로 이루어집니다.

**(1) 사용자**

```
"부산행 KTX표 예매해줘."
```

**(2) LLM (판단)**

```
사용자의 말을 분석해 보니 book_train_ticket이라는 함수가 필요하겠군. 목적지는 '부산'이고, 시간은 명시되지 않았으니 물어봐야지. (또는 기본값 사용)

-> 출력: {"function_call": {"name": "book_train_ticket", "args": {"dest": "Busan"}}}
```

**(3) 애플리케이션 (실행)**

```
어? LLM이 기차 예매 함수를 실행하라고 하네? 내가 진짜 코드를 돌려서 예매하고 결과를 받아야지.

-> 실행 결과: {"status": "success", "seat": "5A"}
```

**(4) LLM (응답)**

```
예매가 성공했구나. 이제 사용자에게 자연스럽게 말해줘야지.

-> 최종 답변: "부산행 KTX 예매를 완료했습니다. 좌석은 5A입니다."
```

이처럼 **LLM**은 **결정(Decision)**을 담당하고, 실제 **실행(Execution)**은 여러분이 짠 코드가 담당하는 구조입니다.

> **📌 참고:** Function Calling의 핵심은 책임의 분리(Separation of Concerns)입니다. LLM은 '무엇을', 애플리케이션은 '어떻게'를 담당합니다.

### 11-1-2 함수 설명(Docstring)의 중요성

LLM이 어떤 함수를 써야 할지 어떻게 알까요? 바로 여러분이 제공한 **함수 설명**을 읽고 판단합니다. 따라서 Function Calling 설계 시 가장 중요한 것은 **명확한 설명**입니다.

**나쁜 예:**

```python
def get_weather(loc): ... (설명 없음)
```

**좋은 예:**

```python
def get_weather(location: str):
    """
    특정 도시의 현재 날씨와 기온 정보를 조회합니다.

    Args:
        location (str): 도시 이름 (예: Seoul, Tokyo)
    """
```

**Gemini**는 이 설명을 읽고 "아, 날씨를 물어보면 이 함수를 location 인자와 함께 써야겠구나"라고 이해합니다.

> 📷 **[이미지]** User -> LLM -> App(Function 실행) -> LLM -> User로 이어지는 Function Calling의 4단계 루프. LLM이 직접 외부 API를 찌르는 게 아니라, App에게 '요청서(JSON)'를 건네주는 모습을 강조.

> **⚠️ 주의:** Docstring의 명확성이 LLM의 함수 선택 정확성을 크게 좌우합니다.

---

## 11-2 외부 REST API·DB 조회를 통한 실시간 데이터 결합

**RAG**가 **과거의 지식(문서)**를 다룬다면, **Function Calling**은 **현재의 정보(API)**를 다룹니다. 이 두 가지가 결합될 때 비로소 강력한 애플리케이션이 탄생합니다.

### 11-2-1 외부 REST API 연동

가장 흔한 활용 사례는 외부 서비스의 API를 연결하는 것입니다.

- **주식 정보**: "삼성전자 현재 주가 얼마야?"
  - `get_stock_price("005930")`

- **환율 정보**: "100달러는 한국 돈으로 얼마야?"
  - `get_exchange_rate("USD", "KRW")`

- **날씨 정보**: "제주도 비 와?"
  - `get_weather("Jeju")`

이러한 **실시간 정보**는 임베딩해서 벡터 DB에 넣을 수 없습니다. 매초마다 값이 변하기 때문입니다. **Function Calling**은 필요할 때마다 즉시 API를 호출하여 **최신 값**을 가져오므로 **실시간성 문제**를 완벽하게 해결합니다.

> **💡 핵심 포인트:** Function Calling을 통해 정적인 문서 기반 시스템이 실시간 데이터 기반 시스템으로 진화합니다.

### 11-2-2 사내 데이터베이스(SQL) 조회

API뿐만 아니라, 회사의 **레거시 데이터베이스(DB)**와도 연결할 수 있습니다.

예를 들어 "이번 달 A팀 매출이 얼마야?"라고 물으면, **LLM**이 이를 `SELECT sum(sales) FROM orders WHERE team='A'`와 같은 **SQL 쿼리**로 변환하거나, 미리 정의된 `get_team_sales('A')` 함수를 호출하여 **정확한 수치 데이터**를 가져올 수 있습니다.

**RAG**는 텍스트 검색에 강하지만 숫자의 정확한 연산이나 집계에는 약합니다. **Function Calling**을 통해 **DB 조회 기능**을 붙여주면 이러한 약점을 보완할 수 있습니다.

> 📷 **[이미지]** RAG(정적 문서, Vector DB)와 Function Calling(동적 데이터, API/SQL)이 서로 상호보완적으로 작동하는 아키텍처 다이어그램. 사용자 질문 유형에 따라 어느 쪽 경로를 탈지 분기되는 모습.

---

## 11-3 여러 도구를 조합한 경량 에이전트 구조

단순히 하나의 함수만 쓰는 것이 아니라, 여러 개의 **도구(Tool)**를 쥐어 주고 "네가 알아서 필요한 걸 골라 써"라고 맡길 수도 있습니다. 이를 **에이전트(Agent)**라고 부릅니다.

### 11-3-1 도구(Tool)의 개념

에이전트에게는 **Toolbox(도구 상자)**가 주어집니다. 이 상자 안에는 [계산기, 검색엔진, 캘린더, 이메일] 등 다양한 도구가 들어 있습니다.

사용자가 "내일 서울 날씨 확인하고, 비 오면 오후 3시 회의 취소 메일 보내줘"라고 복잡한 명령을 내리면, 에이전트는 다음과 같이 사고합니다.

```
Step 1: 날씨 확인이 필요하네?
-> get_weather("Seoul", "tomorrow") 호출.

Step 2: 결과가 'Rain'이네. 그럼 메일을 보내야지.
-> send_email("Meeting Canceled", ...) 호출.
```

> **💡 핵심 포인트:** 복합 작업을 해결하기 위해 여러 도구를 순차적 또는 병렬적으로 사용하는 것이 에이전트의 핵심 역할입니다.

### 11-3-2 순차적 실행과 병렬 실행

**Gemini**는 똑똑해서 한 번에 여러 함수를 호출하기도 합니다(**Parallel Function Calling**).

동시에 2곳의 날씨가 궁금할 때 "서울이랑 부산 날씨 알려줘"라고 질문할 수 있습니다. 그렇다면 같은 함수에 2곳의 이름을 주고 묻게 됩니다.

`get_weather("Seoul")`과 `get_weather("Busan")` 두 개의 요청을 동시에 생성하여 **응답 속도**를 획기적으로 줄여줍니다.

> **📌 참고:** 병렬 실행은 특히 독립적인 작업이 여러 개 있을 때 성능을 크게 향상시킵니다.

### 11-3-3 경량 에이전트 설계

**LangChain** 같은 무거운 프레임워크 없이도, **Vertex AI SDK**만으로도 충분히 훌륭한 에이전트를 만들 수 있습니다. 핵심은 **시스템 프롬프트에 도구 사용 전략을 잘 심어주는 것**입니다.

```
"너는 도구를 적극적으로 사용하는 비서다. 모르는 게 있으면 짐작하지 말고 도구
를 써라"라고 페르소나를 부여하면 도구 활용률이 높아집니다.
```

> 📷 **[이미지]** 로봇(Agent)이 앞에 놓인 여러 가지 도구(망치, 스패너, 드라이버 등) 중 상황에 맞는 도구를 집어 드는 일러스트. 복합적인 작업(Multi-step Task)을 해결하기 위해 도구를 순서대로 사용하는 개념을 시각화.

> **⚠️ 주의:** 에이전트에게 도구 사용을 강력하게 권장하는 페르소나가 중요합니다.

---

## 11-4 날씨/캘린더/사내 API를 활용한 일정 추천 봇 구현

이제 이론을 바탕으로 실제 작동하는 **스마트 일정 비서**를 만들어 보겠습니다. 이 봇은 **날씨**를 확인하고, **내 스케줄**을 고려하여 **최적의 활동**을 추천해 줍니다.

### 11-4-1 가짜(Mock) 함수 정의

실제 네이버 날씨 API나 구글 캘린더 API를 연동하려면 복잡한 인증 키가 필요하므로, 여기서는 실습을 위해 **가짜 데이터**를 반환하는 함수를 정의하여 먼저 원리 학습을 하도록 하겠습니다.

```python
import vertexai
from vertexai.generative_models import GenerativeModel, Tool, FunctionDeclaration

# 1. 날씨 조회 함수
def get_current_weather(location: str):
    """지정된 도시의 현재 날씨를 조회합니다."""
    # 실제로는 API를 호출해야 하지만, 여기선 하드코딩
    if "서울" in location:
        return {"weather": "맑음", "temp": 25}
    elif "부산" in location:
        return {"weather": "비", "temp": 20}
    else:
        return {"weather": "흐림", "temp": 22}

# 2. 일정 조회 함수
def get_my_schedule(date: str):
    """특정 날짜의 사용자 일정을 조회합니다."""
    return ["오후 2시: 팀 미팅", "오후 6시: 헬스장"]
```

> **📌 참고:** Mock 함수는 실제 환경 구성 없이 Function Calling의 동작 원리를 빠르게 학습할 수 있게 해줍니다.

### 11-4-2 도구(Tool) 선언 및 모델 바인딩

이제 이 파이썬 함수들을 **Gemini**가 알아볼 수 있는 **Tool 객체**로 변환하여 모델에 장착합니다.

```python
# 함수 선언부 정의 (JSON 스키마 자동 생성)
weather_func = FunctionDeclaration(
    name="get_current_weather",
    description="Get the current weather in a given location",
    parameters={
        "type": "object",
        "properties": {
            "location": {"type": "string", "description": "The city name"}
        }
    }
)

schedule_func = FunctionDeclaration(
    name="get_my_schedule",
    description="Get the user's schedule for a specific date",
    parameters={
        "type": "object",
        "properties": {
            "date": {"type": "string", "description": "Date to check (YYYY-MM-DD)"}
        }
    }
)

# 도구 상자(Tool) 생성
my_tools = Tool(
    function_declarations=[weather_func, schedule_func]
)

# 모델에 도구 장착
model = GenerativeModel(
    "gemini-1.5-pro-001",
    tools=[my_tools]
)
```

> **⚠️ 주의:** FunctionDeclaration의 description은 LLM의 함수 선택을 돕는 중요한 메타데이터입니다.

### 11-4-3 채팅 세션 실행 및 자동 함수 호출

이제 사용자가 질문을 던지면 **Gemini**가 도구를 호출하고, 우리가 그 결과를 다시 넣어주는 과정을 코드로 구현합니다. (Vertex AI SDK의 **Automatic Function Calling** 기능을 쓰면 이 과정을 자동화할 수도 있습니다.)

```python
chat = model.start_chat()

# 질문: 복합적인 추론이 필요한 질문
query = "나 오늘 서울에 있는데, 오후에 야외 운동 해도 될까? 내 일정 확인해서 알려줘."

response = chat.send_message(query)

# Gemini가 함수 호출을 요청했는지 확인
if response.candidates[0].function_calls:
    for function_call in response.candidates[0].function_calls:
        name = function_call.name
        args = function_call.args
        print(f"Gemini가 함수 호출을 요청함: {name}({args})")

        # 실제 함수 실행 (매핑 로직 필요)
        if name == "get_current_weather":
            api_result = get_current_weather(args["location"])
        elif name == "get_my_schedule":
            api_result = get_my_schedule("today")  # 날짜 파싱은 생략

        # 결과를 다시 모델에게 전달
        print(f"API 결과 전송: {api_result}")
        response = chat.send_message(
            Part.from_function_response(
                name=name,
                response={"content": api_result}
            )
        )

# 최종 답변 출력
print(f"최종 답변: {response.text}")
```

> **📌 참고:** 함수 호출 결과를 다시 모델에게 전달하는 과정이 Function Calling의 핵심 루프입니다.

### 11-4-4 실행 결과 해석

코드를 실행하면 다음과 같은 흐름으로 대화가 진행됩니다.

① **Gemini**는 먼저 `get_current_weather(location="서울")`을 호출합니다. (결과: 맑음, 25도)

② 이어서 `get_my_schedule`을 호출할 수도 있습니다. (결과: 2시 미팅, 6시 헬스장)

③ 최종적으로 모든 정보를 종합하여 답변합니다.

```
"서울은 현재 맑고 25도라 야외 운동하기 좋습니다. 하지만 오후 6시
에 헬스장 일정이 잡혀 있으니, 헬스장 대신 야외 조깅을 하시는 건
어떨까요?"
```

이처럼 **Function Calling**을 사용하면 단순히 정보를 검색하는 것을 넘어, **사용자의 상황**을 파악하고 **실질적인 조언**을 해주는 **고차원의 서비스**가 가능해집니다.

> 📷 **[이미지]** ![](https://static.wikidocs.net/images/page/327195/11%EC%9E%A5.jpg)

> **💡 핵심 포인트:** Function Calling은 AI를 단순한 정보 제공자에서 실질적인 조언자로 격상시킵니다.

---

이것으로 Chapter 11을 마칩니다. 우리는 **Function Calling**을 통해 AI에게 **손과 발**을 달아주었습니다. 이제 **Gemini**는 단순히 채팅창 안에 갇힌 존재가 아니라, 외부의 **날씨**를 보고, **내 일정**을 관리하며, **회사 시스템**과 소통하는 **유능한 비서**가 되었습니다.

지금까지 우리는 **텍스트 기반의 터미널 환경**에서 모든 실습을 진행했습니다. 하지만 일반 사용자에게 이런 검은 화면을 보여줄 수는 없습니다.

다음 Chapter 12에서는 Python만으로도 멋진 웹 사이트를 만들 수 있는 **Streamlit**을 사용하여, 우리가 만든 **RAG + Function Calling 봇**에 예쁜 옷(**UI**)을 입혀보겠습니다.

> **💡 핵심 포인트:** RAG와 Function Calling이 결합되면 과거의 지식과 현재의 정보를 자유롭게 오가는 진정한 지능형 에이전트가 탄생합니다.
