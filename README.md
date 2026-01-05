# Reddit Shorts Factory

Reddit TIFU 스토리를 자동으로 TikTok 쇼츠 영상으로 변환하는 AI 파이프라인

## 개요

매일 자동으로:
1. Reddit에서 인기 있는 TIFU 스토리를 가져옴
2. AI로 60초 영상용 스크립트 각색
3. AI로 장면 이미지 4장 생성
4. 음성 나레이션 생성
5. 비디오 합성
6. TikTok에 자동 업로드

## 아키텍처

```
┌─────────────┐
│   Reddit    │
│     API     │
└──────┬──────┘
       │
       ↓
┌─────────────┐    ┌──────────────┐
│     n8n     │───→│   Bedrock    │
│  (EC2)      │    │   Claude     │
└──────┬──────┘    └──────────────┘
       │
       ↓
┌─────────────┐    ┌──────────────┐
│   Lambda    │───→│   Bedrock    │
│  Functions  │    │ Stable Diff. │
└──────┬──────┘    └──────────────┘
       │
       │           ┌──────────────┐
       ├──────────→│    Polly     │
       │           │    (TTS)     │
       │           └──────────────┘
       ↓
┌─────────────┐    ┌──────────────┐
│   FFmpeg    │───→│      S3      │
│  Container  │    │              │
└──────┬──────┘    └──────────────┘
       │
       ↓
┌─────────────┐
│   TikTok    │
│     API     │
└─────────────┘
```

## 기술 스택

- **Infrastructure**: AWS CDK (TypeScript)
- **Orchestration**: n8n (Self-hosted)
- **Compute**: AWS Lambda (Container Images)
- **AI Models**:
  - Claude 3.5 Sonnet (스토리 각색)
  - Stable Diffusion XL (이미지 생성)
  - Polly Neural (음성 합성)
- **Storage**: S3
- **Region**: us-west-2 (Oregon)

## 프로젝트 구조

```
reddit-shorts-factory/
├── cdk/                    # AWS CDK 인프라 코드
├── lambdas/                # Lambda 함수들
│   ├── story-adapter/
│   ├── content-generator/
│   ├── video-compositor/
│   └── tiktok-uploader/
├── n8n-workflows/          # n8n 워크플로우
├── scripts/                # 빌드/배포 스크립트
└── docs/                   # 문서
```

## 빠른 시작

### 사전 준비

1. **TikTok Developer API 신청** (1-2주 소요)
2. **AWS 계정** 및 Bedrock 모델 활성화
3. **Reddit API** 앱 등록
4. **로컬 환경**:
   - Docker Desktop
   - AWS CLI v2
   - Node.js 18+
   - Python 3.12

자세한 내용은 [docs/PREREQUISITES.md](docs/PREREQUISITES.md) 참고

### 설치 및 배포

```bash
# 1. 저장소 클론
git clone <repository-url>
cd reddit-shorts-factory

# 2. CDK 의존성 설치
cd cdk
npm install

# 3. CDK 부트스트랩 (최초 1회)
cdk bootstrap aws://ACCOUNT-ID/us-west-2

# 4. Lambda 이미지 빌드 & ECR 푸시
cd ..
./scripts/build-lambdas.sh

# 5. 인프라 배포
cd cdk
cdk deploy

# 6. n8n 워크플로우 Import
# n8n UI에서 n8n-workflows/reddit-shorts-pipeline.json Import
```

## 비용

예상 비용 (월 30개 영상 기준):
- Lambda: $10.00
- Bedrock: $5.34
- Polly: $0.24
- S3: $0.70
- **총계: ~$16.28/월**

## 주요 문서

- [DESIGN.md](DESIGN.md) - 상세 아키텍처 설계
- [PROJECT_STRUCTURE.md](PROJECT_STRUCTURE.md) - 프로젝트 구조
- [docs/PREREQUISITES.md](docs/PREREQUISITES.md) - 준비사항
- [docs/REGION_CONSIDERATIONS.md](docs/REGION_CONSIDERATIONS.md) - 리전 선택
- [.claude.md](.claude.md) - Claude Code 컨텍스트

## 라이선스

MIT

## 주의사항

- Reddit 콘텐츠 사용 시 원작자 크레딧 표시
- AI 생성 콘텐츠임을 명시
- 부적절한 콘텐츠 필터링 필수
- TikTok 커뮤니티 가이드라인 준수
