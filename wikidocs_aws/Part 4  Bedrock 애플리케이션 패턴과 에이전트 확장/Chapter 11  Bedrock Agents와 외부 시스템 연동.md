지난 Chapter 10에서 우리는 Few-shot, Chain-of-Thought, System Prompt 설계 등 **프롬프트 고도화** 기법을 익혔습니다. 이제 모델이 똑똑하게 답변하는 것은 물론이고, 환각도 줄이는 방법까지 알게 되었습니다. 하지만 여전히 한 가지 치명적인 한계가 남아 있습니다. 바로 **행동력**입니다.

"지금 서울 날씨 어때?"라고 물으면, 학습된 데이터만으로는 답할 수 없습니다. "내일 오후 2시에 회의 일정 잡아줘"라고 해도, AI는 "잡았습니다"라고 말만 할 뿐 실제 캘린더에 등록하지는 못합니다. "고객 주문 상태 확인해줘"라고 해도, AI는 주문 시스템에 접근할 수 없습니다.

이러한 한계를 극복하기 위해 AWS는 **Bedrock Agents**라는 완전관리형 서비스를 제공합니다. 단순히 "함수를 호출해 주세요"라고 요청하는 수준이 아니라, **스스로 추론하고, 계획을 세우고, 외부 시스템과 상호작용하며, 결과를 종합하는 자율적인 에이전트**를 구축할 수 있습니다.

이번 챕터에서는 **Bedrock Agents**의 핵심 동작 원리인 **ReAct 루프**부터, **Action Group** 설계, **Lambda 함수** 연동, 그리고 안전한 서비스 운영을 위한 **Guardrails**까지 전체 에이전트 생태계를 다루겠습니다.

> **💡 핵심 포인트:** Bedrock Agents는 LLM에게 "손과 발"을 달아주는 서비스입니다. 추론(Reasoning)과 행동(Action)을 반복하며 복잡한 업무를 자율적으로 수행합니다.

---

## 11-1 Agent 동작 원리(ReAct 루프)와 설계 패턴

Bedrock Agents를 이해하려면 먼저 **에이전트가 어떻게 생각하고 행동하는지** 그 내부 메커니즘을 알아야 합니다.

### 11-1-1 ReAct 프레임워크: 추론(Reasoning)과 행동(Action)의 반복

**ReAct**는 "**Re**asoning + **Act**ing"의 줄임말로, LLM이 **생각(추론)**과 **행동(도구 사용)**을 번갈아 수행하는 프레임워크입니다. Bedrock Agents의 핵심 동작 방식이 바로 이 ReAct 루프입니다.

사용자가 "서울 날씨 확인하고, 비가 오면 내일 야외 미팅을 취소해줘"라고 요청하면, 에이전트는 다음과 같이 동작합니다.

**Step 1 — Thought(추론)**

```
사용자가 날씨 확인과 조건부 미팅 취소를 요청했다.
먼저 서울의 현재 날씨를 확인해야 한다.
-> get_weather 액션을 호출하자.
```

**Step 2 — Action(행동)**

```
Action: get_weather(location="Seoul")
Result: {"weather": "Rain", "temp": 12}
```

**Step 3 — Observation(관찰)**

```
서울은 현재 비가 오고 있다. 사용자의 조건(비가 오면)에 해당하므로
내일 야외 미팅을 취소해야 한다.
-> cancel_meeting 액션을 호출하자.
```

**Step 4 — Action(행동)**

```
Action: cancel_meeting(date="2026-03-20", type="outdoor")
Result: {"status": "cancelled", "meeting_id": "MTG-0042"}
```

**Step 5 — Final Response(최종 응답)**

```
"서울은 현재 비가 오고 있습니다(12도). 내일 야외 미팅(MTG-0042)을
취소 처리했습니다."
```

이처럼 ReAct 루프는 **Thought → Action → Observation** 사이클을 목표가 달성될 때까지 반복합니다. 단순한 1회성 함수 호출이 아니라, **다단계 추론과 행동을 자율적으로 수행**하는 것이 핵심입니다.

> **📌 참고:** ReAct 루프는 2022년 Yao et al.의 논문 "ReAct: Synergizing Reasoning and Acting in Language Models"에서 제안된 프레임워크로, 현재 대부분의 AI 에이전트 시스템의 기반이 되고 있습니다.

### 11-1-2 Agent 아키텍처 상세 분석

Bedrock Agent의 전체 아키텍처는 다음 구성 요소로 이루어져 있습니다.

**1) Foundation Model (두뇌)**

에이전트의 추론 엔진입니다. Claude, Llama 등 Bedrock이 지원하는 모델 중 하나를 선택합니다. 사용자 요청을 분석하고, 어떤 Action Group을 호출할지 판단하며, 최종 답변을 생성합니다.

