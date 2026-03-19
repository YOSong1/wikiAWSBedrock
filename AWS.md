# AWS Bedrock 활용 심화 - 도서 목차 계획

> **참고**: GCP Vertex AI 도서(wikidocs_19059)의 구성 흐름을 따르며, KTDS 심화 제안서의 01 AWS Bedrock 커리큘럼(2일, 16시간)을 기반으로 확장한 목차입니다.

---

## Part 1. Amazon Bedrock과 생성형 AI 서비스 큰 그림

Part 1에서는 생성형 AI의 핵심 개념과 AWS가 제공하는 생성형 AI 서비스 전반을 다룹니다. Amazon Bedrock의 위치와 역할을 이해하고, 다른 클라우드 서비스와의 차이점을 파악합니다.

### Chapter 01. 생성형 AI와 AWS 기반 서비스 이해

- 1-1 생성형 AI의 개념과 주요 활용 패턴
  - 1-1-1 생성형 AI란 무엇인가?
  - 1-1-2 생성형 AI의 5가지 핵심 활용 패턴 (생성, 요약, 변환, 추출, 대화형 에이전트)
  - 1-1-3 멀티모달(Multimodal)과 에이전트(Agent)
- 1-2 예측형 ML vs 생성형 AI 비교
- 1-3 AWS 생성형 AI 서비스 생태계
  - 1-3-1 Amazon Bedrock, SageMaker, Q Developer 포지셔닝
  - 1-3-2 Bedrock vs Vertex AI vs Azure OpenAI Service 비교

### Chapter 02. Amazon Bedrock 아키텍처와 서비스 지도

- 2-1 Bedrock 주요 컴포넌트 개요
  - 2-1-1 Foundation Model Playground: 모델 탐색 및 실험
  - 2-1-2 Model Access & Provisioned Throughput: 모델 접근과 배포
  - 2-1-3 Knowledge Bases: RAG 통합 데이터 레이어
  - 2-1-4 Agents: 업무 자동화 오케스트레이션
  - 2-1-5 Guardrails: 안전한 AI 서비스 운영
  - 2-1-6 Model Evaluation: 모델 품질 평가
- 2-2 Bedrock 핵심 서비스 관계 및 활용
  - 2-2-1 서비스 간 상호 작용 개요
  - 2-2-2 지원 모델 패밀리: Claude, Llama, Titan, Mistral 비교
  - 2-2-3 InvokeModel API vs Converse API 차이와 선택 기준

---

## Part 2. AWS 환경 구성과 Bedrock API 기본기

Part 2에서는 AWS 계정과 IAM 설정, Bedrock 접근 권한 구성, SDK 기반 개발 환경 설정까지 실습합니다. Foundation Model을 API로 호출하고 Streaming 응답과 멀티모달 처리를 다룹니다.

### Chapter 03. AWS 계정·IAM·Billing 셋업

- 3-1 AWS 계정과 리소스 계층 구조 이해
  - 3-1-1 개인 개발자 관점
  - 3-1-2 기업/조직 관점 (AWS Organizations)
  - 3-1-3 리전과 가용 영역(AZ) 개념
- 3-2 IAM 역할(Role)과 Bedrock 접근 보안 전략
  - 3-2-1 IAM 정책의 3가지 유형 (AWS 관리형, 고객 관리형, 인라인)
  - 3-2-2 Bedrock 서비스 접근 권한 설정 (bedrock:InvokeModel 등)
  - 3-2-3 서비스 역할과 최소 권한 원칙
- 3-3 Billing 관리와 비용 폭탄 방지책
  - 3-3-1 AWS Cost Explorer와 비용 대시보드
  - 3-3-2 예산(Budget) 알림과 CloudWatch Billing Alarm 설정
  - 3-3-3 Bedrock 요금 체계 이해 (On-demand vs Provisioned Throughput)
- 3-4 Bedrock 리전 전략과 모델 접근 권한(Model Access) 활성화
  - 3-4-1 리전별 모델 가용성 확인
  - 3-4-2 Model Access 요청 및 승인 흐름

### Chapter 04. Bedrock 개발 환경 만들기

- 4-1 AWS Cloud9, 로컬 환경, SageMaker Studio 비교
  - 4-1-1 Cloud9: 클라우드 기반 IDE
  - 4-1-2 로컬 환경 (VS Code + AWS Toolkit)
  - 4-1-3 SageMaker Studio: 노트북 기반 실험
