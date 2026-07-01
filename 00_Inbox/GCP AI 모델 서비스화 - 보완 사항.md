# GCP AI 모델 서비스화 - 보완 사항


## 🔐 보안 (Security)

### 1. 인증 / 인가

- **API Key 또는 OAuth 2.0** 도입 → 익명 요청 차단
- GCP라면 **API Gateway + IAM** 조합이 자연스러움
- 클라이언트 유형별 권한 분리 (읽기 전용 vs 쓰기 가능 등)

### 2. 입력값 검증 (Input Validation)

- 요청 페이로드 스키마 검증 (Pydantic, JSON Schema 등)
- **Prompt Injection 방어** — AI 모델 특유의 공격 벡터
- 입력 길이 제한, 허용 문자 필터링

### 3. Rate Limiting / DDoS 방어

- 클라이언트별 요청 수 제한 (ex. 100 req/min)
- GCP Cloud Armor 또는 API Gateway 레벨에서 설정
- 비용 폭탄 방지에도 직결됨

### 4. 네트워크 격리

- 모델 서버를 **VPC 내부**에 두고 외부 직접 노출 차단
- Cloud Run / GKE라면 Internal Ingress만 허용
- 필요한 IP/CIDR만 허용하는 방화벽 규칙

### 5. 시크릿 관리

- API 키, DB 비밀번호 등을 코드/환경변수에 하드코딩 금지
- **GCP Secret Manager** 사용

---

## ⚙️ 서비스 안정성 (Reliability)

### 1. 타임아웃 & 에러 핸들링

- 모델 추론 타임아웃 명시적 설정
- 표준화된 에러 응답 포맷 (HTTP 상태코드 + 에러 코드 + 메시지)
- Retry 로직 (지수 백오프)

### 2. 모니터링 & 로깅

- **Cloud Monitoring** — 지연시간(latency), 에러율, 처리량(TPS) 추적
- **Cloud Logging** — 요청/응답 로그 (민감 정보 마스킹 필수)
- 알람 설정 (에러율 5% 초과 시 Slack 알림 등)

### 3. 부하 분산 & 스케일링

- Cloud Run이라면 min/max instance 설정
- Cold Start 문제 → min instance 1 이상 유지
- GPU 모델이면 스케일링 전략이 더 중요 (비쌈)

### 4. 버전 관리 & 배포 전략

- `/v1/predict`, `/v2/predict` 형태로 **API 버전 관리**
- 모델 버전도 별도 관리 (Model Registry)
- **Blue/Green 배포** 또는 카나리 배포로 무중단 업데이트

### 5. 비용 관리

- 요청당 토큰/연산량 트래킹
- 이상 사용량 감지 → 자동 알람 또는 차단

---

## 우선순위 추천

|우선순위|항목|이유|
|---|---|---|
|🔴 즉시|인증/인가 + Rate Limiting|무방비 상태의 가장 큰 리스크|
|🔴 즉시|입력값 검증|AI 서비스 특유의 취약점|
|🟡 단기|모니터링 + 로깅|문제 발생 시 원인 파악 불가|
|🟡 단기|타임아웃 / 에러 핸들링|장애 전파 방지|
|🟢 중기|VPC 네트워크 격리|인프라 보안 강화|
|🟢 중기|버전 관리 / 배포 전략|운영 성숙도 향상|
