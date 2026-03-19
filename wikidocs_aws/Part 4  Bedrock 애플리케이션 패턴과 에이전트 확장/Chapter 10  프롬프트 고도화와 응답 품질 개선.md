앞선 챕터에서 우리는 Knowledge Bases와 Converse API를 조합하여 검색된 문서를 근거로 답변하는 **RAG 기반 Q&A 시스템**을 구축했습니다. 하지만 "기능이 동작한다"는 것과 "사용자가 만족한다"는 것은 전혀 다른 차원의 문제입니다. 때로는 AI가 엉뚱한 동문서답을 하기도 하고, 비즈니스 매너에 맞지 않는 가벼운 말투를 쓰기도 하며, 심지어는 없던 사실을 지어내기도 합니다.

이러한 문제들을 해결하기 위해 모델 자체를 새로 학습시키는 것은 비용과 시간이 많이 듭니다. 대신 우리는 **프롬프트(Prompt)**, 즉 AI에게 지시를 내리는 방식만 정교하게 다듬어도 놀라운 성능 향상을 이뤄낼 수 있습니다. 이를 **프롬프트 엔지니어링(Prompt Engineering)**이라고 합니다.

이번 챕터에서는 단순한 지시를 넘어, 예시를 통해 학습시키는 **Few-shot** 기법, Claude가 특히 잘 활용하는 **XML 태그 기반 Chain-of-Thought(CoT)**, 가장 치명적인 문제인 **환각(Hallucination)**을 제어하는 고급 프롬프트 설계 전략, 그리고 Bedrock이 제공하는 **Model Evaluation** 기능을 활용한 체계적인 품질 평가까지 다루겠습니다.

> **💡 핵심 포인트:** 프롬프트 엔지니어링은 모델 재학습 없이 응답 품질을 획기적으로 개선할 수 있는 가장 비용 효율적인 기법입니다.

---

## 10-1 Few-shot, Chain-of-Thought, 단계적 추론 적용

**Claude**와 같은 **거대 언어 모델(LLM)**은 기본적으로 매우 똑똑하지만, 때로는 구체적인 가이드라인이 없으면 엉뚱한 방향으로 답변을 생성합니다. 이를 보정하기 위한 가장 강력한 두 가지 무기가 바로 **Few-shot(퓨샷)**과 **Chain-of-Thought(사고의 사슬)**입니다.

### 10-1-1 Zero-shot vs Few-shot 실험

우리가 지금까지 사용한 방식은 예시 없이 질문만 던지는 **Zero-shot** 방식이었습니다. 하지만 신입 사원에게 업무를 지시할 때를 생각해 보십시오. "보고서 써와"라고 말하는 것보다, "지난달에 김 대리가 쓴 이 보고서를 참고해서 비슷하게 써와"라고 예시를 주는 것이 훨씬 효과적입니다.

**Few-shot 프롬프팅**은 프롬프트 안에 **질문**과 **이상적인 답변**의 예시를 몇 가지(Few) 포함시켜서 모델에게 전달하는 기법입니다.

**Zero-shot 예시:**

```
"이 리뷰의 감정을 분석해줘: '음식이 식어서 왔어요.'"
```

**Few-shot 예시:**

```
다음은 고객 리뷰에 대한 감정 분석 예시입니다.

리뷰: "배송이 정말 빠르네요!"
감정: 긍정

리뷰: "박스가 찌그러져서 왔어요."
감정: 부정

리뷰: "음식이 식어서 왔어요."
감정:
```

이렇게 예시를 주면 모델은 "아, 이런 형식으로 짧게 답하라는 거구나"라고 패턴을 즉각적으로 파악하여, "부정"이라고 정확하게 답변합니다.

> 📷 **[이미지]** Zero-shot 프롬프팅과 Few-shot 프롬프팅의 구조 비교. Zero-shot은 [지시문]->[질문]으로 바로 이어지지만, Few-shot은 [지시문]->[예시1]->[예시2]->[질문]으로 중간 단계가 있음을 시각화.

Bedrock Converse API에서는 `messages` 배열에 user/assistant 쌍으로 예시를 넣어 Few-shot을 구현합니다. 실제 코드를 살펴보겠습니다.

