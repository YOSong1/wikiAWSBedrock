Chapter 13에서 우리는 FastAPI 서버를 도커 이미지로 포장하고, Amazon ECR에 안전하게 업로드하는 과정까지 완료했습니다. 이제 이 이미지를 실제 클라우드 환경에 배포하여 누구나 접속할 수 있는 서비스로 만들 차례입니다.

AWS에서 애플리케이션을 배포하는 방식은 크게 두 가지 갈래로 나뉩니다. 하나는 **AWS Lambda**를 활용한 서버리스 배포이고, 다른 하나는 **ECS/Fargate**를 활용한 컨테이너 기반 배포입니다. Lambda는 "함수 하나만 딱 실행"하는 경량 방식이고, ECS/Fargate는 "도커 컨테이너를 통째로 띄우는" 본격적인 서비스 운영 방식입니다.

이번 챕터에서는 두 가지 방식을 모두 실습하고, 마지막으로 Bedrock의 **Custom Model Training(Fine-tuning)** 기능을 통해 모델 자체를 우리 도메인에 맞게 튜닝하는 방법까지 살펴보겠습니다.



---

## 14-1 Lambda 기반 서버리스 배포

**AWS Lambda**는 서버를 관리할 필요 없이 코드만 업로드하면 실행되는 서버리스 컴퓨팅 서비스입니다. "서버리스"란 서버가 없다는 뜻이 아니라, **서버의 프로비저닝, 스케일링, 패치를 AWS가 전부 관리해준다**는 의미입니다. 우리는 비즈니스 로직에만 집중하면 됩니다.

Lambda의 핵심 특징은 다음과 같습니다.

- **이벤트 기반 실행:** HTTP 요청, S3 파일 업로드, SQS 메시지 등 다양한 이벤트에 반응하여 자동 실행됩니다.
- **사용한 만큼만 과금:** 함수가 실행된 시간(밀리초 단위)과 요청 횟수에 대해서만 비용이 발생합니다.
- **자동 스케일링:** 동시 요청이 1건이든 1,000건이든 AWS가 자동으로 처리합니다.

### 14-1-1 Lambda 함수로 Bedrock 호출 래핑

Lambda 함수는 `handler`라는 진입점 함수를 통해 동작합니다. API Gateway나 다른 AWS 서비스가 Lambda를 호출하면, 이 handler 함수가 `event`(요청 데이터)와 `context`(실행 환경 정보)를 인자로 받아 처리합니다.

Bedrock을 호출하는 Lambda 함수를 작성해 보겠습니다.

```python
# lambda_function.py
import json
import boto3

bedrock_runtime = boto3.client("bedrock-runtime", region_name="us-east-1")

def lambda_handler(event, context):
    """
    API Gateway에서 전달된 요청을 받아 Bedrock Claude 모델을 호출합니다.
    """
    # API Gateway에서 전달된 body 파싱
    body = json.loads(event.get("body", "{}"))
    user_message = body.get("message", "안녕하세요")

    # Bedrock Converse API 호출
    response = bedrock_runtime.converse(
        modelId="anthropic.claude-3-haiku-20240307-v1:0",
        messages=[
            {
                "role": "user",
                "content": [{"text": user_message}]
            }
        ],
        inferenceConfig={
            "maxTokens": 1024,
            "temperature": 0.7
        }
    )

    # 응답 추출
    assistant_message = response["output"]["message"]["content"][0]["text"]

    return {
        "statusCode": 200,
        "headers": {
            "Content-Type": "application/json",
            "Access-Control-Allow-Origin": "*"
        },
        "body": json.dumps({
            "response": assistant_message
        }, ensure_ascii=False)
    }
```

이 코드에서 주목할 점은 다음과 같습니다.

