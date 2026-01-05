# 프로젝트 시작 전 필수 확인사항

## 🚨 Critical Priority (즉시 처리 필요)

### 1. TikTok API 접근 권한 신청

**상태**: ⚠️ 미신청 - **지금 즉시 신청 필요**

**필요 이유**:
- TikTok에 영상을 자동 업로드하려면 Content Posting API 필수
- 승인까지 **1-2주** 소요
- 승인 없이는 Phase 1 완성 불가

**신청 절차**:

#### Step 1: TikTok 개발자 계정 생성
```
1. https://developers.tiktok.com/ 접속
2. "Register" 클릭
3. TikTok 계정으로 로그인
4. 개발자 약관 동의
```

#### Step 2: App 등록
```
1. Developer Portal → "Manage apps" → "Connect an app"
2. App 정보 입력:
   - App name: Reddit Shorts Factory
   - App description: Automated short-form video creation from Reddit stories
   - Category: Video
   - Platform: Server-side (API only)
```

#### Step 3: Content Posting API 신청
```
1. App 설정 → "Add products" → "Content Posting API"
2. Use case 작성 (영문):

   "We create educational/entertainment short videos by:
   1. Sourcing trending stories from Reddit
   2. Adapting them using AI
   3. Generating images and voiceovers
   4. Posting to TikTok automatically

   Expected volume: 1 video per day (30/month)
   Target audience: General entertainment seekers"

3. Redirect URI 입력:
   - https://oauth.tiktok.com/callback (임시)
   - 나중에 실제 도메인으로 변경 가능

4. Submit for review
```

#### Step 4: 승인 대기 중 준비사항
```
✅ Reddit API 연동 테스트
✅ Bedrock 모델 테스트
✅ 로컬에서 비디오 생성 파이프라인 구축
✅ S3에 샘플 비디오 저장

승인 후:
✅ TikTok OAuth 토큰 발급
✅ Lambda 함수 배포
✅ n8n 워크플로우 완성
```

**대체 방안 (승인 전 테스트용)**:
- Mock TikTok Uploader: S3에만 저장하고 성공 반환
- 수동 업로드: 비디오 생성 후 링크를 Slack으로 받아 수동 업로드

---

### 2. AWS Bedrock 모델 접근 권한

**상태**: 확인 필요

Bedrock 모델은 **명시적으로 활성화**해야 사용 가능합니다.

**확인 방법**:
```bash
# AWS Console
1. Bedrock 콘솔 접속 (us-west-2 리전)
2. "Model access" 메뉴
3. 활성화 필요 모델:
   - Anthropic Claude 3.5 Sonnet
   - Stability AI Stable Diffusion XL

# CLI로 확인
aws bedrock list-foundation-models --region us-west-2 \
  --query 'modelSummaries[?contains(modelId, `claude`) || contains(modelId, `stable-diffusion`)].[modelId, modelArn]' \
  --output table
```

**활성화 절차**:
```
1. Bedrock Console → Model access → Manage model access
2. 체크박스 선택:
   ✅ Anthropic - Claude 3.5 Sonnet
   ✅ Stability AI - Stable Diffusion XL
3. "Request model access" 클릭
4. Use case 입력 (선택사항)
```

**승인 시간**:
- Claude: 즉시 (자동 승인)
- Stable Diffusion: 즉시~수 시간

**비용 관련**:
- 모델 접근 자체는 무료
- 사용한 만큼만 과금

---

### 3. Reddit API 인증 설정

**상태**: 확인 필요

**필요 정보**:
- Client ID
- Client Secret
- User Agent

**신청 절차**:
```
1. https://www.reddit.com/prefs/apps 접속
2. "create another app" 클릭
3. 정보 입력:
   - name: reddit-shorts-bot
   - type: script
   - description: Fetches trending stories for video creation
   - redirect uri: http://localhost:8080 (script 타입이므로 미사용)
4. "create app" 클릭
5. Client ID, Secret 복사
```

**Rate Limit**:
- OAuth: 60 requests/minute
- 하루 1번 실행이므로 문제 없음

**저장 위치**:
```bash
# AWS Secrets Manager에 저장 (권장)
aws secretsmanager create-secret \
  --name reddit-shorts/reddit-api \
  --secret-string '{
    "client_id": "YOUR_CLIENT_ID",
    "client_secret": "YOUR_CLIENT_SECRET",
    "user_agent": "reddit-shorts-bot/1.0"
  }' \
  --region us-west-2

# 또는 n8n Credentials에 직접 입력
```

---

## ⚠️ High Priority (구현 전 결정 필요)

### 4. 저작권 및 법적 이슈

#### Reddit 콘텐츠 사용 권한
**Reddit TOS 검토**:
- ✅ 공개 게시물은 크롤링 가능
- ⚠️ 상업적 사용은 제한적
- ✅ 원작자 크레딧 표시 권장

