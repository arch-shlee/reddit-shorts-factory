# Reddit Shorts Factory - CDK 프로젝트 구조

## 디렉토리 구조

```
reddit-shorts-factory/
├── README.md
├── DESIGN.md
├── PROJECT_STRUCTURE.md (이 파일)
│
├── cdk/                         # AWS CDK (TypeScript)
│   ├── bin/
│   │   └── reddit-shorts.ts    # CDK App 엔트리포인트
│   ├── lib/
│   │   ├── reddit-shorts-stack.ts          # 메인 스택
│   │   ├── constructs/
│   │   │   ├── lambda-functions.ts         # Lambda 함수들
│   │   │   ├── storage.ts                  # S3 버킷들
│   │   │   └── iam-roles.ts                # IAM 역할 (n8n EC2용 포함)
│   │   └── config/
│   │       └── environment.ts              # 환경별 설정
│   ├── test/
│   │   └── reddit-shorts-stack.test.ts
│   ├── package.json
│   ├── tsconfig.json
│   └── cdk.json
│
├── lambdas/                     # Lambda 함수들 (Python)
│   ├── story-adapter/
│   │   ├── src/
│   │   │   ├── handler.py              # Lambda 핸들러
│   │   │   ├── prompts.py              # Claude 프롬프트
│   │   │   └── utils/
│   │   │       ├── bedrock_client.py   # Bedrock 클라이언트
│   │   │       └── s3_client.py        # S3 헬퍼
│   │   ├── tests/
│   │   │   ├── test_handler.py
│   │   │   └── fixtures/
│   │   │       └── reddit_post.json
│   │   ├── Dockerfile
│   │   └── requirements.txt
│   │
│   ├── content-generator/
│   │   ├── src/
│   │   │   ├── handler.py
│   │   │   ├── image_generator.py      # Stable Diffusion
│   │   │   ├── tts_generator.py        # Polly
│   │   │   └── utils/
│   │   │       ├── bedrock_client.py
│   │   │       ├── polly_client.py
│   │   │       └── s3_client.py
│   │   ├── tests/
│   │   ├── Dockerfile
│   │   └── requirements.txt
│   │
│   ├── video-compositor/
│   │   ├── src/
│   │   │   ├── handler.py
│   │   │   ├── compositor.py           # FFmpeg 로직
│   │   │   └── utils/
│   │   │       ├── s3_client.py
│   │   │       └── ffmpeg_helper.py
│   │   ├── tests/
│   │   ├── Dockerfile                  # FFmpeg 포함
│   │   └── requirements.txt
│   │
│   └── tiktok-uploader/
│       ├── src/
│       │   ├── handler.py
│       │   ├── tiktok_client.py        # TikTok API
│       │   └── utils/
│       │       └── s3_client.py
│       ├── tests/
│       ├── Dockerfile
│       └── requirements.txt
│
├── n8n-workflows/               # n8n 워크플로우 정의
│   ├── reddit-shorts-pipeline.json     # 메인 워크플로우
│   ├── README.md                       # 워크플로우 설치 가이드
│   └── credentials/
│       └── credentials-template.json   # 크레덴셜 템플릿
│
├── scripts/                     # 유틸리티 스크립트
│   ├── setup.sh                # 초기 환경 설정
│   ├── build-lambdas.sh        # Lambda 이미지 빌드 & 푸시
│   ├── deploy.sh               # CDK 배포
│   ├── test-local.sh           # 로컬 Lambda 테스트
│   └── cleanup.sh              # 리소스 정리
│
├── docs/                        # 문서
│   ├── 01-SETUP.md             # 초기 설정 가이드
│   ├── 02-DEVELOPMENT.md       # 개발 가이드
│   ├── 03-DEPLOYMENT.md        # 배포 가이드
│   ├── 04-N8N-SETUP.md         # n8n 설정 가이드
│   └── 05-TROUBLESHOOTING.md   # 트러블슈팅
│
└── .github/                     # (옵션) CI/CD
    └── workflows/
        └── deploy.yml          # GitHub Actions
```

---

## 주요 파일 설명

### CDK 구조

#### `cdk/bin/reddit-shorts.ts`
```typescript
#!/usr/bin/env node
import 'source-map-support/register';
import * as cdk from 'aws-cdk-lib';
import { RedditShortsStack } from '../lib/reddit-shorts-stack';

const app = new cdk.App();
new RedditShortsStack(app, 'RedditShortsStack', {
  env: {
    account: process.env.CDK_DEFAULT_ACCOUNT,
    region: 'us-west-2'  // Oregon - Bedrock 이미지 모델 완전 지원, 비용 효율적
  }
});
```

#### `cdk/lib/reddit-shorts-stack.ts`
메인 스택 - 모든 리소스 조합:
- S3 버킷 3개
- ECR 레포지토리 4개
- Lambda 함수 4개
- IAM 역할 (Lambda용, n8n EC2용)

#### `cdk/lib/constructs/lambda-functions.ts`
Lambda 함수 정의:
```typescript
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as ecr from 'aws-cdk-lib/aws-ecr';

export class LambdaFunctions extends Construct {
  public readonly storyAdapter: lambda.DockerImageFunction;
  public readonly contentGenerator: lambda.DockerImageFunction;
  // ...
}
```