```python
import boto3
import json

client = boto3.client("bedrock-runtime", region_name="us-east-1")

# Few-shot 감정 분석 실험
messages = [
    # 예시 1 (user-assistant 쌍)
    {
        "role": "user",
        "content": [{"text": "리뷰: '배송이 정말 빠르네요!'\n감정:"}]
    },
    {
        "role": "assistant",
        "content": [{"text": "긍정"}]
    },
    # 예시 2
    {
        "role": "user",
        "content": [{"text": "리뷰: '박스가 찌그러져서 왔어요.'\n감정:"}]
    },
    {
        "role": "assistant",
        "content": [{"text": "부정"}]
    },
    # 실제 질문
    {
        "role": "user",
        "content": [{"text": "리뷰: '음식이 식어서 왔어요.'\n감정:"}]
    }
]

response = client.converse(
    modelId="anthropic.claude-sonnet-4-20250514",
    messages=messages,
    inferenceConfig={"maxTokens": 100, "temperature": 0.0}
)

print(response["output"]["message"]["content"][0]["text"])
# 출력: 부정
```

> **📌 참고:** Converse API의 `messages`에 user/assistant 쌍을 번갈아 넣으면 자연스럽게 Few-shot 예시가 됩니다. 이 방식은 모델에 독립적이므로, Claude뿐 아니라 Llama, Mistral 등 어떤 모델이든 동일하게 적용됩니다.

Few-shot을 효과적으로 사용하기 위한 팁을 정리하면 다음과 같습니다.

| 원칙 | 설명 |
|------|------|
| 예시는 2~5개가 적절 | 너무 적으면 패턴 학습 부족, 너무 많으면 토큰 낭비 |
| 다양한 케이스 포함 | 긍정/부정/중립 등 모든 카테고리를 예시에 포함 |
| 실제 데이터 활용 | 가상 예시보다 실제 데이터가 더 효과적 |
| 일관된 형식 유지 | 예시 간 형식이 달라지면 모델이 혼란 |

### 10-1-2 Chain-of-Thought (CoT): XML 태그 구조화 추론

복잡한 수학 문제나 추론 문제를 풀 때, 단순히 정답만 요구하면 AI도 틀릴 확률이 높습니다. 하지만 "단계적으로 생각해서 풀이 과정을 적어줘"라고 지시하면 정답률이 비약적으로 상승합니다. 이것이 바로 **Chain-of-Thought(CoT)** 기법입니다.

사람도 암산보다 연습장에 풀이를 적을 때 실수가 적은 것처럼, 모델에게도 답변을 생성하기 전에 **생각할 시간(토큰)**을 부여하는 원리입니다.

특히 **Claude 모델**은 **XML 태그**를 활용한 구조화된 추론에 매우 뛰어난 성능을 보입니다. `<thinking>`과 `<answer>` 태그를 사용하면 모델이 내부 추론 과정과 최종 답변을 명확히 분리하도록 유도할 수 있습니다.

```python
import boto3

client = boto3.client("bedrock-runtime", region_name="us-east-1")

system_prompt = """당신은 논리적 문제 해결 전문가입니다.
모든 질문에 다음 형식으로 답변하세요:

<thinking>
여기에 단계별 추론 과정을 적으세요.
- 주어진 조건을 정리합니다.
- 각 단계를 논리적으로 전개합니다.
- 중간 결과를 검증합니다.
</thinking>

<answer>
여기에 최종 답변만 간결하게 적으세요.
</answer>"""

messages = [
    {
        "role": "user",
        "content": [{"text": """한 과수원에 사과나무가 15그루 있습니다.
각 나무에서 평균 40개의 사과가 열리는데,
올해는 병충해로 전체의 20%가 상했습니다.
상하지 않은 사과는 총 몇 개인가요?"""}]
    }
]

response = client.converse(
    modelId="anthropic.claude-sonnet-4-20250514",
    messages=messages,
    system=[{"text": system_prompt}],
    inferenceConfig={"maxTokens": 1024, "temperature": 0.0}
)

result = response["output"]["message"]["content"][0]["text"]
print(result)
```

**출력 예시:**

```xml
<thinking>
주어진 조건을 정리합니다:
- 사과나무: 15그루
- 나무당 평균 사과: 40개
- 병충해로 상한 비율: 20%

1단계: 전체 사과 수 계산
15그루 × 40개 = 600개

2단계: 상한 사과 수 계산
600개 × 20% = 120개

3단계: 상하지 않은 사과 수 계산
600개 - 120개 = 480개

검증: 상하지 않은 비율은 80%이므로
600 × 0.8 = 480 ✓
</thinking>

<answer>
상하지 않은 사과는 총 480개입니다.
</answer>
```

