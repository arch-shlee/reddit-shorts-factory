# Reddit Shorts Factory - Phase 1 설계

## 목표
Reddit TIFU 스토리를 TikTok 쇼츠로 자동 변환하는 MVP 파이프라인 구축

## 아키텍처 개요

```
n8n Cloud
    ↓
[Cron: 매일 09:00 KST]
    ↓
[Node 1: Reddit API]
    ↓
[Node 2: Lambda - Story Adapter]
    ↓
[Node 3: Lambda - Content Generator]
    ↓
[Node 4: Lambda - Video Compositor]
    ↓
[Node 5: TikTok Upload]
    ↓
[Node 6: Slack Notification]
```

## 기술 스택

### n8n
- **배포**: EC2 Self-hosted (기존 인스턴스 사용)
- **인증**: EC2 IAM Role
- **네트워크**: Public Subnet + EIP
- **DB**: SQLite (로컬)
- **노드 수**: 6개
- **실행 주기**: 매일 1회

### AWS 서비스
- **Lambda**: 4개 함수 (모두 Container Image)
- **S3**: 3개 버킷
  - `reddit-shorts-raw`: Reddit 원본 데이터
  - `reddit-shorts-assets`: 이미지/오디오/비디오 파일
  - `reddit-shorts-processed`: 중복 체크용 메타데이터
- **Bedrock**: Claude 3.5 Sonnet, Stable Diffusion XL
- **Polly**: Neural TTS (Joanna/Matthew 음성)
- **ECR**: Docker 이미지 저장소 (Lambda Container Images)

### Infrastructure as Code
- **AWS CDK**: TypeScript로 인프라 정의
- **언어**: TypeScript (CDK), Python 3.12 (Lambda)

### 외부 API
- **Reddit API**: OAuth 2.0 인증
- **TikTok API**: Content Posting API

---

## 세부 설계

### 1. n8n 워크플로우

#### Node 1: Reddit API Call
**타입**: HTTP Request
**설정**:
```json
{
  "method": "GET",
  "url": "https://oauth.reddit.com/r/tifu/hot",
  "authentication": "OAuth2",
  "qs": {
    "limit": 10,
    "t": "day"
  }
}
```

**출력 필터**:
- `score >= 1000`
- `selftext.length >= 500 && <= 2000`
- `removed_by_category == null`

**중복 체크**:
- S3에서 `processed/{post_id}.json` 존재 확인
- 존재하면 스킵

---

#### Node 2: Lambda - Story Adapter
**함수명**: `reddit-shorts-story-adapter`
**런타임**: Python 3.12 Container Image
**메모리**: 512MB
**타임아웃**: 60초

**입력**:
```json
{
  "post_id": "abc123",
  "title": "TIFU by...",
  "content": "원본 텍스트...",
  "author": "username"
}
```

**처리 로직**:
1. Bedrock Claude 호출
2. 프롬프트: "다음 Reddit 스토리를 60초 TikTok 영상용 4장면 스크립트로 변환"
3. 출력 형식:
```json
{
  "scenes": [
    {
      "scene_number": 1,
      "duration": 15,
      "narration": "음성 텍스트",
      "image_prompt": "Stable Diffusion 프롬프트",
      "subtitle": "자막 텍스트"
    }
  ],
  "metadata": {
    "total_duration": 60,
    "estimated_words": 150
  }
}
```

**S3 저장**: `raw/{post_id}_script.json`

---

#### Node 3: Lambda - Content Generator
**함수명**: `reddit-shorts-content-generator`
**런타임**: Python 3.12 Container Image
**메모리**: 3GB
**타임아웃**: 300초 (5분)

**처리 로직**:
1. **이미지 생성** (4장면 순차)
   ```python
   for scene in scenes:
       image = bedrock.invoke_model(
           modelId="stability.stable-diffusion-xl-v1",
           body={
               "text_prompts": [{"text": scene.image_prompt}],
               "cfg_scale": 7,
               "steps": 30,
               "width": 1080,
               "height": 1920  # 9:16 세로
           }
       )
       s3.put_object(Key=f"assets/{post_id}/scene_{i}.png")
   ```

2. **음성 생성** (전체 나레이션 합침)
   ```python
   full_narration = " ".join([s.narration for s in scenes])
   audio = polly.synthesize_speech(
       Text=full_narration,
       VoiceId="Joanna",
       Engine="neural",
       OutputFormat="mp3"
   )
   s3.put_object(Key=f"assets/{post_id}/narration.mp3")
   ```

**출력**:
```json
{
  "post_id": "abc123",
  "images": [
    "s3://bucket/assets/abc123/scene_0.png",
    ...
  ],
  "audio": "s3://bucket/assets/abc123/narration.mp3",
  "scenes": [...],
  "durations": [15, 15, 15, 15]
}
```

---

#### Node 4: Lambda - Video Compositor
**함수명**: `reddit-shorts-video-compositor`
**런타임**: Python 3.12 Container Image (FFmpeg 포함)
**메모리**: 10GB
**임시 스토리지**: 10GB
**타임아웃**: 900초 (15분)