1. **`boto3.client` 초기화를 handler 바깥에 배치합니다.** Lambda는 동일한 실행 환경을 재사용할 수 있으므로(Warm Start), 클라이언트를 전역으로 선언하면 매 호출마다 재생성하는 비용을 아낄 수 있습니다.
2. **반환값의 형식**이 API Gateway가 기대하는 구조(`statusCode`, `headers`, `body`)를 따릅니다.
3. **CORS 헤더**(`Access-Control-Allow-Origin`)를 포함하여 프런트엔드에서 직접 호출할 수 있도록 합니다.

> **💡 핵심 포인트:** Lambda 함수에서 Bedrock을 호출하려면, Lambda의 **실행 역할(Execution Role)**에 `bedrock:InvokeModel` 권한이 반드시 포함되어야 합니다. IAM 콘솔에서 다음 정책을 연결합니다.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "bedrock:InvokeModel",
                "bedrock:InvokeModelWithResponseStream"
            ],
            "Resource": "arn:aws:bedrock:us-east-1::foundation-model/*"
        }
    ]
}
```

**Lambda 배포 방법**은 여러 가지가 있지만, AWS CLI를 사용하는 방법이 가장 직관적입니다.

```bash
# 1. 코드를 zip으로 패키징
zip function.zip lambda_function.py

# 2. Lambda 함수 생성
aws lambda create-function \
    --function-name bedrock-chat-function \
    --runtime python3.12 \
    --role arn:aws:iam::123456789012:role/lambda-bedrock-role \
    --handler lambda_function.lambda_handler \
    --zip-file fileb://function.zip \
    --timeout 30 \
    --memory-size 256
```

> **⚠️ 주의:** Bedrock 모델 호출은 일반적인 API 호출보다 시간이 오래 걸릴 수 있습니다. Lambda의 기본 타임아웃은 3초이므로, 반드시 `--timeout`을 30초 이상으로 설정해야 합니다. 최대 15분(900초)까지 설정 가능합니다.

**Lambda Layer를 활용한 의존성 관리**도 알아두면 좋습니다. `boto3`는 Lambda 런타임에 기본 포함되어 있지만, `langchain`이나 `chromadb` 같은 외부 라이브러리가 필요한 경우 **Lambda Layer**로 패키징하여 연결할 수 있습니다.

```bash
# Layer용 디렉토리 구성
mkdir -p python/lib/python3.12/site-packages
pip install langchain -t python/lib/python3.12/site-packages/
zip -r layer.zip python/

# Layer 생성
aws lambda publish-layer-version \
    --layer-name bedrock-dependencies \
    --zip-file fileb://layer.zip \
    --compatible-runtimes python3.12

# 함수에 Layer 연결
aws lambda update-function-configuration \
    --function-name bedrock-chat-function \
    --layers arn:aws:lambda:us-east-1:123456789012:layer:bedrock-dependencies:1
```

### 14-1-2 API Gateway 연동

Lambda 함수를 만들었지만, 아직 외부에서 HTTP로 접근할 수는 없습니다. Lambda에 인터넷 관문을 달아주는 것이 바로 **Amazon API Gateway**입니다. API Gateway는 HTTPS 엔드포인트를 생성하고, 들어오는 요청을 Lambda 함수로 라우팅해 줍니다.

> 📷 **[이미지]** [Client/Browser] -> (HTTPS) -> [API Gateway] -> (Invoke) -> [Lambda Function] -> (API Call) -> [Amazon Bedrock]으로 이어지는 요청 흐름도

**HTTP API 생성 (AWS CLI)**

```bash
# 1. HTTP API 생성
aws apigatewayv2 create-api \
    --name bedrock-chat-api \
    --protocol-type HTTP \
    --target arn:aws:lambda:us-east-1:123456789012:function:bedrock-chat-function

# 2. Lambda 리소스 기반 정책 추가 (API Gateway가 Lambda를 호출할 수 있도록)
aws lambda add-permission \
    --function-name bedrock-chat-function \
    --statement-id apigateway-invoke \
    --action lambda:InvokeFunction \
    --principal apigateway.amazonaws.com
