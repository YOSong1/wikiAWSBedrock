우리는 지난 Chapter 8을 통해 **Knowledge Bases**와 **벡터 인덱스**를 구성하고, **Retrieve API**와 **RetrieveAndGenerate API**로 문서를 검색하는 방법을 익혔습니다. 하지만 검색된 결과는 아직 기계적인 텍스트 조각들의 나열일 뿐입니다.

일반 사용자에게 "관련 문서 청크 3개를 찾았습니다"라고 던져주는 것은 친절한 서비스가 아닙니다.

우리가 원하는 것은 **"검색된 문서를 바탕으로, 사용자의 질문에 대해 사람처럼 자연스럽게 설명해 주는 답변"**입니다. 이를 위해 검색된 텍스트를 **LLM(Claude)**에게 전달하고, "이 내용을 읽고 답변을 작성해 줘"라고 시키는 과정이 필요합니다.

이번 챕터에서는 **검색기(Retriever)**와 **생성기(Generator)**를 결합하여 완전한 **RAG(Retrieval-Augmented Generation)** 파이프라인을 구축하고, 답변의 신뢰도를 높이기 위해 **근거(출처)**를 함께 제시하는 방법을 상세히 알아보겠습니다. 나아가 **멀티턴 대화**까지 지원하는 실무형 Q&A 시스템을 직접 구현해 봅니다.

> **💡 핵심 포인트:** 검색과 생성이 결합된 RAG 시스템은 사용자의 질문에 근거 기반의 자연스러운 답변을 제공합니다. Bedrock은 이를 위한 두 가지 경로 -- RetrieveAndGenerate(원콜 방식)와 Retrieve + Converse(커스텀 방식) -- 를 모두 지원합니다.

---

## 9-1 Retriever-Generator 구조 상세 분석

RAG 아키텍처는 크게 정보를 찾는 **검색기(Retriever)**와 정보를 바탕으로 글을 쓰는 **생성기(Generator)**라는 두 개의 두뇌로 이루어져 있습니다. Bedrock은 이 두 요소를 **Knowledge Bases** 서비스 안에서 네이티브로 통합합니다.

### 9-1-1 Bedrock Knowledge Bases의 RAG 아키텍처

Chapter 8에서 우리는 Knowledge Bases를 생성하고 데이터를 인제스트했습니다. 이제 이 Knowledge Base 위에서 RAG가 어떻게 동작하는지 내부 구조를 들여다보겠습니다.

Bedrock Knowledge Bases의 RAG는 크게 두 가지 API 경로로 나뉩니다.

**경로 1: RetrieveAndGenerate API (원콜 방식)**

`bedrock-agent-runtime` 클라이언트의 `retrieve_and_generate()` 메서드를 호출하면, Bedrock이 내부적으로 검색과 생성을 모두 처리합니다. 개발자는 질문만 넘기면 됩니다.

- 장점: 구현이 매우 간단하고, Citation 메타데이터가 자동으로 포함됩니다.
- 단점: 프롬프트를 세밀하게 커스터마이징하기 어렵습니다.

**경로 2: Retrieve API + Converse API (커스텀 방식)**

`retrieve()`로 관련 문서 청크를 먼저 가져온 뒤, 직접 프롬프트를 설계하고 `bedrock-runtime`의 `converse()` API로 Claude에게 전달합니다.

- 장점: 프롬프트, 모델 파라미터, 출력 형식을 완전히 제어할 수 있습니다.
- 단점: 구현 코드가 길어지고 Citation 로직을 직접 작성해야 합니다.

> **📌 참고:** 실무에서는 빠른 프로토타입에 경로 1을, 프로덕션 수준의 커스터마이징이 필요할 때 경로 2를 선택합니다. 이번 챕터에서는 두 경로를 모두 다룹니다.

### 9-1-2 데이터 흐름의 5단계

두 경로 모두 내부적으로는 동일한 5단계 데이터 흐름을 따릅니다.