- 4-2 AWS CLI 기본 설정과 프로파일 구성
  - 4-2-1 AWS CLI 설치 및 aws configure
  - 4-2-2 Named Profile과 SSO 연동
  - 4-2-3 자격 증명(Credentials) 관리 모범 사례
- 4-3 Python 가상환경과 Boto3/Bedrock SDK 설치
  - 4-3-1 프로젝트 폴더 생성 및 가상환경 구성
  - 4-3-2 boto3, botocore, bedrock-runtime 클라이언트 구성
- 4-4 Jupyter와 VS Code를 이용한 개발 워크플로
  - 4-4-1 VS Code에서 Jupyter Notebook 실행하기
  - 4-4-2 첫 번째 Bedrock 모델 호출 (InvokeModel API)

### Chapter 05. Foundation Model API와 프롬프트 기초

- 5-1 Bedrock 지원 모델 패밀리 이해
  - 5-1-1 Claude (Anthropic): 장문 처리와 안전성
  - 5-1-2 Llama (Meta): 오픈소스 기반 유연성
  - 5-1-3 Titan (Amazon): AWS 네이티브 통합
  - 5-1-4 Mistral: 경량 고성능 모델
  - 5-1-5 모델 성능·비용·응답 품질 비교표 작성
- 5-2 InvokeModel API로 텍스트 생성/요약/변환 호출하기
  - 5-2-1 모델별 요청 본문(Body) 구조 차이
  - 5-2-2 생성 제어 파라미터 (temperature, top_p, max_tokens)
  - 5-2-3 Streaming 응답(InvokeModelWithResponseStream) 처리
- 5-3 Converse API로 멀티턴 대화 구현
  - 5-3-1 Converse API의 장점: 모델 독립적 인터페이스
  - 5-3-2 멀티턴 대화와 대화 이력 관리
  - 5-3-3 멀티모달 입력 (이미지, 문서) 처리
- 5-4 System Prompt와 기본 프롬프트 설계 패턴
  - 5-4-1 System Prompt 설계 원칙
  - 5-4-2 Few-shot / Chain-of-Thought / XML 태그 구조화
  - 5-4-3 모델 선택 기준표 작성 실습

---

## Part 3. 임베딩과 RAG를 위한 데이터 레이어 설계

Part 3에서는 Knowledge Bases 기반 RAG 파이프라인을 구성합니다. S3 데이터 소스 연동, 임베딩 모델 선택, 청킹 전략, 시맨틱 검색까지 데이터 레이어 전체를 설계합니다.

### Chapter 06. Embeddings로 의미 기반 검색 준비하기

- 6-1 임베딩(Embeddings)의 개념과 활용 사례
  - 6-1-1 키워드 검색 vs 의미 기반 검색
  - 6-1-2 벡터 공간에서의 의미 거리
  - 6-1-3 주요 활용 사례
- 6-2 Bedrock 임베딩 모델 선택
  - 6-2-1 Titan Embeddings 모델 특성
  - 6-2-2 Cohere Embed 모델과 비교
  - 6-2-3 임베딩 차원 수와 성능 트레이드오프
- 6-3 Bedrock Embeddings API 호출 및 실습
  - 6-3-1 InvokeModel로 임베딩 생성
  - 6-3-2 실행 결과 해석
  - 6-3-3 유사도 비교 실습 (코사인 유사도)
- 6-4 임베딩 결과 저장 형식(벡터 + 메타데이터) 설계
  - 6-4-1 벡터와 메타데이터의 결합
  - 6-4-2 S3 기반 저장 전략

### Chapter 07. 문서 전처리와 청킹 파이프라인

- 7-1 PDF/텍스트 문서 파싱 전략
  - 7-1-1 PDF 파싱의 어려움
  - 7-1-2 도구 선택: LangChain Document Loaders와 Amazon Textract
- 7-2 청크 크기, 중첩, 메타데이터 설계 기준
  - 7-2-1 청킹의 필요성
  - 7-2-2 고정 크기 청킹 vs 시맨틱 청킹 비교 실습
  - 7-2-3 중첩(Chunk Overlap) 전략
  - 7-2-4 메타데이터(Metadata) 설계
- 7-3 S3 데이터 소스 구성과 동기화
  - 7-3-1 S3 버킷 구성과 데이터 업로드
  - 7-3-2 Knowledge Bases 데이터 소스 연동
  - 7-3-3 자동 동기화(Sync) 설정