```

명령이 성공하면 `https://{api-id}.execute-api.us-east-1.amazonaws.com` 형태의 엔드포인트가 생성됩니다. 이제 어디서든 이 URL로 요청을 보낼 수 있습니다.

```bash
# API 호출 테스트
curl -X POST https://abc123def.execute-api.us-east-1.amazonaws.com/ \
    -H "Content-Type: application/json" \
    -d '{"message": "AWS Bedrock이란 무엇인가요?"}'
```

> **📌 참고:** API Gateway에는 **REST API**와 **HTTP API** 두 가지 유형이 있습니다. HTTP API가 더 저렴하고 빠르며, 대부분의 사용 사례에 충분합니다. 요청 검증, 캐싱, API 키 관리 같은 고급 기능이 필요하다면 REST API를 선택합니다.

### 14-1-3 Cold Start 최적화와 동시성 제어

Lambda의 가장 큰 단점은 바로 **Cold Start(콜드 스타트)**입니다. Lambda 함수가 한동안 호출되지 않으면 실행 환경이 제거되고, 다음 요청이 들어올 때 새로운 환경을 만드는 데 시간이 걸립니다. 이 초기 지연 시간이 Cold Start입니다.

Bedrock 호출 자체가 1~5초 정도 걸리는데, Cold Start까지 더해지면 사용자 경험이 크게 저하될 수 있습니다. 이를 해결하는 방법을 알아봅시다.

**방법 1: Provisioned Concurrency (프로비저닝된 동시성)**

지정한 수만큼의 Lambda 인스턴스를 항상 "따뜻하게(Warm)" 유지해주는 기능입니다. Cold Start를 완전히 제거할 수 있지만, 유지 비용이 발생합니다.

```bash
# 10개의 인스턴스를 항상 준비 상태로 유지
aws lambda put-provisioned-concurrency-config \
    --function-name bedrock-chat-function \
    --qualifier prod \
    --provisioned-concurrent-executions 10
```

**방법 2: SnapStart**

Lambda SnapStart는 함수의 초기화 단계를 스냅샷으로 저장해두고, 호출 시 이 스냅샷에서 복원하여 시작 시간을 단축합니다. 현재 Java 런타임에서 지원되며, Python에서는 Provisioned Concurrency를 활용하는 것이 현실적입니다.

**방법 3: 예약된 동시성(Reserved Concurrency)**

특정 함수가 사용할 수 있는 동시 실행 수의 상한을 설정합니다. 이는 Cold Start 해결보다는 **비용 제어**와 **다운스트림 보호**에 유용합니다.

```bash
# 최대 동시 실행 수를 50으로 제한
aws lambda put-function-concurrency \
    --function-name bedrock-chat-function \
    --reserved-concurrent-executions 50
```

> **💡 핵심 포인트:** Lambda + Bedrock 조합은 **간헐적이고 가벼운 AI 호출**(슬랙봇, 알림, 단발성 요약 등)에 최적입니다. 반면 지속적인 트래픽이 예상되는 서비스라면, 다음 절에서 다루는 ECS/Fargate가 더 적합합니다.



---

## 14-2 ECS/Fargate 기반 컨테이너 배포

Lambda가 "함수 단위의 실행"이라면, **Amazon ECS(Elastic Container Service)**는 **도커 컨테이너 단위의 실행**입니다. Chapter 13에서 ECR에 업로드한 FastAPI 서버 이미지를 ECS를 통해 본격적인 프로덕션 서비스로 운영할 수 있습니다.

ECS에서 컨테이너를 실행하는 방식은 두 가지입니다.

- **EC2 시작 유형:** 직접 EC2 인스턴스를 관리하며 그 위에서 컨테이너를 실행합니다.
- **Fargate 시작 유형:** 서버 관리 없이 컨테이너만 정의하면 AWS가 알아서 인프라를 할당합니다.

