# Chapter 08. Knowledge Bases와 벡터 인덱스 구성

앞선 Chapter 7에서 우리는 PDF 문서를 파싱하고, 적절한 크기의 청크(Chunk)로 분할하며, 메타데이터를 설계하고, S3 데이터 소스까지 구성하는 전처리 파이프라인을 완성했습니다. 이제 우리 앞에는 수백, 수천 개의 텍스트 조각이 S3 버킷에 정돈되어 놓여 있습니다. 하지만 이 조각들을 단순히 S3에 저장해 두는 것만으로는 부족합니다. 사용자가 질문을 던졌을 때, 수천 개의 조각 중에서 **의미적으로 가장 관련 있는 정답**을 밀리초 단위로 찾아낼 수 있어야 하기 때문입니다.

이를 위해서는 텍스트 조각들을 **벡터(숫자 배열)**로 변환하여 특별한 데이터베이스에 저장해야 합니다. 이 데이터베이스가 바로 **벡터 스토어(Vector Store)**이고, AWS에서는 **Knowledge Bases for Amazon Bedrock**이라는 완전 관리형 서비스를 통해 벡터 스토어 구축부터 검색, 생성까지 한 번에 처리할 수 있습니다.

이번 챕터에서는 벡터 스토어의 원리를 이해하고, Knowledge Bases를 생성하여 데이터를 인제스트(Ingest)하고, 시맨틱 검색과 RAG 호출까지 End-to-End로 실습하겠습니다.

> **💡 핵심 포인트:**
> Knowledge Bases for Amazon Bedrock은 벡터 스토어 관리, 임베딩 생성, 시맨틱 검색, RAG 호출을 하나의 관리형 서비스로 통합합니다. 개발자는 인프라 관리 대신 비즈니스 로직에 집중할 수 있습니다.

## 8-1 벡터 스토어 개념과 선택

### 8-1-1 벡터 스토어란 무엇인가?

우리가 흔히 사용하는 SQL 데이터베이스(MySQL, PostgreSQL 등)는 **키워드 매칭**에 최적화되어 있습니다. 예를 들어 `SELECT * FROM docs WHERE text LIKE '%배터리%'`와 같은 쿼리를 실행하면, '배터리'라는 단어가 정확히 포함된 문서만 찾아줍니다. 하지만 사용자가 "전원이 빨리 닳아요"라고 검색하면, '배터리'라는 단어가 없기 때문에 검색에 실패합니다.

**벡터 스토어**는 이 한계를 극복합니다. 텍스트를 고차원의 **벡터 공간(Vector Space)**에 점으로 찍어두고, 질문이 들어오면 그 질문의 벡터 위치와 **가장 가까운 점들**을 찾아냅니다. "전원이 빨리 닳아요"와 "배터리 수명 관리"가 벡터 공간에서 서로 가까이 위치하기 때문에, 키워드가 달라도 의미적으로 관련된 문서를 찾아낼 수 있는 것입니다.

벡터 스토어의 핵심 동작을 정리하면 다음과 같습니다.

1. **저장(Indexing)**: 텍스트 청크를 임베딩 모델로 벡터 변환 후 인덱스에 저장
2. **검색(Search)**: 사용자 질문을 벡터로 변환하여 가장 가까운 k개의 벡터를 반환
3. **필터링(Filtering)**: 메타데이터 조건을 결합하여 검색 범위를 좁힘

> **📌 참고:**
> SQL 데이터베이스는 정확한 키워드 매칭에 강하고, 벡터 스토어는 의미 기반 유사도 검색에 강합니다. RAG 시스템에서는 두 가지를 조합한 **하이브리드 검색**도 활용됩니다.

### 8-1-2 Amazon OpenSearch Serverless vs Pinecone vs 기타 옵션

Knowledge Bases for Amazon Bedrock은 다양한 벡터 스토어 백엔드를 지원합니다. 프로젝트의 규모, 예산, 운영 방식에 따라 적절한 선택이 필요합니다.

#### Amazon OpenSearch Serverless (기본 권장)

Knowledge Bases를 생성할 때 **기본(Default)**으로 선택되는 벡터 스토어입니다.

