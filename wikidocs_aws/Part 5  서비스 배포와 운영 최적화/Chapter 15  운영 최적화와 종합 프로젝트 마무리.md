# Chapter 15. 운영 최적화와 종합 프로젝트 마무리

지난 챕터에서 우리는 **Lambda 서버리스 배포**, **API Gateway 연동**, **ECS/Fargate 컨테이너 배포**, 그리고 **Fine-tuning 개요**까지 다루며 서비스를 실제 운영 환경에 올리는 방법을 익혔습니다. 이제 여러분의 Bedrock 기반 AI 서비스는 인터넷에 공개되어 누구나 접속할 수 있는 상태가 되었습니다.

하지만 "배포했다"는 것이 곧 "완성했다"는 의미는 아닙니다. 진정한 운영은 배포 버튼을 누른 직후부터 시작됩니다. "비용이 예상보다 훨씬 나오면 어떡하지?", "서버가 갑자기 느려지면 어떻게 감지하지?", "보안 사고가 나면 어떤 조치를 취해야 하지?"와 같은 운영 이슈들이 기다리고 있기 때문입니다.

이번 마지막 챕터에서는 **비용 최적화 전략**부터 **CloudWatch 모니터링**, **운영 체크리스트와 보안 점검**, 그리고 지금까지 배운 모든 것을 종합하는 **실전 프로젝트 설계**까지 다루겠습니다. 이 책의 대미를 장식하는 챕터인 만큼, 여러분이 실무에서 바로 활용할 수 있는 실용적인 가이드를 제공하겠습니다.



---

## 15-1 Provisioned Throughput vs On-demand 비용 비교

Amazon Bedrock의 요금 체계를 이해하는 것은 운영 최적화의 첫 번째 단계입니다. 잘못된 과금 방식을 선택하면 같은 서비스를 운영하면서도 비용이 몇 배나 차이 날 수 있습니다.

### 15-1-1 요금 체계 상세 분석

Bedrock은 크게 **두 가지 과금 방식**을 제공합니다.

**1) On-demand (온디맨드) 방식**

사용한 만큼만 비용을 지불하는 방식입니다. 입력 토큰(Input Token)과 출력 토큰(Output Token)에 각각 단가가 적용됩니다.

| 모델 | 입력 토큰 (1K당) | 출력 토큰 (1K당) |
|------|-----------------|-----------------|
| Claude 3.5 Sonnet | $0.003 | $0.015 |
| Claude 3 Haiku | $0.00025 | $0.00125 |
| Llama 3.1 70B | $0.00099 | $0.00099 |
| Titan Text Express | $0.0002 | $0.0006 |

> **📌 참고:** 위 가격은 예시이며, AWS 공식 요금 페이지에서 최신 가격을 반드시 확인하세요. 리전에 따라 가격이 다를 수 있습니다.

온디맨드 방식의 **월간 비용 추정** 예시를 살펴보겠습니다.

```
# 일일 사용 패턴 예시 (Claude 3.5 Sonnet 기준)
- 하루 평균 1,000건의 API 호출
- 건당 평균 입력 토큰: 2,000 (RAG 컨텍스트 포함)
- 건당 평균 출력 토큰: 500

# 일일 비용 계산
입력: 1,000 x 2,000 / 1,000 x $0.003 = $6.00
출력: 1,000 x 500 / 1,000 x $0.015 = $7.50
일일 합계: $13.50

# 월간 비용 (30일 기준)
$13.50 x 30 = $405/월
```

**2) Provisioned Throughput (프로비저닝된 처리량) 방식**

일정 처리량을 미리 확보하고 시간 단위로 비용을 지불하는 방식입니다. **Model Unit(모델 유닛)**이라는 단위로 처리 용량을 구매합니다.

- **1개월 약정**: 온디맨드 대비 약 50% 할인
- **6개월 약정**: 온디맨드 대비 약 66% 할인
- **약정 없음**: 시간 단위 과금 (가장 비싸지만 유연)

```
# Provisioned Throughput 비용 예시
- Claude 3.5 Sonnet, 1 Model Unit
- 약정 없음: ~$39.60/시간
- 1개월 약정: ~$24.00/시간
- 6개월 약정: ~$16.80/시간
```

> **💡 핵심 포인트:** Provisioned Throughput는 처리량(throughput)을 보장받는 대신, 사용하지 않아도 비용이 발생합니다. 반면 On-demand는 호출한 만큼만 과금되지만, 트래픽이 몰릴 때 스로틀링(throttling)이 발생할 수 있습니다.

### 15-1-2 워크로드별 최적 선택 가이드

어떤 과금 방식을 선택할지는 **워크로드의 특성**에 따라 달라집니다.

**On-demand가 적합한 경우:**
- 개발/테스트 환경
- 트래픽이 불규칙하고 예측이 어려운 서비스
- 일일 호출 수가 수백 건 이하인 소규모 서비스
- 프로토타입 단계의 프로젝트