이 책에서는 서버 관리 부담이 없는 **Fargate**를 사용합니다. Lambda와 마찬가지로 서버리스이지만, Lambda보다 훨씬 자유로운 실행 환경(긴 실행 시간, 자유로운 포트 설정, 풀 컨테이너 지원)을 제공합니다.

> 📷 **[이미지]** Lambda와 ECS/Fargate의 비교 다이어그램. Lambda는 "함수 코드"를 입력받고, ECS/Fargate는 "Docker 컨테이너 이미지"를 입력받는 구조. 실행 시간, 메모리 제한, 적합한 사용 사례를 표로 비교

### 14-2-1 ECS 클러스터 및 태스크 정의

ECS의 핵심 개념을 먼저 정리합니다.

- **클러스터(Cluster):** 컨테이너들이 실행되는 논리적 그룹입니다. 하나의 프로젝트 또는 서비스 단위로 생성합니다.
- **태스크 정의(Task Definition):** 컨테이너의 설계도입니다. 어떤 이미지를 사용하고, CPU/메모리를 얼마나 할당하고, 어떤 포트를 열지 등을 JSON으로 정의합니다.
- **서비스(Service):** 태스크 정의를 기반으로 실제 컨테이너를 실행하고 관리하는 단위입니다. 원하는 태스크 수를 유지해 줍니다.

**1단계: 클러스터 생성**

```bash
aws ecs create-cluster --cluster-name bedrock-service-cluster
```

**2단계: 태스크 실행 역할 생성**

ECS 태스크가 ECR에서 이미지를 가져오고 Bedrock을 호출하려면 적절한 IAM 역할이 필요합니다.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "bedrock:InvokeModel",
                "bedrock:InvokeModelWithResponseStream"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "*"
        }
    ]
}
```

> **⚠️ 주의:** ECS에는 두 종류의 IAM 역할이 있습니다. **Task Execution Role**은 ECR 이미지 풀, 로그 전송 등 ECS 인프라 작업에 사용되고, **Task Role**은 컨테이너 내부 애플리케이션이 AWS 서비스(Bedrock 등)를 호출할 때 사용됩니다. Bedrock 호출 권한은 **Task Role**에 부여해야 합니다.

**3단계: 태스크 정의 등록**

다음 JSON 파일을 `task-definition.json`으로 저장합니다.

```json
{
    "family": "bedrock-fastapi-task",
    "networkMode": "awsvpc",
    "requiresCompatibilities": ["FARGATE"],
    "cpu": "512",
    "memory": "1024",
    "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
    "taskRoleArn": "arn:aws:iam::123456789012:role/ecsTaskRole",
    "containerDefinitions": [
        {
            "name": "bedrock-fastapi",
            "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/bedrock-rag-server:latest",
            "portMappings": [
                {
                    "containerPort": 8000,
                    "protocol": "tcp"
                }
            ],
            "environment": [
                {
                    "name": "AWS_DEFAULT_REGION",
                    "value": "us-east-1"
                }
            ],
            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-group": "/ecs/bedrock-fastapi",
                    "awslogs-region": "us-east-1",
                    "awslogs-stream-prefix": "ecs"
                }
            },
            "essential": true
        }
    ]
}
```

```bash
# 태스크 정의 등록
aws ecs register-task-definition --cli-input-json file://task-definition.json
```

이 정의에서 주요 설정을 살펴보면 다음과 같습니다.

| 항목 | 값 | 설명 |
|------|-----|------|
| `cpu` | 512 | 0.5 vCPU (Fargate 최소 단위) |
| `memory` | 1024 | 1GB 메모리 |
| `networkMode` | awsvpc | Fargate 필수 - 각 태스크에 고유 ENI 할당 |
| `containerPort` | 8000 | FastAPI 서버의 기본 포트 |
| `logConfiguration` | awslogs | CloudWatch Logs로 컨테이너 로그 전송 |

### 14-2-2 Fargate 서비스 배포와 오토스케일링

태스크 정의가 완료되었으니, 이제 **서비스**를 생성하여 실제로 컨테이너를 실행합니다.

```bash
# ECS 서비스 생성
aws ecs create-service \
    --cluster bedrock-service-cluster \
    --service-name bedrock-fastapi-service \
    --task-definition bedrock-fastapi-task \
    --desired-count 2 \
    --launch-type FARGATE \
    --network-configuration "awsvpcConfiguration={
        subnets=[subnet-0123456789abcdef0,subnet-0123456789abcdef1],
        securityGroups=[sg-0123456789abcdef0],
        assignPublicIp=ENABLED
    }"