> 📷 **[이미지]** CoT가 적용되었을 때 모델 내부의 처리 과정. [입력] -> [<thinking> 중간 추론 단계 생성] -> [<answer> 최종 답변]으로 이어지는 흐름을 도식화.

XML 태그를 활용한 CoT의 핵심 장점은 다음과 같습니다.

1. **추론과 답변의 분리**: `<thinking>` 내용은 디버깅에 활용하고, `<answer>` 내용만 사용자에게 보여줄 수 있습니다.
2. **파싱의 용이성**: 정규표현식이나 XML 파서로 답변 부분만 쉽게 추출할 수 있습니다.
3. **Claude 최적화**: Claude 모델은 XML 태그 기반 지시를 특히 잘 따릅니다.

```python
import re

# <answer> 태그 내용만 추출하는 유틸리티
def extract_answer(response_text):
    match = re.search(r"<answer>(.*?)</answer>", response_text, re.DOTALL)
    return match.group(1).strip() if match else response_text

final_answer = extract_answer(result)
print(final_answer)
# 출력: 상하지 않은 사과는 총 480개입니다.
```

RAG 시스템에서도 "먼저 `<analysis>` 태그 안에서 검색된 문서를 분석하고, 그 다음에 `<answer>` 태그 안에 답변을 작성해"라고 단계를 나누어 지시하면 훨씬 더 정확한 결과를 얻을 수 있습니다.

> **💡 핵심 포인트:** Claude에서 XML 태그 기반 CoT를 사용하면, 추론 과정은 디버깅에 활용하고 최종 답변만 사용자에게 깔끔하게 전달할 수 있습니다.

---

## 10-2 도메인 전용 지시문과 스타일 프리셋 설계

RAG 시스템이 범용적인 챗봇이 아니라, 특정 비즈니스 도메인(법률, 의료, 금융 등)의 전문가처럼 행동하게 하려면 그에 맞는 **페르소나(Persona)**와 **스타일(Style)**을 입혀야 합니다.

### 10-2-1 System Prompt 설계 원칙 심화

Chapter 5에서 잠깐 다루었던 **System Prompt**를 심화하여 적용해 보겠습니다. Converse API의 `system` 파라미터는 모델에게 대화 전체에 걸쳐 적용되는 역할과 규칙을 설정하는 데 사용됩니다. 단순히 "친절하게 답해" 수준이 아니라, **도메인 특화 지식**을 전제 조건으로 깔아주는 것이 핵심입니다.

효과적인 System Prompt를 작성하기 위한 **5가지 설계 원칙**을 소개합니다.

| 원칙 | 설명 | 예시 |
|------|------|------|
| 역할(Role) 명시 | 모델이 수행할 구체적 역할 정의 | "당신은 10년 차 AWS 솔루션즈 아키텍트입니다" |
| 대상(Audience) 정의 | 답변의 수준을 결정하는 기준 | "비전문가인 마케팅 팀을 대상으로 설명합니다" |
| 제약(Constraints) 설정 | 하지 말아야 할 행동 명시 | "추측으로 답변하지 마세요" |
| 형식(Format) 지정 | 출력 구조를 미리 결정 | "항상 번호 매긴 리스트로 답변하세요" |
| 어조(Tone) 규정 | 말투와 분위기 설정 | "전문적이되 친근한 어조를 유지하세요" |

예를 들어 '사내 IT 헬프데스크 봇'을 만든다면 다음과 같이 구체적인 시스템 프롬프트를 작성해야 합니다.

```python
import boto3

client = boto3.client("bedrock-runtime", region_name="us-east-1")

system_prompt = """당신은 10년 차 IT 시스템 관리자입니다.
비전문가인 직원들에게 기술적인 문제를 아주 쉽게 설명해야 합니다.
답변 시 다음 원칙을 따르세요:

<rules>
1. 전문 용어(예: DNS, DHCP)를 사용할 때는 반드시 괄호 안에 풀어서 설명하세요.
2. 해결 방법은 항상 1, 2, 3 번호가 매겨진 리스트 형태로 제공하세요.
3. 각 단계는 비전공자도 따라할 수 있도록 구체적으로 작성하세요.
4. 마지막에는 "해결되지 않으면 IT 지원팀(내선 1234)으로 연락주세요."를 덧붙이세요.
5. 제공된 사내 문서에 없는 내용은 답변하지 마세요.
</rules>"""

messages = [
    {
        "role": "user",
        "content": [{"text": "인터넷이 안 돼요. 어떻게 해야 하나요?"}]
    }
]

response = client.converse(
    modelId="anthropic.claude-sonnet-4-20250514",
    messages=messages,
    system=[{"text": system_prompt}],
    inferenceConfig={"maxTokens": 1024, "temperature": 0.3}
)

print(response["output"]["message"]["content"][0]["text"])
```