1. **질문(Query)**: 사용자가 "Knowledge Bases의 청킹 전략은 어떻게 설정하나요?"라고 묻습니다.
2. **임베딩 변환(Embed)**: 질문 텍스트를 Titan Embeddings 또는 Cohere Embed 모델로 벡터화합니다.
3. **검색(Retrieve)**: 벡터화된 질문을 OpenSearch Serverless(또는 다른 벡터 스토어)에서 유사도 검색하여 관련 청크 3~5개를 가져옵니다.
4. **프롬프트 조립(Prompt Assembly)**: 검색된 청크들을 시스템 프롬프트와 함께 하나의 메시지로 조립합니다. "다음 문맥을 참고하여 질문에 답하세요"라는 지시와 함께 묶습니다.
5. **생성(Generate)**: 조립된 프롬프트를 Claude 모델에게 전송하고, 최종 답변을 생성합니다.

이 과정에서 가장 중요한 것은 4번 단계인 **프롬프트 조립**입니다. 검색된 내용을 어떻게 포장해서 AI에게 주느냐에 따라 답변 품질이 천차만별로 달라지기 때문입니다.

> 📷 **[이미지]** [User Query] -> [Embedding Model] -> [Vector Store 검색] -> [Retrieved Chunks] -> [Prompt Template 조립] -> [Claude 생성] -> [Answer + Citations]로 이어지는 Bedrock RAG 파이프라인의 전체 흐름도. Retrieved Chunks가 Prompt Template 안으로 삽입되는 모습을 시각적으로 강조.

---

## 9-2 검색 결과를 프롬프트에 삽입하는 템플릿 설계

RetrieveAndGenerate API를 사용하면 Bedrock이 내부적으로 프롬프트를 조립해 주지만, Retrieve + Converse 조합을 사용할 때는 우리가 직접 **프롬프트 템플릿**을 설계해야 합니다. 이 설계가 답변 품질을 좌우합니다.

### 9-2-1 기본 프롬프트 템플릿 구조

가장 널리 사용되는 RAG 프롬프트 템플릿은 다음과 같은 구조를 가집니다.

```
[시스템 지침 (System Prompt)]
당신은 기업 내부 문서를 기반으로 답변하는 전문 어시스턴트입니다.
반드시 아래의 <context> 태그 안에 있는 내용만을 바탕으로 답변하세요.

<context>
{검색된 문서 청크들}
</context>

[사용자 질문]
{사용자의 원래 질문}
```

여기서 `{검색된 문서 청크들}` 자리에는 **Retrieve API**가 반환한 문서 내용들이 자동으로 들어가고, `{사용자의 원래 질문}` 자리에는 원래 질문이 들어갑니다.

> **📌 참고:** Claude 모델은 XML 태그를 활용한 구조화된 프롬프트를 특히 잘 이해합니다. `<context>`, `<question>`, `<instructions>` 같은 태그로 각 섹션을 구분하면 답변 품질이 향상됩니다.

Converse API에서는 `system` 파라미터와 `messages` 파라미터를 분리하여 전달할 수 있으므로, 다음과 같이 자연스럽게 구성합니다.

```python
import boto3

bedrock_runtime = boto3.client('bedrock-runtime', region_name='us-east-1')

def build_rag_prompt(query, retrieved_chunks):
    """검색 결과를 Converse API 형식으로 조립합니다."""

    # 검색된 청크들을 하나의 문맥 문자열로 결합
    context_text = ""
    for i, chunk in enumerate(retrieved_chunks, 1):
        source = chunk.get('location', {}).get('s3Location', {}).get('uri', '알 수 없음')
        content = chunk.get('content', {}).get('text', '')
        context_text += f"[문서 {i}] (출처: {source})\n{content}\n\n"

    # 시스템 프롬프트 정의
    system_prompt = """당신은 기업 내부 문서를 기반으로 답변하는 전문 어시스턴트입니다.
반드시 아래 <context> 태그 안의 내용만을 바탕으로 답변하세요.
만약 문맥에서 답을 찾을 수 없다면 "제공된 문서에서 해당 정보를 찾을 수 없습니다"라고 답하세요.
답변 시 어떤 문서를 참고했는지 언급해 주세요."""

    # 사용자 메시지에 문맥 포함
    user_message = f"""<context>
{context_text}
</context>

질문: {query}"""

    return system_prompt, user_message
```

### 9-2-2 환각(Hallucination) 방지 전략

RAG의 핵심 목표 중 하나는 AI가 거짓말을 하지 않게 하는 것입니다. Claude는 매우 강력한 모델이지만, 주어진 문맥 밖의 내용을 그럴듯하게 지어낼 수 있습니다. 이를 **환각(Hallucination)**이라 합니다.

