# Chapter 07. 문서 전처리와 청킹 파이프라인

지난 Chapter 6에서 우리는 텍스트를 AI가 이해할 수 있는 숫자(벡터)로 바꾸는 **임베딩** 기술을 익혔습니다. Bedrock의 Titan Embeddings API로 문장을 벡터로 변환하고, 코사인 유사도로 문장 간 의미적 거리를 측정하는 실습도 해보았습니다. 하지만 현실 세계의 데이터는 우리가 실습했던 깔끔한 한 문장짜리 텍스트가 아닙니다. 회사 내의 데이터는 복잡한 표와 서식이 섞인 PDF 파일, 수십 페이지에 달하는 보고서, 혹은 엉성하게 작성된 메모 형태로 존재합니다.

이런 거칠고 방대한 원본 데이터를 그대로 **임베딩 모델**에 넣을 수는 없습니다. 마치 요리를 하기 전에 재료를 씻고 다듬는 과정이 필요하듯, AI에게 데이터를 먹이기 전에도 **전처리(Preprocessing)**와 **청킹(Chunking)**이라는 과정이 반드시 필요합니다.

이번 챕터에서는 **RAG** 시스템의 성능을 좌우하는 데이터 파이프라인을 구축해보겠습니다. PDF 문서를 텍스트로 추출하고, 이를 AI가 읽기 좋은 크기로 자르고, 최종적으로 **Amazon S3**와 **Knowledge Bases for Amazon Bedrock**에 연동하는 전략을 상세히 다룹니다.

> **💡 핵심 포인트:**
> 원본 데이터의 품질과 청킹 방식이 RAG 시스템의 검색 정확도를 결정하는 가장 중요한 요소입니다. 아무리 좋은 임베딩 모델과 LLM을 쓰더라도, 데이터 전처리가 엉망이면 결과도 엉망입니다.

---

## 7-1 PDF/텍스트 문서 파싱 전략

데이터 파이프라인의 첫 단계는 **파싱(Parsing)**입니다. 파싱이란 사람이 보는 문서 파일(PDF, PPTX, DOCX 등)에서 컴퓨터가 처리할 수 있는 순수한 텍스트 데이터만을 뽑아내는 과정을 말합니다.

### 7-1-1 PDF 파싱의 어려움

특히 비즈니스 환경에서 가장 많이 쓰이는 **PDF** 파일은 AI에게 가장 까다로운 상대입니다. PDF는 텍스트의 구조(문단, 제목)보다는 '보여지는 위치' 정보를 중심으로 저장되기 때문입니다. 단순히 텍스트를 긁어오면 단의 순서가 뒤섞이거나, 머리말/꼬리말이 본문 중간에 끼어들어가 문맥을 망치기도 합니다.

PDF 파싱이 어려운 대표적인 이유를 정리하면 다음과 같습니다.

| 문제 유형 | 설명 | 영향 |
|-----------|------|------|
| 2단/다단 레이아웃 | 왼쪽 단과 오른쪽 단의 텍스트가 뒤섞임 | 문맥이 완전히 깨짐 |
| 머리말/꼬리말 | 페이지 번호, 문서 제목 등이 본문에 섞임 | 임베딩 품질 저하 |
| 표(Table) | 행/열 구조가 줄글로 풀려 의미 손실 | 데이터 정확도 하락 |
| 스캔 이미지 PDF | 텍스트가 이미지로 되어 있어 추출 불가 | OCR 필요 |

> **⚠️ 주의:**
> PDF 파일에서 텍스트를 추출할 때는 레이아웃과 구조 정보가 손실될 수 있으므로, 파싱 결과를 반드시 육안으로 검수하는 습관을 들이세요.

### 7-1-2 도구 선택: LangChain Document Loaders와 Amazon Textract

이러한 복잡함을 해결하기 위해 두 가지 접근법을 소개합니다.

**접근법 1: LangChain Document Loaders (오픈소스)**

**LangChain**은 다양한 포맷의 문서를 손쉽게 불러올 수 있는 **Document Loader** 기능을 제공합니다.

- **PyPDFLoader**: 가장 기본적이고 빠릅니다. 텍스트 위주의 간단한 PDF에 적합합니다.
- **PDFPlumber**: 표나 복잡한 레이아웃 처리에 좀 더 강력합니다.
- **UnstructuredLoader**: 이미지, 표, 텍스트가 섞인 복잡한 문서에 강력하지만 설치가 조금 더 까다롭습니다.