이 지시문이 적용되면, 사용자가 "인터넷이 안 돼요"라고 물었을 때 AI는 자동으로 **1, 2, 3 단계**의 해결책을 제시하고 **연락처**까지 안내하게 됩니다.

> **📌 참고:** Claude 모델에서 System Prompt 안에 `<rules>`, `<context>`, `<instructions>` 같은 XML 태그를 사용하면 지시사항의 경계가 명확해져 모델이 규칙을 더 정확하게 따릅니다.

### 10-2-2 출력 스타일 프리셋 (JSON, Markdown, 표 등)

답변의 내용뿐만 아니라 **형식**도 중요합니다. 개발자를 위한 봇이라면 코드는 반드시 마크다운 코드 블록으로 감싸야 하고, 데이터 분석가를 위한 봇이라면 결과를 **JSON**이나 **표(Table)** 형태로 출력해야 할 것입니다.

프롬프트에 **출력 템플릿**을 미리 정의해 주면 **일관된 스타일**을 유지할 수 있습니다. 아래는 세 가지 대표적인 스타일 프리셋 예시입니다.

**프리셋 1: JSON 구조화 출력**

```python
system_json = """당신은 데이터 분석 어시스턴트입니다.
모든 답변을 다음 JSON 형식으로만 출력하세요. 다른 텍스트는 포함하지 마세요.

<output_format>
{
  "summary": "한 줄 요약",
  "details": ["세부 사항 1", "세부 사항 2"],
  "confidence": "high | medium | low",
  "sources": ["참조 출처"]
}
</output_format>"""
```

**프리셋 2: Markdown 보고서 형식**

```python
system_markdown = """당신은 비즈니스 보고서 작성 전문가입니다.
모든 답변을 다음 Markdown 형식으로 작성하세요.

<output_format>
## 주요 안건
(한 줄 요약)

## 결정 사항
- (글머리 기호로 나열)

## Action Item
| 담당자 | 할 일 | 기한 |
|--------|-------|------|
| OOO | OOO | OOO |
</output_format>"""
```

**프리셋 3: 단계별 가이드 형식**

```python
system_guide = """당신은 AWS 기술 지원 전문가입니다.
모든 답변을 다음 형식으로 작성하세요.

<output_format>
🔍 문제 진단: (문제의 원인을 한 줄로 요약)

📋 해결 단계:
1단계: (구체적 행동)
2단계: (구체적 행동)
3단계: (구체적 행동)

⚡ 추가 팁: (관련된 유용한 정보)
</output_format>"""
```

이렇게 형식을 강제하면, 사용자는 매번 똑같은 구조의 깔끔한 결과물을 받아볼 수 있어 **업무 효율**이 크게 향상됩니다. 특히 JSON 프리셋은 후속 프로그램에서 응답을 파싱할 때 매우 유용합니다.

> 📷 **[이미지]** 동일한 질문("AWS 비용을 줄이는 방법은?")에 대해, JSON 프리셋, Markdown 프리셋, 단계별 가이드 프리셋을 각각 적용했을 때의 출력 결과를 나란히 비교.

---

## 10-3 환각(Hallucination) 완화를 위한 프롬프트 패턴

생성형 AI의 가장 큰 리스크는 사실이 아닌 것을 사실처럼 말하는 **환각(Hallucination)** 현상입니다. 특히 기업용 서비스에서 잘못된 정보 제공은 치명적일 수 있습니다. 이를 100% 막을 수는 없지만, **프롬프트 엔지니어링**을 통해 발생 빈도를 크게 줄일 수 있습니다.

> **⚠️ 주의:** 환각은 모델의 학습 과정에서 발생하는 근본적인 문제로, 프롬프트 설계만으로 완전히 제거할 수는 없습니다. 프롬프트 패턴, RAG, Guardrails를 다층적으로 적용하는 것이 최선의 전략입니다.