**2) Instruction (지침)**

에이전트의 역할과 행동 규칙을 정의하는 **시스템 프롬프트**입니다. "너는 고객 지원 담당자다. 주문 조회와 환불 처리를 수행할 수 있다"처럼 에이전트의 페르소나와 업무 범위를 지정합니다.

**3) Action Group (손과 발)**

에이전트가 수행할 수 있는 **구체적인 작업**의 모음입니다. 각 Action Group은 **OpenAPI 스키마**로 정의되며, 실제 실행은 **AWS Lambda 함수**가 담당합니다.

**4) Knowledge Base (기억)**

Chapter 8에서 구축한 Knowledge Base를 에이전트에 연결할 수 있습니다. 에이전트가 RAG 검색이 필요하다고 판단하면 자동으로 Knowledge Base를 조회합니다.

**5) Guardrails (안전 장치)**

에이전트의 입출력을 필터링하여 유해 콘텐츠, 민감 정보 노출, 허용되지 않은 주제 등을 차단합니다. 이 장의 후반부에서 자세히 다룹니다.

> 📷 **[이미지]** Bedrock Agent 아키텍처 다이어그램. 중앙에 Foundation Model(두뇌)이 있고, 위에서 User Request가 들어오며, 왼쪽으로 Action Group(Lambda), 오른쪽으로 Knowledge Base(S3 + Vector Store), 아래에 Guardrails가 배치된 구성도.

> **⚠️ 주의:** Agent의 Instruction(지침)이 모호하면 에이전트가 불필요한 Action을 반복하거나, 엉뚱한 Action Group을 호출할 수 있습니다. Chapter 10에서 배운 프롬프트 설계 원칙을 여기서도 적용하세요.

---

## 11-2 Action Group 설계와 OpenAPI 스키마 작성

에이전트에게 "무엇을 할 수 있는지" 알려주는 것이 **Action Group**입니다. 이것은 에이전트의 도구 상자(Toolbox)라고 생각하면 됩니다.

### 11-2-1 Action Group 개념과 역할

**Action Group**은 관련된 작업들을 하나의 그룹으로 묶은 것입니다. 예를 들어 고객 지원 에이전트라면 다음과 같이 구성할 수 있습니다.

- **주문 관리 Action Group**: 주문 조회, 주문 취소, 배송 추적
- **고객 정보 Action Group**: 회원 정보 조회, 적립금 확인
- **환불 처리 Action Group**: 환불 요청, 환불 상태 확인

각 Action Group은 두 가지 핵심 요소로 구성됩니다.

1. **OpenAPI 스키마**: 어떤 작업이 가능한지, 어떤 파라미터가 필요한지를 정의하는 API 명세서
2. **Lambda 함수**: 실제 작업을 수행하는 실행 코드

에이전트의 Foundation Model은 **OpenAPI 스키마를 읽고** 상황에 맞는 액션을 선택하며, 선택된 액션은 **Lambda 함수를 통해** 실행됩니다.

> **💡 핵심 포인트:** Action Group의 OpenAPI 스키마는 에이전트가 "어떤 도구가 있는지" 이해하는 설명서입니다. 스키마의 description이 명확할수록 에이전트의 도구 선택 정확도가 높아집니다.

### 11-2-2 OpenAPI 스키마(JSON/YAML) 작성법

Bedrock Agents는 **OpenAPI 3.0** 사양을 따르는 스키마를 사용합니다. 날씨 조회와 일정 관리 API를 정의하는 예시를 살펴보겠습니다.

```yaml
openapi: "3.0.0"
info:
  title: "WeatherAndScheduleAPI"
  version: "1.0.0"
  description: "날씨 조회 및 일정 관리를 위한 API"
paths:
  /get-weather:
    get:
      summary: "특정 도시의 현재 날씨를 조회합니다"
      description: >
        도시 이름을 입력받아 현재 날씨, 기온, 습도 정보를 반환합니다.
        사용자가 날씨를 묻거나 야외 활동 가능 여부를 확인할 때 사용합니다.
      operationId: "getWeather"
      parameters:
        - name: "location"
          in: "query"
          description: "도시 이름 (예: Seoul, Busan, Jeju)"
          required: true
          schema:
            type: "string"
      responses:
        "200":
          description: "날씨 정보 반환 성공"
          content:
            application/json:
              schema:
                type: "object"
                properties:
                  weather:
                    type: "string"
                    description: "날씨 상태 (맑음, 흐림, 비 등)"
                  temperature:
                    type: "number"
                    description: "현재 기온 (섭씨)"
  /cancel-meeting:
    post:
      summary: "특정 날짜의 미팅을 취소합니다"
      description: >
        날짜와 미팅 유형을 입력받아 해당 미팅을 취소 처리합니다.
        사용자가 일정 취소를 요청할 때 사용합니다.
      operationId: "cancelMeeting"
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: "object"
              properties:
                date:
                  type: "string"
                  description: "취소할 미팅 날짜 (YYYY-MM-DD 형식)"
                meeting_type:
                  type: "string"
                  description: "미팅 유형 (indoor, outdoor, online)"
              required:
                - date
      responses:
        "200":
          description: "미팅 취소 성공"
```