- AWS 콘솔에서 Knowledge Base를 만들면, OpenSearch Serverless 컬렉션이 자동으로 프로비저닝됩니다.
- 별도의 클러스터 관리 없이 자동 스케일링되며, 서버리스 요금 모델로 사용한 만큼만 비용을 지불합니다.
- AWS IAM과 자연스럽게 통합되어 보안 설정이 간편합니다.
- 대규모 데이터(수백만 건 이상)에서도 안정적인 검색 성능을 제공합니다.

#### Amazon Aurora PostgreSQL (pgvector)

이미 Aurora PostgreSQL을 사용 중인 조직이라면 **pgvector** 확장을 활용할 수 있습니다.

- 기존 RDS/Aurora 인프라를 그대로 활용하므로 추가 서비스 없이 벡터 검색을 도입할 수 있습니다.
- SQL과 벡터 검색을 하나의 데이터베이스에서 통합 운영할 수 있습니다.
- 다만, 대규모 벡터 검색에서는 OpenSearch Serverless 대비 성능 튜닝이 필요할 수 있습니다.

#### Pinecone (외부 서비스)

AWS 외부의 전문 벡터 데이터베이스입니다.

- 벡터 검색에 특화된 SaaS로, 글로벌 규모의 검색 워크로드에 적합합니다.
- Knowledge Bases와의 연동이 공식 지원되지만, API 키 관리와 네트워크 설정이 추가로 필요합니다.
- 멀티 클라우드 환경에서 벡터 스토어를 통합하고 싶을 때 유용합니다.

#### Redis Enterprise Cloud

실시간 캐싱과 벡터 검색을 동시에 필요로 하는 경우에 적합합니다.

| 벡터 스토어 | 관리 편의성 | AWS 통합 | 비용 모델 | 추천 시나리오 |
|---|---|---|---|---|
| OpenSearch Serverless | 매우 높음 (자동 생성) | 네이티브 | 서버리스 (OCU 기반) | 기본 선택, 대부분의 프로젝트 |
| Aurora pgvector | 보통 | 높음 | RDS 인스턴스 비용 | 기존 Aurora 사용 조직 |
| Pinecone | 보통 | API 연동 | SaaS 구독 | 멀티 클라우드, 대규모 검색 |
| Redis Enterprise | 보통 | API 연동 | 구독 기반 | 실시간 캐싱 + 검색 |

이번 실습에서는 가장 간편하고 AWS 네이티브 통합이 뛰어난 **Amazon OpenSearch Serverless**를 사용하겠습니다. Knowledge Bases 생성 시 자동으로 프로비저닝되므로 별도 설정이 필요 없습니다.

> 📷 **[이미지]** Knowledge Bases for Amazon Bedrock을 중심으로, S3 데이터 소스에서 Titan Embeddings를 거쳐 OpenSearch Serverless/Pinecone/Aurora pgvector 등 다양한 벡터 스토어로 연결되는 아키텍처 다이어그램

---

## 8-2 Knowledge Bases 생성 및 구성

이제 실제로 **Knowledge Bases for Amazon Bedrock**을 생성하고, 데이터를 인제스트하여 검색 가능한 상태로 만들어 보겠습니다.

### 8-2-1 Knowledge Base 생성 흐름 (콘솔 + API)

#### AWS 콘솔을 통한 생성

가장 직관적인 방법은 AWS Management Console을 사용하는 것입니다. 전체 흐름은 다음과 같습니다.

1. **Amazon Bedrock 콘솔** 접속 후 좌측 메뉴에서 **Knowledge bases** 클릭
2. **Create knowledge base** 버튼 클릭
3. Knowledge base 이름과 설명 입력 (예: `my-company-docs-kb`)
4. IAM 역할 선택: **Create and use a new service role** 권장
5. 데이터 소스로 **Amazon S3** 선택 후 버킷 URI 입력 (예: `s3://my-rag-documents/`)
6. 임베딩 모델 선택: **Titan Embeddings G1 - Text v2** 권장
7. 벡터 스토어: **Quick create a new vector store** (OpenSearch Serverless 자동 생성)
8. **Create knowledge base** 클릭

> **⚠️ 주의:**
> Knowledge Base 생성 시 자동으로 만들어지는 OpenSearch Serverless 컬렉션에는 비용이 발생합니다. 실습이 끝나면 반드시 리소스를 정리하세요. OpenSearch Serverless의 최소 OCU(OpenSearch Compute Unit) 비용은 시간당 약 $0.24입니다.