---

## Lambda 공통 구조

각 Lambda 함수는 동일한 패턴:

```
lambda-name/
├── src/
│   ├── handler.py           # Lambda 핸들러
│   │   def handler(event, context):
│   │       # 1. 입력 검증
│   │       # 2. 비즈니스 로직
│   │       # 3. S3 저장
│   │       # 4. 반환
│   │
│   ├── [domain].py          # 핵심 로직
│   └── utils/               # 공통 유틸리티
│       ├── aws_clients.py   # boto3 클라이언트
│       └── logger.py        # 로깅
│
├── tests/
│   ├── test_handler.py      # 핸들러 테스트
│   └── fixtures/            # 테스트 데이터
│
├── Dockerfile               # Container Image
│   FROM public.ecr.aws/lambda/python:3.12
│   COPY requirements.txt .
│   RUN pip install -r requirements.txt
│   COPY src/ ${LAMBDA_TASK_ROOT}/
│   CMD ["handler.handler"]
│
└── requirements.txt         # Python 의존성
    boto3==1.34.x
    botocore==1.34.x
```

---

## n8n 워크플로우 구조

### `n8n-workflows/reddit-shorts-pipeline.json`

n8n에서 Export한 JSON 파일:
```json
{
  "name": "Reddit Shorts Factory",
  "nodes": [
    {
      "name": "Cron Trigger",
      "type": "n8n-nodes-base.cron",
      "parameters": {
        "triggerTimes": {
          "item": [
            {
              "mode": "everyDay",
              "hour": 9,
              "minute": 0
            }
          ]
        }
      }
    },
    {
      "name": "Reddit API",
      "type": "n8n-nodes-base.httpRequest",
      "parameters": {
        "url": "https://oauth.reddit.com/r/tifu/hot",
        "authentication": "oAuth2",
        "options": {
          "qs": {
            "limit": 10
          }
        }
      }
    },
    {
      "name": "Lambda - Story Adapter",
      "type": "n8n-nodes-base.awsLambda",
      "parameters": {
        "functionName": "reddit-shorts-story-adapter",
        "payload": "={{ JSON.stringify($json) }}"
      },
      "credentials": {
        "aws": {
          "name": "AWS IAM Role"
        }
      }
    }
    // ... 나머지 노드들
  ],
  "connections": {
    "Cron Trigger": {
      "main": [[{"node": "Reddit API", "type": "main", "index": 0}]]
    }
    // ...
  }
}
```

---

## 개발 워크플로우

### 1. 초기 설정
```bash
# CDK 설치
npm install -g aws-cdk

# 프로젝트 클론
cd reddit-shorts-factory

# CDK 의존성 설치
cd cdk && npm install

# CDK 부트스트랩 (최초 1회)
cdk bootstrap aws://ACCOUNT-ID/us-east-1
```

### 2. Lambda 개발
```bash
# 로컬 테스트 (Docker 필요)
cd lambdas/story-adapter
docker build -t story-adapter-test .
docker run --rm \
  -e AWS_ACCESS_KEY_ID=test \
  -e AWS_SECRET_ACCESS_KEY=test \
  -v $(pwd)/tests:/tests \
  story-adapter-test python -m pytest /tests
```

### 3. Lambda 배포
```bash
# ECR에 이미지 빌드 & 푸시
./scripts/build-lambdas.sh

# CDK 배포
cd cdk && cdk deploy
```

### 4. n8n 설정
```bash
# n8n 워크플로우 Import
# n8n UI에서 reddit-shorts-pipeline.json Import
# Credentials 설정 (AWS IAM Role 자동 사용)
```

---

## CDK 배포 플로우

```
1. cdk synth              # CloudFormation 템플릿 생성
2. cdk diff               # 변경사항 확인
3. cdk deploy             # 배포 실행
   ↓
   - ECR 레포지토리 생성
   - S3 버킷 생성
   - Lambda 함수 생성 (ECR 이미지 참조)
   - IAM 역할/정책 생성
   - n8n EC2 IAM Role 업데이트 (Lambda 호출 권한)
```

---

## 환경 변수

### Lambda 환경 변수 (CDK에서 설정)
```typescript
new lambda.DockerImageFunction(this, 'StoryAdapter', {
  environment: {
    RAW_BUCKET: rawBucket.bucketName,
    ASSETS_BUCKET: assetsBucket.bucketName,
    BEDROCK_MODEL_ID: 'anthropic.claude-3-5-sonnet-20241022',
    LOG_LEVEL: 'INFO'
  }
});
```

### n8n 환경 변수 (EC2에서 설정)
```bash
# /etc/systemd/system/n8n.service 또는 ~/.bashrc
export N8N_BASIC_AUTH_ACTIVE=true
export N8N_BASIC_AUTH_USER=admin
export N8N_BASIC_AUTH_PASSWORD=your-password
export AWS_REGION=us-east-1
```

---

## 다음 단계

1. ✅ 프로젝트 구조 생성
2. 🔄 CDK 프로젝트 초기화
3. 🔄 Lambda 함수 구현
4. 🔄 n8n 워크플로우 작성
5. 🔄 통합 테스트
6. 🔄 프로덕션 배포