스키마 작성 시 핵심 원칙은 다음과 같습니다.

- **description을 상세히 작성**: 에이전트의 Foundation Model이 이 설명을 읽고 언제 이 API를 호출할지 판단합니다
- **operationId를 명확하게**: 기능을 한눈에 알 수 있는 이름으로 작성합니다
- **파라미터에 예시 포함**: description에 구체적인 예시를 넣으면 정확도가 올라갑니다

> **⚠️ 주의:** OpenAPI 스키마의 description 필드가 LLM의 함수 선택 정확성을 결정합니다. "Gets weather"보다 "특정 도시의 현재 날씨를 조회합니다. 사용자가 날씨를 묻거나 야외 활동 가능 여부를 확인할 때 사용합니다"처럼 상세하게 작성하세요.

### 11-2-3 Lambda 함수 연동 및 외부 API 호출

OpenAPI 스키마가 "메뉴판"이라면, Lambda 함수는 실제로 요리를 만드는 "주방"입니다. 에이전트가 액션을 선택하면 Bedrock은 해당 Lambda 함수를 호출하고, Lambda는 외부 API나 데이터베이스에 접근하여 결과를 반환합니다.

다음은 날씨 조회와 미팅 취소를 처리하는 Lambda 함수 예시입니다.

```python
import json

def lambda_handler(event, context):
    """Bedrock Agent Action Group Lambda 핸들러"""

    # 에이전트가 호출한 API 정보 추출
    api_path = event.get("apiPath")
    http_method = event.get("httpMethod")
    parameters = event.get("parameters", [])
    request_body = event.get("requestBody", {})

    # 파라미터를 딕셔너리로 변환
    params = {p["name"]: p["value"] for p in parameters}

    if api_path == "/get-weather" and http_method == "GET":
        location = params.get("location", "Seoul")
        # 실제로는 외부 날씨 API를 호출
        result = {
            "weather": "Rain",
            "temperature": 12,
            "humidity": 85,
            "location": location
        }

    elif api_path == "/cancel-meeting" and http_method == "POST":
        body = request_body.get("content", {}).get("application/json", {})
        body_props = body.get("properties", [])
        date = next((p["value"] for p in body_props if p["name"] == "date"), None)
        meeting_type = next(
            (p["value"] for p in body_props if p["name"] == "meeting_type"),
            "all"
        )
        # 실제로는 일정 관리 시스템 API를 호출
        result = {
            "status": "cancelled",
            "meeting_id": "MTG-0042",
            "date": date,
            "type": meeting_type
        }

    else:
        result = {"error": f"Unknown API: {api_path}"}

    # Bedrock Agent가 기대하는 응답 형식
    response = {
        "messageVersion": "1.0",
        "response": {
            "actionGroup": event.get("actionGroup"),
            "apiPath": api_path,
            "httpMethod": http_method,
            "httpStatusCode": 200,
            "responseBody": {
                "application/json": {
                    "body": json.dumps(result, ensure_ascii=False)
                }
            }
        }
    }

    return response
```

> **📌 참고:** Lambda 함수의 event 객체에는 `apiPath`, `httpMethod`, `parameters`, `requestBody` 등 에이전트가 전달하는 정보가 포함됩니다. 이 구조를 이해하는 것이 Action Group 개발의 핵심입니다.

> 📷 **[이미지]** User -> Bedrock Agent -> OpenAPI 스키마 분석 -> Lambda 함수 호출 -> 외부 API/DB -> 결과 반환 -> Agent 추론 -> 최종 응답의 전체 흐름도.

---

## 11-3 Agent 고급 기능

기본 동작을 이해했으니, 이제 에이전트의 고급 기능들을 살펴보겠습니다.

### 11-3-1 Memory(대화 기억)와 Knowledge Base 연계

Bedrock Agents는 **세션 메모리(Session Memory)** 기능을 지원합니다. 이를 통해 에이전트는 이전 대화 내용을 기억하고, 문맥을 유지하면서 대화할 수 있습니다.