- 7-4 품질 좋은 RAG를 위한 데이터 구성 팁
  - 7-4-1 불필요한 요소 제거 (Cleaning)
  - 7-4-2 표(Table) 데이터 처리
  - 7-4-3 FAQ·매뉴얼 문서의 최적 청킹 전략

### Chapter 08. Knowledge Bases와 벡터 인덱스 구성

- 8-1 벡터 스토어 개념과 선택
  - 8-1-1 벡터 스토어란 무엇인가?
  - 8-1-2 Amazon OpenSearch Serverless vs Pinecone vs 기타 옵션
- 8-2 Knowledge Bases 생성 및 구성
  - 8-2-1 Knowledge Base 생성 흐름 (콘솔 + API)
  - 8-2-2 임베딩 모델과 벡터 스토어 연결
  - 8-2-3 데이터 인제스트(Ingest)와 인덱싱
- 8-3 시맨틱 검색과 Metadata 필터링
  - 8-3-1 기본 유사도 검색 (Retrieve API)
  - 8-3-2 Metadata 필터링으로 검색 정확도 튜닝
  - 8-3-3 검색 결과 스코어 해석
- 8-4 RetrieveAndGenerate API로 기본 RAG 호출하기
  - 8-4-1 Retrieve vs RetrieveAndGenerate 차이
  - 8-4-2 기본 RAG 파이프라인 End-to-End 테스트
  - 8-4-3 검색 품질 개선 보고서 작성

---

## Part 4. Bedrock 애플리케이션 패턴과 에이전트 확장

Part 4에서는 RAG 기반 질의응답 구조를 심화하고, Agents를 활용한 업무 자동화, Guardrails를 통한 안전한 서비스 구조, 모델 평가와 Fine-tuning까지 다룹니다.

### Chapter 09. RAG 기반 질의응답 애플리케이션 설계

- 9-1 Retriever-Generator 구조 상세 분석
  - 9-1-1 Bedrock Knowledge Bases의 RAG 아키텍처
  - 9-1-2 데이터 흐름의 5단계
- 9-2 검색 결과를 프롬프트에 삽입하는 템플릿 설계
  - 9-2-1 기본 프롬프트 템플릿 구조
  - 9-2-2 환각(Hallucination) 방지 전략
- 9-3 근거(출처)를 포함하는 답변 포맷
  - 9-3-1 Citation 메타데이터 활용
  - 9-3-2 출처 표기 전략
- 9-4 FAQ·매뉴얼 문서 기반 Q&A 시스템 구현
  - 9-4-1 Knowledge Base + Converse API 조합
  - 9-4-2 멀티턴 RAG 대화 흐름 설계
  - 9-4-3 최종 테스트와 응답 품질 검증

### Chapter 10. 프롬프트 고도화와 응답 품질 개선

- 10-1 Few-shot, Chain-of-Thought, 단계적 추론 적용
  - 10-1-1 Zero-shot vs Few-shot 실험
  - 10-1-2 Chain-of-Thought (CoT): XML 태그 구조화 추론
- 10-2 도메인 전용 지시문과 스타일 프리셋 설계
  - 10-2-1 System Prompt 설계 원칙 심화
  - 10-2-2 출력 스타일 프리셋 (JSON, Markdown, 표 등)
- 10-3 환각(Hallucination) 완화를 위한 프롬프트 패턴
  - 10-3-1 지식의 경계 설정
  - 10-3-2 근거 기반 추론 유도 (Grounding)
  - 10-3-3 사실 검증 단계 추가 (Self-Correction)
- 10-4 모델 평가(Model Evaluation) 실습
  - 10-4-1 Bedrock Model Evaluation 기능 활용
  - 10-4-2 정확성·관련성·안전성 평가 기준 정의
  - 10-4-3 RAG 응답 품질 평가와 개선 사이클

### Chapter 11. Bedrock Agents와 외부 시스템 연동

- 11-1 Agent 동작 원리(ReAct 루프)와 설계 패턴
  - 11-1-1 ReAct 프레임워크: 추론(Reasoning)과 행동(Action)의 반복
  - 11-1-2 Agent 아키텍처 상세 분석
- 11-2 Action Group 설계와 OpenAPI 스키마 작성
  - 11-2-1 Action Group 개념과 역할
  - 11-2-2 OpenAPI 스키마(JSON/YAML) 작성법
  - 11-2-3 Lambda 함수 연동 및 외부 API 호출
- 11-3 Agent 고급 기능
  - 11-3-1 Memory(대화 기억)와 Knowledge Base 연계
  - 11-3-2 멀티 Action Group 오케스트레이션
  - 11-3-3 에이전트 End-to-End 구현 실습