**접근법 2: Amazon Textract (AWS 관리형 서비스)**

AWS 환경에서는 **Amazon Textract**라는 강력한 문서 분석 서비스를 활용할 수 있습니다. Textract는 단순 텍스트 추출을 넘어, 표(Table)와 양식(Form)의 구조까지 인식하는 AWS 네이티브 OCR/문서 분석 서비스입니다.

| 기능 | LangChain (PyPDF) | Amazon Textract |
|------|-------------------|-----------------|
| 텍스트 추출 | O | O |
| 표 구조 인식 | 제한적 | O (행/열 단위) |
| 스캔 이미지 PDF (OCR) | X | O |
| 양식(Key-Value) 추출 | X | O |
| 비용 | 무료 (오픈소스) | 페이지당 과금 |
| 설치 복잡도 | 낮음 | AWS 자격증명 필요 |

먼저 LangChain을 사용하는 기본적인 파싱 코드를 살펴보겠습니다.

```bash
pip install langchain langchain-community pypdf boto3
```

```python
from langchain_community.document_loaders import PyPDFLoader

# PDF 파일 로드
FILE_PATH = "./sample_data/company_manual.pdf"
loader = PyPDFLoader(FILE_PATH)
documents = loader.load()

print(f"총 페이지 수: {len(documents)}")
print(f"\n=== 1페이지 내용 (앞 300자) ===")
print(documents[0].page_content[:300])
print(f"\n=== 메타데이터 ===")
print(documents[0].metadata)
# 출력 예: {'source': './sample_data/company_manual.pdf', 'page': 0}
```

이번에는 Amazon Textract를 boto3로 호출하는 코드입니다. S3에 업로드된 PDF를 분석할 수 있습니다.

```python
import boto3

# Textract 클라이언트 생성
textract_client = boto3.client("textract", region_name="us-east-1")

# S3에 있는 PDF 분석 (비동기 방식 - 여러 페이지 PDF에 적합)
response = textract_client.start_document_analysis(
    DocumentLocation={
        "S3Object": {
            "Bucket": "my-rag-documents",
            "Name": "company_manual.pdf"
        }
    },
    FeatureTypes=["TABLES", "FORMS"]  # 표와 양식 구조도 함께 추출
)

job_id = response["JobId"]
print(f"Textract 분석 작업 시작: {job_id}")
```

```python
import time

# 분석 완료 대기
while True:
    result = textract_client.get_document_analysis(JobId=job_id)
    status = result["JobStatus"]
    if status in ["SUCCEEDED", "FAILED"]:
        break
    print(f"분석 중... (상태: {status})")
    time.sleep(5)

if status == "SUCCEEDED":
    # 추출된 텍스트 블록 확인
    blocks = result["Blocks"]
    text_blocks = [b for b in blocks if b["BlockType"] == "LINE"]
    print(f"추출된 텍스트 라인 수: {len(text_blocks)}")
    for block in text_blocks[:5]:
        print(block["Text"])
```

> **📌 참고:**
> 일반적인 텍스트 PDF라면 PyPDFLoader로 충분합니다. 스캔 문서이거나 표 구조가 중요한 경우에만 Amazon Textract를 사용하세요. Textract는 페이지당 과금되므로 비용 효율성도 고려해야 합니다.

> 📷 **[이미지]** 다양한 문서 아이콘(PDF, Word, HTML)이 두 갈래(LangChain Loader / Amazon Textract)를 통과하여 균일한 텍스트 데이터로 변환되는 개념도. 각 경로의 장단점이 표시된 그림

---

## 7-2 청크 크기, 중첩, 메타데이터 설계 기준

문서에서 텍스트를 추출했다면, 이제 이를 적절한 크기로 잘라야 합니다. 이 과정을 **청킹(Chunking)**이라고 합니다. 청킹은 RAG 시스템의 검색 정확도를 결정하는 매우 중요한 변수입니다.

### 7-2-1 청킹의 필요성

왜 문서를 잘라야 할까요? 두 가지 핵심 이유가 있습니다.

- **입력 길이 제한**: 임베딩 모델과 LLM은 한 번에 처리할 수 있는 토큰 수에 한계가 있습니다. Amazon Titan Embeddings V2 모델의 경우 최대 8,192 토큰까지 처리 가능하지만, 책 한 권을 통째로 넣을 수는 없습니다.

