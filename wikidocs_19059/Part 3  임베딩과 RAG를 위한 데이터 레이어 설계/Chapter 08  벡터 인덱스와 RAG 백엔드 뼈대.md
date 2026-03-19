앞선 Chapter 7에서 우리는 방대한 PDF 문서를 문단 단위의 작은 조각(Chunk)으로 자르고 다듬는 데이터 전처리 과정을 마쳤습니다. 이제 우리 앞에는 수백, 수천 개의 텍스트 조각들이 놓여 있습니다. 하지만 이 조각들을 단순히 폴더에 파일로 저장해 두는 것만으로는 부족합니다. 사용자가 질문을 던졌을 때, 수천 개의 조각 중에서 가장 적절한 정답을 0.1초 안에 찾아낼 수 있어야 하기 때문입니다.

이를 위해서는 이 텍스트 조각들을 특별한 형태의 데이터베이스에 저장해야 합니다. 일반적인 데이터베이스가 '글자'를 저장한다면, 우리가 사용할 데이터베이스는 텍스트의 '의미(숫자 벡터)'를 저장합니다. 이를 **벡터 스토어(Vector Store)**라고 부릅니다.

이번 챕터에서는 **RAG(검색 증강 생성)** 시스템의 허리 역할을 하는 **벡터 스토어**의 원리를 깊이 있게 이해하고, **LangChain**과 **Vertex AI**를 연동하여 실제 검색 엔진을 구축하는 과정을 단계별로 상세히 다루겠습니다.

> **💡 핵심 포인트:**
> 벡터 스토어는 의미 기반 검색을 가능하게 하는 RAG 시스템의 핵심 인프라입니다.

## 8-1 벡터 스토어 개념과 선택

### 8-1-1 벡터 스토어란 무엇인가?

우리가 흔히 사용하는 SQL 데이터베이스(MySQL, Oracle 등)는 "키워드 매칭"에 최적화되어 있습니다. 예를 들어 `SELECT * FROM docs WHERE text LIKE '%배터리%'`와 같은 쿼리를 날리면, '배터리'라는 단어가 정확히 포함된 문서만 찾아줍니다. 하지만 사용자가 "전원이 빨리 닳아요"라고 검색한다면, '배터리'라는 단어가 없기 때문에 검색에 실패하게 됩니다.

반면에 **벡터 스토어**는 단어의 **의미적 거리**를 계산하는 데이터베이스입니다. 텍스트를 고차원의 **벡터 공간(Vector Space)**에 점으로 찍어두고, 질문이 들어오면 그 질문의 위치와 가장 가까운 점들을 찾아냅니다. 덕분에 "전원이 빨리 닳아요"라는 질문과 "배터리 수명 관리"라는 문서가 의미적으로 가깝다는 것을 인식하고 찾아낼 수 있는 것입니다.

> **📌 참고:**
> SQL 데이터베이스는 정확한 키워드 매칭에 강하고, 벡터 스토어는 의미 기반 유사도 검색에 강합니다.

### 8-1-2 어떤 벡터 스토어를 선택해야 할까?

벡터 스토어를 구축하는 방법은 크게 두 가지가 있습니다. 프로젝트의 규모와 예산, 운영 방식에 따라 적절한 선택이 필요합니다.

#### 첫 번째: 설치형(Self-hosted) 방식

대표적으로 **ChromaDB**나 **FAISS**가 있습니다.

- 이 방식은 개발자의 PC나 서버에 라이브러리 형태로 직접 설치하여 사용합니다.
- 별도의 클라우드 비용이 들지 않고, 설치 즉시 사용할 수 있어 개발 초기 단계나 프로토타입을 만들 때 매우 유용합니다.
- 데이터가 파일 형태로 내 컴퓨터에 저장되므로 관리가 직관적입니다.
- 하지만 데이터가 수천만 건을 넘어가면 메모리 부족 현상이 발생할 수 있어 대규모 서비스에는 적합하지 않을 수 있습니다.