```python
import boto3
import json
import uuid

# 에이전트 런타임 클라이언트 생성
bedrock_agent_runtime = boto3.client("bedrock-agent-runtime", region_name="us-east-1")

# 세션 ID 생성 (동일 세션 ID를 유지하면 대화 기억이 유지됨)
session_id = str(uuid.uuid4())

def invoke_agent(input_text, agent_id, agent_alias_id, session_id):
    """에이전트를 호출하고 응답을 반환합니다."""
    response = bedrock_agent_runtime.invoke_agent(
        agentId=agent_id,
        agentAliasId=agent_alias_id,
        sessionId=session_id,
        inputText=input_text,
        enableTrace=True,      # 추론 과정 추적 활성화
        memoryId="user-123"    # 장기 메모리 식별자
    )

    # 스트리밍 응답 처리
    completion = ""
    traces = []

    for event in response["completion"]:
        if "chunk" in event:
            chunk_text = event["chunk"]["bytes"].decode("utf-8")
            completion += chunk_text
        if "trace" in event:
            traces.append(event["trace"])

    return completion, traces


# 첫 번째 대화
answer1, _ = invoke_agent(
    "내 주문번호 ORD-2024-0315의 배송 상태를 알려줘.",
    agent_id="AGENT_ID",
    agent_alias_id="ALIAS_ID",
    session_id=session_id
)
print(f"[응답 1] {answer1}")

# 두 번째 대화 - 이전 문맥 기억
answer2, _ = invoke_agent(
    "그 주문을 환불 처리해줘.",  # "그 주문" = ORD-2024-0315
    agent_id="AGENT_ID",
    agent_alias_id="ALIAS_ID",
    session_id=session_id       # 동일 세션 ID 유지
)
print(f"[응답 2] {answer2}")
```

또한 에이전트에 **Knowledge Base**를 연결하면, 에이전트가 스스로 판단하여 필요할 때 RAG 검색을 수행합니다.

```python
# 에이전트 생성 시 Knowledge Base 연결 (boto3 bedrock-agent 클라이언트)
bedrock_agent = boto3.client("bedrock-agent", region_name="us-east-1")

response = bedrock_agent.associate_agent_knowledge_base(
    agentId="AGENT_ID",
    agentVersion="DRAFT",
    knowledgeBaseId="KB_ID",
    description="고객 FAQ 및 제품 매뉴얼 문서 검색용 Knowledge Base"
)
```

에이전트는 사용자 질문의 성격에 따라 **Action Group 호출**(실시간 데이터 조회)과 **Knowledge Base 검색**(문서 기반 답변)을 자동으로 선택합니다.

> **💡 핵심 포인트:** Action Group은 "현재의 실시간 데이터"를, Knowledge Base는 "축적된 문서 지식"을 담당합니다. 에이전트는 두 가지를 상황에 맞게 조합하여 최적의 답변을 생성합니다.

### 11-3-2 멀티 Action Group 오케스트레이션

하나의 에이전트에 여러 Action Group을 연결하면, 에이전트는 복합적인 업무를 **자율적으로 오케스트레이션**합니다.

예를 들어 "고객 지원 에이전트"에 다음 3개의 Action Group이 있다고 가정합니다.

- **OrderAPI**: 주문 조회, 주문 취소
- **PaymentAPI**: 환불 처리, 결제 내역 확인
- **NotificationAPI**: 이메일 발송, SMS 알림

사용자가 "주문번호 ORD-0042 환불 처리하고, 완료되면 이메일로 알려줘"라고 요청하면, 에이전트는 다음과 같이 동작합니다.

```
Thought 1: 먼저 주문 정보를 확인해야 한다.
Action 1: OrderAPI - getOrderDetails(orderId="ORD-0042")
Observation 1: 주문 금액 59,000원, 결제 수단 신용카드

Thought 2: 주문 정보를 확인했다. 이제 환불을 처리하자.
Action 2: PaymentAPI - processRefund(orderId="ORD-0042", amount=59000)
Observation 2: 환불 처리 완료, 환불 ID: REF-1234

Thought 3: 환불이 완료되었다. 이메일로 알림을 보내자.
Action 3: NotificationAPI - sendEmail(to="customer@email.com",
           subject="환불 완료 안내", body="ORD-0042 환불(59,000원) 처리 완료")
Observation 3: 이메일 발송 성공

Final: "주문 ORD-0042에 대한 환불(59,000원)이 완료되었습니다.
        고객님 이메일로 안내 메일을 발송했습니다."
```

이처럼 에이전트는 여러 Action Group을 **순서대로 호출**하면서 복잡한 업무를 처리합니다. 개발자는 각 Action Group만 잘 정의해 놓으면, 오케스트레이션 로직은 에이전트가 알아서 수행합니다.