- **검색의 정확도**: 문서 전체를 하나의 벡터로 만들면 주제가 희석됩니다. 사용자가 "연차 신청 방법"을 물었을 때, 100페이지짜리 인사규정 전체보다는 해당 내용이 적힌 '한 문단'을 찾아주는 것이 훨씬 정확합니다.

### 7-2-2 고정 크기 청킹 vs 시맨틱 청킹 비교 실습

청킹 방식은 크게 두 가지로 나뉩니다.

**고정 크기 청킹 (Fixed-size Chunking)**

가장 직관적인 방법입니다. 지정한 글자(또는 토큰) 수만큼 일정하게 자릅니다. LangChain의 `RecursiveCharacterTextSplitter`가 대표적입니다. 이 분할기는 문단(`\n\n`) → 문장(`\n`) → 단어(공백) 순서로 최대한 의미 단위가 깨지지 않도록 자릅니다.

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

# 고정 크기 분할기 설정
fixed_splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,       # 청크당 약 500자
    chunk_overlap=50,     # 50자 정도 겹치게 설정
    separators=["\n\n", "\n", " ", ""]  # 자르는 우선순위
)

# 문서 분할 실행
fixed_chunks = fixed_splitter.split_documents(documents)
print(f"고정 크기 청킹 결과: {len(fixed_chunks)}개 청크")

# 첫 번째 청크 확인
print(f"\n=== 청크 1 ({len(fixed_chunks[0].page_content)}자) ===")
print(fixed_chunks[0].page_content[:200])
```

**시맨틱 청킹 (Semantic Chunking)**

시맨틱 청킹은 텍스트의 의미적 유사도를 기반으로 자릅니다. 연속된 문장들의 임베딩 유사도를 계산하여, 의미가 크게 달라지는 지점에서 청크를 나눕니다. Bedrock의 Knowledge Bases에서도 시맨틱 청킹을 기본 옵션으로 제공합니다.

LangChain의 시맨틱 청킹을 Bedrock Titan Embeddings와 함께 사용하는 예제입니다.

```python
import json
import boto3
from langchain_experimental.text_splitter import SemanticChunker
from langchain.embeddings.base import Embeddings

# Bedrock 임베딩을 LangChain과 연동하기 위한 래퍼 클래스
class BedrockTitanEmbeddings(Embeddings):
    def __init__(self, region_name="us-east-1"):
        self.client = boto3.client("bedrock-runtime", region_name=region_name)
        self.model_id = "amazon.titan-embed-text-v2:0"

    def embed_documents(self, texts):
        embeddings = []
        for text in texts:
            response = self.client.invoke_model(
                modelId=self.model_id,
                body=json.dumps({"inputText": text})
            )
            result = json.loads(response["body"].read())
            embeddings.append(result["embedding"])
        return embeddings

    def embed_query(self, text):
        return self.embed_documents([text])[0]

# 시맨틱 청킹 설정
bedrock_embeddings = BedrockTitanEmbeddings()
semantic_splitter = SemanticChunker(
    bedrock_embeddings,
    breakpoint_threshold_type="percentile"  # 유사도 하위 N%에서 분할
)

# 텍스트 분할
sample_text = documents[0].page_content
semantic_chunks = semantic_splitter.create_documents([sample_text])
print(f"시맨틱 청킹 결과: {len(semantic_chunks)}개 청크")
```

두 방식의 차이를 정리하면 다음과 같습니다.

| 비교 항목 | 고정 크기 청킹 | 시맨틱 청킹 |
|-----------|---------------|------------|
| 분할 기준 | 글자/토큰 수 | 의미적 유사도 변화 |
| 청크 크기 | 일정함 | 불규칙 (내용에 따라 다름) |
| 구현 난이도 | 쉬움 | 중간 (임베딩 모델 필요) |
| 비용 | 낮음 | 높음 (임베딩 API 호출 필요) |
| 품질 | 보통 | 높음 (의미 단위 보존) |
| 추천 상황 | 대량 문서, 빠른 처리 | 고품질 검색이 필요한 경우 |

> **💡 핵심 포인트:**
> Knowledge Bases for Amazon Bedrock에서는 콘솔에서 청킹 전략을 선택할 수 있습니다. **Default chunking**(고정 크기, 약 300 토큰), **Fixed-size chunking**(사용자 지정 크기), **Semantic chunking**, **No chunking** 중 선택 가능합니다. 대부분의 경우 고정 크기 청킹(500~1000 토큰)으로 시작한 후, 검색 품질을 보면서 시맨틱 청킹으로 전환하는 것을 추천합니다.

### 7-2-3 중첩(Chunk Overlap) 전략

문서를 단순히 칼로 무 자르듯 뚝뚝 끊으면 문제가 발생합니다. 하필 중요한 문장이 잘리는 경계선에 위치한다면 어떻게 될까요? 의미가 반으로 나뉘어 검색되지 않을 것입니다.

이를 방지하기 위해 **중첩(Overlap)**을 둡니다. 앞 청크의 끝부분을 뒷 청크의 앞부분에 10~20% 정도 겹치게 포함시키는 것입니다. 마치 슬라이딩 윈도우처럼 겹쳐가며 이동하는 방식입니다.

```
[====== 청크 1 (500자) ======]
                    [====== 청크 2 (500자) ======]
                                        [====== 청크 3 (500자) ======]
                    ↑                   ↑
               50자 중첩           50자 중첩