**Provisioned Throughput가 적합한 경우:**
- 24시간 안정적으로 운영되는 프로덕션 서비스
- 일일 호출 수가 수만 건 이상인 대규모 서비스
- 응답 시간 SLA(Service Level Agreement)가 엄격한 서비스
- 월 비용이 $1,000 이상 예상되는 경우

```
# 손익분기점 계산 예시 (Claude 3.5 Sonnet)
# Provisioned Throughput 1 Model Unit, 1개월 약정
월간 고정 비용: $24.00 x 24시간 x 30일 = $17,280/월

# On-demand로 같은 비용이면 몇 건 호출 가능한가?
# (입력 2K + 출력 500 토큰 기준, 건당 약 $0.0135)
$17,280 / $0.0135 ≈ 약 1,280,000건/월 ≈ 약 42,667건/일

# 결론: 하루 4만 건 이상이면 Provisioned Throughput 고려
```

> **⚠️ 주의:** 위 계산은 단순화한 예시입니다. 실제 Provisioned Throughput의 Model Unit당 처리 가능한 토큰 수는 모델과 입출력 비율에 따라 달라집니다. AWS 공식 문서와 가격 계산기(AWS Pricing Calculator)로 정확한 비교를 수행하세요.

> 📷 **[이미지]** AWS Pricing Calculator에서 Bedrock On-demand vs Provisioned Throughput를 비교하는 화면. 월간 예상 호출 수를 입력하면 양쪽 비용을 나란히 보여주는 그래프



---

## 15-2 CloudWatch 모니터링 설정

서비스가 돌아가기 시작하면, "지금 잘 돌아가고 있는가?"를 실시간으로 파악할 수 있어야 합니다. AWS에서는 **CloudWatch**가 이 역할을 담당합니다. CloudWatch는 AWS의 모든 서비스에서 발생하는 지표(Metrics), 로그(Logs), 알람(Alarms)을 통합 관리하는 모니터링 허브입니다.

### 15-2-1 Bedrock 호출 지표 모니터링

Amazon Bedrock은 자동으로 CloudWatch에 주요 지표를 전송합니다. 별도의 코드 없이도 다음 지표들을 확인할 수 있습니다.

| 지표 이름 | 설명 | 단위 |
|-----------|------|------|
| `Invocations` | 총 API 호출 수 | Count |
| `InvocationLatency` | API 호출 응답 시간 | Milliseconds |
| `InvocationClientErrors` | 클라이언트 오류 (4xx) | Count |
| `InvocationServerErrors` | 서버 오류 (5xx) | Count |
| `InvocationThrottles` | 스로틀링(요청 제한) 횟수 | Count |
| `InputTokenCount` | 입력 토큰 수 | Count |
| `OutputTokenCount` | 출력 토큰 수 | Count |

AWS CLI로 Bedrock 지표를 조회하는 방법을 살펴보겠습니다.

```bash
# 최근 1시간 동안의 Bedrock 호출 수 조회
aws cloudwatch get-metric-statistics \
  --namespace "AWS/Bedrock" \
  --metric-name "Invocations" \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Sum \
  --region us-east-1

# 최근 1시간 동안의 평균 지연 시간 조회
aws cloudwatch get-metric-statistics \
  --namespace "AWS/Bedrock" \
  --metric-name "InvocationLatency" \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average \
  --region us-east-1
```

> **💡 핵심 포인트:** `--period 300`은 5분 간격으로 데이터를 집계한다는 의미입니다. 더 세밀한 모니터링이 필요하면 60(1분)으로 줄일 수 있지만, 비용이 증가할 수 있습니다.

Python(boto3)으로 지표를 프로그래밍 방식으로 조회할 수도 있습니다.

```python
import boto3
from datetime import datetime, timedelta

cloudwatch = boto3.client('cloudwatch', region_name='us-east-1')

response = cloudwatch.get_metric_statistics(
    Namespace='AWS/Bedrock',
    MetricName='InvocationLatency',
    StartTime=datetime.utcnow() - timedelta(hours=1),
    EndTime=datetime.utcnow(),
    Period=300,
    Statistics=['Average', 'p99'],  # 평균과 99 퍼센타일
    Dimensions=[
        {
            'Name': 'ModelId',
            'Value': 'anthropic.claude-3-5-sonnet-20241022-v2:0'
        }
    ]
)

for datapoint in sorted(response['Datapoints'], key=lambda x: x['Timestamp']):
    print(f"시간: {datapoint['Timestamp']}, 평균 지연: {datapoint['Average']:.0f}ms")
```

### 15-2-2 지연 시간(Latency)과 에러율 알림 설정

지표를 눈으로 확인하는 것만으로는 부족합니다. 문제가 발생했을 때 자동으로 알림을 받을 수 있도록 **CloudWatch Alarms**를 설정해야 합니다.

**1) 지연 시간 알람 설정**

평균 응답 시간이 5초를 초과하면 알림을 보내는 알람을 만들어 보겠습니다.