### 10-3-1 지식의 경계 설정

가장 기본적이면서도 효과적인 방법은 AI에게 **모름을 인정하는 법**을 가르치는 것입니다. AI는 기본적으로 질문에 답을 하려는 성향이 강하기 때문에, 이를 억제하는 **명시적인 지시**가 필요합니다.

```python
system_prompt_with_boundary = """당신은 사내 문서 기반 Q&A 어시스턴트입니다.

<rules>
- 반드시 아래 <context>에 제공된 정보만을 사용하여 답변하세요.
- context에 답변에 필요한 정보가 없다면, 절대로 외부 지식을 사용하지 마세요.
- 정보가 부족한 경우 정확히 다음 문구로 답변하세요:
  "죄송합니다. 제공된 문서에서 해당 정보를 찾을 수 없습니다.
   추가 자료가 필요하시면 담당 부서에 문의해 주세요."
- 부분적으로만 답변 가능한 경우, 답변 가능한 부분만 제공하고
  나머지는 "확인이 필요합니다"라고 명시하세요.
</rules>

<context>
{retrieved_documents}
</context>"""
```

이 패턴을 RAG 시스템과 결합하면 다음과 같이 동작합니다.

```python
import boto3

client = boto3.client("bedrock-runtime", region_name="us-east-1")

def rag_query_with_boundary(question, retrieved_docs):
    """지식 경계가 설정된 RAG 질의"""

    system_text = f"""당신은 사내 문서 기반 Q&A 어시스턴트입니다.

<rules>
- 반드시 아래 <context>에 제공된 정보만을 사용하여 답변하세요.
- context에 정보가 없으면 "제공된 문서에서 해당 정보를 찾을 수 없습니다."라고만 답변하세요.
- 추측하거나 외부 지식으로 보충하지 마세요.
</rules>

<context>
{retrieved_docs}
</context>"""

    response = client.converse(
        modelId="anthropic.claude-sonnet-4-20250514",
        messages=[
            {"role": "user", "content": [{"text": question}]}
        ],
        system=[{"text": system_text}],
        inferenceConfig={"maxTokens": 1024, "temperature": 0.0}
    )

    return response["output"]["message"]["content"][0]["text"]

# 문서에 없는 질문을 했을 때
context = "당사의 연차 휴가는 입사 1년 후 15일이 부여됩니다."
answer = rag_query_with_boundary("점심 메뉴 추천해줘", context)
print(answer)
# 출력: "죄송합니다. 제공된 문서에서 해당 정보를 찾을 수 없습니다."
```

> **📌 참고:** `temperature`를 0.0으로 설정하면 모델의 창의성(=환각 가능성)을 최소화할 수 있습니다. 사실 기반 답변이 필요한 RAG 시스템에서는 낮은 temperature가 필수입니다.

### 10-3-2 근거 기반 추론 유도 (Grounding)

모델에게 답변을 생성할 때 반드시 **문서의 어느 부분에서 가져왔는지**를 인용하도록 강제하는 방법입니다. 근거 없는 문장을 생성하기 어렵게 만드는 구조적 장치입니다.

```python
system_grounding = """당신은 근거 기반으로만 답변하는 어시스턴트입니다.

<instructions>
1. 답변의 각 핵심 문장 끝에 반드시 [출처: 문서명, 페이지] 형식으로 근거를 표기하세요.
2. 근거를 찾을 수 없는 내용은 문장을 생성하지 마세요.
3. 여러 문서에서 정보를 종합한 경우, 모든 출처를 표기하세요.
4. 답변 마지막에 참조한 문서 목록을 정리하세요.
</instructions>

<output_format>
[답변 본문 - 각 문장에 출처 태그 포함]

---
📚 참조 문서:
- 문서명 1 (페이지)
- 문서명 2 (페이지)
</output_format>

<context>
{retrieved_documents}
</context>"""
```

이렇게 하면 모델은 근거가 확실하지 않은 문장은 스스로 생성을 포기하게 되어, 결과적으로 **신뢰할 수 있는 정보**만 남게 됩니다. Chapter 9에서 다뤘던 Citation 메타데이터와 결합하면 더욱 강력한 출처 추적 시스템을 만들 수 있습니다.