```

적절한 중첩 크기는 청크 크기의 **10~20%** 입니다. 청크가 500자라면 50~100자 정도의 중첩이 적당합니다.

```python
# 중첩 유무에 따른 차이 실험
splitter_no_overlap = RecursiveCharacterTextSplitter(
    chunk_size=500, chunk_overlap=0
)
splitter_with_overlap = RecursiveCharacterTextSplitter(
    chunk_size=500, chunk_overlap=100
)

chunks_no = splitter_no_overlap.split_documents(documents)
chunks_yes = splitter_with_overlap.split_documents(documents)

print(f"중첩 없음: {len(chunks_no)}개 청크")
print(f"중첩 100자: {len(chunks_yes)}개 청크")
print(f"청크 수 증가: {len(chunks_yes) - len(chunks_no)}개 (+{(len(chunks_yes)/len(chunks_no)-1)*100:.1f}%)")
```

> **📌 참고:**
> 중첩을 사용하면 전체 청크 수와 저장 공간이 증가하지만, 문맥 끊김으로 인한 검색 실패를 크게 줄여줍니다. 특히 Knowledge Bases for Bedrock에서 Fixed-size chunking을 설정할 때 overlap 비율을 함께 지정할 수 있습니다.

### 7-2-4 메타데이터(Metadata) 설계

잘라낸 텍스트 조각에는 반드시 '꼬리표'를 붙여야 합니다. 이것이 **메타데이터**입니다. 메타데이터가 있어야 나중에 LLM이 답변할 때 "이 내용은 인사규정 문서 15쪽에서 참고했습니다"라고 근거를 제시할 수 있습니다.

권장하는 메타데이터 필드는 다음과 같습니다.

| 필드명 | 예시 | 용도 |
|--------|------|------|
| `source` | "2025_HR_Guide.pdf" | 원본 파일 추적 |
| `page` | 15 | 페이지 번호 |
| `category` | "인사규정" | 문서 분류 (필터링용) |
| `created_date` | "2025-03-01" | 문서 최신성 판단 |
| `department` | "인사팀" | 부서별 필터링 |

LangChain에서 커스텀 메타데이터를 추가하는 방법입니다.

```python
# 기존 메타데이터에 커스텀 필드 추가
for chunk in fixed_chunks:
    chunk.metadata["category"] = "인사규정"
    chunk.metadata["department"] = "인사팀"
    chunk.metadata["created_date"] = "2025-03-01"

