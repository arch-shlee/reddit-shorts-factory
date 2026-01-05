# us-west-2(오레곤) 리전 선택 가이드

## 결정 사항
**모든 AWS 리소스를 us-west-2(오레곤) 리전에 배포**

---

## 선택 이유

### 1. Bedrock 모델 가용성
- ✅ Stable Diffusion XL 완전 지원
- ✅ Claude 3.5 Sonnet 지원
- ✅ 최신 모델 우선 출시 리전
- ✅ 높은 할당량 (트래픽 분산)

### 2. 비용 효율성
| 항목 | us-west-2 | ap-northeast-2 (서울) | 절감 |
|------|-----------|----------------------|------|
| Lambda (GB-초) | $0.0000166667 | $0.0000185185 | 11% |
| S3 (GB/월) | $0.023 | $0.025 | 8% |
| 데이터 전송 (리전 간) | $0 | $0.02/GB | 100% |

**월 30개 영상 기준**: us-west-2 = $16.28, ap-northeast-2 = $18.50

### 3. 아키텍처 단순화
- 모든 서비스가 같은 리전 → 크로스 리전 복잡성 제거
- VPC Peering, Transit Gateway 불필요
- 데이터 전송 비용 $0

---

## 단점 및 완화 전략

### 1. 한국에서의 관리 UI 레이턴시

**영향**:
- n8n UI 접속: 150-200ms RTT
- CloudWatch 대시보드: 100-150ms

**완화**:
- ✅ UI는 가끔만 접속 (주 1-2회 정도)
- ✅ 자동화된 워크플로우는 레이턴시 무관
- ✅ Slack 알림으로 모니터링 대체

**측정 방법**:
```bash
# 한국에서 오레곤까지 레이턴시 확인
ping ec2.us-west-2.amazonaws.com

# 예상 결과: ~150ms
```

### 2. TikTok 업로드 경로 최적화

**현재 아키텍처**:
```
Lambda (us-west-2) → TikTok CDN (지역별 엔드포인트)
```

**TikTok이 제공하는 upload_url 예시**:
- 아시아: `upload-sg.tiktok.com` (싱가포르)
- 미국: `upload-va.tiktok.com` (버지니아)
- 유럽: `upload-ie.tiktok.com` (아일랜드)

**시나리오 분석**:

#### 시나리오 A: TikTok이 아시아 CDN 반환
```
Lambda (us-west-2) --150ms--> TikTok SG
비디오 12MB 업로드: 약 3초
```

#### 시나리오 B: TikTok이 미국 CDN 반환
```
Lambda (us-west-2) --20ms--> TikTok VA
비디오 12MB 업로드: 약 1초
```

**결론**:
- TikTok API가 Lambda 위치 기반으로 최적 CDN 선택
- us-west-2에서 미국 CDN 사용 시 최적
- 최악의 경우(아시아 CDN)도 3초 이내로 수용 가능

**모니터링**:
```python
# Lambda 함수에서 로깅
import time
start = time.time()
response = upload_to_tiktok(video_url)
duration = time.time() - start
logger.info(f"TikTok upload took {duration}s, endpoint: {response['upload_url']}")
```

### 3. 긴급 대응 시 시간대 차이

**문제**: 한국 시간 09:00 = 오레곤 시간 16:00 (전날)
**영향**: AWS Support 케이스, 긴급 디버깅

**완화**:
- CloudWatch Alarms → SNS → Slack (실시간 알림)
- Lambda Dead Letter Queue 설정
- 자동 재시도 로직
```typescript
new lambda.Function(this, 'StoryAdapter', {
  deadLetterQueue: dlq,
  retryAttempts: 2,
  maxEventAge: Duration.hours(6)
});
```

---

## 리전 변경이 필요한 경우

### 한국 리전(ap-northeast-2)으로 변경 고려 시점

1. **Bedrock 모델 가용성 확대**
   - 서울 리전에서 Stable Diffusion XL 지원 시작
   - 확인: https://docs.aws.amazon.com/bedrock/latest/userguide/models-regions.html

2. **비용 최적화**
   - 크로스 리전 전송이 월 $10 이상 발생 시
   - 현재: 거의 $0 (TikTok 업로드만 인터넷 전송)

3. **규제 요구사항**
   - 한국 데이터 거주성 법률 변경 시

### 마이그레이션 절차