```

`--desired-count 2`는 항상 2개의 태스크(컨테이너)를 유지하겠다는 의미입니다. 하나가 비정상 종료되면 ECS가 자동으로 새 태스크를 시작하여 2개를 유지합니다.

**오토스케일링 설정**

트래픽이 급증할 때 자동으로 태스크 수를 늘리려면 **Application Auto Scaling**을 설정합니다.

```bash
# 1. 스케일링 대상 등록
aws application-autoscaling register-scalable-target \
    --service-namespace ecs \
    --resource-id service/bedrock-service-cluster/bedrock-fastapi-service \
    --scalable-dimension ecs:service:DesiredCount \
    --min-capacity 2 \
    --max-capacity 10

# 2. CPU 사용률 기반 스케일링 정책
aws application-autoscaling put-scaling-policy \
    --service-namespace ecs \
    --resource-id service/bedrock-service-cluster/bedrock-fastapi-service \
    --scalable-dimension ecs:service:DesiredCount \
    --policy-name cpu-scaling-policy \
    --policy-type TargetTrackingScaling \
    --target-tracking-scaling-policy-configuration '{
        "TargetValue": 70.0,
        "PredefinedMetricSpecification": {
            "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
        },
        "ScaleOutCooldown": 60,
        "ScaleInCooldown": 120
    }'
```

이 설정은 **CPU 사용률이 70%를 넘으면 태스크를 추가하고, 70% 아래로 떨어지면 점차 줄이는** 정책입니다. 최소 2개, 최대 10개의 태스크를 유지합니다.

> **💡 핵심 포인트:** Bedrock 호출은 CPU보다 네트워크 대기 시간이 지배적입니다. CPU 기반 스케일링 외에도 **요청 수 기반 스케일링**(ALBRequestCountPerTarget)을 함께 설정하면 더 정확한 오토스케일링이 가능합니다.

### 14-2-3 ALB(Application Load Balancer) 연동

현재 각 Fargate 태스크는 개별 IP를 가지고 있어서 직접 접근이 어렵습니다. **ALB(Application Load Balancer)**를 앞에 배치하면 단일 엔드포인트로 여러 태스크에 트래픽을 분산할 수 있습니다.

> 📷 **[이미지]** [Client] -> (HTTPS) -> [ALB] -> (분산) -> [Fargate Task 1] / [Fargate Task 2] / [Fargate Task N]으로 트래픽이 분산되는 아키텍처 다이어그램. 각 태스크는 Bedrock을 호출하는 화살표를 포함

**ALB 구성 순서**

```bash
# 1. ALB 생성
aws elbv2 create-load-balancer \
    --name bedrock-alb \
    --subnets subnet-0123456789abcdef0 subnet-0123456789abcdef1 \
    --security-groups sg-0123456789abcdef0 \
    --scheme internet-facing \
    --type application

# 2. 대상 그룹(Target Group) 생성
aws elbv2 create-target-group \
    --name bedrock-targets \
    --protocol HTTP \
    --port 8000 \
    --vpc-id vpc-0123456789abcdef0 \
    --target-type ip \
    --health-check-path "/health" \
    --health-check-interval-seconds 30

# 3. 리스너 생성 (80 포트 -> 대상 그룹으로 전달)
aws elbv2 create-listener \
    --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/bedrock-alb/abc123 \
    --protocol HTTP \
    --port 80 \
    --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/bedrock-targets/abc123