# 결과 확인
print(fixed_chunks[0].metadata)
# 출력 예:
# {
#   'source': './sample_data/company_manual.pdf',
#   'page': 0,
#   'category': '인사규정',
#   'department': '인사팀',
#   'created_date': '2025-03-01'
# }
```

> **💡 핵심 포인트:**
> Knowledge Bases for Bedrock에서는 S3에 문서와 함께 `.metadata.json` 파일을 두면 자동으로 메타데이터가 연동됩니다. 이를 활용하면 검색 시 메타데이터 필터링으로 특정 카테고리나 날짜 범위의 문서만 검색할 수 있습니다.

> 📷 **[이미지]** 긴 텍스트 띠를 일정한 간격(Chunk Size)으로 자르되, 연결 부위가 겹치도록(Overlap) 겹쳐서 자르는 슬라이딩 윈도우 방식의 다이어그램. 각 청크에 메타데이터 꼬리표가 달린 모습

---

## 7-3 S3 데이터 소스 구성과 동기화

앞서 로컬 환경에서 직접 파싱하고 청킹하는 방법을 배웠습니다. 하지만 AWS 환경에서는 **Knowledge Bases for Amazon Bedrock**이 이 과정을 자동으로 처리해줍니다. 우리가 할 일은 S3 버킷에 문서를 올리고, 데이터 소스로 연결하는 것뿐입니다.

### 7-3-1 S3 버킷 구성과 데이터 업로드

먼저 RAG용 문서를 저장할 S3 버킷을 구성합니다. 폴더 구조를 체계적으로 설계하면 나중에 메타데이터 필터링이 수월해집니다.

```
my-rag-documents/
├── hr/                          # 인사 관련 문서
│   ├── 2025_HR_Guide.pdf
│   ├── 2025_HR_Guide.pdf.metadata.json
│   ├── vacation_policy.pdf
│   └── vacation_policy.pdf.metadata.json
├── it/                          # IT 관련 문서
│   ├── security_manual.pdf
│   └── security_manual.pdf.metadata.json
└── finance/                     # 재무 관련 문서
    ├── expense_guide.pdf
    └── expense_guide.pdf.metadata.json
```

boto3로 S3 버킷을 생성하고 파일을 업로드하는 코드입니다.

```python
import boto3
import json

s3_client = boto3.client("s3", region_name="us-east-1")
bucket_name = "my-rag-documents"

# 1. S3 버킷 생성
s3_client.create_bucket(Bucket=bucket_name)
print(f"버킷 생성 완료: {bucket_name}")

# 2. PDF 문서 업로드
s3_client.upload_file(
    Filename="./sample_data/company_manual.pdf",
    Bucket=bucket_name,
    Key="hr/company_manual.pdf"
)
print("PDF 업로드 완료")

# 3. 메타데이터 파일 생성 및 업로드
metadata = {
    "metadataAttributes": {
        "category": {
            "value": "인사규정",
            "type": "STRING"
        },
        "department": {
            "value": "인사팀",
            "type": "STRING"
        },
        "year": {
            "value": 2025,
            "type": "NUMBER"
        }
    }
}

s3_client.put_object(
    Bucket=bucket_name,
    Key="hr/company_manual.pdf.metadata.json",
    Body=json.dumps(metadata, ensure_ascii=False),
    ContentType="application/json"
)
print("메타데이터 업로드 완료")
```

> **⚠️ 주의:**
> 메타데이터 JSON 파일의 이름은 반드시 `원본파일명.metadata.json` 형식이어야 합니다. 예를 들어 `report.pdf`의 메타데이터는 `report.pdf.metadata.json`이어야 Knowledge Bases가 자동으로 인식합니다.

### 7-3-2 Knowledge Bases 데이터 소스 연동

S3에 문서가 준비되었으면, Knowledge Bases에 데이터 소스로 연결합니다. boto3의 `bedrock-agent` 클라이언트를 사용합니다.

```python
bedrock_agent = boto3.client("bedrock-agent", region_name="us-east-1")

# Knowledge Base에 S3 데이터 소스 추가
response = bedrock_agent.create_data_source(
    knowledgeBaseId="YOUR_KB_ID",  # 기존에 생성한 Knowledge Base ID
    name="hr-documents",
    description="인사팀 문서 데이터 소스",
    dataSourceConfiguration={
        "type": "S3",
        "s3Configuration": {
            "bucketArn": f"arn:aws:s3:::{bucket_name}",
            "inclusionPrefixes": ["hr/"]  # hr/ 폴더만 포함
        }
    },
    vectorIngestionConfiguration={
        "chunkingConfiguration": {
            "chunkingStrategy": "FIXED_SIZE",
            "fixedSizeChunkingConfiguration": {
                "maxTokens": 512,        # 청크당 최대 512 토큰
                "overlapPercentage": 20   # 20% 중첩
            }
        }
    }
)