```bash
# 지연 시간 알람 생성
aws cloudwatch put-metric-alarm \
  --alarm-name "Bedrock-HighLatency-Alarm" \
  --alarm-description "Bedrock 평균 응답 시간이 5초를 초과할 때 알림" \
  --namespace "AWS/Bedrock" \
  --metric-name "InvocationLatency" \
  --statistic Average \
  --period 300 \
  --threshold 5000 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:bedrock-alerts \
  --region us-east-1
```

> **📌 참고:** `--alarm-actions`에 지정하는 ARN은 **SNS(Simple Notification Service) 토픽**입니다. 미리 SNS 토픽을 생성하고 이메일 구독을 설정해 두면, 알람이 울릴 때 이메일로 알림을 받을 수 있습니다.

**2) 에러율 알람 설정**

서버 오류(5xx)가 5분간 10건 이상 발생하면 알림을 보내는 알람입니다.

```bash
# 에러율 알람 생성
aws cloudwatch put-metric-alarm \
  --alarm-name "Bedrock-HighErrorRate-Alarm" \
  --alarm-description "Bedrock 서버 에러가 5분간 10건 이상일 때 알림" \
  --namespace "AWS/Bedrock" \
  --metric-name "InvocationServerErrors" \
  --statistic Sum \
  --period 300 \
  --threshold 10 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:bedrock-alerts \
  --region us-east-1
```

**3) 스로틀링 알람 설정**

스로틀링은 요청이 너무 많아 AWS가 일부 요청을 거부하는 현상입니다. 이것이 자주 발생하면 Provisioned Throughput로의 전환을 고려해야 합니다.

```bash
# 스로틀링 알람 생성
aws cloudwatch put-metric-alarm \
  --alarm-name "Bedrock-Throttling-Alarm" \
  --alarm-description "Bedrock 스로틀링이 5분간 5건 이상일 때 알림" \
  --namespace "AWS/Bedrock" \
  --metric-name "InvocationThrottles" \
  --statistic Sum \
  --period 300 \
  --threshold 5 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:bedrock-alerts \
  --region us-east-1
```

### 15-2-3 비용 알림과 대시보드 구성

**1) AWS Budgets를 이용한 비용 알림**

예상 비용을 초과하기 전에 미리 경고를 받으려면 **AWS Budgets**를 활용합니다.

```bash
# 월 $500 예산 알림 생성 (80%, 100% 도달 시 알림)
aws budgets create-budget \
  --account-id 123456789012 \
  --budget '{
    "BudgetName": "Bedrock-Monthly-Budget",
    "BudgetLimit": {
      "Amount": "500",
      "Unit": "USD"
    },
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST",
    "CostFilters": {
      "Service": ["Amazon Bedrock"]
    }
  }' \
  --notifications-with-subscribers '[
    {
      "Notification": {
        "NotificationType": "ACTUAL",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 80,
        "ThresholdType": "PERCENTAGE"
      },
      "Subscribers": [
        {
          "SubscriptionType": "EMAIL",
          "Address": "admin@example.com"
        }
      ]
    },
    {
      "Notification": {
        "NotificationType": "ACTUAL",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 100,
        "ThresholdType": "PERCENTAGE"
      },
      "Subscribers": [
        {
          "SubscriptionType": "EMAIL",
          "Address": "admin@example.com"
        }
      ]
    }
  ]'
```

**2) CloudWatch 대시보드 구성**

여러 지표를 한눈에 볼 수 있는 커스텀 대시보드를 만들어 봅시다.

```bash
# CloudWatch 대시보드 생성
aws cloudwatch put-dashboard \
  --dashboard-name "Bedrock-Operations" \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric",
        "x": 0, "y": 0, "width": 12, "height": 6,
        "properties": {
          "title": "API 호출 수",
          "metrics": [
            ["AWS/Bedrock", "Invocations", {"stat": "Sum", "period": 300}]
          ],
          "region": "us-east-1"
        }
      },
      {
        "type": "metric",
        "x": 12, "y": 0, "width": 12, "height": 6,
        "properties": {
          "title": "응답 지연 시간 (ms)",
          "metrics": [
            ["AWS/Bedrock", "InvocationLatency", {"stat": "Average", "period": 300}],
            ["AWS/Bedrock", "InvocationLatency", {"stat": "p99", "period": 300}]
          ],
          "region": "us-east-1"
        }
      },
      {
        "type": "metric",
        "x": 0, "y": 6, "width": 12, "height": 6,
        "properties": {
          "title": "에러 및 스로틀링",
          "metrics": [
            ["AWS/Bedrock", "InvocationClientErrors", {"stat": "Sum", "period": 300}],
            ["AWS/Bedrock", "InvocationServerErrors", {"stat": "Sum", "period": 300}],
            ["AWS/Bedrock", "InvocationThrottles", {"stat": "Sum", "period": 300}]
          ],
          "region": "us-east-1"
        }
      },
      {
        "type": "metric",
        "x": 12, "y": 6, "width": 12, "height": 6,
        "properties": {
          "title": "토큰 사용량",
          "metrics": [
            ["AWS/Bedrock", "InputTokenCount", {"stat": "Sum", "period": 300}],
            ["AWS/Bedrock", "OutputTokenCount", {"stat": "Sum", "period": 300}]
          ],
          "region": "us-east-1"
        }
      }
    ]
  }'
```