```

ALB를 생성한 후에는 ECS 서비스를 업데이트하여 ALB와 연결합니다. 이때 ECS 서비스를 새로 생성하면서 `--load-balancers` 옵션을 지정하는 것이 가장 깔끔합니다.

```bash
aws ecs create-service \
    --cluster bedrock-service-cluster \
    --service-name bedrock-fastapi-service-v2 \
    --task-definition bedrock-fastapi-task \
    --desired-count 2 \
    --launch-type FARGATE \
    --network-configuration "awsvpcConfiguration={
        subnets=[subnet-0123456789abcdef0,subnet-0123456789abcdef1],
        securityGroups=[sg-0123456789abcdef0],
        assignPublicIp=ENABLED
    }" \
    --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/bedrock-targets/abc123,containerName=bedrock-fastapi,containerPort=8000"
```

> **📌 참고:** FastAPI 서버에 `/health` 엔드포인트를 반드시 구현해 두어야 합니다. ALB는 이 경로로 주기적으로 요청을 보내 태스크가 정상인지 확인합니다. 응답이 없으면 해당 태스크를 비정상으로 판단하고 트래픽을 보내지 않습니다.

```python
# FastAPI 헬스체크 엔드포인트
@app.get("/health")
async def health_check():
    return {"status": "healthy"}
```

배포가 완료되면 ALB의 DNS 이름(예: `bedrock-alb-123456.us-east-1.elb.amazonaws.com`)으로 접속하여 서비스를 테스트할 수 있습니다.

```bash
curl http://bedrock-alb-123456.us-east-1.elb.amazonaws.com/health
# {"status": "healthy"}

curl -X POST http://bedrock-alb-123456.us-east-1.elb.amazonaws.com/chat \
    -H "Content-Type: application/json" \
    -d '{"message": "Amazon Bedrock의 장점을 알려주세요"}'
```



---

## 14-3 Fine-tuning 개요와 데이터셋 준비

지금까지 우리는 Bedrock이 제공하는 **기본(Foundation) 모델**을 그대로 사용해 왔습니다. 프롬프트 엔지니어링과 RAG를 통해 성능을 끌어올리는 방법도 배웠습니다. 하지만 특정 도메인에서 일관된 어조, 형식, 전문 지식을 요구하는 경우, 모델 자체를 추가 학습시키는 **Fine-tuning**이 더 효과적일 수 있습니다.

Amazon Bedrock은 **Custom Model Training** 기능을 통해 Foundation Model을 커스텀 데이터셋으로 미세 조정(Fine-tuning)할 수 있습니다.

### 14-3-1 Bedrock Custom Model Training 프로세스

Fine-tuning의 전체 흐름은 다음과 같습니다.

1. **데이터셋 준비:** 도메인에 맞는 학습 데이터를 JSONL 형식으로 작성합니다.
2. **S3 업로드:** 데이터셋 파일을 S3 버킷에 업로드합니다.
3. **Training Job 생성:** Bedrock 콘솔 또는 API로 Fine-tuning 작업을 시작합니다.
4. **모델 학습:** AWS가 지정된 Foundation Model을 데이터셋으로 추가 학습합니다.
5. **커스텀 모델 생성:** 학습이 완료되면 고유한 모델 ID가 부여됩니다.
6. **Provisioned Throughput 구매:** 커스텀 모델을 사용하려면 전용 처리량을 구매해야 합니다.
7. **호출:** 기존 InvokeModel/Converse API에 커스텀 모델 ID를 지정하여 호출합니다.

> 📷 **[이미지]** Fine-tuning 파이프라인 흐름도: [JSONL 데이터셋] -> [S3 버킷] -> [Bedrock Training Job] -> [Custom Model] -> [Provisioned Throughput] -> [InvokeModel API]

**AWS CLI로 Training Job 생성하기**

```bash
aws bedrock create-model-customization-job \
    --job-name my-finetuning-job \
    --custom-model-name my-custom-claude \
    --role-arn arn:aws:iam::123456789012:role/BedrockFineTuningRole \
    --base-model-identifier anthropic.claude-3-haiku-20240307-v1:0 \
    --training-data-config '{"s3Uri": "s3://my-bucket/training/train.jsonl"}' \
    --validation-data-config '{"validators": [{"s3Uri": "s3://my-bucket/training/validation.jsonl"}]}' \
    --output-data-config '{"s3Uri": "s3://my-bucket/output/"}' \
    --hyper-parameters '{"epochCount": "3", "batchSize": "1", "learningRate": "0.00001"}'