#### boto3 API를 통한 생성

프로그래밍 방식으로 Knowledge Base를 생성하려면 `bedrock-agent` 클라이언트를 사용합니다. 아래 코드는 전체 생성 흐름을 보여줍니다.

```python
import boto3
import json
import time

# Bedrock Agent 클라이언트 생성
bedrock_agent = boto3.client('bedrock-agent', region_name='us-east-1')

# 1. Knowledge Base 생성
response = bedrock_agent.create_knowledge_base(
    name='my-company-docs-kb',
    description='회사 내부 문서 기반 RAG 지식 베이스',
    roleArn='arn:aws:iam::123456789012:role/AmazonBedrockExecutionRoleForKB',
    knowledgeBaseConfiguration={
        'type': 'VECTOR',
        'vectorKnowledgeBaseConfiguration': {
            'embeddingModelArn': 'arn:aws:bedrock:us-east-1::foundation-model/amazon.titan-embed-text-v2:0'
        }
    },
    storageConfiguration={
        'type': 'OPENSEARCH_SERVERLESS',
        'opensearchServerlessConfiguration': {
            'collectionArn': 'arn:aws:aoss:us-east-1:123456789012:collection/my-kb-collection',
            'vectorIndexName': 'bedrock-knowledge-base-default-index',
            'fieldMapping': {
                'vectorField': 'bedrock-knowledge-base-default-vector',
                'textField': 'AMAZON_BEDROCK_TEXT_CHUNK',
                'metadataField': 'AMAZON_BEDROCK_METADATA'
            }
        }
    }
)

knowledge_base_id = response['knowledgeBase']['knowledgeBaseId']
print(f"Knowledge Base 생성 완료! ID: {knowledge_base_id}")
```

> **📌 참고:**
> 콘솔에서 "Quick create" 옵션을 사용하면 OpenSearch Serverless 컬렉션, 인덱스, IAM 역할이 자동으로 생성됩니다. API로 직접 생성할 때는 이 리소스들을 사전에 준비해야 합니다. 입문 단계에서는 콘솔 방식을 권장합니다.

### 8-2-2 임베딩 모델과 벡터 스토어 연결

Knowledge Base를 생성할 때 가장 중요한 설정 중 하나가 **임베딩 모델** 선택입니다. Chapter 6에서 학습한 임베딩 개념이 여기서 실제로 적용됩니다.

Knowledge Bases에서 지원하는 주요 임베딩 모델은 다음과 같습니다.

| 모델 | 차원 수 | 최대 토큰 | 특징 |
|---|---|---|---|
| Titan Embeddings G1 - Text v2 | 1024 (기본) | 8,192 | AWS 네이티브, 다국어 지원 |
| Titan Embeddings G1 - Text v1 | 1536 | 8,192 | 이전 버전, 호환성 유지 |
| Cohere Embed - English v3 | 1024 | 512 | 영어 특화, 높은 정확도 |
| Cohere Embed - Multilingual v3 | 1024 | 512 | 다국어, 100+ 언어 지원 |

한국어 문서를 다루는 프로젝트라면 **Titan Embeddings G1 - Text v2**가 가장 균형 잡힌 선택입니다. AWS 네이티브 모델이라 추가 비용 없이 Knowledge Bases와 자연스럽게 통합되며, 다국어(한국어 포함) 성능도 우수합니다.

임베딩 모델이 생성하는 벡터의 차원 수는 벡터 스토어의 인덱스 설정과 정확히 일치해야 합니다. Titan Embeddings v2의 기본 차원인 1024차원을 사용한다면, OpenSearch Serverless 인덱스도 1024차원으로 설정되어야 합니다. Knowledge Bases의 자동 설정을 사용하면 이 매핑이 자동으로 처리됩니다.

### 8-2-3 데이터 인제스트(Ingest)와 인덱싱

Knowledge Base를 생성하고 S3 데이터 소스를 연결했다면, 이제 **데이터 인제스트(Ingest)**를 실행하여 문서를 벡터로 변환하고 인덱스에 저장해야 합니다.