프롬프트에 다음과 같은 제약 조건을 명시적으로 포함해야 합니다.

- **"제공된 문맥(context)에 있는 내용으로만 대답하세요."** -- 가장 기본적인 제약입니다.
- **"사전 학습된 지식을 사용하지 마세요."** -- 모델이 훈련 과정에서 학습한 일반 지식을 섞는 것을 방지합니다.
- **"모르면 모른다고 말하세요."** -- "정보가 부족하여 알 수 없습니다"라고 답하도록 명시합니다.
- **"추측하거나 가정하지 마세요."** -- 불확실한 내용을 확신 있게 말하는 것을 방지합니다.

실무에서 효과적인 환각 방지 프롬프트의 예시는 다음과 같습니다.

```python
ANTI_HALLUCINATION_SYSTEM_PROMPT = """당신은 기업 내부 문서 전문 어시스턴트입니다.

<rules>
1. 반드시 <context> 태그 안의 내용만을 바탕으로 답변하세요.
2. 사전 학습된 지식이나 외부 정보를 사용하지 마세요.
3. 문맥에서 답을 찾을 수 없는 경우, "제공된 문서에서 해당 정보를 찾을 수 없습니다"라고 명확히 답하세요.
4. 추측이나 가정을 포함하지 마세요.
5. 답변에 사용한 문서 번호를 함께 언급하세요.
</rules>"""
```

> **⚠️ 주의:** 환각 방지 지침이 없으면 Claude가 검색된 기업 내부 정보와 자신의 일반 지식을 섞어서 사실과 다른 답변을 생성할 수 있습니다. 특히 보안 문서나 사내 규정처럼 정확성이 중요한 도메인에서는 반드시 강력한 제약 조건을 포함하세요.

> 📷 **[이미지]** '나쁜 프롬프트'(단순히 질문만 던짐)와 '좋은 프롬프트'(역할, 제약조건, XML 태그로 구조화된 문맥이 정리됨)를 좌우로 비교. 좋은 프롬프트 쪽에는 system, context, question 각 섹션이 박스로 구분되어 깔끔하게 정리된 모습.

---

## 9-3 근거(출처)를 포함하는 답변 포맷

기업용 검색 시스템에서 가장 중요한 것은 **신뢰성**입니다. AI가 그럴듯한 답변을 내놓더라도, 사용자는 "이거 진짜 맞는 말이야?"라고 의심할 수 있습니다. 이때 답변과 함께 **출처(Source)**를 표기해 준다면 신뢰도가 비약적으로 상승합니다.

### 9-3-1 Citation 메타데이터 활용

Bedrock Knowledge Bases의 가장 큰 강점 중 하나는 **Citation 메타데이터를 자동으로 제공**한다는 점입니다. RetrieveAndGenerate API를 호출하면 응답에 `citations` 필드가 포함되어, 답변의 어떤 부분이 어떤 문서에서 왔는지 추적할 수 있습니다.

RetrieveAndGenerate API의 응답 구조를 살펴보겠습니다.

```python
import boto3
import json

bedrock_agent_runtime = boto3.client('bedrock-agent-runtime', region_name='us-east-1')

response = bedrock_agent_runtime.retrieve_and_generate(
    input={'text': 'Knowledge Bases의 청킹 전략은 어떤 것들이 있나요?'},
    retrieveAndGenerateConfiguration={
        'type': 'KNOWLEDGE_BASE',
        'knowledgeBaseConfiguration': {
            'knowledgeBaseId': 'YOUR_KB_ID',
            'modelArn': 'arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-sonnet-4-20250514'
        }
    }
)

# 생성된 답변 텍스트
answer = response['output']['text']
print(f"[답변]\n{answer}\n")

# Citation 정보 파싱
print("[출처 정보]")
for citation in response.get('citations', []):
    # 이 Citation이 커버하는 답변 텍스트 범위
    generated_part = citation.get('generatedResponsePart', {})
    text_range = generated_part.get('textResponsePart', {}).get('text', '')

    # 참조한 문서들
    for ref in citation.get('retrievedReferences', []):
        source_uri = ref.get('location', {}).get('s3Location', {}).get('uri', '알 수 없음')
        content_snippet = ref.get('content', {}).get('text', '')[:100]
        print(f"  - 출처: {source_uri}")
        print(f"    내용 미리보기: {content_snippet}...")
        print()
```