> 📷 **[이미지]** CloudWatch 대시보드 화면. 4개의 위젯이 2x2 격자로 배치되어 있으며, API 호출 수, 응답 지연 시간, 에러/스로틀링 현황, 토큰 사용량을 실시간으로 보여주는 모습



---

## 15-3 운영 체크리스트와 보안 점검

서비스를 운영 환경에 올렸다면, 체계적인 점검 절차가 필요합니다. 이 절에서는 배포 직후 확인해야 할 항목과, 지속적으로 관리해야 할 보안/비용/품질 가이드를 정리합니다.

### 15-3-1 배포 후 기본 동작 점검 체크리스트

배포 직후 가장 먼저 해야 할 일은 서비스가 '살아있는지' 확인하는 것입니다. **ECS/Fargate**나 **ALB**는 헬스 체크 엔드포인트를 주기적으로 호출하여 서비스 상태를 확인합니다. 이 엔드포인트가 정상 응답하지 않으면 컨테이너를 재시작하거나 트래픽을 차단합니다.

**헬스 체크 엔드포인트 구현 (FastAPI)**

```python
from fastapi import FastAPI
import boto3
from botocore.exceptions import ClientError

app = FastAPI()

@app.get("/health")
async def health_check():
    """
    ALB/ECS가 서비스 상태를 확인하기 위해 호출하는 엔드포인트.
    단순 생존 확인이므로 빠르게 응답해야 합니다.
    """
    return {"status": "ok", "service": "bedrock-rag-api"}

@app.get("/health/deep")
async def deep_health_check():
    """
    Bedrock 연결 상태까지 확인하는 심층 헬스 체크.
    주기적 모니터링용으로, ALB 헬스 체크에는 사용하지 마세요.
    (외부 서비스 의존성 때문에 느릴 수 있습니다)
    """
    try:
        client = boto3.client('bedrock-runtime', region_name='us-east-1')
        # 가장 가벼운 모델로 최소한의 호출 테스트
        response = client.invoke_model(
            modelId='amazon.titan-text-lite-v1',
            body='{"inputText": "hi", "textGenerationConfig": {"maxTokenCount": 1}}'
        )
        return {
            "status": "ok",
            "service": "bedrock-rag-api",
            "bedrock_connection": "healthy"
        }
    except ClientError as e:
        return {
            "status": "degraded",
            "service": "bedrock-rag-api",
            "bedrock_connection": "unhealthy",
            "error": str(e)
        }
```

> **💡 핵심 포인트:** 헬스 체크 엔드포인트는 두 가지를 분리하는 것이 좋습니다. `/health`는 ALB용으로 빠르고 가벼워야 하고, `/health/deep`은 외부 의존성(Bedrock 연결)까지 확인하는 용도로 별도 모니터링에 활용합니다.

**배포 직후 점검 체크리스트:**

| 순번 | 점검 항목 | 확인 방법 | 기대 결과 |
|------|----------|----------|----------|
| 1 | 서비스 기동 확인 | `curl https://<서비스URL>/health` | `{"status": "ok"}` |
| 2 | Bedrock 연결 확인 | `curl https://<서비스URL>/health/deep` | `bedrock_connection: healthy` |
| 3 | 기본 질의 테스트 | 실제 질문을 보내고 답변 확인 | 정상 답변 반환 |
| 4 | 스트리밍 응답 확인 | 스트리밍 엔드포인트 호출 | 토큰이 순차적으로 도착 |
| 5 | 에러 핸들링 확인 | 빈 질문, 너무 긴 입력 전송 | 적절한 에러 메시지 반환 |
| 6 | CloudWatch 로그 확인 | AWS 콘솔에서 로그 그룹 조회 | 요청/응답 로그 정상 기록 |
| 7 | 알람 동작 확인 | 테스트 알람 트리거 | 이메일/Slack 알림 수신 |

### 15-3-2 보안/비용/품질 관점의 운영 가이드

운영 환경에서는 세 가지 축 -- **보안**, **비용**, **품질** -- 을 균형 있게 관리해야 합니다.

**보안 체크리스트**

```
[보안] IAM 최소 권한 원칙
 - Bedrock 호출용 IAM Role에 bedrock:InvokeModel만 부여
 - bedrock:* 같은 와일드카드 권한은 절대 사용 금지
 - Resource ARN도 특정 모델로 제한

[보안] VPC 엔드포인트를 통한 프라이빗 통신
 - Bedrock용 VPC 엔드포인트(PrivateLink) 구성
 - 인터넷을 경유하지 않고 AWS 내부 네트워크로 통신
 - 민감한 데이터가 공용 인터넷에 노출되지 않도록 보호

[보안] 데이터 암호화
 - S3 버킷: SSE-KMS 또는 SSE-S3로 저장 시 암호화
 - 전송 중 암호화: TLS 1.2 이상 (Bedrock API는 기본 적용)
 - Knowledge Base 벡터 스토어의 암호화 설정 확인
```