인제스트 과정에서 내부적으로 일어나는 일은 다음과 같습니다.

1. S3 버킷에서 문서를 읽어옴
2. 문서를 설정된 청킹 전략에 따라 분할
3. 각 청크를 임베딩 모델(Titan Embeddings)로 벡터 변환
4. 벡터와 원본 텍스트, 메타데이터를 벡터 스토어에 저장

#### 콘솔에서 인제스트 실행

Knowledge Base 상세 페이지에서 **Data source** 섹션의 **Sync** 버튼을 클릭하면 인제스트가 시작됩니다.

#### API로 인제스트 실행

```python
# 데이터 소스 생성 (S3 연결)
ds_response = bedrock_agent.create_data_source(
    knowledgeBaseId=knowledge_base_id,
    name='company-docs-s3',
    dataSourceConfiguration={
        'type': 'S3',
        's3Configuration': {
            'bucketArn': 'arn:aws:s3:::my-rag-documents'
        }
    },
    vectorIngestionConfiguration={
        'chunkingConfiguration': {
            'chunkingStrategy': 'FIXED_SIZE',
            'fixedSizeChunkingConfiguration': {
                'maxTokens': 512,
                'overlapPercentage': 20
            }
        }
    }
)

data_source_id = ds_response['dataSource']['dataSourceId']

# 인제스트(동기화) 시작
ingestion_response = bedrock_agent.start_ingestion_job(
    knowledgeBaseId=knowledge_base_id,
    dataSourceId=data_source_id
)

ingestion_job_id = ingestion_response['ingestionJob']['ingestionJobId']
print(f"인제스트 작업 시작! Job ID: {ingestion_job_id}")

# 인제스트 상태 확인
while True:
    status_response = bedrock_agent.get_ingestion_job(
        knowledgeBaseId=knowledge_base_id,
        dataSourceId=data_source_id,
        ingestionJobId=ingestion_job_id
    )
    status = status_response['ingestionJob']['status']
    print(f"현재 상태: {status}")

    if status in ['COMPLETE', 'FAILED']:
        break
    time.sleep(10)

if status == 'COMPLETE':
    stats = status_response['ingestionJob']['statistics']
    print(f"처리된 문서 수: {stats['numberOfDocumentsScanned']}")
    print(f"인덱싱된 청크 수: {stats['numberOfNewDocumentsIndexed']}")
    print(f"실패한 문서 수: {stats['numberOfDocumentsFailed']}")
```

> **💡 핵심 포인트:**
> 인제스트는 **증분(Incremental)** 방식으로 동작합니다. S3에 새 문서를 추가한 후 다시 Sync를 실행하면, 변경된 문서만 처리합니다. 전체 데이터를 다시 인덱싱할 필요가 없어 효율적입니다.

> 📷 **[이미지]** S3 버킷의 문서가 Knowledge Bases를 통해 청킹 → 임베딩 → OpenSearch Serverless 인덱스 저장으로 이어지는 인제스트 파이프라인 흐름도

---

## 8-3 시맨틱 검색과 Metadata 필터링

데이터 인제스트가 완료되었다면, 이제 가장 핵심적인 기능인 **검색**을 테스트할 차례입니다. Knowledge Bases는 `bedrock-agent-runtime` 클라이언트의 **Retrieve API**를 통해 시맨틱 검색을 제공합니다.

### 8-3-1 기본 유사도 검색 (Retrieve API)

**Retrieve API**는 사용자의 질문을 벡터로 변환하고, 인덱스에서 가장 유사한 청크를 찾아 반환합니다. 이것이 RAG의 "R(Retrieval)" 단계에 해당합니다.

```python
# bedrock-agent-runtime 클라이언트 생성
bedrock_agent_runtime = boto3.client('bedrock-agent-runtime', region_name='us-east-1')

# 기본 시맨틱 검색
response = bedrock_agent_runtime.retrieve(
    knowledgeBaseId='YOUR_KNOWLEDGE_BASE_ID',
    retrievalQuery={
        'text': 'Amazon Bedrock에서 지원하는 임베딩 모델은 무엇인가요?'
    },
    retrievalConfiguration={
        'vectorSearchConfiguration': {
            'numberOfResults': 5  # 상위 5개 결과 반환
        }
    }
)

# 검색 결과 출력
print(f"검색된 결과 수: {len(response['retrievalResults'])}")
for i, result in enumerate(response['retrievalResults']):
    print(f"\n--- [결과 {i+1}] ---")
    print(f"내용: {result['content']['text'][:200]}...")
    print(f"위치: {result['location']['s3Location']['uri']}")
    print(f"유사도 점수: {result['score']:.4f}")
```