> **💡 핵심 포인트:** `citations` 배열의 각 항목은 답변의 특정 부분(`generatedResponsePart`)과 그 근거가 된 문서(`retrievedReferences`)를 매핑합니다. 이를 통해 "답변의 이 문장은 이 문서에서 왔다"는 추적이 가능합니다.

### 9-3-2 출처 표기 전략

답변에 출처를 보여주는 방법은 크게 두 가지입니다.

**방식 A: 답변 내 인라인 인용 (Inline Citation)**

- AI에게 지시하여 문장 끝에 참조 번호를 달게 하는 방식입니다.
- 예: "Knowledge Bases는 고정 크기, 시맨틱, 계층적 청킹을 지원합니다 [문서 1]."
- 모델 성능에 따라 참조 번호가 누락되거나 잘못 매핑될 수 있습니다.

**방식 B: 답변 하단 부록 (Recommended)**

- LLM이 답변을 생성한 후, Python 코드 레벨에서 검색에 사용된 문서의 메타데이터를 하단에 리스트로 붙여주는 방식입니다.
- 구현이 확실하고 출처 누락이 없어 가장 권장됩니다.

**방식 C: RetrieveAndGenerate의 Citation 활용 (AWS 네이티브)**

- Bedrock이 자동으로 제공하는 Citation 메타데이터를 파싱하여 표시하는 방식입니다.
- 답변의 어떤 부분이 어떤 문서에서 왔는지 정확하게 매핑됩니다.

실무에서는 **방식 B 또는 C**를 채택하는 것이 안정적입니다. 다음은 방식 B를 구현한 출처 표기 함수입니다.

```python
def format_answer_with_sources(answer, retrieved_references):
    """답변 하단에 출처 목록을 추가합니다."""
    output = f"[답변]\n{answer}\n"
    output += "\n" + "-" * 50 + "\n"
    output += "[참고 문서]\n"

    seen_sources = set()
    for i, ref in enumerate(retrieved_references, 1):
        s3_uri = ref.get('location', {}).get('s3Location', {}).get('uri', '알 수 없음')
        # 중복 출처 제거
        if s3_uri not in seen_sources:
            seen_sources.add(s3_uri)
            # S3 URI에서 파일명만 추출
            file_name = s3_uri.split('/')[-1] if '/' in s3_uri else s3_uri
            print(f"  {i}. {file_name}")
            print(f"     전체 경로: {s3_uri}")

    return output
```

> 📷 **[이미지]** 최종 답변 화면 예시. 상단에는 AI의 자연어 답변이 나오고, 하단에는 구분선과 함께 [참고 문서] 1. company_policy.pdf 2. bedrock_guide.pdf 처럼 출처가 깔끔하게 나열된 모습.

---

## 9-4 FAQ 및 매뉴얼 문서 기반 Q&A 시스템 구현

이제 이론적인 준비는 끝났습니다. Chapter 8에서 만든 **Knowledge Base**와 **프롬프트 템플릿**, 그리고 **Claude**를 결합하여 실제 작동하는 질의응답 시스템을 완성해 보겠습니다.

### 9-4-1 Knowledge Base + Converse API 조합

RetrieveAndGenerate API도 편리하지만, 프롬프트를 세밀하게 제어하려면 **Retrieve API로 검색 후 Converse API로 생성**하는 패턴이 필수입니다. 이 조합은 실무에서 가장 많이 사용되는 RAG 구현 패턴입니다.

먼저 필요한 클라이언트를 준비합니다.

```python
import boto3
import json

# 검색용 클라이언트 (Knowledge Base 접근)
bedrock_agent_runtime = boto3.client('bedrock-agent-runtime', region_name='us-east-1')

# 생성용 클라이언트 (Claude 모델 호출)
bedrock_runtime = boto3.client('bedrock-runtime', region_name='us-east-1')

KNOWLEDGE_BASE_ID = "YOUR_KB_ID"  # Chapter 8에서 생성한 Knowledge Base ID
MODEL_ID = "anthropic.claude-sonnet-4-20250514"
```