최소 권한 IAM 정책의 예시를 보겠습니다.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowBedrockInvoke",
            "Effect": "Allow",
            "Action": [
                "bedrock:InvokeModel",
                "bedrock:InvokeModelWithResponseStream"
            ],
            "Resource": [
                "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-5-sonnet-*",
                "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-haiku-*"
            ]
        },
        {
            "Sid": "AllowKnowledgeBaseRetrieve",
            "Effect": "Allow",
            "Action": [
                "bedrock:Retrieve",
                "bedrock:RetrieveAndGenerate"
            ],
            "Resource": "arn:aws:bedrock:us-east-1:123456789012:knowledge-base/KB_ID"
        }
    ]
}
```

> **⚠️ 주의:** API 키나 AWS 자격 증명을 코드에 하드코딩하지 마세요. ECS Task Role이나 Lambda Execution Role을 통해 자격 증명을 자동으로 부여받도록 구성해야 합니다. 환경 변수로 관리하더라도 AWS Secrets Manager를 활용하는 것이 모범 사례입니다.

**비용 관리 가이드**

```
[비용] 토큰 사용량 최적화
 - 프롬프트에 불필요한 컨텍스트를 넣지 않기
 - RAG 검색 결과 수(top_k)를 적절히 조절 (3~5개 권장)
 - 용도에 맞는 모델 선택 (간단한 분류는 Haiku, 복잡한 추론은 Sonnet)

[비용] 캐싱 전략
 - 동일 질문에 대한 응답을 캐싱 (ElastiCache, DynamoDB)
 - 임베딩 결과를 캐싱하여 중복 임베딩 API 호출 방지

[비용] 정기적 비용 리뷰
 - 주간 비용 리포트 확인 (AWS Cost Explorer)
 - 사용하지 않는 Provisioned Throughput 해제
 - 토큰 사용량 추이 모니터링
```

**품질 관리 가이드**

```
[품질] 응답 품질 모니터링
 - 사용자 피드백(좋아요/싫어요) 수집 체계 구축
 - 주기적 응답 샘플링 및 수동 검토 (주 1회 이상)
 - 환각(Hallucination) 비율 추적

[품질] Knowledge Base 관리
 - 문서 변경 시 S3 동기화(Sync) 재실행
 - 청킹 전략과 검색 품질 정기 점검
 - 오래된 문서 정리 및 메타데이터 업데이트

[품질] 지연 시간 관리
 - P50 목표: 2초 이내
 - P99 목표: 5초 이내
 - 목표 초과 시 원인 분석 (모델 선택, 프롬프트 길이, 네트워크)
```

> 📷 **[이미지]** 운영 체크리스트를 보안/비용/품질 세 축으로 분류한 다이어그램. 각 축에서 핵심 점검 항목이 체크박스 형태로 나열되어 있는 모습



---

## 15-4 종합 프로젝트: 고객 지원 챗봇 또는 사내 문서 Q&A 어시스턴트

이제 이 책에서 배운 모든 것을 하나의 프로젝트로 통합할 시간입니다. 이번 절에서는 **실무에서 가장 흔히 요구되는 두 가지 유형** -- 고객 지원 챗봇과 사내 문서 Q&A 어시스턴트 -- 중 하나를 선택하여, 요구사항 정의부터 아키텍처 설계, 구현, 평가까지 End-to-End로 진행하는 종합 프로젝트를 설계합니다.

### 15-4-1 프로젝트 요구사항 정의

**프로젝트 A: 고객 지원 챗봇**

```
[프로젝트명] AI 기반 고객 지원 챗봇
[목적] 고객 문의의 70% 이상을 AI가 자동 응답하여 상담 인력 부담 경감
[대상 사용자] 제품/서비스 이용 고객

[기능 요구사항]
FR-01: 제품 FAQ 기반 자동 답변 (RAG)
FR-02: 주문 상태 조회 (Agent + Action Group)
FR-03: 반품/교환 절차 안내 (RAG + 단계별 가이드)
FR-04: 상담원 연결 에스컬레이션 (AI가 판단하여 전환)
FR-05: 대화 이력 기반 문맥 유지 (멀티턴 대화)

[비기능 요구사항]
NFR-01: 평균 응답 시간 3초 이내
NFR-02: 동시 사용자 100명 지원
NFR-03: 99.9% 가용성
NFR-04: 개인정보(PII) 마스킹 (Guardrails 적용)
NFR-05: 월 운영 비용 $500 이내
```

**프로젝트 B: 사내 문서 Q&A 어시스턴트**

```
[프로젝트명] 사내 문서 Q&A 어시스턴트
[목적] 사내 규정, 매뉴얼, 기술 문서를 AI로 빠르게 검색/요약
[대상 사용자] 사내 임직원