```

> **⚠️ 주의:** 모든 Foundation Model이 Fine-tuning을 지원하는 것은 아닙니다. 2025년 기준으로 **Amazon Titan** 시리즈와 **Meta Llama** 시리즈, 일부 **Cohere** 모델이 Fine-tuning을 지원합니다. Anthropic Claude 모델의 Fine-tuning 지원은 제한적이므로, 사전에 AWS 공식 문서에서 최신 지원 현황을 반드시 확인하세요.

### 14-3-2 데이터셋 준비 기준과 포맷

Fine-tuning의 성패는 **데이터셋의 품질**에 달려 있습니다. 아무리 좋은 모델이라도 부정확하거나 일관성 없는 데이터로 학습하면 오히려 성능이 떨어집니다.

**JSONL 데이터 형식**

Bedrock Fine-tuning용 데이터셋은 **JSONL(JSON Lines)** 형식을 사용합니다. 한 줄이 하나의 학습 예시입니다.

```jsonl
{"prompt": "고객이 반품을 요청합니다. 배송 후 7일 이내이고 제품이 미개봉입니다.", "completion": "안녕하세요, 고객님. 반품 규정을 확인한 결과, 배송 후 7일 이내 미개봉 제품은 전액 환불이 가능합니다. 반품 접수 도와드리겠습니다. 주문번호를 알려주시겠습니까?"}
{"prompt": "고객이 제품 교환을 문의합니다. 사이즈가 맞지 않다고 합니다.", "completion": "안녕하세요, 고객님. 사이즈 교환은 배송일로부터 14일 이내에 가능합니다. 원하시는 사이즈를 말씀해 주시면 교환 절차를 안내해 드리겠습니다."}
{"prompt": "고객이 배송 지연에 대해 불만을 표출합니다.", "completion": "고객님, 배송 지연으로 불편을 드려 진심으로 사과드립니다. 현재 주문 상태를 즉시 확인해 보겠습니다. 잠시만 기다려 주시겠습니까?"}
```

**데이터셋 품질 기준**

| 기준 | 권장사항 |
|------|----------|
| **최소 데이터 수** | 최소 100개 이상, 권장 500~1,000개 |
| **데이터 다양성** | 다양한 시나리오와 엣지 케이스를 포함 |
| **일관된 형식** | 모든 응답이 동일한 어조와 구조를 유지 |
| **정확성** | 사실 관계가 정확한 데이터만 포함 |
| **적절한 길이** | 너무 짧거나 너무 긴 응답을 피하고 실제 서비스와 유사한 길이 유지 |
| **검증 데이터** | 전체의 10~20%를 검증용으로 분리 |

**데이터셋을 S3에 업로드**

```bash
# 학습용 데이터와 검증용 데이터를 S3에 업로드
aws s3 cp train.jsonl s3://my-bucket/training/train.jsonl
aws s3 cp validation.jsonl s3://my-bucket/training/validation.jsonl
```

> **💡 핵심 포인트:** Fine-tuning을 시작하기 전에, 먼저 프롬프트 엔지니어링과 RAG로 충분한 성능이 나오는지 검토하세요. Fine-tuning은 시간과 비용이 상당히 소요되므로, "프롬프트 튜닝 -> RAG -> Fine-tuning" 순서로 단계적으로 접근하는 것이 현명합니다.

### 14-3-3 비용/성능 트레이드오프 분석

Fine-tuning은 강력한 기능이지만, 비용과 운영 복잡성을 함께 가져옵니다. 세 가지 접근법을 비교해 봅시다.

| 구분 | 프롬프트 엔지니어링 | RAG | Fine-tuning |
|------|---------------------|-----|-------------|
| **초기 비용** | 거의 없음 | 벡터 DB 구축 비용 | 학습 비용 + Provisioned Throughput |
| **추가 지식 반영** | 프롬프트에 직접 삽입 | 문서 업로드로 실시간 반영 | 재학습 필요 |
| **응답 품질** | 기본 모델 수준 | 검색 품질에 의존 | 도메인 특화 최고 품질 |
| **유지보수** | 프롬프트 수정만 | 데이터 동기화 관리 | 데이터 갱신 시 재학습 |
| **적합한 경우** | 범용 질의응답 | 최신 정보 기반 응답 | 특정 어조/형식 필요 |

**Provisioned Throughput 비용 구조**

커스텀 모델은 On-demand로 호출할 수 없고, 반드시 **Provisioned Throughput**을 구매해야 합니다. 이는 시간당 과금되며, 모델과 처리량 단위(Model Unit)에 따라 비용이 달라집니다.

```bash
# Provisioned Throughput 생성
aws bedrock create-provisioned-model-throughput \
    --model-units 1 \
    --provisioned-model-name my-custom-endpoint \
    --model-id arn:aws:bedrock:us-east-1:123456789012:custom-model/my-custom-claude