- 11-4 Guardrails 설계와 적용
  - 11-4-1 유해 콘텐츠 차단 정책 설정
  - 11-4-2 민감정보(PII) 보호 필터
  - 11-4-3 입출력 통제(Topic Denial, Word Filters)
  - 11-4-4 Guardrail 정책 초안 및 테스트

---

## Part 5. 서비스 배포와 운영 최적화

Part 5에서는 구축한 AI 서비스를 운영 환경에 배포하고, 비용 최적화와 모니터링, 종합 프로젝트 설계까지 마무리합니다.

### Chapter 12. 웹 UI와 서비스 아키텍처 설계

- 12-1 Streamlit/Gradio 기반 빠른 프로토타입 UI
  - 12-1-1 Gradio: 모델 시연에 최적화된 도구
  - 12-1-2 Streamlit: 대시보드와 챗봇 인터페이스
- 12-2 입력-결과-근거를 보여주는 레이아웃 설계
  - 12-2-1 2단 분할 레이아웃 (질문 + 출처)
  - 12-2-2 RAG 컨텍스트와 메타데이터 시각화
- 12-3 고객 지원 챗봇 또는 사내 문서 Q&A 어시스턴트 UI 구현
  - 12-3-1 세션 상태 관리
  - 12-3-2 채팅 인터페이스와 Knowledge Base 연동

### Chapter 13. 백엔드 컨테이너화와 배포

- 13-1 FastAPI 기반 Bedrock API 서버 구조
  - 13-1-1 API 서버의 필요성
  - 13-1-2 FastAPI + Bedrock Runtime 서버 코드 설계
- 13-2 Dockerfile 작성과 ECR 푸시
  - 13-2-1 Dockerfile 핵심 명령어
  - 13-2-2 Amazon ECR 이미지 빌드 및 푸시
- 13-3 환경 변수/비밀 키 관리 (AWS Secrets Manager)
  - 13-3-1 환경 변수 활용
  - 13-3-2 AWS Secrets Manager 연계
- 13-4 로컬 Docker 실행으로 엔드투엔드 테스트
  - 13-4-1 도커 컨테이너 실행
  - 13-4-2 API 테스트 (curl, Postman)

### Chapter 14. AWS Lambda와 ECS/Fargate로 서비스 올리기

- 14-1 Lambda 기반 서버리스 배포
  - 14-1-1 Lambda 함수로 Bedrock 호출 래핑
  - 14-1-2 API Gateway 연동
  - 14-1-3 Cold Start 최적화와 동시성 제어
- 14-2 ECS/Fargate 기반 컨테이너 배포
  - 14-2-1 ECS 클러스터 및 태스크 정의
  - 14-2-2 Fargate 서비스 배포와 오토스케일링
  - 14-2-3 ALB(Application Load Balancer) 연동
- 14-3 Fine-tuning 개요와 데이터셋 준비
  - 14-3-1 Bedrock Custom Model Training 프로세스
  - 14-3-2 데이터셋 준비 기준과 포맷
  - 14-3-3 비용/성능 트레이드오프 분석

### Chapter 15. 운영 최적화와 종합 프로젝트 마무리

- 15-1 Provisioned Throughput vs On-demand 비용 비교
  - 15-1-1 요금 체계 상세 분석
  - 15-1-2 워크로드별 최적 선택 가이드
- 15-2 CloudWatch 모니터링 설정
  - 15-2-1 Bedrock 호출 지표 모니터링
  - 15-2-2 지연 시간(Latency)과 에러율 알림 설정
  - 15-2-3 비용 알림과 대시보드 구성
- 15-3 운영 체크리스트와 보안 점검
  - 15-3-1 배포 후 기본 동작 점검 체크리스트
  - 15-3-2 보안·비용·품질 관점의 운영 가이드
- 15-4 종합 프로젝트: 고객 지원 챗봇 또는 사내 문서 Q&A 어시스턴트
  - 15-4-1 프로젝트 요구사항 정의
  - 15-4-2 아키텍처 설계서 작성
  - 15-4-3 End-to-End 구현 및 발표
  - 15-4-4 평가 기준표와 피드백

---

## 부록

- A. AWS Bedrock 서비스 요금표 (2025년 기준)
- B. Bedrock 지원 모델 전체 목록과 리전별 가용성
- C. Bedrock vs Vertex AI vs Azure OpenAI 기능 비교표
- D. 실습 코드 저장소 안내