[기능 요구사항]
FR-01: 사내 규정집 기반 질의응답 (RAG)
FR-02: 기술 매뉴얼 검색 및 요약 (RAG + 요약)
FR-03: 답변에 출처(문서명, 페이지) 표기 (Citation)
FR-04: 부서별 문서 접근 권한 필터링 (Metadata Filter)
FR-05: 문서 업데이트 시 자동 재인덱싱

[비기능 요구사항]
NFR-01: 평균 응답 시간 2초 이내
NFR-02: 동시 사용자 50명 지원
NFR-03: 사내 네트워크에서만 접근 가능 (VPC)
NFR-04: 문서 데이터 암호화 (KMS)
NFR-05: 월 운영 비용 $300 이내
```

> **💡 핵심 포인트:** 요구사항을 작성할 때는 반드시 **기능 요구사항(FR)**과 **비기능 요구사항(NFR)**을 분리하세요. 특히 NFR의 비용 목표와 성능 목표는 아키텍처 선택에 직접적인 영향을 미칩니다.

### 15-4-2 아키텍처 설계서 작성

두 프로젝트 모두 유사한 아키텍처 패턴을 따릅니다. 아래는 **사내 문서 Q&A 어시스턴트**를 기준으로 한 아키텍처입니다.

**전체 아키텍처 흐름:**

```
[사용자]
    → [ALB (Application Load Balancer)]
    → [ECS/Fargate - FastAPI 서버]
    → [Bedrock Agent]
        → [Knowledge Base] → [OpenSearch Serverless (벡터 스토어)]
        → [S3 (문서 원본)]
    → [Bedrock Foundation Model (Claude 3.5 Sonnet)]
    → [응답 반환]

[부가 서비스]
    - CloudWatch: 모니터링 + 로깅 + 알람
    - Secrets Manager: API 키, DB 비밀번호 관리
    - IAM: 최소 권한 역할 관리
    - VPC + PrivateLink: 프라이빗 네트워크 통신
    - Guardrails: PII 마스킹 + 유해 콘텐츠 차단
```

> 📷 **[이미지]** 종합 프로젝트 아키텍처 다이어그램. 좌측의 사용자로부터 ALB, ECS/Fargate, Bedrock Agent, Knowledge Base, S3까지의 데이터 흐름이 화살표로 연결되어 있고, 하단에 CloudWatch, IAM, Guardrails 등 부가 서비스가 점선으로 연결된 모습

**기술 스택 선정 근거:**

| 구성 요소 | 선택 기술 | 선정 이유 |
|-----------|----------|----------|
| 컴퓨팅 | ECS/Fargate | 컨테이너 기반, 서버 관리 불필요, 오토스케일링 |
| AI 모델 | Claude 3.5 Sonnet | 한국어 성능 우수, 장문 처리 능력 |
| 벡터 스토어 | OpenSearch Serverless | Knowledge Base와 네이티브 통합, 관리 부담 최소 |
| 문서 저장소 | S3 | 비용 효율적, Knowledge Base 데이터 소스로 직접 연동 |
| 로드 밸런서 | ALB | HTTP/HTTPS 라우팅, 헬스 체크 지원 |
| 모니터링 | CloudWatch | AWS 네이티브 통합, 알람 자동화 |
| 보안 | Guardrails + IAM + VPC | 다층 보안 체계 |

**아키텍처 설계서 작성 시 포함해야 할 항목:**

```
1. 시스템 개요
   - 프로젝트 목적과 범위
   - 핵심 기능 목록

2. 아키텍처 다이어그램
   - 전체 시스템 구성도
   - 데이터 흐름도
   - 네트워크 구성도 (VPC, 서브넷, 보안 그룹)

3. 기술 스택 상세
   - 각 구성 요소별 선택 기술과 버전
   - 선정 근거와 대안 비교

4. 데이터 아키텍처
   - 문서 저장 구조 (S3 버킷 설계)
   - 청킹 전략 (크기, 중첩, 메타데이터)
   - 벡터 인덱스 설계

5. 보안 아키텍처
   - IAM 역할 및 정책
   - 네트워크 보안 (VPC, 보안 그룹)
   - 데이터 암호화 전략

6. 운영 아키텍처
   - 모니터링 전략 (지표, 알람, 대시보드)
   - 로깅 전략
   - 비용 관리 전략

7. 확장 계획
   - 스케일링 전략
   - 향후 기능 로드맵
```

### 15-4-3 End-to-End 구현 및 발표

프로젝트 구현은 **4단계**로 나누어 진행합니다. 각 단계가 완료될 때마다 동작을 확인하고 다음 단계로 넘어가는 **점진적 빌드(Incremental Build)** 방식을 추천합니다.

**1단계: 데이터 레이어 구축 (Part 3에서 학습)**

```python
# S3에 문서 업로드 및 Knowledge Base 연동
import boto3

s3 = boto3.client('s3')

# 문서 업로드
s3.upload_file(
    Filename='company_policy.pdf',
    Bucket='my-rag-documents',
    Key='policies/company_policy.pdf'
)