#### 두 번째: 완전 관리형(Managed Service) 방식

**Google Cloud**의 **Vertex AI Vector Search**나 **Pinecone** 등이 여기에 해당합니다.

- 클라우드 제공 업체가 인프라 관리를 대신 해주기 때문에, 수억 건 이상의 데이터를 저장해도 속도 저하가 거의 없습니다.
- 또한 24시간 안정적인 서비스를 보장합니다.
- 하지만 사용량에 따라 비용이 발생하며, 인덱스를 처음 배포하는 데 20~30분 정도의 시간이 소요되는 등 초기 설정이 다소 무거울 수 있습니다.

이번 강의에서는 여러분이 실습을 빠르고 비용 부담 없이 진행할 수 있도록, 설치형인 **ChromaDB**를 사용하여 로컬 환경에 벡터 검색 엔진을 구축해 보겠습니다.

> 📷 **[이미지]** 좌측에는 '사용자 PC(ChromaDB)', 우측에는 'Google Cloud(Vertex AI Vector Search)'를 배치하고, 각각의 특징(비용, 성능, 관리 편의성)을 비교하는 저울 모양의 인포그래픽

---

## 8-2 Vertex AI Search 또는 외부 벡터 DB 연동하기

이제 실제로 **벡터 스토어**를 구축해 보겠습니다. **LangChain** 라이브러리는 다양한 **벡터 스토어**를 일관된 코드로 다룰 수 있게 도와주므로, 우리는 복잡한 내부 알고리즘을 몰라도 손쉽게 DB를 만들 수 있습니다.

### 8-2-1 라이브러리 설치

가장 먼저, VS Code 터미널에서 다음 명령어를 입력하여 **ChromaDB**와 **LangChain** 연동 패키지를 설치합니다. `langchain-chroma`는 **LangChain**에서 **ChromaDB**를 쉽게 쓰기 위한 전용 패키지입니다.

```bash
pip install langchain-chroma chromadb
```

### 8-2-2 벡터 스토어 생성 및 데이터 저장

이제 Chapter 7에서 준비했던 `split_docs`(쪼개진 문서 리스트)와 Chapter 6에서 실습한 `embedding_model`을 하나로 합치는 작업을 진행합니다. 아래 코드는 텍스트 데이터를 벡터로 변환하여 **ChromaDB**에 저장하는 과정을 수행합니다.

```python
from langchain_chroma import Chroma
from langchain_google_vertexai import VertexAIEmbeddings

# 1. Vertex AI 임베딩 모델을 LangChain 래퍼로 설정합니다.
# 이 객체는 텍스트가 들어오면 자동으로 Google API를 호출해 벡터로 바꿔주는 역할을 합니다.
embeddings = VertexAIEmbeddings(model_name="text-multilingual-embedding-002")

# 2. ChromaDB 생성 및 데이터 입력
# persist_directory는 데이터를 내 PC의 폴더에 파일로 영구 저장하겠다는 의미입니다.
vector_store = Chroma.from_documents(
    documents=split_docs,  # Ch7에서 만든 청크 리스트
    embedding=embeddings,  # 위에서 설정한 임베딩 모델
    persist_directory="./chroma_db"
)
print("벡터 스토어 구축 완료! 데이터가 './chroma_db' 폴더에 저장되었습니다.")
```

이 코드가 실행되면 내부적으로 다음과 같은 일이 일어납니다.

1. `split_docs`에 있는 텍스트 조각들이 하나씩 `embeddings` 모델로 전달됩니다.
2. **Google Vertex AI** 서버가 이 텍스트를 768차원의 숫자 벡터로 변환하여 돌려줍니다.
3. **ChromaDB**는 이 벡터와 원본 텍스트, 메타데이터를 묶어서 지정된 폴더(`./chroma_db`)에 바이너리 파일 형태로 저장합니다.

이제 프로그램을 껐다 켜도 데이터는 사라지지 않고 언제든 검색할 수 있는 상태가 됩니다.