위 코드를 실행하면, 질문과 의미적으로 가장 가까운 5개의 텍스트 청크가 유사도 점수와 함께 반환됩니다. 각 결과에는 원본 문서의 S3 위치 정보도 포함되어 있어, 출처 추적이 가능합니다.

### 8-3-2 Metadata 필터링으로 검색 정확도 튜닝

벡터 유사도 검색만으로는 해결하기 어려운 요구사항이 있습니다. "2024년 매뉴얼에서만 찾아줘"라거나 "인사팀 문서만 검색해줘" 같은 조건입니다. 이때 Chapter 7에서 정성껏 설계한 **메타데이터**가 빛을 발합니다.

Knowledge Bases의 Retrieve API는 `filter` 파라미터를 통해 메타데이터 기반 필터링을 지원합니다.

```python
# 메타데이터 필터링을 적용한 검색
response = bedrock_agent_runtime.retrieve(
    knowledgeBaseId='YOUR_KNOWLEDGE_BASE_ID',
    retrievalQuery={
        'text': '연차 휴가 정책은 어떻게 되나요?'
    },
    retrievalConfiguration={
        'vectorSearchConfiguration': {
            'numberOfResults': 5,
            'filter': {
                'andAll': [
                    {
                        'equals': {
                            'key': 'department',
                            'value': 'HR'
                        }
                    },
                    {
                        'greaterThanOrEquals': {
                            'key': 'year',
                            'value': 2024
                        }
                    }
                ]
            }
        }
    }
)

for i, result in enumerate(response['retrievalResults']):
    print(f"[결과 {i+1}] 점수: {result['score']:.4f}")
    print(f"  내용: {result['content']['text'][:150]}...")
    metadata = result.get('location', {}).get('s3Location', {})
    print(f"  출처: {metadata.get('uri', 'N/A')}")
```

Knowledge Bases에서 지원하는 필터 연산자는 다음과 같습니다.

| 연산자 | 설명 | 예시 |
|---|---|---|
| `equals` | 정확히 일치 | `{'key': 'department', 'value': 'HR'}` |
| `notEquals` | 불일치 | `{'key': 'status', 'value': 'archived'}` |
| `greaterThan` | 초과 | `{'key': 'year', 'value': 2023}` |
| `greaterThanOrEquals` | 이상 | `{'key': 'year', 'value': 2024}` |
| `lessThan` | 미만 | `{'key': 'page', 'value': 100}` |
| `lessThanOrEquals` | 이하 | `{'key': 'page', 'value': 50}` |
| `in` | 목록 포함 | `{'key': 'category', 'value': ['policy', 'guide']}` |
| `notIn` | 목록 미포함 | `{'key': 'type', 'value': ['draft']}` |
| `startsWith` | 접두사 일치 | `{'key': 'title', 'value': '2024'}` |
| `stringContains` | 문자열 포함 | `{'key': 'title', 'value': '매뉴얼'}` |

필터 조건은 `andAll`(AND 조합)과 `orAll`(OR 조합)로 복합 조건을 구성할 수 있습니다.

> **💡 핵심 포인트:**
> 메타데이터 필터링은 벡터 검색 **이전에** 적용됩니다. 즉, 필터 조건에 맞는 문서들만 대상으로 유사도 검색이 수행되므로, 검색 정확도가 크게 향상되고 불필요한 결과가 줄어듭니다.

### 8-3-3 검색 결과 스코어 해석

Retrieve API가 반환하는 `score` 값은 **유사도 점수**입니다. 이 점수를 올바르게 해석하는 것이 검색 품질 관리의 핵심입니다.

- **점수 범위**: 0.0 ~ 1.0 (1에 가까울수록 유사도가 높음)
- **높은 점수 (0.7 이상)**: 질문과 매우 관련 있는 문서
- **중간 점수 (0.4 ~ 0.7)**: 부분적으로 관련 있는 문서
- **낮은 점수 (0.4 미만)**: 관련성이 낮은 문서