> **📌 참고:** Action Group을 설계할 때는 "하나의 그룹 = 하나의 도메인"으로 분리하는 것이 좋습니다. 주문, 결제, 알림처럼 역할을 분리하면 에이전트의 추론 정확도가 높아지고, 유지보수도 쉬워집니다.

### 11-3-3 에이전트 End-to-End 구현 실습

이제 실제로 Bedrock Agent를 **처음부터 끝까지** 생성하는 과정을 코드로 살펴보겠습니다.

**Step 1: 에이전트 생성**

```python
import boto3
import json
import time

bedrock_agent = boto3.client("bedrock-agent", region_name="us-east-1")

# 에이전트 생성
create_response = bedrock_agent.create_agent(
    agentName="customer-support-agent",
    foundationModel="anthropic.claude-3-5-sonnet-20241022-v2:0",
    instruction="""당신은 친절한 고객 지원 담당자입니다.

주요 역할:
1. 고객의 주문 상태를 조회하고 안내합니다.
2. 날씨 정보를 확인하여 야외 배송 일정에 참고합니다.
3. 필요한 경우 주문 취소 및 환불 처리를 수행합니다.

행동 원칙:
- 항상 정확한 정보를 제공하세요.
- 확인되지 않은 정보는 추측하지 마세요.
- 작업 완료 후 결과를 명확히 안내하세요.""",
    agentResourceRoleArn="arn:aws:iam::123456789012:role/BedrockAgentRole",
    idleSessionTTLInSeconds=1800
)

agent_id = create_response["agent"]["agentId"]
print(f"에이전트 생성 완료: {agent_id}")
```

**Step 2: Action Group 추가**

```python
# OpenAPI 스키마 정의
openapi_schema = {
    "openapi": "3.0.0",
    "info": {"title": "CustomerSupportAPI", "version": "1.0.0"},
    "paths": {
        "/get-order-status": {
            "get": {
                "summary": "주문 상태를 조회합니다",
                "description": "주문번호를 입력받아 현재 배송 상태를 반환합니다.",
                "operationId": "getOrderStatus",
                "parameters": [{
                    "name": "orderId",
                    "in": "query",
                    "description": "주문번호 (예: ORD-2024-0315)",
                    "required": True,
                    "schema": {"type": "string"}
                }],
                "responses": {
                    "200": {"description": "주문 상태 반환 성공"}
                }
            }
        }
    }
}

# Action Group 생성
action_group_response = bedrock_agent.create_agent_action_group(
    agentId=agent_id,
    agentVersion="DRAFT",
    actionGroupName="OrderManagement",
    description="주문 조회, 취소, 환불 등 주문 관련 작업을 수행합니다",
    actionGroupExecutor={
        "lambda": "arn:aws:lambda:us-east-1:123456789012:function:order-handler"
    },
    apiSchema={
        "payload": json.dumps(openapi_schema)
    }
)

print(f"Action Group 생성 완료: {action_group_response['agentActionGroup']['actionGroupId']}")
```

**Step 3: 에이전트 준비(Prepare) 및 별칭(Alias) 생성**

```python
# 에이전트 준비 (변경사항 반영)
bedrock_agent.prepare_agent(agentId=agent_id)
print("에이전트 준비 중...")
time.sleep(15)  # 준비 완료 대기

# 별칭(Alias) 생성 - 에이전트 호출 시 사용
alias_response = bedrock_agent.create_agent_alias(
    agentId=agent_id,
    agentAliasName="production-v1"
)

alias_id = alias_response["agentAlias"]["agentAliasId"]
print(f"에이전트 별칭 생성 완료: {alias_id}")
```

**Step 4: 에이전트 호출 및 테스트**

```python
bedrock_agent_runtime = boto3.client("bedrock-agent-runtime", region_name="us-east-1")

# 에이전트 호출
response = bedrock_agent_runtime.invoke_agent(
    agentId=agent_id,
    agentAliasId=alias_id,
    sessionId="test-session-001",
    inputText="주문번호 ORD-2024-0315 배송 상태를 확인해주세요.",
    enableTrace=True
)

# 응답 스트리밍 처리
print("=== 에이전트 응답 ===")
for event in response["completion"]:
    if "chunk" in event:
        text = event["chunk"]["bytes"].decode("utf-8")
        print(text, end="")

    # ReAct 추론 과정 출력 (디버깅용)
    if "trace" in event:
        trace = event["trace"]["trace"]
        if "orchestrationTrace" in trace:
            orch = trace["orchestrationTrace"]
            if "rationale" in orch:
                print(f"\n[Thought] {orch['rationale']['text']}")
            if "invocationInput" in orch:
                print(f"[Action] {orch['invocationInput']}")

print("\n=== 응답 끝 ===")
```