> **📌 참고:** 출처 표기를 강제하면 모델의 자기 검증 능력이 작동합니다. 근거를 찾아야 하는 부담이 환각 억제의 자연스러운 안전장치가 됩니다.

### 10-3-3 사실 검증 단계 추가 (Self-Correction)

고급 기법으로, 모델에게 답변을 생성하게 한 뒤 "방금 한 답변이 문맥에 비추어 사실인지 스스로 검증해봐"라고 한 번 더 묻는 **자기 교정(Self-Correction)** 과정을 추가합니다.

이 패턴은 하나의 프롬프트 안에서 **생성-검증-수정**의 3단계를 수행하도록 설계합니다.

```python
system_self_correction = """당신은 정확성을 최우선으로 하는 Q&A 어시스턴트입니다.

다음 3단계를 반드시 순서대로 수행하세요:

<step1_generate>
제공된 context를 바탕으로 질문에 대한 초안 답변을 작성하세요.
</step1_generate>

<step2_verify>
초안 답변의 각 문장을 context와 대조하여 검증하세요.
- 각 문장이 context에서 뒷받침되는가?
- 과장되거나 추론이 지나친 부분은 없는가?
- context에 없는 정보를 추가하지 않았는가?
</step2_verify>

<step3_final>
검증 결과를 반영하여 최종 답변을 작성하세요.
문제가 발견된 문장은 수정하거나 제거하세요.
</step3_final>

출력 형식:
<draft>초안 답변</draft>
<verification>검증 결과</verification>
<final_answer>최종 답변</final_answer>"""
```

이 방식은 토큰을 더 소비하지만, 정확성이 매우 중요한 금융, 의료, 법률 도메인에서는 충분히 그 비용을 정당화할 수 있습니다. `<draft>`와 `<verification>` 태그는 내부 처리용으로 활용하고, `<final_answer>` 내용만 사용자에게 전달하면 됩니다.

> 📷 **[이미지]** Self-Correction 흐름도. [질문+Context] -> [1단계: 초안 생성] -> [2단계: 사실 검증] -> [3단계: 수정된 최종 답변]의 3단계 파이프라인을 시각화.

> **💡 핵심 포인트:** 환각 방지는 단일 기법이 아니라, 지식 경계 설정 + 근거 기반 인용 + 자기 교정을 다층적으로 적용해야 효과적입니다. 여기에 Bedrock Guardrails의 콘텐츠 필터를 추가하면 한층 더 안전한 시스템을 구축할 수 있습니다.

---

## 10-4 모델 평가(Model Evaluation) 실습

지금까지 배운 다양한 기법들을 적용하여 프롬프트를 수정했습니다. 그런데, 수정된 프롬프트가 이전보다 실제로 더 좋다는 것을 어떻게 확신할 수 있을까요? 감에 의존하지 않고 **데이터**로 증명해야 합니다. Amazon Bedrock은 이를 위한 **Model Evaluation** 기능을 기본 제공합니다.

### 10-4-1 Bedrock Model Evaluation 기능 활용

Amazon Bedrock 콘솔에서 **Model Evaluation** 기능을 사용하면, 여러 모델과 프롬프트 조합의 성능을 체계적으로 비교할 수 있습니다. 이 기능은 두 가지 평가 방식을 지원합니다.

| 평가 방식 | 설명 | 적합한 상황 |
|-----------|------|-------------|
| **자동 평가 (Automatic)** | 사전 정의된 지표로 자동 측정 | 대규모 테스트, 모델 간 비교 |
| **사람 평가 (Human)** | 평가자가 직접 응답 품질 판단 | 뉘앙스가 중요한 서비스, 최종 검증 |

**콘솔에서 자동 평가를 설정하는 흐름은 다음과 같습니다:**

1. AWS 콘솔에서 **Amazon Bedrock** > **Model evaluation** 진입
2. **Create evaluation job** 클릭
3. 평가 유형 선택: Automatic evaluation
4. 평가할 모델 선택 (예: Claude Sonnet vs Claude Haiku)
5. 평가 데이터셋 업로드 (JSONL 형식)
6. 평가 지표 선택 (정확성, 독성, 견고성 등)
7. 평가 실행 및 결과 확인

평가 데이터셋은 다음과 같은 JSONL 형식으로 준비합니다.