**Container Image 구성**:
```dockerfile
FROM public.ecr.aws/lambda/python:3.12

# FFmpeg 설치
RUN yum install -y wget tar xz && \
    wget https://johnvansickle.com/ffmpeg/releases/ffmpeg-release-amd64-static.tar.xz && \
    tar xvf ffmpeg-release-amd64-static.tar.xz && \
    mv ffmpeg-*-static/ffmpeg /usr/local/bin/ && \
    chmod +x /usr/local/bin/ffmpeg

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY app.py ${LAMBDA_TASK_ROOT}
CMD ["app.handler"]
```

**처리 로직**:
```python
# 1. S3에서 에셋 다운로드
for img in images:
    download_to_tmp(img)
download_to_tmp(audio)

# 2. FFmpeg 명령 구성
# - 각 이미지 15초씩 표시 (Ken Burns 효과)
# - 오디오 오버레이
# - 자막 임베딩 (SRT 파일 사용)
# - 출력: 1080x1920, 60fps, H.264

ffmpeg_cmd = f"""
ffmpeg -y \
  -loop 1 -t 15 -i /tmp/scene_0.png \
  -loop 1 -t 15 -i /tmp/scene_1.png \
  -loop 1 -t 15 -i /tmp/scene_2.png \
  -loop 1 -t 15 -i /tmp/scene_3.png \
  -i /tmp/narration.mp3 \
  -filter_complex "[0:v]scale=1080:1920,zoompan=z='min(zoom+0.0015,1.5)':d=375:s=1080x1920[v0]; \
                   [1:v]scale=1080:1920,zoompan=z='min(zoom+0.0015,1.5)':d=375:s=1080x1920[v1]; \
                   [2:v]scale=1080:1920,zoompan=z='min(zoom+0.0015,1.5)':d=375:s=1080x1920[v2]; \
                   [3:v]scale=1080:1920,zoompan=z='min(zoom+0.0015,1.5)':d=375:s=1080x1920[v3]; \
                   [v0][v1][v2][v3]concat=n=4:v=1:a=0[vout]" \
  -map "[vout]" -map 4:a \
  -c:v libx264 -preset medium -crf 23 \
  -c:a aac -b:a 128k \
  -movflags +faststart \
  /tmp/output.mp4
"""

# 3. S3에 업로드
s3.upload_file("/tmp/output.mp4", f"assets/{post_id}/final.mp4")
```

**출력**:
```json
{
  "post_id": "abc123",
  "video_url": "s3://bucket/assets/abc123/final.mp4",
  "duration": 60,
  "size_mb": 12.3
}
```

---

#### Node 5: TikTok Upload
**타입**: HTTP Request → Lambda
**함수명**: `reddit-shorts-tiktok-uploader`

**TikTok API 플로우**:
```python
# 1. 비디오 업로드 초기화
init_response = requests.post(
    "https://open.tiktokapis.com/v2/post/publish/video/init/",
    headers={"Authorization": f"Bearer {access_token}"},
    json={
        "post_info": {
            "title": title[:150],  # 최대 150자
            "privacy_level": "PUBLIC_TO_EVERYONE",
            "disable_comment": False,
            "disable_duet": False,
            "disable_stitch": False,
            "video_cover_timestamp_ms": 1000
        },
        "source_info": {
            "source": "FILE_UPLOAD",
            "video_size": file_size,
            "chunk_size": 10485760,  # 10MB
            "total_chunk_count": total_chunks
        }
    }
)

upload_url = init_response.json()["data"]["upload_url"]
publish_id = init_response.json()["data"]["publish_id"]

# 2. 청크 업로드
with open(video_path, "rb") as f:
    chunk_index = 0
    while chunk := f.read(10485760):
        requests.put(
            upload_url,
            headers={"Content-Range": f"bytes {start}-{end}/{total}"},
            data=chunk
        )
        chunk_index += 1

# 3. 게시 확인
status = requests.post(
    "https://open.tiktokapis.com/v2/post/publish/status/fetch/",
    json={"publish_id": publish_id}
)
```

**메타데이터 저장**:
```json
{
  "post_id": "abc123",
  "tiktok_video_id": "7123456789",
  "status": "published",
  "url": "https://tiktok.com/@username/video/7123456789",
  "timestamp": "2026-01-05T09:30:00Z"
}
```
→ S3: `processed/{post_id}.json`

---

#### Node 6: Slack Notification
**타입**: Slack 노드
**설정**:
```json
{
  "channel": "#reddit-shorts-bot",
  "message": {
    "blocks": [
      {
        "type": "section",
        "text": {
          "type": "mrkdwn",
          "text": "*새 쇼츠 업로드 완료!* :rocket:"
        }
      },
      {
        "type": "section",
        "fields": [
          {"type": "mrkdwn", "text": "*Reddit:*\n<{{reddit_url}}|{{title}}>"},
          {"type": "mrkdwn", "text": "*TikTok:*\n<{{tiktok_url}}|영상 보기>"},
          {"type": "mrkdwn", "text": "*Score:* {{score}}"},
          {"type": "mrkdwn", "text": "*Duration:* {{duration}}s"}
        ]
      },
      {
        "type": "image",
        "image_url": "{{thumbnail_url}}",
        "alt_text": "썸네일"
      }
    ]
  }
}
```