> **📌 참고:** `bedrock-agent-runtime`은 Knowledge Bases, Agents 등 Bedrock의 에이전트 관련 기능에 접근하는 클라이언트이고, `bedrock-runtime`은 Foundation Model을 직접 호출하는 클라이언트입니다. 두 클라이언트의 역할을 구분해 두세요.

다음으로, 검색과 생성을 결합하는 핵심 함수를 작성합니다.

```python
def retrieve_context(query, kb_id, top_k=5):
    """Knowledge Base에서 관련 문서를 검색합니다."""
    response = bedrock_agent_runtime.retrieve(
        knowledgeBaseId=kb_id,
        retrievalQuery={'text': query},
        retrievalConfiguration={
            'vectorSearchConfiguration': {
                'numberOfResults': top_k
            }
        }
    )

    results = []
    for item in response.get('retrievalResults', []):
        results.append({
            'content': item.get('content', {}).get('text', ''),
            'source': item.get('location', {}).get('s3Location', {}).get('uri', ''),
            'score': item.get('score', 0)
        })

    return results


def generate_answer(query, context_chunks):
    """검색 결과를 바탕으로 Claude에게 답변을 생성하게 합니다."""

    # 문맥 문자열 조립
    context_text = ""
    for i, chunk in enumerate(context_chunks, 1):
        context_text += f"<document index=\"{i}\" source=\"{chunk['source']}\">\n"
        context_text += f"{chunk['content']}\n"
        context_text += f"</document>\n\n"

    # 시스템 프롬프트 (환각 방지 포함)
    system_prompt = """당신은 기업 내부 문서를 기반으로 답변하는 전문 어시스턴트입니다.

<rules>
1. 반드시 <context> 태그 안의 문서 내용만을 바탕으로 답변하세요.
2. 사전 학습된 지식이나 외부 정보를 절대 사용하지 마세요.
3. 문맥에서 답을 찾을 수 없으면 "제공된 문서에서 해당 정보를 찾을 수 없습니다"라고 답하세요.
4. 답변에 참고한 문서 번호(예: [문서 1])를 함께 표기하세요.
5. 한국어로 친절하고 명확하게 답변하세요.
</rules>"""

    # 사용자 메시지
    user_message = f"""<context>
{context_text}
</context>

질문: {query}"""

    # Converse API 호출
    response = bedrock_runtime.converse(
        modelId=MODEL_ID,
        system=[{'text': system_prompt}],
        messages=[
            {
                'role': 'user',
                'content': [{'text': user_message}]
            }
        ],
        inferenceConfig={
            'maxTokens': 1024,
            'temperature': 0,
            'topP': 0.9
        }
    )

    answer = response['output']['message']['content'][0]['text']
    return answer


def ask_question(query):
    """검색 + 생성을 결합한 RAG Q&A 함수입니다."""

    # 1단계: Knowledge Base에서 검색
    print(f"\n{'='*60}")
    print(f"[질문] {query}")
    print(f"{'='*60}")

    context_chunks = retrieve_context(query, KNOWLEDGE_BASE_ID)
    print(f"\n[검색 결과] {len(context_chunks)}개의 관련 문서를 찾았습니다.")

    if not context_chunks:
        print("[답변] 관련 문서를 찾지 못했습니다.")
        return

    # 2단계: Claude로 답변 생성
    answer = generate_answer(query, context_chunks)

    print(f"\n[답변]\n{answer}")

    # 3단계: 출처 표시
    print(f"\n{'-'*50}")
    print("[참고 문서]")
    seen = set()
    for i, chunk in enumerate(context_chunks, 1):
        source = chunk['source']
        if source not in seen:
            seen.add(source)
            file_name = source.split('/')[-1] if '/' in source else source
            score = chunk['score']
            print(f"  {i}. {file_name} (유사도: {score:.4f})")

    return answer
```

이제 질문을 던져보겠습니다.

```python
# 테스트 실행
ask_question("Knowledge Bases에서 지원하는 청킹 전략에는 어떤 것들이 있나요?")
```

**실행 결과 예시:**