> **⚠️ 주의:** 에이전트를 수정한 후에는 반드시 `prepare_agent()`를 호출하여 변경사항을 반영해야 합니다. prepare 없이 호출하면 이전 버전의 에이전트가 동작합니다.

> 📷 **[이미지]** Bedrock 콘솔에서 Agent 테스트 화면. 좌측에 채팅 인터페이스, 우측에 ReAct 추론 과정(Trace)이 단계별로 표시되는 모습.

---

## 11-4 Guardrails 설계와 적용

에이전트가 강력해질수록 **안전 장치**의 중요성도 커집니다. Bedrock Guardrails는 AI 모델의 입출력을 실시간으로 감시하고 필터링하는 서비스입니다. 에이전트뿐 아니라 일반 모델 호출에도 적용할 수 있습니다.

### 11-4-1 유해 콘텐츠 차단 정책 설정

Guardrails의 **콘텐츠 필터(Content Filters)**는 유해하거나 부적절한 콘텐츠를 자동으로 차단합니다. 각 카테고리별로 필터링 강도를 설정할 수 있습니다.

```python
bedrock = boto3.client("bedrock", region_name="us-east-1")

# Guardrail 생성
guardrail_response = bedrock.create_guardrail(
    name="customer-support-guardrail",
    description="고객 지원 에이전트용 안전 가드레일",

    # 콘텐츠 필터 정책
    contentPolicyConfig={
        "filtersConfig": [
            {
                "type": "SEXUAL",
                "inputStrength": "HIGH",    # 입력 필터링 강도
                "outputStrength": "HIGH"    # 출력 필터링 강도
            },
            {
                "type": "VIOLENCE",
                "inputStrength": "HIGH",
                "outputStrength": "HIGH"
            },
            {
                "type": "HATE",
                "inputStrength": "HIGH",
                "outputStrength": "HIGH"
            },
            {
                "type": "INSULTS",
                "inputStrength": "MEDIUM",
                "outputStrength": "HIGH"
            },
            {
                "type": "MISCONDUCT",
                "inputStrength": "HIGH",
                "outputStrength": "HIGH"
            },
            {
                "type": "PROMPT_ATTACK",
                "inputStrength": "HIGH",
                "outputStrength": "NONE"    # 출력에는 적용하지 않음
            }
        ]
    },

    # 차단 시 보여줄 메시지
    blockedInputMessaging="죄송합니다. 해당 요청은 서비스 정책에 따라 처리할 수 없습니다.",
    blockedOutputsMessaging="죄송합니다. 해당 내용은 서비스 정책에 따라 제공할 수 없습니다."
)

guardrail_id = guardrail_response["guardrailId"]
print(f"Guardrail 생성 완료: {guardrail_id}")
```

필터링 강도는 **NONE**, **LOW**, **MEDIUM**, **HIGH** 4단계로 설정할 수 있으며, HIGH로 설정할수록 더 엄격하게 필터링합니다. 특히 `PROMPT_ATTACK` 타입은 **프롬프트 인젝션 공격**을 탐지하여 차단합니다.

> **💡 핵심 포인트:** 콘텐츠 필터의 강도는 서비스 특성에 맞게 조정하세요. 고객 서비스용은 HIGH, 창작 도구용은 MEDIUM이 적절할 수 있습니다.

### 11-4-2 민감정보(PII) 보호 필터

고객 지원 에이전트가 주민등록번호, 신용카드 번호 같은 **개인식별정보(PII)**를 노출하면 심각한 보안 사고가 됩니다. Guardrails의 **민감정보 필터**로 이를 방지할 수 있습니다.

```python
# Guardrail에 민감정보 보호 정책 추가
bedrock.update_guardrail(
    guardrailIdentifier=guardrail_id,
    name="customer-support-guardrail",
    description="고객 지원 에이전트용 안전 가드레일",

    # 민감정보(PII) 필터 정책
    sensitiveInformationPolicyConfig={
        "piiEntitiesConfig": [
            {
                "type": "EMAIL",
                "action": "ANONYMIZE"       # 마스킹 처리 (예: j***@email.com)
            },
            {
                "type": "PHONE",
                "action": "ANONYMIZE"
            },
            {
                "type": "CREDIT_DEBIT_CARD_NUMBER",
                "action": "BLOCK"           # 완전 차단
            },
            {
                "type": "US_SOCIAL_SECURITY_NUMBER",
                "action": "BLOCK"
            },
            {
                "type": "NAME",
                "action": "ANONYMIZE"
            }
        ],
        # 정규식 기반 커스텀 민감정보 탐지
        "regexesConfig": [
            {
                "name": "KoreanResidentId",
                "description": "한국 주민등록번호 패턴 탐지",
                "pattern": "\\d{6}-[1-4]\\d{6}",
                "action": "BLOCK"
            },
            {
                "name": "InternalOrderId",
                "description": "내부 주문 시스템 ID 패턴",
                "pattern": "INT-\\d{8}-[A-Z]{3}",
                "action": "ANONYMIZE"
            }
        ]
    },

    blockedInputMessaging="민감한 개인정보가 포함된 요청입니다. 개인정보를 제거 후 다시 시도해주세요.",
    blockedOutputsMessaging="죄송합니다. 개인정보 보호 정책에 따라 해당 정보를 표시할 수 없습니다."
)
```