```bash
# 1. CDK 코드 수정
cd cdk
# bin/reddit-shorts.ts에서 region: 'ap-northeast-2'로 변경

# 2. 새 리전에 배포
cdk bootstrap aws://ACCOUNT/ap-northeast-2
cdk deploy --all

# 3. S3 데이터 복제 (필요시)
aws s3 sync s3://reddit-shorts-assets-us-west-2 \
             s3://reddit-shorts-assets-ap-northeast-2 \
             --source-region us-west-2 \
             --region ap-northeast-2

# 4. n8n 워크플로우 업데이트
# Lambda ARN을 새 리전으로 변경

# 5. 기존 리전 리소스 정리
cdk destroy --all  # us-west-2 스택
```

---

## 글로벌 확장 시나리오 (Phase 3+)

만약 서비스가 성장하여 여러 국가를 지원한다면:

### Multi-Region 아키텍처

```
사용자 위치별 라우팅:
- 아시아 → ap-northeast-2 (서울)
- 미국 → us-west-2 (오레곤)
- 유럽 → eu-west-1 (아일랜드)

공통:
- Route 53 Geolocation Routing
- DynamoDB Global Tables (중복 방지)
- S3 Cross-Region Replication
```

**비용**: 현재의 3배 (리전당 $16 × 3 = $48/월)
**시점**: 일일 영상 생성 100개 이상 시

---

## 체크리스트

### 배포 전 확인사항

- [ ] Bedrock 모델 가용성 확인
  ```bash
  aws bedrock list-foundation-models --region us-west-2 \
    | jq '.modelSummaries[] | select(.modelId | contains("stable-diffusion"))'
  ```

- [ ] Service Quotas 확인
  ```bash
  aws service-quotas get-service-quota \
    --service-code bedrock \
    --quota-code L-F4A3F1F7 \  # Claude quotas
    --region us-west-2
  ```

- [ ] n8n EC2를 us-west-2에 생성
  ```bash
  # AMI: Ubuntu 22.04 LTS
  # Instance Type: t3.micro (Free Tier)
  # VPC: Default VPC in us-west-2
  ```

- [ ] CDK Bootstrap
  ```bash
  cdk bootstrap aws://ACCOUNT-ID/us-west-2
  ```

### 배포 후 확인사항

- [ ] Lambda → Bedrock 호출 성공
  ```bash
  aws logs tail /aws/lambda/reddit-shorts-story-adapter \
    --region us-west-2 --follow
  ```

- [ ] S3 버킷이 us-west-2에 생성됨
  ```bash
  aws s3api get-bucket-location --bucket reddit-shorts-assets
  # 결과: "LocationConstraint": "us-west-2"
  ```

- [ ] TikTok 업로드 속도 측정
  ```bash
  # CloudWatch Logs Insights 쿼리
  fields @timestamp, @message
  | filter @message like /TikTok upload took/
  | stats avg(duration) by bin(5m)
  ```

- [ ] 전체 파이프라인 E2E 테스트
  - Reddit → 스크립트 → 이미지 → 비디오 → TikTok
  - 예상 소요 시간: 5-8분

---

## FAQ

### Q1: us-east-1 (버지니아)는 왜 안 쓰나요?
**A**: Stable Diffusion XL 모델 가용성과 할당량이 us-west-2가 더 좋습니다. 또한 us-west-2가 신규 Bedrock 기능이 먼저 출시되는 리전입니다.

### Q2: 한국에서 접속 시 체감 속도는?
**A**: n8n UI 로딩은 2-3초 정도로 느껴질 수 있지만, 자동화된 워크플로우 실행에는 영향 없습니다.

### Q3: 나중에 리전 변경이 어렵나요?
**A**: CDK 코드 1줄 수정 + 재배포로 30분 이내 완료 가능합니다. S3 데이터 이전만 시간이 걸립니다.

### Q4: Bedrock 비용이 리전마다 다른가요?
**A**: 아니요, Bedrock 가격은 모든 리전에서 동일합니다.

### Q5: 만약 오레곤 리전에 장애가 생기면?
**A**:
- AWS Health Dashboard 모니터링
- Multi-Region 백업 (Phase 2+)
- 현재는 수동 복구 (CDK로 다른 리전 재배포)

---

## 참고 자료

- [AWS Bedrock Regions](https://docs.aws.amazon.com/bedrock/latest/userguide/bedrock-regions.html)
- [AWS Regional Services List](https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/)
- [AWS Pricing Calculator](https://calculator.aws/)
- [Latency Monitoring Tools](https://www.cloudping.info/)