```
============================================================
[질문] Knowledge Bases에서 지원하는 청킹 전략에는 어떤 것들이 있나요?
============================================================

[검색 결과] 5개의 관련 문서를 찾았습니다.

[답변]
Knowledge Bases에서는 다음과 같은 청킹 전략을 지원합니다 [문서 1][문서 3]:

1. **고정 크기 청킹(Fixed-size Chunking)**: 문서를 일정한 토큰 수 단위로 분할합니다.
   청크 간 중첩(overlap)을 설정하여 문맥 손실을 줄일 수 있습니다.

2. **시맨틱 청킹(Semantic Chunking)**: 문서의 의미적 단위를 기준으로 분할합니다.
   관련 내용이 하나의 청크에 포함되어 검색 품질이 높아집니다.

3. **계층적 청킹(Hierarchical Chunking)**: 부모-자식 관계의 계층 구조로 분할합니다.

--------------------------------------------------
[참고 문서]
  1. bedrock_chunking_guide.pdf (유사도: 0.8721)
  2. knowledge_bases_manual.pdf (유사도: 0.8156)
```

> **💡 핵심 포인트:** `retrieve()` + `converse()` 조합은 코드가 길어지지만, 시스템 프롬프트를 자유롭게 커스터마이징할 수 있어 프로덕션 환경에서 선호됩니다.

### 9-4-2 멀티턴 RAG 대화 흐름 설계

실제 Q&A 시스템에서 사용자는 한 번의 질문으로 끝나지 않습니다. 첫 번째 답변을 보고 "좀 더 자세히 설명해 줘", "그럼 비용은 어떻게 되는데?" 같은 후속 질문을 이어갑니다. 이를 **멀티턴 대화**라고 합니다.

멀티턴 RAG에서 핵심은 **대화 이력을 유지하면서, 매 턴마다 새로운 검색을 수행**하는 것입니다.

```python
class MultiTurnRAGChat:
    """멀티턴 대화를 지원하는 RAG 챗봇 클래스입니다."""

    def __init__(self, kb_id, model_id):
        self.kb_id = kb_id
        self.model_id = model_id
        self.conversation_history = []  # Converse API 형식의 대화 이력
        self.bedrock_agent_runtime = boto3.client('bedrock-agent-runtime', region_name='us-east-1')
        self.bedrock_runtime = boto3.client('bedrock-runtime', region_name='us-east-1')

        self.system_prompt = """당신은 기업 내부 문서를 기반으로 답변하는 전문 어시스턴트입니다.

<rules>
1. <context> 태그 안의 문서 내용만을 바탕으로 답변하세요.
2. 이전 대화 맥락을 고려하되, 새로 제공된 문서 내용을 우선합니다.
3. 문맥에서 답을 찾을 수 없으면 "제공된 문서에서 해당 정보를 찾을 수 없습니다"라고 답하세요.
4. 한국어로 친절하고 명확하게 답변하세요.
</rules>"""

    def _retrieve(self, query, top_k=5):
        """Knowledge Base에서 관련 문서를 검색합니다."""
        response = self.bedrock_agent_runtime.retrieve(
            knowledgeBaseId=self.kb_id,
            retrievalQuery={'text': query},
            retrievalConfiguration={
                'vectorSearchConfiguration': {
                    'numberOfResults': top_k
                }
            }
        )
        return response.get('retrievalResults', [])

    def _build_user_message(self, query, retrieved_results):
        """검색 결과를 포함한 사용자 메시지를 구성합니다."""
        context_text = ""
        for i, item in enumerate(retrieved_results, 1):
            content = item.get('content', {}).get('text', '')
            source = item.get('location', {}).get('s3Location', {}).get('uri', '')
            context_text += f"<document index=\"{i}\" source=\"{source}\">\n{content}\n</document>\n\n"

        return f"""<context>
{context_text}
</context>

질문: {query}"""

    def chat(self, query):
        """한 턴의 대화를 처리합니다."""

        # 1. 검색
        retrieved_results = self._retrieve(query)

        # 2. 사용자 메시지 구성 (검색 결과 포함)
        user_message = self._build_user_message(query, retrieved_results)

        # 3. 대화 이력에 사용자 메시지 추가
        self.conversation_history.append({
            'role': 'user',
            'content': [{'text': user_message}]
        })

        # 4. Converse API 호출 (전체 대화 이력 전달)
        response = self.bedrock_runtime.converse(
            modelId=self.model_id,
            system=[{'text': self.system_prompt}],
            messages=self.conversation_history,
            inferenceConfig={
                'maxTokens': 1024,
                'temperature': 0,
                'topP': 0.9
            }
        )

        # 5. 어시스턴트 응답 추출 및 이력에 추가
        assistant_message = response['output']['message']
        self.conversation_history.append(assistant_message)

        answer = assistant_message['content'][0]['text']

        # 6. 출처 정보 수집
        sources = []
        seen = set()
        for item in retrieved_results:
            uri = item.get('location', {}).get('s3Location', {}).get('uri', '')
            if uri and uri not in seen:
                seen.add(uri)
                sources.append(uri.split('/')[-1])

        return {
            'answer': answer,
            'sources': sources,
            'num_chunks': len(retrieved_results)
        }

    def reset(self):
        """대화 이력을 초기화합니다."""
        self.conversation_history = []
        print("대화 이력이 초기화되었습니다.")
```