**권장 조치**:
```python
# 비디오에 크레딧 추가
def add_credits_overlay(video_path, reddit_author, post_url):
    """
    비디오 마지막에 크레딧 오버레이:
    'Original story by u/username
     Source: reddit.com/r/tifu/...'
    """
    # FFmpeg overlay 필터 사용
```

**TikTok 설명란**:
```
이 영상은 Reddit의 실제 이야기를 각색했습니다.
원글: reddit.com/r/tifu/...
작성자: u/username

#redditstories #tifu #storytime
```

#### AI 생성 콘텐츠 표시
**TikTok 정책** (2024년 기준):
- AI 생성 콘텐츠는 명시해야 함

**구현 방법**:
```json
// TikTok API 요청 시
{
  "post_info": {
    "title": "...",
    "privacy_level": "PUBLIC_TO_EVERYONE",
    "video_cover_timestamp_ms": 1000,
    "brand_content_toggle": false,
    "brand_organic_toggle": false,
    "ai_generated_toggle": true  // ← AI 생성 표시
  }
}
```

**비디오 워터마크**:
```
우측 하단에 작게 표시:
"AI Generated"
```

---

### 5. 비용 모니터링 설정

**예상치 못한 비용 방지**

#### CloudWatch Billing Alarm 설정
```typescript
// CDK 코드
import * as cloudwatch from 'aws-cdk-lib/aws-cloudwatch';
import * as sns from 'aws-cdk-lib/aws-sns';
import * as subscriptions from 'aws-cdk-lib/aws-sns-subscriptions';

// SNS 토픽 생성
const billingTopic = new sns.Topic(this, 'BillingAlerts', {
  displayName: 'Reddit Shorts Billing Alerts'
});

billingTopic.addSubscription(
  new subscriptions.EmailSubscription('your-email@example.com')
);

// 월 $50 초과 시 알림
const billingAlarm = new cloudwatch.Alarm(this, 'BillingAlarm', {
  metric: new cloudwatch.Metric({
    namespace: 'AWS/Billing',
    metricName: 'EstimatedCharges',
    dimensionsMap: {
      Currency: 'USD'
    },
    statistic: 'Maximum',
    period: Duration.hours(6)
  }),
  threshold: 50,
  evaluationPeriods: 1,
  alarmDescription: 'Alert when monthly charges exceed $50'
});

billingAlarm.addAlarmAction(new cloudwatchActions.SnsAction(billingTopic));
```

#### Bedrock 할당량 제한
```typescript
// Lambda 환경 변수로 일일 제한 설정
environment: {
  MAX_DAILY_GENERATIONS: '5',  // 하루 최대 5개 영상
  BEDROCK_TIMEOUT: '30000'     // 30초 타임아웃
}
```

```python
# Lambda 함수 내부
import boto3
from datetime import datetime

def check_daily_limit():
    """S3에 저장된 오늘 생성 횟수 확인"""
    today = datetime.now().strftime('%Y-%m-%d')
    s3 = boto3.client('s3')

    try:
        response = s3.list_objects_v2(
            Bucket='reddit-shorts-processed',
            Prefix=f'{today}/'
        )
        count = response.get('KeyCount', 0)

        if count >= int(os.environ['MAX_DAILY_GENERATIONS']):
            raise Exception(f'Daily limit reached: {count} videos')

        return count
    except Exception as e:
        logger.error(f'Error checking daily limit: {e}')
        raise
```

---

### 6. 부적절한 콘텐츠 필터링

**문제**: Reddit TIFU에는 성인 콘텐츠, 폭력, 혐오 표현 포함 가능

**해결책**:

#### Option A: Reddit API 필터 (기본)
```python
# Reddit API 호출 시
params = {
    'limit': 10,
    't': 'day',
    'over_18': 'false'  # NSFW 제외
}
```

#### Option B: AI 기반 콘텐츠 검토 (권장)
```python
# Story Adapter Lambda에서
def check_content_safety(text: str) -> dict:
    """
    Bedrock Guardrails 또는 Claude로 콘텐츠 검토
    """
    bedrock = boto3.client('bedrock-runtime')

    response = bedrock.invoke_model(
        modelId='anthropic.claude-3-5-sonnet-20241022',
        body=json.dumps({
            "anthropic_version": "bedrock-2023-05-31",
            "messages": [{
                "role": "user",
                "content": f"""다음 텍스트가 TikTok에 게시하기 적절한지 판단해주세요:

                {text}

                다음 기준으로 평가:
                - 폭력적 내용
                - 성인 콘텐츠
                - 혐오 표현
                - 불법 활동

                JSON 형식으로 응답:
                {{"safe": true/false, "reason": "이유"}}"""
            }],
            "max_tokens": 200,
            "temperature": 0
        })
    )

    result = json.loads(response['body'].read())
    return json.loads(result['content'][0]['text'])

# 사용
safety_check = check_content_safety(reddit_post['selftext'])
if not safety_check['safe']:
    logger.warning(f"Unsafe content detected: {safety_check['reason']}")
    return {
        'statusCode': 400,
        'body': 'Content filtered'
    }
```