data_source_id = response["dataSource"]["dataSourceId"]
print(f"데이터 소스 생성 완료: {data_source_id}")
```

> **📌 참고:**
> `chunkingStrategy`에는 `FIXED_SIZE`, `SEMANTIC`, `HIERARCHICAL`, `NONE` 중 선택할 수 있습니다. `SEMANTIC`을 선택하면 Bedrock이 자체적으로 임베딩 유사도 기반 청킹을 수행합니다. `HIERARCHICAL`은 부모-자식 구조로 청킹하여 더 풍부한 문맥을 제공합니다.

### 7-3-3 자동 동기화(Sync) 설정

데이터 소스를 연결한 후에는 **동기화(Sync)**를 실행하여 S3의 문서가 청킹되고 임베딩되어 벡터 인덱스에 저장되도록 해야 합니다.

```python
# 데이터 소스 동기화(인제스트) 시작
sync_response = bedrock_agent.start_ingestion_job(
    knowledgeBaseId="YOUR_KB_ID",
    dataSourceId=data_source_id
)

ingestion_job_id = sync_response["ingestionJob"]["ingestionJobId"]
print(f"동기화 작업 시작: {ingestion_job_id}")
```

```python
import time

# 동기화 상태 확인
while True:
    job = bedrock_agent.get_ingestion_job(
        knowledgeBaseId="YOUR_KB_ID",
        dataSourceId=data_source_id,
        ingestionJobId=ingestion_job_id
    )
    status = job["ingestionJob"]["status"]
    if status in ["COMPLETE", "FAILED"]:
        break
    print(f"동기화 진행 중... (상태: {status})")
    time.sleep(10)

if status == "COMPLETE":
    stats = job["ingestionJob"]["statistics"]
    print(f"동기화 완료!")
    print(f"  - 스캔된 문서 수: {stats.get('numberOfDocumentsScanned', 0)}")
    print(f"  - 인덱싱된 문서 수: {stats.get('numberOfNewDocumentsIndexed', 0)}")
    print(f"  - 수정된 문서 수: {stats.get('numberOfModifiedDocumentsIndexed', 0)}")
    print(f"  - 삭제된 문서 수: {stats.get('numberOfDocumentsDeleted', 0)}")
else:
    print(f"동기화 실패: {job['ingestionJob'].get('failureReasons', 'Unknown')}")
```

S3에 새 문서를 추가하거나 기존 문서를 수정할 때마다 동기화를 다시 실행하면 됩니다. 변경된 문서만 증분으로 처리되므로 효율적입니다.

> **💡 핵심 포인트:**
> Knowledge Bases의 동기화는 현재 수동 트리거 방식입니다. 자동화하려면 S3 이벤트 알림 + Lambda 함수를 조합하여, 파일이 업로드될 때마다 `start_ingestion_job`을 호출하는 파이프라인을 구성할 수 있습니다.

> 📷 **[이미지]** S3 버킷에서 Knowledge Bases로 이어지는 데이터 동기화 흐름도. S3 → 파싱 → 청킹 → 임베딩 → 벡터 인덱스 저장까지의 파이프라인이 표시된 아키텍처 다이어그램

---

## 7-4 품질 좋은 RAG를 위한 데이터 구성 팁

마지막으로, 단순히 자르는 것을 넘어 현업에서 검색 품질을 높이기 위해 사용하는 몇 가지 '노하우'를 공유합니다. 이 부분은 실전에서 가장 체감이 큰 영역이니 꼭 챙겨가시기 바랍니다.

### 7-4-1 불필요한 요소 제거 (Cleaning)

PDF의 매 페이지마다 반복되는 "2025년 사업보고서 - 대외비" 같은 머리말이나 바닥글, 페이지 번호, 저작권 고지 등은 제거하는 것이 좋습니다. 이런 문구가 모든 청크에 들어가면, 임베딩 모델이 이 반복 문구에 과도하게 주목하여 검색 품질을 떨어뜨릴 수 있습니다.

```python
import re

def clean_text(text):
    """PDF에서 추출한 텍스트를 정리하는 함수"""

    # 1. 머리말/꼬리말 패턴 제거
    text = re.sub(r"2025년 사업보고서\s*-\s*대외비", "", text)
    text = re.sub(r"Page\s*\d+\s*(of\s*\d+)?", "", text)
    text = re.sub(r"^\d+\s*$", "", text, flags=re.MULTILINE)  # 페이지 번호만 있는 줄

    # 2. 과도한 공백/줄바꿈 정리
    text = re.sub(r"\n{3,}", "\n\n", text)  # 3줄 이상 빈 줄 → 2줄로
    text = re.sub(r" {2,}", " ", text)       # 연속 공백 → 1개로

    # 3. 특수문자 노이즈 제거
    text = re.sub(r"[■□●○▶▷]", "", text)   # 장식용 특수문자

    return text.strip()