```python
def retrieve_with_threshold(knowledge_base_id, query, threshold=0.5, top_k=5):
    """
    유사도 임계값(threshold)을 적용한 검색 함수.
    점수가 임계값 이하인 결과는 제외합니다.
    """
    response = bedrock_agent_runtime.retrieve(
        knowledgeBaseId=knowledge_base_id,
        retrievalQuery={'text': query},
        retrievalConfiguration={
            'vectorSearchConfiguration': {
                'numberOfResults': top_k
            }
        }
    )

    # 임계값 이상인 결과만 필터링
    filtered_results = [
        r for r in response['retrievalResults']
        if r['score'] >= threshold
    ]

    print(f"전체 결과: {len(response['retrievalResults'])}개")
    print(f"임계값({threshold}) 통과: {len(filtered_results)}개")

    if not filtered_results:
        print("관련 문서를 찾을 수 없습니다. 질문을 다시 표현해 보세요.")

    return filtered_results


# 사용 예시
results = retrieve_with_threshold(
    knowledge_base_id='YOUR_KNOWLEDGE_BASE_ID',
    query='Bedrock에서 Claude 모델의 최대 토큰 수는?',
    threshold=0.5
)
```

> **⚠️ 주의:**
> 유사도 점수가 낮은 결과를 그대로 LLM에 전달하면, 관련 없는 컨텍스트로 인해 **환각(Hallucination)**이 발생할 수 있습니다. 반드시 임계값을 설정하여 품질이 낮은 검색 결과를 걸러내는 로직을 추가하세요.

> 📷 **[이미지]** 벡터 공간에서 질문(Query)을 중심으로 가장 가까운 점 5개(Top-k)가 선택되는 다이어그램. 유사도 점수 0.5를 기준선으로 표시하여, 기준선 위(고득점)의 결과만 RAG 파이프라인으로 전달되는 흐름

---

## 8-4 RetrieveAndGenerate API로 기본 RAG 호출하기

지금까지 우리는 검색(Retrieval) 단계를 집중적으로 다루었습니다. 하지만 RAG의 진정한 가치는 검색된 문서를 **LLM에 전달하여 자연스러운 답변을 생성**하는 데 있습니다. AWS는 이 전체 과정을 **단 한 번의 API 호출**로 처리하는 `RetrieveAndGenerate` API를 제공합니다.

### 8-4-1 Retrieve vs RetrieveAndGenerate 차이

두 API의 차이를 명확히 이해하는 것이 중요합니다.

| 비교 항목 | Retrieve API | RetrieveAndGenerate API |
|---|---|---|
| **반환값** | 관련 문서 청크 목록 | LLM이 생성한 자연어 답변 |
| **LLM 호출** | 하지 않음 (검색만) | 자동으로 LLM 호출 포함 |
| **프롬프트 제어** | 개발자가 직접 구성 | 내부적으로 자동 구성 |
| **유연성** | 높음 (커스터마이징 자유) | 보통 (간편하지만 제한적) |
| **비용** | 임베딩 검색 비용만 | 임베딩 검색 + LLM 추론 비용 |
| **주요 용도** | 검색 결과를 직접 가공할 때 | 빠르게 RAG 프로토타입을 만들 때 |

**Retrieve API**는 검색 결과만 반환하므로, 개발자가 프롬프트를 자유롭게 설계하고 LLM을 별도로 호출해야 합니다. 반면 **RetrieveAndGenerate API**는 검색부터 답변 생성까지 한 번에 처리하므로, 빠른 프로토타이핑에 적합합니다.

> **💡 핵심 포인트:**
> 프로토타입이나 PoC 단계에서는 `RetrieveAndGenerate`로 빠르게 검증하고, 프로덕션 환경에서는 `Retrieve` + 별도 LLM 호출로 전환하여 프롬프트와 응답 품질을 세밀하게 제어하는 것이 일반적인 패턴입니다.

### 8-4-2 기본 RAG 파이프라인 End-to-End 테스트

이제 `RetrieveAndGenerate` API를 사용하여 질문부터 답변까지 End-to-End RAG 파이프라인을 테스트해 보겠습니다.