PII 필터는 두 가지 동작 모드를 제공합니다.

- **ANONYMIZE**: 민감정보를 마스킹 처리하여 응답에 포함 (예: `010-****-5678`)
- **BLOCK**: 민감정보가 감지되면 응답 자체를 차단

> **⚠️ 주의:** `regexesConfig`를 활용하면 한국 주민등록번호처럼 AWS 기본 PII 탐지에 포함되지 않는 국가별 민감정보도 커스텀 패턴으로 보호할 수 있습니다.

### 11-4-3 입출력 통제(Topic Denial, Word Filters)

특정 주제에 대한 응답을 아예 거부하거나, 금지어를 필터링할 수도 있습니다.

**Topic Denial (주제 거부)**

에이전트가 다루지 말아야 할 주제를 명시적으로 정의합니다.

```python
# 주제 거부 정책
topic_policy_config = {
    "topicsConfig": [
        {
            "name": "Investment_Advice",
            "definition": "투자 조언, 주식 추천, 금융 상품 권유 등 재무적 조언을 제공하는 것",
            "examples": [
                "어떤 주식을 사야 하나요?",
                "비트코인에 투자해도 될까요?",
                "내 돈을 어디에 넣는 게 좋을까?"
            ],
            "type": "DENY"
        },
        {
            "name": "Medical_Advice",
            "definition": "의료 진단, 약물 처방, 치료 방법 등 의학적 조언을 제공하는 것",
            "examples": [
                "두통이 심한데 어떤 약을 먹어야 해?",
                "이 증상이면 어떤 병인가요?"
            ],
            "type": "DENY"
        }
    ]
}
```

**Word Filters (단어 필터)**

특정 단어나 구문을 직접 차단합니다.

```python
# 단어 필터 정책
word_policy_config = {
    "wordsConfig": [
        {"text": "경쟁사A"},
        {"text": "경쟁사B"},
        {"text": "내부기밀"}
    ],
    "managedWordListsConfig": [
        {"type": "PROFANITY"}    # AWS 관리형 비속어 목록
    ]
}
```

> **📌 참고:** `PROFANITY` 관리형 단어 목록은 AWS가 미리 구축한 비속어 사전으로, 영어를 포함한 다국어 비속어를 자동으로 필터링합니다.

### 11-4-4 Guardrail 정책 초안 및 테스트

모든 정책을 조합하여 완성된 Guardrail을 만들고 테스트해 보겠습니다.

```python
# 완성된 Guardrail 생성 (모든 정책 포함)
full_guardrail = bedrock.create_guardrail(
    name="production-guardrail-v1",
    description="프로덕션 환경용 종합 가드레일",

    contentPolicyConfig={
        "filtersConfig": [
            {"type": "SEXUAL", "inputStrength": "HIGH", "outputStrength": "HIGH"},
            {"type": "VIOLENCE", "inputStrength": "HIGH", "outputStrength": "HIGH"},
            {"type": "HATE", "inputStrength": "HIGH", "outputStrength": "HIGH"},
            {"type": "PROMPT_ATTACK", "inputStrength": "HIGH", "outputStrength": "NONE"}
        ]
    },

    sensitiveInformationPolicyConfig={
        "piiEntitiesConfig": [
            {"type": "EMAIL", "action": "ANONYMIZE"},
            {"type": "PHONE", "action": "ANONYMIZE"},
            {"type": "CREDIT_DEBIT_CARD_NUMBER", "action": "BLOCK"}
        ],
        "regexesConfig": [
            {
                "name": "KoreanResidentId",
                "description": "한국 주민등록번호",
                "pattern": "\\d{6}-[1-4]\\d{6}",
                "action": "BLOCK"
            }
        ]
    },

    topicPolicyConfig={
        "topicsConfig": [
            {
                "name": "Investment_Advice",
                "definition": "투자 조언이나 금융 상품 권유",
                "examples": ["어떤 주식을 사야 하나요?"],
                "type": "DENY"
            }
        ]
    },

    wordPolicyConfig={
        "wordsConfig": [{"text": "경쟁사A"}],
        "managedWordListsConfig": [{"type": "PROFANITY"}]
    },

    blockedInputMessaging="요청이 서비스 정책에 의해 차단되었습니다.",
    blockedOutputsMessaging="응답이 서비스 정책에 의해 차단되었습니다."
)

guardrail_id = full_guardrail["guardrailId"]
guardrail_version = full_guardrail["version"]

print(f"Guardrail 생성 완료: {guardrail_id} (v{guardrail_version})")
```