```json
{"prompt": "AWS Lambda의 최대 실행 시간은?", "referenceResponse": "AWS Lambda 함수의 최대 실행 시간은 15분(900초)입니다."}
{"prompt": "S3 버킷 이름의 제약 조건은?", "referenceResponse": "S3 버킷 이름은 3~63자, 소문자와 숫자, 하이픈만 사용 가능하며 전 세계적으로 고유해야 합니다."}
{"prompt": "Bedrock에서 지원하는 임베딩 모델은?", "referenceResponse": "Amazon Titan Embeddings와 Cohere Embed 모델을 지원합니다."}
```

> **📌 참고:** Model Evaluation의 자동 평가는 텍스트 요약, 질의응답, 텍스트 분류 등 다양한 태스크 유형에 대한 내장 지표를 제공합니다. 프롬프트 변경 전후의 성능을 정량적으로 비교할 때 매우 유용합니다.

### 10-4-2 정확성 관련성 안전성 평가 기준 정의

무엇이 **좋은 답변**인지 명확한 기준을 세워야 합니다. 다음 세 가지 축을 중심으로 평가 프레임워크를 설계합니다.

**1. 정확성(Accuracy)**: 문서의 내용과 일치하는가?
- 사실 오류 없이 정확한 정보를 제공하는가
- 숫자, 날짜, 고유명사가 원본과 일치하는가

**2. 관련성(Relevance)**: 질문에 적합한 답변인가?
- 질문의 핵심을 파악하고 답변했는가
- 불필요한 정보 없이 간결하게 답변했는가

**3. 안전성(Safety)**: 유해하거나 부적절한 내용은 없는가?
- 편향적이거나 차별적인 표현이 없는가
- 환각으로 생성된 허위 정보가 없는가

> **⚠️ 주의:** Bedrock Guardrails를 함께 사용하면 유해 콘텐츠 차단, 민감정보(PII) 보호, 주제 거부(Topic Denial) 등의 안전성 기준을 시스템 수준에서 강제할 수 있습니다. 프롬프트 패턴과 Guardrails는 보완적으로 사용하는 것이 좋습니다.

### 10-4-3 RAG 응답 품질 평가와 개선 사이클

일일이 사람이 읽어보고 점수를 매기는 것은 힘든 일입니다. 놀랍게도, 평가 자체를 **LLM**에게 맡길 수 있습니다. 이를 **LLM-as-a-Judge** 기법이라고 합니다. Bedrock의 Claude 모델을 평가자로 활용하여, 프롬프트 A와 B의 품질을 자동으로 비교해 보겠습니다.

```python
import boto3
import json

client = boto3.client("bedrock-runtime", region_name="us-east-1")

def call_claude(system_text, user_text, temperature=0.0):
    """Claude 호출 헬퍼 함수"""
    response = client.converse(
        modelId="anthropic.claude-sonnet-4-20250514",
        messages=[
            {"role": "user", "content": [{"text": user_text}]}
        ],
        system=[{"text": system_text}],
        inferenceConfig={"maxTokens": 2048, "temperature": temperature}
    )
    return response["output"]["message"]["content"][0]["text"]

# --- 1단계: 두 가지 프롬프트 버전으로 답변 생성 ---
context = """AWS Lambda는 서버리스 컴퓨팅 서비스입니다.
최대 실행 시간은 15분이며, 메모리는 128MB~10,240MB를 설정할 수 있습니다.
지원 런타임은 Python, Node.js, Java, Go, .NET 등입니다."""

question = "Lambda 함수의 제한 사항을 알려주세요."

# Prompt A: 기본형
prompt_a_system = "제공된 문서를 보고 답변해주세요."
answer_a = call_claude(
    f"{prompt_a_system}\n\n<context>\n{context}\n</context>",
    question
)

# Prompt B: 고도화형 (CoT + 지식 경계 + 출력 형식)
prompt_b_system = """당신은 AWS 공인 솔루션즈 아키텍트입니다.

<rules>
- 반드시 <context>에 제공된 정보만 사용하세요.
- 정보가 없으면 "문서에서 확인할 수 없습니다"라고 답변하세요.
- 답변은 Markdown 표 형식으로 정리하세요.
</rules>

<thinking>
먼저 context에서 관련 정보를 찾고, 정확한 수치를 확인한 후 답변을 작성합니다.
</thinking>"""

answer_b = call_claude(
    f"{prompt_b_system}\n\n<context>\n{context}\n</context>",
    question
)

# --- 2단계: LLM-as-a-Judge로 평가 ---
eval_system = """당신은 AI 응답 품질 평가 전문가입니다.
두 답변을 다음 기준으로 1~5점 척도로 평가하고 승자를 선택하세요.

<criteria>
- 정확성: 문서 내용과 일치하는가? (1~5)
- 완결성: 질문에 충분히 답했는가? (1~5)
- 형식: 읽기 쉽고 구조화되어 있는가? (1~5)
- 안전성: 환각이나 허위 정보가 없는가? (1~5)
</criteria>

반드시 다음 JSON 형식으로만 출력하세요:
{
  "answer_a_scores": {"정확성": N, "완결성": N, "형식": N, "안전성": N},
  "answer_b_scores": {"정확성": N, "완결성": N, "형식": N, "안전성": N},
  "winner": "A 또는 B",
  "reasoning": "판단 근거"
}"""

eval_question = f"""질문: {question}

참조 문서:
{context}

답변 A:
{answer_a}

답변 B:
{answer_b}"""

evaluation = call_claude(eval_system, eval_question)
print(evaluation)
```