# 모든 청크에 클리닝 적용
for chunk in fixed_chunks:
    chunk.page_content = clean_text(chunk.page_content)

print("클리닝 완료!")
print(f"예시: {fixed_chunks[0].page_content[:200]}")
```

> **⚠️ 주의:**
> 클리닝 규칙은 문서마다 다릅니다. 반드시 샘플을 눈으로 확인한 후 규칙을 작성하세요. 과도한 클리닝은 오히려 유의미한 정보를 삭제할 수 있습니다.

### 7-4-2 표(Table) 데이터 처리

RAG 시스템의 가장 큰 난적은 '표'입니다. 표를 단순히 줄글로 풀면 행과 열의 구조가 깨져서 "매출액"이 "1분기"인지 "2분기"인지 알 수 없게 됩니다.

**전략 1: 마크다운 변환**

표를 마크다운 형식으로 변환하면 구조를 어느 정도 유지할 수 있습니다.

```python
# Amazon Textract로 추출한 표를 마크다운으로 변환하는 예시
def table_to_markdown(textract_table):
    """Textract 표 결과를 마크다운으로 변환"""
    rows = textract_table["Rows"]
    md_lines = []

    for i, row in enumerate(rows):
        cells = [cell["Text"] for cell in row["Cells"]]
        md_lines.append("| " + " | ".join(cells) + " |")
        if i == 0:  # 헤더 구분선
            md_lines.append("| " + " | ".join(["---"] * len(cells)) + " |")

    return "\n".join(md_lines)

# 변환 결과 예시:
# | 구분 | 1분기 | 2분기 | 3분기 | 4분기 |
# | --- | --- | --- | --- | --- |
# | 매출액 | 100억 | 120억 | 115억 | 140억 |
```

**전략 2: LLM 요약**

복잡한 표는 LLM에게 요약을 맡기고, 그 요약본을 임베딩하는 것도 효과적입니다.

```python
import json

bedrock_runtime = boto3.client("bedrock-runtime", region_name="us-east-1")

table_text = """
| 구분 | 1분기 | 2분기 | 3분기 | 4분기 |
| 매출액 | 100억 | 120억 | 115억 | 140억 |
| 영업이익 | 20억 | 25억 | 22억 | 30억 |
"""

prompt = f"""다음 표의 핵심 내용을 검색에 유리하도록 자연어 문장으로 요약해주세요.

{table_text}
"""

response = bedrock_runtime.converse(
    modelId="anthropic.claude-3-haiku-20240307-v1:0",
    messages=[{"role": "user", "content": [{"text": prompt}]}]
)

summary = response["output"]["message"]["content"][0]["text"]
print(f"표 요약: {summary}")
# 예: "2025년 분기별 실적에서 매출액은 1분기 100억에서 4분기 140억으로 꾸준히 성장했으며,
#      영업이익도 1분기 20억에서 4분기 30억으로 증가했다."
```

> **💡 핵심 포인트:**
> 표 데이터는 원본 마크다운 + LLM 요약본을 함께 저장하면 가장 좋은 검색 결과를 얻을 수 있습니다. 원본은 정확한 수치 조회용, 요약본은 의미 기반 검색용으로 활용됩니다.

### 7-4-3 FAQ/매뉴얼 문서의 최적 청킹 전략

문서의 유형에 따라 최적의 청킹 전략이 다릅니다.

**FAQ 문서**

FAQ는 이미 "질문-답변" 쌍으로 구조화되어 있으므로, 각 Q&A 쌍을 하나의 청크로 만드는 것이 가장 효과적입니다.

```python
def chunk_faq_document(text):
    """FAQ 문서를 Q&A 단위로 청킹"""
    # Q: 또는 질문: 패턴으로 분리
    qa_pairs = re.split(r"\n(?=Q:|질문:|\d+\.)", text)
    chunks = []
    for qa in qa_pairs:
        qa = qa.strip()
        if len(qa) > 20:  # 너무 짧은 조각은 제외
            chunks.append(qa)
    return chunks