```python
import boto3
import json

# bedrock-agent-runtime 클라이언트 생성
bedrock_agent_runtime = boto3.client('bedrock-agent-runtime', region_name='us-east-1')

def rag_query(knowledge_base_id, question, model_id='anthropic.claude-3-sonnet-20240229-v1:0'):
    """
    Knowledge Base를 활용한 RAG 질의응답 함수.
    검색 + 생성을 한 번의 API 호출로 처리합니다.
    """
    response = bedrock_agent_runtime.retrieve_and_generate(
        input={
            'text': question
        },
        retrieveAndGenerateConfiguration={
            'type': 'KNOWLEDGE_BASE',
            'knowledgeBaseConfiguration': {
                'knowledgeBaseId': knowledge_base_id,
                'modelArn': f'arn:aws:bedrock:us-east-1::foundation-model/{model_id}',
                'retrievalConfiguration': {
                    'vectorSearchConfiguration': {
                        'numberOfResults': 5
                    }
                }
            }
        }
    )

    return response


# RAG 질의응답 실행
knowledge_base_id = 'YOUR_KNOWLEDGE_BASE_ID'
question = 'Amazon Bedrock에서 지원하는 임베딩 모델의 종류와 특징을 설명해주세요.'

response = rag_query(knowledge_base_id, question)

# 생성된 답변 출력
print("=" * 60)
print(f"질문: {question}")
print("=" * 60)
print(f"\n답변:\n{response['output']['text']}")

# 참조된 출처(Citation) 정보 출력
print("\n--- 참조 출처 ---")
for i, citation in enumerate(response.get('citations', [])):
    for ref in citation.get('retrievedReferences', []):
        source = ref.get('location', {}).get('s3Location', {}).get('uri', 'N/A')
        print(f"[출처 {i+1}] {source}")
```

위 코드를 실행하면, Knowledge Base에서 관련 문서를 검색한 뒤, Claude 모델이 검색 결과를 바탕으로 자연스러운 한국어 답변을 생성합니다. 또한 `citations` 필드에는 답변의 근거가 된 원본 문서의 S3 위치가 포함되어 있어, **출처 추적**이 가능합니다.

#### 멀티턴 대화 지원

`RetrieveAndGenerate` API는 `sessionId`를 활용한 멀티턴 대화도 지원합니다.

```python
# 첫 번째 질문 (세션 시작)
response1 = bedrock_agent_runtime.retrieve_and_generate(
    input={'text': 'Bedrock Knowledge Bases의 주요 기능은 무엇인가요?'},
    retrieveAndGenerateConfiguration={
        'type': 'KNOWLEDGE_BASE',
        'knowledgeBaseConfiguration': {
            'knowledgeBaseId': knowledge_base_id,
            'modelArn': f'arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-sonnet-20240229-v1:0'
        }
    }
)

session_id = response1['sessionId']
print(f"세션 ID: {session_id}")
print(f"답변: {response1['output']['text'][:200]}...")

# 두 번째 질문 (같은 세션에서 후속 질문)
response2 = bedrock_agent_runtime.retrieve_and_generate(
    input={'text': '그중에서 비용이 가장 적게 드는 방법은?'},
    sessionId=session_id,  # 이전 세션 ID를 전달하여 대화 이어가기
    retrieveAndGenerateConfiguration={
        'type': 'KNOWLEDGE_BASE',
        'knowledgeBaseConfiguration': {
            'knowledgeBaseId': knowledge_base_id,
            'modelArn': f'arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-sonnet-20240229-v1:0'
        }
    }
)

print(f"\n후속 답변: {response2['output']['text'][:200]}...")
```

### 8-4-3 검색 품질 개선 보고서 작성

RAG 시스템을 구축한 후에는 반드시 **검색 품질을 체계적으로 평가**해야 합니다. 아래는 검색 품질을 진단하고 개선 방향을 도출하는 실습 코드입니다.