> **⚠️ 주의:** 멀티턴 대화에서 대화 이력이 길어지면 토큰 한도를 초과할 수 있습니다. 실무에서는 최근 N턴만 유지하거나, 오래된 이력을 요약하는 전략이 필요합니다. Claude Sonnet은 200K 토큰 컨텍스트를 지원하지만, 비용 효율을 위해 대화 이력을 적절히 관리하는 것이 좋습니다.

이제 멀티턴 대화를 실행해 보겠습니다.

```python
# 멀티턴 RAG 챗봇 초기화
chatbot = MultiTurnRAGChat(
    kb_id="YOUR_KB_ID",
    model_id="anthropic.claude-sonnet-4-20250514"
)

# 첫 번째 질문
result = chatbot.chat("Knowledge Bases의 데이터 소스로 어떤 것을 사용할 수 있나요?")
print(f"[답변] {result['answer']}")
print(f"[참고 문서] {', '.join(result['sources'])}")

# 후속 질문 (이전 맥락을 이어감)
result = chatbot.chat("그중에서 S3를 사용할 때 주의할 점은 뭐가 있어?")
print(f"[답변] {result['answer']}")
print(f"[참고 문서] {', '.join(result['sources'])}")

# 또 다른 후속 질문
result = chatbot.chat("비용은 어떻게 청구되나요?")
print(f"[답변] {result['answer']}")
print(f"[참고 문서] {', '.join(result['sources'])}")
```

**실행 결과 예시:**

```
[답변] Knowledge Bases의 데이터 소스로는 다음을 사용할 수 있습니다:
1. Amazon S3 버킷: PDF, TXT, HTML, DOCX, CSV 등 다양한 형식의 파일을 저장
2. 웹 크롤러: 특정 웹사이트의 콘텐츠를 자동으로 수집
3. Confluence: Atlassian Confluence 워크스페이스 연동
4. Salesforce: Salesforce 데이터 연동
5. SharePoint: Microsoft SharePoint 연동
[참고 문서] data_source_guide.pdf, kb_setup_manual.pdf

[답변] S3를 데이터 소스로 사용할 때 주의할 점은 다음과 같습니다:
1. 버킷은 Knowledge Base와 같은 리전에 있어야 합니다.
2. IAM 역할에 S3 읽기 권한이 부여되어야 합니다.
3. 파일 크기가 너무 크면 처리 시간이 길어질 수 있으므로 적절히 분할하세요.
...
[참고 문서] s3_best_practices.pdf, kb_setup_manual.pdf
```

멀티턴 대화에서는 두 번째 질문의 "그중에서"가 첫 번째 답변의 데이터 소스 목록을 참조하고 있음을 Claude가 대화 이력을 통해 이해합니다.

### 9-4-3 최종 테스트와 응답 품질 검증

시스템이 완성되었으면, 다양한 유형의 질문으로 응답 품질을 검증해야 합니다. 다음 세 가지 시나리오를 반드시 테스트하세요.

**시나리오 1: 문서에 답이 있는 질문**

```python
result = chatbot.chat("Bedrock에서 지원하는 임베딩 모델은 무엇인가요?")
# 기대: 문서 내용을 바탕으로 정확한 답변 + 출처 표시
```

**시나리오 2: 문서에 답이 없는 질문**