생성된 Guardrail은 에이전트에 연결하거나, 일반 모델 호출 시에도 적용할 수 있습니다.

```python
# 방법 1: 에이전트에 Guardrail 연결
bedrock_agent.update_agent(
    agentId=agent_id,
    agentName="customer-support-agent",
    foundationModel="anthropic.claude-3-5-sonnet-20241022-v2:0",
    instruction="당신은 고객 지원 담당자입니다...",
    agentResourceRoleArn="arn:aws:iam::123456789012:role/BedrockAgentRole",
    guardrailConfiguration={
        "guardrailIdentifier": guardrail_id,
        "guardrailVersion": guardrail_version
    }
)

# 방법 2: Converse API에서 직접 Guardrail 적용
bedrock_runtime = boto3.client("bedrock-runtime", region_name="us-east-1")

response = bedrock_runtime.converse(
    modelId="anthropic.claude-3-5-sonnet-20241022-v2:0",
    messages=[
        {"role": "user", "content": [{"text": "비트코인에 투자해도 될까요?"}]}
    ],
    guardrailConfig={
        "guardrailIdentifier": guardrail_id,
        "guardrailVersion": guardrail_version
    }
)

# Guardrail 차단 여부 확인
if response.get("stopReason") == "guardrail_intervened":
    print("Guardrail에 의해 차단됨!")
    print(response["output"]["message"]["content"][0]["text"])
else:
    print("정상 응답:")
    print(response["output"]["message"]["content"][0]["text"])
```

Guardrail이 적용된 상태에서 "비트코인에 투자해도 될까요?"라고 물으면, Topic Denial 정책에 의해 차단되고 미리 설정한 차단 메시지가 반환됩니다.

> **💡 핵심 포인트:** Guardrails는 "방어의 마지막 보루"입니다. 프롬프트 설계로 1차 방어를 하고, Guardrails로 2차 방어를 하는 **이중 안전 장치** 전략을 권장합니다.

> 📷 **[이미지]** Guardrail 테스트 결과 화면. 정상 질문에는 응답이 반환되고, 차단 대상 질문에는 "요청이 서비스 정책에 의해 차단되었습니다" 메시지가 표시되는 비교 화면.

---

## 마치며

이번 챕터에서는 **Bedrock Agents**의 핵심 원리부터 실전 구현까지 전체 생태계를 살펴보았습니다. 정리하면 다음과 같습니다.

- **ReAct 루프**: 에이전트는 Thought → Action → Observation을 반복하며 복잡한 작업을 자율적으로 수행합니다
- **Action Group + OpenAPI 스키마**: 에이전트의 "능력"을 정의하는 API 명세서이며, Lambda 함수가 실제 실행을 담당합니다
- **Memory + Knowledge Base**: 대화 기억과 문서 검색을 결합하여 더 똑똑한 에이전트를 만들 수 있습니다
- **Guardrails**: 콘텐츠 필터, PII 보호, 주제 거부, 단어 필터로 안전한 서비스를 운영합니다

Chapter 10의 프롬프트 고도화가 에이전트의 "두뇌"를 업그레이드했다면, 이번 챕터의 Agents와 Guardrails는 에이전트에게 "손발"과 "안전벨트"를 달아준 것입니다.

지금까지 우리는 터미널과 코드 환경에서 모든 실습을 진행해왔습니다. 하지만 실제 서비스에서는 일반 사용자가 검은 화면을 마주할 수 없습니다. 다음 Chapter 12에서는 **Streamlit**을 활용하여 우리가 구축한 RAG + Agent 시스템에 **웹 UI**를 입혀보겠습니다. 코드 한 줄 없이도 누구나 사용할 수 있는 멋진 챗봇 인터페이스를 만들어 보겠습니다.

> **💡 핵심 포인트:** Bedrock Agents + Knowledge Base + Guardrails, 이 세 가지가 결합되면 과거의 지식(문서)과 현재의 데이터(API)를 오가며, 안전하게 동작하는 진정한 엔터프라이즈급 AI 에이전트가 탄생합니다.