```python
def evaluate_retrieval_quality(knowledge_base_id, test_queries):
    """
    테스트 질문 세트를 사용하여 검색 품질을 평가하는 함수.
    각 질문에 대해 검색 결과의 평균 점수와 결과 수를 분석합니다.
    """
    report = []

    for query_info in test_queries:
        query = query_info['question']
        expected_keywords = query_info.get('expected_keywords', [])

        response = bedrock_agent_runtime.retrieve(
            knowledgeBaseId=knowledge_base_id,
            retrievalQuery={'text': query},
            retrievalConfiguration={
                'vectorSearchConfiguration': {
                    'numberOfResults': 5
                }
            }
        )

        results = response['retrievalResults']
        scores = [r['score'] for r in results]
        avg_score = sum(scores) / len(scores) if scores else 0

        # 기대 키워드가 결과에 포함되어 있는지 확인
        keyword_hits = 0
        for result in results:
            text = result['content']['text'].lower()
            if any(kw.lower() in text for kw in expected_keywords):
                keyword_hits += 1

        report.append({
            'question': query,
            'num_results': len(results),
            'avg_score': round(avg_score, 4),
            'top_score': round(max(scores), 4) if scores else 0,
            'keyword_hit_rate': f"{keyword_hits}/{len(results)}"
        })

    return report


# 테스트 질문 세트 정의
test_queries = [
    {
        'question': 'Bedrock에서 사용할 수 있는 모델 종류는?',
        'expected_keywords': ['claude', 'titan', 'llama', 'mistral']
    },
    {
        'question': '임베딩 벡터의 차원 수는 얼마인가요?',
        'expected_keywords': ['1024', '1536', '차원', 'dimension']
    },
    {
        'question': 'Knowledge Base 생성 방법을 알려주세요.',
        'expected_keywords': ['knowledge base', 'create', '생성']
    }
]

# 평가 실행 및 보고서 출력
report = evaluate_retrieval_quality('YOUR_KNOWLEDGE_BASE_ID', test_queries)

print("\n===== 검색 품질 평가 보고서 =====\n")
for entry in report:
    print(f"질문: {entry['question']}")
    print(f"  결과 수: {entry['num_results']}")
    print(f"  평균 점수: {entry['avg_score']}")
    print(f"  최고 점수: {entry['top_score']}")
    print(f"  키워드 적중률: {entry['keyword_hit_rate']}")
    print()
```

이 보고서를 통해 다음과 같은 개선 방향을 도출할 수 있습니다.

- **평균 점수가 전반적으로 낮다면**: 청킹 크기를 조정하거나, 문서 전처리 품질을 개선해야 합니다 (Chapter 7 참조)
- **키워드 적중률이 낮다면**: 메타데이터 필터링을 추가하여 검색 범위를 좁혀 보세요
- **특정 질문에서만 점수가 낮다면**: 해당 주제의 문서가 누락되었거나, 청킹 경계에서 맥락이 잘려 나갔을 수 있습니다
- **상위 점수와 하위 점수의 격차가 크다면**: `numberOfResults`를 줄여 노이즈를 제거하세요

> **📌 참고:**
> 검색 품질 평가는 일회성이 아닌 **지속적인 개선 사이클**입니다. 사용자의 실제 질문 로그를 수집하고, 주기적으로 테스트 세트를 업데이트하여 품질을 모니터링하는 것이 중요합니다.

---

## 마치며

이번 Chapter 8에서 우리는 RAG 시스템의 핵심 인프라를 구축했습니다. 벡터 스토어의 원리를 이해하고, **Knowledge Bases for Amazon Bedrock**을 생성하여 S3의 문서를 인제스트했습니다. **Retrieve API**로 시맨틱 검색과 메타데이터 필터링을 수행하고, **RetrieveAndGenerate API**로 검색부터 답변 생성까지 한 번에 처리하는 End-to-End RAG 파이프라인을 완성했습니다.

하지만 `RetrieveAndGenerate`가 제공하는 기본 RAG는 시작점일 뿐입니다. 실제 서비스에서는 프롬프트를 세밀하게 설계하고, 환각을 방지하며, 출처를 명확하게 표기하는 등 **응답 품질을 정교하게 다듬는 과정**이 필요합니다. 다음 **Chapter 9**에서는 `Retrieve` API로 가져온 검색 결과를 직접 프롬프트에 삽입하고, **Claude** 모델과 결합하여 출처가 포함된 고품질 Q&A 시스템을 설계하는 과정을 다루겠습니다.