> 📷 **[이미지]** 텍스트 청크들이 믹서기(Embedding Model)를 통과해 액체(Vector)가 되고, 이것이 'ChromaDB'라는 통에 차곡차곡 담기는 과정을 묘사한 개념도. 입력 데이터가 변환되어 저장소에 안착하는 흐름

---

## 8-3 유사도 검색(Top-k)과 필터링, 스코어 활용

데이터를 저장했다면, 가장 중요한 기능인 '검색'이 잘 동작하는지 검증해 볼 차례입니다. 벡터 검색은 단순히 결과를 가져오는 것을 넘어, 품질을 높이기 위한 다양한 기법들이 존재합니다.

### 8-3-1 기본 유사도 검색 (Similarity Search)

가장 기본적인 검색 방식입니다. 사용자의 질문을 벡터로 변환한 뒤, 저장된 벡터들 중 거리가 가장 가까운 k개의 문서를 가져오는 것입니다. 이를 **Top-k 검색**이라고 합니다.

```python
query = "Vertex AI의 주요 기능은 무엇인가요?"

# 상위 3개(k=3) 문서 검색
results = vector_store.similarity_search(query, k=3)

print(f"검색된 문서 개수: {len(results)}")
for i, doc in enumerate(results):
    print(f"--- [문서 {i+1}] ---")
    print(doc.page_content[:100] + "...")  # 내용이 기니까 앞부분만 출력
```

### 8-3-2 유사도 점수 확인 (Score)

검색 결과를 신뢰할 수 있는지 판단하려면, "얼마나 비슷한지"를 나타내는 점수(Score)를 확인해야 합니다. **ChromaDB**는 기본적으로 **거리(Distance)**를 점수로 반환합니다. 거리가 0에 가까울수록 두 문장이 거의 완벽하게 일치한다는 뜻이고, 값이 클수록 서로 관련이 없다는 뜻입니다.

> **⚠️ 주의:**
> 거리 점수가 높은 결과는 신뢰도가 낮으므로, 임계값(threshold)을 설정하여 필터링하는 것이 좋습니다.

```python
# 점수(거리)와 함께 검색
results_with_score = vector_store.similarity_search_with_score(query, k=3)

for doc, score in results_with_score:
    print(f"거리(Distance): {score:.4f} | 내용: {doc.page_content[:50]}...")
```

만약 거리가 특정 임계값(예: 0.8)보다 크다면, AI가 "관련 정보를 찾을 수 없습니다"라고 대답하도록 로직을 짤 수 있습니다. 이는 AI의 **환각(Hallucination)** 현상을 줄이는 데 매우 중요한 기법입니다.

### 8-3-3 메타데이터 필터링 (Metadata Filtering)

벡터 검색만으로는 해결하기 어려운 요구사항도 있습니다. 예를 들어 "2024년도 문서 중에서만 검색해줘"라거나 "인사팀 문서만 찾아줘" 같은 조건입니다. 이때 Chapter 7에서 정성껏 붙여둔 **메타데이터**가 빛을 발합니다.

```python
# 메타데이터의 'page' 정보가 10 이상인 문서 중에서만 검색
results = vector_store.similarity_search(
    query,
    k=2,
    filter={"page": {"$gte": 10}}  # MongoDB 스타일의 쿼리 문법 사용
)
```

이처럼 **벡터 검색**(의미 찾기)과 **메타데이터 필터**(조건 거르기)를 결합하면, 훨씬 정교하고 사용자의 의도에 꼭 맞는 검색 시스템을 만들 수 있습니다.

> 📷 **[이미지]** 과녁판(Vector Space)에 화살(Query)이 꽂혀 있고, 그 화살 주변에 가장 가까이 있는 점 3개(Top-k)가 선택되는 그림. 동시에 바깥쪽에는 '연도 <2023' 같은 거름망(Filter)이 있어서 조건에 안 맞는 점들은 제외되는 모습