# 예시
faq_text = """
Q: 연차는 어떻게 신청하나요?
A: 사내 포털의 '근태관리' 메뉴에서 연차 신청이 가능합니다. 최소 3일 전에 신청해야 하며, 팀장 승인이 필요합니다.

Q: 재택근무 신청 절차는?
A: 주 2회까지 재택근무가 가능합니다. 전주 금요일까지 팀장에게 사전 보고 후 사내 포털에서 신청하세요.
"""

faq_chunks = chunk_faq_document(faq_text)
for i, chunk in enumerate(faq_chunks):
    print(f"--- FAQ 청크 {i+1} ---")
    print(chunk[:100])
```

**매뉴얼/가이드 문서**

매뉴얼은 계층적 제목 구조(1장 > 1.1절 > 1.1.1항)를 활용하여, 섹션 단위로 청킹하되 상위 제목을 컨텍스트로 포함시키는 것이 좋습니다.

```python
from langchain.text_splitter import MarkdownHeaderTextSplitter

# 마크다운 헤더 기반 분할
headers_to_split_on = [
    ("#", "chapter"),
    ("##", "section"),
    ("###", "subsection"),
]

md_splitter = MarkdownHeaderTextSplitter(
    headers_to_split_on=headers_to_split_on
)

# 마크다운 문서 분할
md_text = """
# 제1장 근태관리
## 1.1 출퇴근
### 1.1.1 출근 시간
정규 출근 시간은 오전 9시입니다. 유연근무제 적용 시 8시~10시 사이 출근 가능합니다.
### 1.1.2 퇴근 시간
정규 퇴근 시간은 오후 6시입니다.
## 1.2 연차
### 1.2.1 연차 일수
입사 1년 미만은 월 1일, 1년 이상은 연 15일의 연차가 부여됩니다.
"""

md_chunks = md_splitter.split_text(md_text)
for chunk in md_chunks:
    print(f"내용: {chunk.page_content[:80]}")
    print(f"메타: {chunk.metadata}")
    print()
# 출력 예:
# 내용: 정규 출근 시간은 오전 9시입니다...
# 메타: {'chapter': '제1장 근태관리', 'section': '1.1 출퇴근', 'subsection': '1.1.1 출근 시간'}
```

이처럼 상위 제목이 메타데이터로 자동 포함되므로, 검색 시 문맥을 훨씬 잘 파악할 수 있습니다.

> 📷 **[이미지]** 문서 유형별 최적 청킹 전략 비교 다이어그램. FAQ(Q&A 쌍 단위), 매뉴얼(계층적 섹션 단위), 보고서(고정 크기 + 중첩) 세 가지 방식이 시각적으로 비교된 그림

---

## 마치며

이것으로 Chapter 7을 마칩니다. 우리는 원석과 같은 PDF 문서를 캐내어, AI가 소화하기 좋게 다듬고 자르는 데이터 가공 파이프라인을 구축했습니다. 핵심 내용을 정리하면 다음과 같습니다.

- **파싱**: LangChain Document Loaders(간단한 텍스트 PDF)와 Amazon Textract(표/스캔 문서)를 상황에 맞게 선택합니다.
- **청킹**: 고정 크기 청킹으로 시작하되, 품질이 중요하면 시맨틱 청킹으로 전환합니다. Knowledge Bases for Bedrock에서 설정 가능합니다.
- **중첩**: 청크 크기의 10~20%를 중첩하여 문맥 끊김을 방지합니다.
- **메타데이터**: source, page, category 등을 설계하여 검색 필터링과 출처 제시에 활용합니다.
- **S3 연동**: 체계적인 폴더 구조와 `.metadata.json` 파일로 Knowledge Bases와 자연스럽게 통합합니다.
- **클리닝**: 반복 문구 제거, 표 처리, 문서 유형별 최적 전략이 RAG 품질을 크게 좌우합니다.

이제 우리에게는 깔끔하게 정리된 수백 개의 **텍스트 조각(Chunk)**과 이를 설명하는 **메타데이터**, 그리고 이를 숫자로 바꿀 **임베딩 모델**이 준비되었습니다. 다음 **Chapter 8**에서는 이 수많은 벡터 조각들을 효율적으로 저장하고, 빠르게 검색해 낼 수 있는 **Knowledge Bases와 벡터 인덱스 구성**에 대해 알아보겠습니다. OpenSearch Serverless와 같은 벡터 스토어를 연결하고, 실제로 시맨틱 검색을 수행하는 실습을 진행할 예정입니다.