```

> **📌 참고:** Provisioned Throughput의 최소 약정 기간은 1개월 또는 6개월입니다. 테스트 목적이라면 약정 없는 옵션(No commitment, 시간당 과금)을 선택하되, 비용이 상대적으로 높다는 점을 유의하세요.

**실용적인 의사결정 가이드**

Fine-tuning이 적합한 경우는 다음과 같습니다.

- 고객 응대 챗봇에서 **회사 고유의 어조와 매뉴얼 스타일**을 일관되게 유지해야 할 때
- 특수한 **도메인 용어**(의료, 법률, 금융)에 대한 이해도를 높여야 할 때
- **출력 형식**(특정 JSON 스키마, 보고서 양식)을 정확하게 지켜야 할 때
- 프롬프트 엔지니어링만으로는 원하는 품질에 도달하지 못할 때

반면, 다음 상황에서는 Fine-tuning보다 RAG가 더 나은 선택입니다.

- 정보가 자주 변경되는 경우 (뉴스, 주가, 재고 현황 등)
- 출처(Citation)를 함께 제공해야 하는 경우
- 대량의 문서를 기반으로 질의응답을 하는 경우



---

## 마치며

이번 챕터에서는 Bedrock 기반 AI 서비스를 실제 운영 환경에 올리는 두 가지 핵심 방법을 배웠습니다. **Lambda + API Gateway** 조합으로 서버 관리 없이 가볍게 API를 배포하는 방법과, **ECS/Fargate + ALB** 조합으로 본격적인 컨테이너 기반 서비스를 운영하는 방법을 각각 실습했습니다. 그리고 Fine-tuning을 통해 모델 자체를 도메인에 맞게 커스텀하는 방법과, 프롬프트 엔지니어링/RAG/Fine-tuning 간의 트레이드오프까지 분석했습니다.

서비스를 클라우드에 올렸다면 절반은 성공한 셈입니다. 하지만 진짜 중요한 것은 **배포 이후의 운영**입니다. Chapter 15에서는 CloudWatch 모니터링 설정, Provisioned Throughput vs On-demand 비용 최적화, 운영 체크리스트와 보안 점검 등 안정적인 서비스 유지를 위한 실전 가이드를 다루겠습니다.