# Knowledge Base 동기화 트리거
bedrock_agent = boto3.client('bedrock-agent')
bedrock_agent.start_ingestion_job(
    knowledgeBaseId='KB_12345',
    dataSourceId='DS_67890'
)
```

**2단계: RAG + Agent 로직 구현 (Part 4에서 학습)**

```python
# Knowledge Base 기반 RAG 질의응답
bedrock_agent_runtime = boto3.client('bedrock-agent-runtime')

def ask_question(question: str, session_id: str) -> dict:
    """Knowledge Base 기반 RAG 질의응답"""
    response = bedrock_agent_runtime.retrieve_and_generate(
        input={'text': question},
        retrieveAndGenerateConfiguration={
            'type': 'KNOWLEDGE_BASE',
            'knowledgeBaseConfiguration': {
                'knowledgeBaseId': 'KB_12345',
                'modelArn': 'arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-5-sonnet-20241022-v2:0',
                'retrievalConfiguration': {
                    'vectorSearchConfiguration': {
                        'numberOfResults': 5
                    }
                },
                'generationConfiguration': {
                    'promptTemplate': {
                        'textPromptTemplate': (
                            '다음 검색 결과를 바탕으로 질문에 답변하세요.\n'
                            '검색 결과에 없는 내용은 "해당 정보를 찾을 수 없습니다"라고 답하세요.\n\n'
                            '검색 결과:\n$search_results$\n\n'
                            '질문: $query$\n\n'
                            '답변:'
                        )
                    }
                }
            }
        },
        sessionId=session_id
    )

    answer = response['output']['text']
    citations = response.get('citations', [])

    return {
        'answer': answer,
        'sources': [
            {
                'text': c['generatedResponsePart']['textResponsePart']['text'],
                'location': c['retrievedReferences'][0]['location']['s3Location']['uri']
            }
            for c in citations if c.get('retrievedReferences')
        ]
    }
```

**3단계: API 서버 + 컨테이너화 (Chapter 13에서 학습)**

```python
# FastAPI 서버 핵심 코드
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import uuid

app = FastAPI(title="Document Q&A Assistant")

class QuestionRequest(BaseModel):
    question: str
    session_id: str | None = None

class AnswerResponse(BaseModel):
    answer: str
    sources: list
    session_id: str

@app.post("/ask", response_model=AnswerResponse)
async def ask(request: QuestionRequest):
    session_id = request.session_id or str(uuid.uuid4())

    try:
        result = ask_question(request.question, session_id)
        return AnswerResponse(
            answer=result['answer'],
            sources=result['sources'],
            session_id=session_id
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"질의 처리 중 오류: {str(e)}")
```

**4단계: 배포 + 모니터링 (Chapter 14, 15에서 학습)**

```bash
# ECR에 이미지 푸시
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

docker build -t doc-qa-assistant .
docker tag doc-qa-assistant:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/doc-qa-assistant:latest
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/doc-qa-assistant:latest

# ECS 서비스 업데이트 (새 이미지 배포)
aws ecs update-service \
  --cluster my-cluster \
  --service doc-qa-service \
  --force-new-deployment
```

**발표 시 포함할 내용:**

프로젝트를 발표할 때는 다음 구조를 따르면 효과적입니다.

```
1. 프로젝트 소개 (2분)
   - 해결하려는 문제와 목표
   - 대상 사용자와 기대 효과

2. 아키텍처 설명 (3분)
   - 전체 시스템 구성도
   - 핵심 기술 선택 근거

3. 라이브 데모 (5분)
   - 실제 질문/답변 시연
   - 출처 표시 기능 시연
   - 에러 처리 시연

4. 운영 현황 (2분)
   - CloudWatch 대시보드 화면
   - 비용 현황

5. 회고 및 향후 계획 (3분)
   - 기술적 도전과 해결 과정
   - 개선하고 싶은 점
   - 향후 로드맵