---

## 8-4 문서 검색 API 및 기본 RAG 백엔드 뼈대 만들기

이제 지금까지 배운 모든 기능을 하나로 통합하여, 실제 애플리케이션에서 사용할 수 있는 **RAG 백엔드** 클래스를 만들어보겠습니다. 코드를 **클래스(Class)** 형태로 정리해 두면, 나중에 웹 UI나 챗봇 서버 등 어디서든 쉽게 가져다 쓸 수 있어 재사용성이 높아집니다.

### 8-4-1 RAGBackend 클래스 설계

우리는 **RAGBackend**라는 이름의 클래스를 만들 것입니다. 이 클래스는 초기화될 때 **벡터 DB**를 로드하고, `retrieve_documents`라는 함수를 통해 검색 기능을 제공합니다.

```python
class RAGBackend:
    def __init__(self, persist_dir="./chroma_db"):
        # 1. 임베딩 모델 설정
        self.embeddings = VertexAIEmbeddings(model_name="text-multilingual-embedding-002")

        # 2. 이미 저장된 DB를 로드 (데이터를 다시 넣는 게 아니라, 읽어오기만 함)
        self.vector_store = Chroma(
            persist_directory=persist_dir,
            embedding_function=self.embeddings
        )

    def retrieve_documents(self, query, k=3):
        """
        사용자의 질문(query)을 받아 가장 관련성 높은 k개의 문서를 반환합니다.
        """
        return self.vector_store.similarity_search(query, k=k)

# 사용 예시: 실제로는 이 부분만 호출해서 쓰면 됩니다.
rag = RAGBackend()
docs = rag.retrieve_documents("임베딩 모델의 가격 정책은 어떻게 되나요?")
print(f"검색된 문서 수: {len(docs)}")
print(f"가장 관련성 높은 내용: \n{docs[0].page_content}")
```

### 8-4-2 Retriever 인터페이스 (LangChain 표준)

**LangChain**은 **검색기(Retriever)**라는 표준 인터페이스를 제공합니다. 우리가 만든 **벡터 스토어**를 `as_retriever()` 함수를 통해 변환하면, **LangChain**의 다른 강력한 기능들(예: 파이프라인 연결)과 블록을 조립하듯 쉽게 연결할 수 있습니다.

```python
# 벡터 스토어를 LangChain 표준 Retriever 객체로 변환
retriever = vector_store.as_retriever(
    search_type="similarity",
    search_kwargs={"k": 3}
)

# 이제 retriever.invoke("질문") 만으로 간편하게 검색 가능
relevant_docs = retriever.invoke("RAG란 무엇인가?")
```

이 `retriever` 객체가 바로 **RAG** 아키텍처의 '검색(Retrieval)' 단계를 담당하는 핵심 엔진이 됩니다. 우리는 이 엔진을 다음 챕터에서 생성형 AI인 **Gemini**와 결합할 것입니다.

> 📷 **[이미지]** VS Code에서 RAGBackend 클래스 코드를 작성한 화면과, 터미널에서 질문을 입력했을 때 관련 문서 내용이 즉시 출력되는 실행 결과

---

## 마치며

이것으로 Chapter 8을 마칩니다. 우리는 쪼개진 텍스트 데이터를 **벡터 스토어**에 저장하여 우리만의 **지식 베이스**를 구축했고, 사용자의 질문에 맞춰 관련 정보를 순식간에 꺼내오는 **검색 엔진(Retriever)**까지 완성했습니다.

하지만 검색된 문서는 그저 '재료'일 뿐입니다. 사용자에게 날것의 텍스트 조각을 그대로 던져주면 읽기 불편할 것입니다. 다음 **Chapter 9**에서는 이 검색된 재료를 **Gemini**에게 전달하여, 사람처럼 자연스럽고 친절한 답변으로 가공해 내는 **RAG** 기반 질의응답 애플리케이션의 완성 과정을 다루겠습니다.