---

## 데이터 플로우

### S3 버킷 구조
```
reddit-shorts-raw/
  └── 2026-01-05/
      └── abc123_original.json

reddit-shorts-assets/
  └── abc123/
      ├── scene_0.png
      ├── scene_1.png
      ├── scene_2.png
      ├── scene_3.png
      ├── narration.mp3
      └── final.mp4

reddit-shorts-processed/
  └── abc123.json  # 중복 체크 + 메타데이터
```

---

## 비용 예측 (월 30개 영상 기준)

### AWS Lambda
- Story Adapter: 512MB × 60s × 30회 = $0.03
- Content Generator: 3GB × 300s × 30회 = $0.90
- Video Compositor: 10GB × 900s × 30회 = $9.00
- TikTok Uploader: 512MB × 30s × 30회 = $0.01
**합계**: ~$10/월

### AWS Bedrock
- Claude 3.5 Sonnet: $3/1M input tokens, $15/1M output tokens
  - 30회 × 2K input + 1K output = $0.54/월
- Stable Diffusion XL: $0.04/이미지
  - 30회 × 4장면 = 120장 = $4.80/월
**합계**: ~$5.34/월

### AWS Polly
- Neural TTS: $16/1M 문자
  - 30회 × 500자 = 15K 문자 = $0.24/월

### AWS S3
- 스토리지: 30GB/월 = $0.69
- PUT 요청: 300회 = $0.002
- GET 요청: 600회 = $0.0002
**합계**: ~$0.70/월

### n8n Cloud
- Free Tier: 5,000 executions/월 (충분)
- $0/월

### 총 비용: **~$16.28/월**

---

## 제약사항 및 해결책

### 1. Lambda 타임아웃 (15분)
**문제**: 긴 영상(60초 이상) 처리 시 초과 가능
**해결책**:
- Phase 1: 60초 이하 영상만 처리
- Phase 2: ECS Fargate로 전환

### 2. Bedrock 할당량
**문제**: Stable Diffusion XL 기본 할당량 제한
**해결책**:
- Service Quotas에서 증량 신청
- 실패 시 재시도 로직 (n8n Error Trigger)

### 3. TikTok API 제한
**문제**: 일일 업로드 제한
**해결책**:
- 하루 1개로 제한 (Cron 09:00 1회)
- 추후 계정 추가

### 4. 중복 처리
**문제**: S3 파일 체크로는 경쟁 조건 가능
**해결책**:
- Phase 1: 문제 발생 시 수동 삭제
- Phase 2: DynamoDB Conditional Write

---

## 테스트 전략

### 단위 테스트
각 Lambda 함수별 로컬 테스트:
```bash
# Story Adapter 테스트
sam local invoke StoryAdapterFunction -e events/reddit_post.json

# Content Generator 테스트
sam local invoke ContentGeneratorFunction -e events/script.json

# Video Compositor 테스트 (Docker 필요)
docker run -v /tmp:/tmp lambda:video-compositor events/assets.json
```

### 통합 테스트
n8n 워크플로우 수동 실행:
1. Reddit API 노드만 실행 → 출력 확인
2. Story Adapter까지 실행 → 스크립트 품질 확인
3. Content Generator까지 → 이미지/음성 확인
4. 전체 파이프라인 → 최종 영상 확인

### E2E 테스트
실제 TikTok 업로드 (Private 계정):
- 영상 품질 확인
- 자막 가독성 확인
- 음성 싱크 확인

---

## 모니터링

### CloudWatch 메트릭
- Lambda 오류율
- Lambda 소요 시간
- Bedrock 호출 실패율

### n8n 모니터링
- 워크플로우 성공/실패율
- 각 노드 실행 시간

### Slack 알림
- 성공: 썸네일 + 링크
- 실패: 에러 메시지 + 스택 트레이스

---

## 다음 단계 (Phase 2)

1. **DynamoDB 통합**: 중복 체크 강화
2. **YouTube Shorts 추가**: 멀티 플랫폼 확장
3. **ECS Fargate**: 긴 영상 처리
4. **A/B 테스트**: 썸네일/제목 최적화
5. **Analytics**: 조회수/참여도 추적

---

## 참고 자료

- [n8n Documentation](https://docs.n8n.io/)
- [Reddit API](https://www.reddit.com/dev/api/)
- [TikTok Content Posting API](https://developers.tiktok.com/doc/content-posting-api-get-started/)
- [AWS Bedrock Pricing](https://aws.amazon.com/bedrock/pricing/)
- [FFmpeg Documentation](https://ffmpeg.org/documentation.html)