```

### 15-4-4 평가 기준표와 피드백

종합 프로젝트의 평가는 네 가지 축으로 이루어집니다.

| 평가 영역 | 배점 | 세부 기준 |
|-----------|------|----------|
| **정확성 (Accuracy)** | 30점 | - RAG 검색 적합도 (관련 문서를 잘 찾는가?) <br> - 답변의 사실 정확성 (환각이 없는가?) <br> - 출처 표기의 정확성 |
| **지연 시간 (Latency)** | 20점 | - P50 응답 시간 2초 이내 <br> - P99 응답 시간 5초 이내 <br> - 스트리밍 첫 토큰 시간 1초 이내 |
| **비용 효율 (Cost)** | 20점 | - 월간 예상 비용이 목표 이내 <br> - 토큰 최적화 전략 적용 여부 <br> - 모델 선택의 적절성 |
| **사용자 경험 (UX)** | 30점 | - UI의 직관성과 완성도 <br> - 에러 상황 대처의 우아함 <br> - 대화 맥락 유지 능력 <br> - 보안/개인정보 보호 |

**자가 평가 체크리스트:**

각 항목을 스스로 점검하며, 부족한 부분을 보완하세요.

```
[ ] Knowledge Base에 문서가 정상적으로 인덱싱되었는가?
[ ] 5개 이상의 테스트 질문에 대해 정확한 답변을 제공하는가?
[ ] 답변에 출처(문서명)가 정확히 표시되는가?
[ ] 관련 없는 질문에 "모르겠다"고 적절히 응답하는가?
[ ] 헬스 체크 엔드포인트가 정상 동작하는가?
[ ] CloudWatch에 로그와 지표가 기록되고 있는가?
[ ] 알람이 설정되어 있고, 테스트해 보았는가?
[ ] IAM 정책이 최소 권한 원칙을 따르는가?
[ ] Guardrails가 적용되어 PII가 마스킹되는가?
[ ] 비용 알림(Budget Alert)이 설정되어 있는가?
```

**피드백 수집 방법:**

프로젝트 완성 후에는 다음과 같은 방법으로 피드백을 수집합니다.

1. **동료 리뷰(Peer Review)**: 다른 팀원이 실제로 사용해 보고 피드백 작성
2. **응답 품질 평가**: 10개 이상의 표준 질문 세트로 정확도 측정
3. **부하 테스트**: 동시 접속자를 점진적으로 늘려가며 성능 한계 확인
4. **보안 점검**: IAM 정책, 네트워크 설정, 암호화 상태 최종 검증



---

## 마치며

15개의 챕터를 거치며 정말 긴 여정을 함께했습니다. 처음에는 "생성형 AI가 뭐지?"라는 질문에서 시작했는데, 이제 여러분은 **AWS Bedrock 기반의 AI 서비스를 설계하고, 구축하고, 배포하고, 운영할 수 있는 역량**을 갖추게 되었습니다.

우리가 함께 걸어온 길을 돌아보겠습니다.

**Part 1 -- Amazon Bedrock과 생성형 AI 서비스 큰 그림**에서는 생성형 AI의 핵심 개념과 AWS 생태계를 이해했습니다. Bedrock이 다른 클라우드 서비스와 어떻게 다른지, 왜 Bedrock을 선택해야 하는지를 배웠습니다.

**Part 2 -- AWS 환경 구성과 Bedrock API 기본기**에서는 AWS 계정 설정부터 IAM 보안, SDK 구성, 그리고 Foundation Model API 호출까지 개발의 기초를 다졌습니다. 처음으로 Bedrock에게 질문을 보내고 답변을 받았던 그 순간의 짜릿함을 기억하시나요?

**Part 3 -- 임베딩과 RAG를 위한 데이터 레이어 설계**에서는 임베딩의 개념, 문서 청킹 전략, Knowledge Bases 구성까지 RAG의 핵심 데이터 레이어를 설계했습니다. AI가 "아는 척"하지 않고 실제 문서에 기반한 답변을 하게 만드는 기술의 핵심을 이해했습니다.

**Part 4 -- Bedrock 애플리케이션 패턴과 에이전트 확장**에서는 RAG 질의응답, 프롬프트 고도화, Agents, Guardrails까지 실무에서 필요한 핵심 기능을 모두 다뤘습니다. AI가 단순한 질의응답을 넘어 외부 시스템과 연동하여 실제 업무를 수행하는 모습을 구현했습니다.

**Part 5 -- 서비스 배포와 운영 최적화**에서는 웹 UI 구축, 컨테이너화, 클라우드 배포, 그리고 이번 챕터에서 다룬 운영 최적화까지, 아이디어를 실제 서비스로 만드는 전 과정을 완성했습니다.

이제 여러분은:

- **Bedrock API**를 자유자재로 호출하고 응답을 처리할 수 있습니다.
- **RAG 아키텍처**를 설계하여 환각을 줄이고 정확한 답변을 제공할 수 있습니다.
- **Agents와 Guardrails**로 안전하고 강력한 AI 서비스를 구축할 수 있습니다.
- **Docker와 ECS/Fargate**로 서비스를 컨테이너화하여 배포할 수 있습니다.
- **CloudWatch**로 서비스를 모니터링하고 비용을 최적화할 수 있습니다.

하지만 이것은 끝이 아니라 **새로운 시작**입니다. 생성형 AI 기술은 하루가 다르게 발전하고 있습니다. 새로운 모델이 출시되고, 새로운 기능이 추가되고, 새로운 패턴이 등장합니다. 이 책에서 배운 기초 위에 끊임없이 새로운 것을 쌓아가시길 바랍니다.

실제 서비스를 운영하다 보면 예상치 못한 문제들을 만나게 될 것입니다. 그때마다 이 책으로 돌아와 기본기를 되짚어 보세요. 그리고 무엇보다, **직접 만들어 보는 것**이 가장 좋은 학습 방법이라는 것을 잊지 마세요. 작은 프로젝트라도 좋으니 직접 서비스를 만들고 배포해 보는 경험이 여러분을 한 단계 더 성장시킬 것입니다.

여러분의 AI 서비스가 실제 사용자에게 가치를 제공하는 멋진 서비스가 되기를 진심으로 응원합니다.

긴 여정을 함께해 주셔서 감사합니다. 행운을 빕니다!