**추가 비용**:
- 콘텐츠 검토 Claude 호출: $0.003/게시물
- 월 30개 × $0.003 = $0.09

---

## 📋 Medium Priority (개발 중 고려)

### 7. 로컬 개발 환경

**필요 도구**:
- ✅ Docker Desktop (Lambda Container Image 테스트)
- ✅ AWS CLI v2 (인증, 배포)
- ✅ Node.js 18+ (CDK)
- ✅ Python 3.12 (Lambda 함수)
- ✅ FFmpeg (로컬 비디오 테스트)

**설치 확인**:
```bash
docker --version        # Docker version 24.0+
aws --version          # aws-cli/2.x
node --version         # v18.0+
python --version       # Python 3.12+
ffmpeg -version        # ffmpeg version 6.0+
```

### 8. Git/버전 관리 전략

**.gitignore 추가 필요**:
```gitignore
# Secrets
.env
cdk/cdk.out/
credentials.json

# Lambda build artifacts
lambdas/*/package/
lambdas/*/.aws-sam/

# Python
__pycache__/
*.pyc
.pytest_cache/

# Node.js
node_modules/
*.log

# OS
.DS_Store
Thumbs.db

# IDE
.vscode/
.idea/
*.swp

# Test outputs
/tmp/
test-videos/
```

**민감 정보 관리**:
```bash
# AWS Secrets Manager 사용 (권장)
# 환경 변수로 절대 커밋하지 말 것

# 로컬 테스트용 .env 파일 (절대 커밋 금지)
cat > .env.example <<EOF
# AWS
AWS_REGION=us-west-2
AWS_ACCOUNT_ID=123456789012

# Reddit (Secrets Manager에서 가져옴)
REDDIT_CLIENT_ID=<from-secrets-manager>
REDDIT_CLIENT_SECRET=<from-secrets-manager>

# TikTok (Secrets Manager에서 가져옴)
TIKTOK_CLIENT_KEY=<from-secrets-manager>
TIKTOK_CLIENT_SECRET=<from-secrets-manager>
EOF
```

---

## ✅ 체크리스트

### 즉시 처리 (1-2일 내)
- [ ] **TikTok Developer 계정 생성 및 API 신청** ← 가장 중요!
- [ ] AWS Bedrock 모델 접근 활성화 (Claude, Stable Diffusion)
- [ ] Reddit API 앱 등록 및 Client ID/Secret 발급
- [ ] AWS Secrets Manager에 API 키 저장
- [ ] CloudWatch Billing Alarm 설정 ($50 threshold)

### 개발 시작 전 (1주일 내)
- [ ] 로컬 개발 환경 설정 (Docker, AWS CLI, Node.js)
- [ ] .gitignore 설정 및 민감 정보 보호 확인
- [ ] 콘텐츠 안전성 필터링 전략 결정
- [ ] 저작권 표시 방법 결정 (크레딧, 워터마크)
- [ ] Phase 1 비용 한도 설정

### TikTok API 승인 대기 중
- [ ] Reddit → 스크립트 변환 파이프라인 구현
- [ ] 이미지 생성 테스트
- [ ] 비디오 합성 테스트
- [ ] S3에 샘플 비디오 저장 (Mock TikTok uploader)

### TikTok API 승인 후
- [ ] OAuth 토큰 발급
- [ ] TikTok Uploader Lambda 구현
- [ ] E2E 테스트 (Private 계정에 업로드)
- [ ] Production 배포

---

## 타임라인 예상

```
Week 1:
- TikTok API 신청 (승인 대기 시작)
- AWS 리소스 활성화
- 로컬 환경 설정
- CDK 프로젝트 초기화

Week 2:
- Lambda 함수 구현 (Story Adapter, Content Generator)
- Bedrock 통합 테스트
- Mock TikTok Uploader 구현

Week 3:
- Video Compositor 구현 (FFmpeg)
- n8n 워크플로우 작성
- E2E 테스트 (TikTok 제외)

Week 4:
- TikTok API 승인 (예상)
- TikTok Uploader 구현
- 전체 파이프라인 테스트
- Production 배포
```

---

## 추가 문의 사항

1. **비즈니스 목적**:
   - 개인 프로젝트인가요, 상업적 용도인가요?
   - 수익화 계획이 있나요? (TikTok Creator Fund 등)

2. **규모**:
   - 하루 몇 개 영상을 생성할 계획인가요?
   - 향후 확장 계획 (YouTube, Instagram 등)

3. **콘텐츠 정책**:
   - 특정 주제만 다룰 건가요? (예: 유머만, 교육만)
   - 연령 제한이 있는 콘텐츠는 어떻게 처리할까요?

이 정보들이 아키텍처 설계에 영향을 줄 수 있습니다.