```python
result = chatbot.chat("오늘 서울 날씨는 어떤가요?")
# 기대: "제공된 문서에서 해당 정보를 찾을 수 없습니다" 응답
```

**시나리오 3: 모호한 질문**

```python
result = chatbot.chat("비용이 얼마나 드나요?")
# 기대: 검색된 문맥에 따라 답변하되, 정보가 부족하면 솔직히 인정
```

응답 품질을 체계적으로 평가하기 위한 간단한 검증 함수도 만들어 둡시다.

```python
def evaluate_response(query, expected_keywords, chatbot):
    """응답 품질을 간단히 검증합니다."""
    result = chatbot.chat(query)
    answer = result['answer']

    # 기대 키워드 포함 여부 확인
    found = [kw for kw in expected_keywords if kw in answer]
    missing = [kw for kw in expected_keywords if kw not in answer]

    print(f"\n[평가] 질문: {query}")
    print(f"  - 포함된 키워드: {found}")
    print(f"  - 누락된 키워드: {missing}")
    print(f"  - 출처 수: {len(result['sources'])}")
    print(f"  - 검색 청크 수: {result['num_chunks']}")

    score = len(found) / len(expected_keywords) * 100 if expected_keywords else 0
    print(f"  - 키워드 매칭률: {score:.0f}%")

    return score

# 테스트 케이스 실행
chatbot.reset()
test_cases = [
    ("임베딩 모델의 종류는?", ["Titan", "Cohere"]),
    ("청킹 전략을 설명해주세요", ["고정 크기", "시맨틱"]),
    ("오늘 날씨는?", ["찾을 수 없습니다"]),
]

scores = []
for query, keywords in test_cases:
    score = evaluate_response(query, keywords, chatbot)
    scores.append(score)
    chatbot.reset()  # 각 테스트 간 대화 이력 초기화

avg_score = sum(scores) / len(scores)
print(f"\n{'='*50}")
print(f"전체 평균 키워드 매칭률: {avg_score:.0f}%")
```

> **💡 핵심 포인트:** 검증 과정에서 매칭률이 낮다면, (1) 검색 결과의 품질(청킹 전략, 임베딩 모델)을 점검하거나 (2) 프롬프트의 지시문을 조정하세요. RAG 시스템의 품질은 검색 품질과 프롬프트 품질의 곱으로 결정됩니다.

---

## 마치며

이번 챕터에서는 **검색기(Retriever)**와 **생성기(Generator)**를 결합한 완전한 RAG 질의응답 시스템을 구축했습니다. 핵심 내용을 정리하면 다음과 같습니다.

- **Bedrock Knowledge Bases의 두 가지 RAG 경로**: 간편한 `RetrieveAndGenerate` API와 커스터마이징 가능한 `Retrieve` + `Converse` API 조합을 배웠습니다.
- **프롬프트 템플릿 설계**: XML 태그를 활용한 구조화된 프롬프트와 환각 방지 전략을 적용했습니다.
- **출처 표기**: Citation 메타데이터를 파싱하여 답변의 신뢰성을 높이는 방법을 구현했습니다.
- **멀티턴 RAG 대화**: 대화 이력을 유지하면서 매 턴마다 새로운 검색을 수행하는 챗봇 클래스를 설계했습니다.

이제 우리의 시스템은 단순한 검색 도구가 아니라, **근거를 가지고 논리적으로 답변하는 지능형 Q&A 어시스턴트**로 진화했습니다.

하지만 아직 개선의 여지가 있습니다. 같은 검색 결과라도 프롬프트를 어떻게 설계하느냐에 따라 답변의 정확도, 형식, 톤이 크게 달라집니다. 다음 **Chapter 10**에서는 **프롬프트 엔지니어링 고도화**를 통해 Few-shot, Chain-of-Thought 등의 기법을 적용하고, 모델 평가(Model Evaluation)로 응답 품질을 체계적으로 개선하는 방법을 다루겠습니다.

> **💡 핵심 포인트:** RAG 시스템의 완성을 위해서는 검색 결과를 효과적으로 프롬프트에 삽입하고, 출처를 명확하게 표시하며, 멀티턴 대화를 안정적으로 지원하는 것이 핵심입니다. 검색 품질(Chapter 8)과 프롬프트 품질(Chapter 10)이 합쳐져 최종 답변 품질이 결정됩니다.