이러한 평가를 여러 질문에 걸쳐 반복 실행하면, "**Prompt B**가 **Prompt A** 대비 정확성 15%, 형식 30% 향상"과 같이 **정량적인 비교**가 가능해집니다.

> 📷 **[이미지]** RAG 응답 품질 개선 사이클 다이어그램. [프롬프트 설계] -> [답변 생성] -> [LLM-as-a-Judge 평가] -> [지표 분석] -> [프롬프트 개선] -> [다시 답변 생성]의 순환 구조.

**RAG 응답 품질 개선 사이클**을 정리하면 다음과 같습니다.

| 단계 | 활동 | 도구 |
|------|------|------|
| 1. 테스트셋 구성 | 대표 질문 20~50개 + 정답 준비 | JSONL 파일 |
| 2. 기준선 측정 | 현재 프롬프트로 답변 생성 및 평가 | Converse API + LLM-as-a-Judge |
| 3. 프롬프트 개선 | Few-shot, CoT, 환각 방지 패턴 적용 | System Prompt 수정 |
| 4. 재측정 | 개선된 프롬프트로 동일 테스트셋 평가 | Bedrock Model Evaluation |
| 5. 비교 분석 | 기준선 대비 개선 폭 확인 | 점수 비교표 |
| 6. 반복 | 목표 달성까지 3~5단계 반복 | - |

> **💡 핵심 포인트:** 프롬프트 엔지니어링은 "수정 -> 평가 -> 개선"의 반복 사이클입니다. Bedrock Model Evaluation과 LLM-as-a-Judge를 결합하면, 이 사이클을 효율적으로 자동화할 수 있습니다.

---

## 마치며

이번 챕터에서는 단순히 답변을 생성하는 것을 넘어, **Few-shot**과 **Chain-of-Thought**로 모델의 추론 능력을 끌어올리고, **도메인 전용 System Prompt**로 전문가 페르소나를 입히며, **환각 방지 프롬프트 패턴**으로 신뢰성을 확보하고, **Bedrock Model Evaluation**으로 이 모든 개선을 정량적으로 검증하는 방법을 배웠습니다.

특히 Claude 모델의 강점인 **XML 태그 기반 구조화 추론**은 Bedrock 환경에서 프롬프트 엔지니어링의 핵심 무기가 됩니다. `<thinking>`, `<answer>`, `<rules>` 같은 태그를 적극 활용하면 모델의 행동을 정밀하게 제어할 수 있습니다.

이제 우리의 **RAG 시스템**은 두뇌(Brain)가 완성되었습니다. 하지만 현실의 업무는 단순히 문서를 검색하고 답변하는 것을 넘어, 외부 시스템과 연동하여 실제 작업을 수행해야 하는 경우가 많습니다. 다음 Chapter 11에서는 **Bedrock Agents**를 활용하여 모델이 외부 API를 호출하고, Lambda 함수를 실행하며, 복잡한 업무를 자동으로 오케스트레이션하는 **에이전트 시스템**을 구축해 보겠습니다.

> **💡 핵심 포인트:** 프롬프트 엔지니어링은 지속적인 반복 개선을 통해 AI 시스템의 품질을 획기적으로 향상시킵니다. "한 번에 완벽한 프롬프트"는 없습니다. 측정하고, 개선하고, 다시 측정하세요.
