# GCP 배포 가이드

Google Cloud Platform에 배당주 퀀트 분석 시스템 배포하기

## 📋 목차

- [개요](#개요)
- [사전 준비](#사전-준비)
- [App Engine 배포](#app-engine-배포)
- [Cloud Run 배포](#cloud-run-배포)
- [Compute Engine 배포](#compute-engine-배포)
- [CI/CD 파이프라인](#cicd-파이프라인)
- [모니터링 및 로깅](#모니터링-및-로깅)
- [비용 최적화](#비용-최적화)

---

## 🎯 개요

GCP는 여러 서비스 옵션을 제공합니다:

| 서비스 | 특징 | 권장 용도 | 비용 |
|--------|------|-----------|------|
| **App Engine** | 완전 관리형, 자동 스케일링 | 빠른 배포, 운영 간소화 | 중간 |
| **Cloud Run** | 컨테이너 기반, 서버리스 | 트래픽 변동 큰 경우 | 낮음 |
| **Compute Engine** | VM 기반, 완전 제어 | 커스터마이징 필요 시 | 높음 |
| **Kubernetes Engine** | 컨테이너 오케스트레이션 | 대규모 마이크로서비스 | 매우 높음 |

**이 가이드는 App Engine과 Cloud Run에 집중합니다.**

---

## 📋 사전 준비

### 1. GCP 프로젝트 생성

```bash
# gcloud CLI 설치 (https://cloud.google.com/sdk/docs/install)

# 인증
gcloud auth login

# 프로젝트 생성
gcloud projects create dividend-analyzer-prod --name="Dividend Analyzer"

# 프로젝트 설정
gcloud config set project dividend-analyzer-prod

# 현재 프로젝트 확인
gcloud config get-value project
```

### 2. 필수 API 활성화

```bash
# App Engine Admin API
gcloud services enable appengine.googleapis.com

# Cloud Build API (CI/CD용)
gcloud services enable cloudbuild.googleapis.com

# Cloud Run API
gcloud services enable run.googleapis.com

# Container Registry API
gcloud services enable containerregistry.googleapis.com

# Secret Manager API (API 키 관리)
gcloud services enable secretmanager.googleapis.com
```

### 3. 결제 계정 연결

GCP Console에서 결제 계정 연결: https://console.cloud.google.com/billing

---

## 🚀 App Engine 배포

### 1. app.yaml 설정

프로젝트 루트에 `app.yaml` 생성:

```yaml
runtime: python310

# 인스턴스 클래스 (F1=무료, F2~F4=유료)
instance_class: F2

# 자동 스케일링 설정
automatic_scaling:
  target_cpu_utilization: 0.65
  min_instances: 0  # 비용 절감
  max_instances: 3
  min_idle_instances: 0
  max_idle_instances: 1

# 환경변수
env_variables:
  PYTHONUNBUFFERED: "1"

# 엔트리포인트
entrypoint: streamlit run app4.py --server.port=$PORT --server.address=0.0.0.0 --server.headless=true
```

### 2. .gcloudignore 설정

불필요한 파일 제외:

```
# Git
.git
.gitignore

# Python
__pycache__/
*.pyc
venv/
env/

# 환경변수 (Secret Manager 사용)
.env
.env.local

# 문서
docs/
*.md
!README.md

# 테스트
tests/

# IDE
.vscode/
.idea/
```

### 3. Secret Manager로 API 키 관리

```bash
# Secret 생성
echo -n "your_groq_api_key" | gcloud secrets create groq-api-key --data-file=-

# App Engine에 권한 부여
gcloud secrets add-iam-policy-binding groq-api-key \
  --member=serviceAccount:dividend-analyzer-prod@appspot.gserviceaccount.com \
  --role=roles/secretmanager.secretAccessor

# app.yaml에 Secret 추가
```

**app.yaml 업데이트**:
```yaml
runtime: python310
instance_class: F2

automatic_scaling:
  target_cpu_utilization: 0.65
  min_instances: 0
  max_instances: 3

# Secret Manager에서 환경변수 로드
env_variables:
  PYTHONUNBUFFERED: "1"

# Secret 참조 (app4.py에서 os.getenv 사용)
beta_settings:
  cloud_sql_instances: []

entrypoint: streamlit run app4.py --server.port=$PORT --server.address=0.0.0.0 --server.headless=true
```

**app4.py 수정**:
```python
import os
from google.cloud import secretmanager

def get_secret(secret_id):
    """Secret Manager에서 시크릿 가져오기"""
    try:
        client = secretmanager.SecretManagerServiceClient()
        project_id = os.getenv('GOOGLE_CLOUD_PROJECT')
        name = f"projects/{project_id}/secrets/{secret_id}/versions/latest"
        response = client.access_secret_version(request={"name": name})
        return response.payload.data.decode('UTF-8')
    except Exception as e:
        return os.getenv(secret_id.upper().replace('-', '_'), '')

# API 키 로드
GROQ_API_KEY = get_secret('groq-api-key') or os.getenv("GROQ_API_KEY", "")
```

### 4. 배포 실행

```bash
# 첫 배포 (App Engine 초기화)
gcloud app create --region=asia-northeast3  # 서울 리전

# 애플리케이션 배포
gcloud app deploy

# 특정 버전 배포
gcloud app deploy --version=v1 --no-promote

# 배포 후 브라우저 열기
gcloud app browse
```

### 5. 트래픽 관리

```bash
# 버전 확인
gcloud app versions list

# 트래픽 분할
gcloud app services set-traffic default --splits v1=50,v2=50

# 특정 버전으로 전환
gcloud app services set-traffic default --splits v2=100

# 이전 버전 삭제
gcloud app versions delete v1
```

---

## ☁️ Cloud Run 배포

Cloud Run은 **비용 효율적이고 자동 스케일링**이 뛰어난 옵션입니다.

### 1. Dockerfile 준비

Cloud Run용으로 PORT 환경변수 사용:

```dockerfile
FROM python:3.10-slim

WORKDIR /app

# 의존성 설치
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 애플리케이션 코드 복사
COPY . .

# Cloud Run은 PORT 환경변수 제공
ENV PORT=8080

# 비루트 사용자 실행
RUN useradd -m -u 1000 streamlit && chown -R streamlit:streamlit /app
USER streamlit

# Cloud Run 헬스체크
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl --fail http://localhost:${PORT}/_stcore/health || exit 1

# Streamlit 실행
CMD streamlit run app4.py \
    --server.port=${PORT} \
    --server.address=0.0.0.0 \
    --server.headless=true \
    --browser.gatherUsageStats=false
```

### 2. Container Registry에 푸시

```bash
# Artifact Registry 저장소 생성 (권장)
gcloud artifacts repositories create dividend-analyzer \
    --repository-format=docker \
    --location=asia-northeast3 \
    --description="Dividend Analyzer Docker images"

# Docker 인증 설정
gcloud auth configure-docker asia-northeast3-docker.pkg.dev

# 이미지 빌드 및 태그
docker build -t asia-northeast3-docker.pkg.dev/dividend-analyzer-prod/dividend-analyzer/app:v1 .

# 푸시
docker push asia-northeast3-docker.pkg.dev/dividend-analyzer-prod/dividend-analyzer/app:v1
```

### 3. Cloud Run 배포

```bash
# 기본 배포
gcloud run deploy dividend-analyzer \
    --image asia-northeast3-docker.pkg.dev/dividend-analyzer-prod/dividend-analyzer/app:v1 \
    --platform managed \
    --region asia-northeast3 \
    --allow-unauthenticated \
    --memory 1Gi \
    --cpu 1 \
    --max-instances 10 \
    --min-instances 0 \
    --set-env-vars GROQ_API_KEY=your_api_key

# Secret Manager 사용
gcloud run deploy dividend-analyzer \
    --image asia-northeast3-docker.pkg.dev/dividend-analyzer-prod/dividend-analyzer/app:v1 \
    --platform managed \
    --region asia-northeast3 \
    --allow-unauthenticated \
    --memory 1Gi \
    --cpu 1 \
    --max-instances 10 \
    --min-instances 0 \
    --set-secrets GROQ_API_KEY=groq-api-key:latest

# 서비스 URL 확인
gcloud run services describe dividend-analyzer --region asia-northeast3 --format 'value(status.url)'
```

### 4. 커스텀 도메인 연결

```bash
# 도메인 매핑
gcloud run domain-mappings create \
    --service dividend-analyzer \
    --domain dividend.yourdomain.com \
    --region asia-northeast3

# DNS 레코드 추가 (출력된 정보 참고)
```

### 5. 자동 스케일링 설정

```bash
# 최소/최대 인스턴스 조정
gcloud run services update dividend-analyzer \
    --min-instances 1 \
    --max-instances 20 \
    --region asia-northeast3

# 동시성 설정 (인스턴스당 요청 수)
gcloud run services update dividend-analyzer \
    --concurrency 80 \
    --region asia-northeast3

# CPU 할당 (비용 최적화)
gcloud run services update dividend-analyzer \
    --cpu-throttling \
    --region asia-northeast3
```

---

## 🖥️ Compute Engine 배포

완전한 제어가 필요한 경우 사용.

### 1. VM 인스턴스 생성

```bash
# e2-medium 인스턴스 (2 vCPU, 4GB RAM)
gcloud compute instances create dividend-analyzer-vm \
    --zone=asia-northeast3-a \
    --machine-type=e2-medium \
    --image-family=ubuntu-2204-lts \
    --image-project=ubuntu-os-cloud \
    --boot-disk-size=20GB \
    --tags=http-server,https-server

# 방화벽 규칙 생성
gcloud compute firewall-rules create allow-streamlit \
    --allow tcp:8501 \
    --target-tags http-server

# SSH 접속
gcloud compute ssh dividend-analyzer-vm --zone=asia-northeast3-a
```

### 2. VM에서 설정

```bash
# 시스템 업데이트
sudo apt-get update && sudo apt-get upgrade -y

# Docker 설치
sudo apt-get install -y docker.io docker-compose
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER

# 재로그인 후
# 코드 클론
git clone https://github.com/yourusername/dividend-quant-analyzer.git
cd dividend-quant-analyzer

# 환경변수 설정
echo "GROQ_API_KEY=your_key" > .env

# Docker Compose 실행
docker-compose up -d

# 또는 직접 Python 설치
sudo apt-get install -y python3.10 python3-pip
pip3 install -r requirements.txt
streamlit run app4.py --server.port=8501 --server.address=0.0.0.0
```

### 3. 외부 IP 확인 및 접속

```bash
# 외부 IP 확인
gcloud compute instances describe dividend-analyzer-vm \
    --zone=asia-northeast3-a \
    --format='get(networkInterfaces[0].accessConfigs[0].natIP)'

# 브라우저에서 접속
# http://<EXTERNAL_IP>:8501
```

---

## 🔄 CI/CD 파이프라인

### Cloud Build 설정

#### 1. cloudbuild.yaml 생성

```yaml
steps:
  # Docker 이미지 빌드
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'build'
      - '-t'
      - 'asia-northeast3-docker.pkg.dev/$PROJECT_ID/dividend-analyzer/app:$SHORT_SHA'
      - '-t'
      - 'asia-northeast3-docker.pkg.dev/$PROJECT_ID/dividend-analyzer/app:latest'
      - '.'

  # Container Registry에 푸시
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'push'
      - 'asia-northeast3-docker.pkg.dev/$PROJECT_ID/dividend-analyzer/app:$SHORT_SHA'

  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'push'
      - 'asia-northeast3-docker.pkg.dev/$PROJECT_ID/dividend-analyzer/app:latest'

  # Cloud Run 배포
  - name: 'gcr.io/cloud-builders/gcloud'
    args:
      - 'run'
      - 'deploy'
      - 'dividend-analyzer'
      - '--image'
      - 'asia-northeast3-docker.pkg.dev/$PROJECT_ID/dividend-analyzer/app:$SHORT_SHA'
      - '--region'
      - 'asia-northeast3'
      - '--platform'
      - 'managed'
      - '--allow-unauthenticated'

timeout: 1200s
options:
  machineType: 'N1_HIGHCPU_8'
```

#### 2. GitHub 연동

```bash
# Cloud Build GitHub 앱 설치
# https://github.com/apps/google-cloud-build

# 트리거 생성
gcloud builds triggers create github \
    --repo-name=dividend-quant-analyzer \
    --repo-owner=yourusername \
    --branch-pattern="^main$" \
    --build-config=cloudbuild.yaml

# 수동 빌드 실행
gcloud builds submit --config cloudbuild.yaml .
```

#### 3. 자동 배포 워크플로우

```
[GitHub Push] → [Cloud Build 트리거] → [Docker 빌드] → [Cloud Run 배포] → [자동 롤아웃]
```

---

## 📊 모니터링 및 로깅

### Cloud Logging

```bash
# 로그 확인
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=dividend-analyzer" --limit 50

# 실시간 로그 스트리밍
gcloud logging tail "resource.type=cloud_run_revision AND resource.labels.service_name=dividend-analyzer"

# 에러 로그만 필터링
gcloud logging read "resource.type=cloud_run_revision AND severity>=ERROR" --limit 20
```

### Cloud Monitoring

```bash
# 대시보드 생성 (웹 콘솔 권장)
# https://console.cloud.google.com/monitoring

# 알림 정책 생성 (CLI)
gcloud alpha monitoring policies create \
    --notification-channels=CHANNEL_ID \
    --display-name="High CPU Alert" \
    --condition-threshold-value=0.8 \
    --condition-threshold-duration=300s
```

### 커스텀 메트릭

**app4.py에 추가**:
```python
from google.cloud import monitoring_v3
import time

def record_metric(metric_name, value):
    """커스텀 메트릭 기록"""
    try:
        client = monitoring_v3.MetricServiceClient()
        project_name = f"projects/{os.getenv('GOOGLE_CLOUD_PROJECT')}"
        
        series = monitoring_v3.TimeSeries()
        series.metric.type = f"custom.googleapis.com/{metric_name}"
        
        point = monitoring_v3.Point()
        point.value.double_value = value
        point.interval.end_time.seconds = int(time.time())
        
        series.points = [point]
        client.create_time_series(name=project_name, time_series=[series])
    except Exception as e:
        print(f"메트릭 기록 실패: {e}")

# 사용 예시
record_metric("dividend_analyzer/user_count", len(df))
```

---

## 💰 비용 최적화

### 1. Cloud Run 비용 절감

```bash
# CPU 스로틀링 활성화 (요청 없을 때 CPU 미사용)
gcloud run services update dividend-analyzer \
    --cpu-throttling \
    --region asia-northeast3

# 최소 인스턴스 0으로 설정
gcloud run services update dividend-analyzer \
    --min-instances 0 \
    --region asia-northeast3

# 메모리 최적화
gcloud run services update dividend-analyzer \
    --memory 512Mi \
    --region asia-northeast3
```

### 2. 예상 비용 계산

**Cloud Run (무료 티어 포함)**:
- 요청: 월 200만 건 무료
- CPU: 월 18만 vCPU초 무료
- 메모리: 월 36만 GiB초 무료

**예시**:
- 일 1000명 방문, 평균 5분 사용
- 월 약 $10~30 예상

### 3. 예산 알림 설정

```bash
# 예산 생성
gcloud billing budgets create \
    --billing-account=BILLING_ACCOUNT_ID \
    --display-name="Dividend Analyzer Budget" \
    --budget-amount=50USD \
    --threshold-rule=percent=50 \
    --threshold-rule=percent=90
```

---

## 🔐 보안 강화

### 1. IAM 역할 최소화

```bash
# 서비스 계정 생성
gcloud iam service-accounts create dividend-sa \
    --display-name="Dividend Analyzer Service Account"

# 필요한 역할만 부여
gcloud projects add-iam-policy-binding dividend-analyzer-prod \
    --member="serviceAccount:dividend-sa@dividend-analyzer-prod.iam.gserviceaccount.com" \
    --role="roles/secretmanager.secretAccessor"
```

### 2. VPC 연결 (고급)

```bash
# VPC 네트워크 생성
gcloud compute networks create dividend-vpc --subnet-mode=custom

# 서브넷 생성
gcloud compute networks subnets create dividend-subnet \
    --network=dividend-vpc \
    --region=asia-northeast3 \
    --range=10.0.0.0/24

# Cloud Run에 VPC 연결
gcloud run services update dividend-analyzer \
    --vpc-connector=dividend-connector \
    --region asia-northeast3
```

### 3. Cloud Armor (DDoS 방어)

```bash
# 보안 정책 생성
gcloud compute security-policies create dividend-security-policy \
    --description="DDoS protection"

# 규칙 추가
gcloud compute security-policies rules create 1000 \
    --security-policy=dividend-security-policy \
    --expression="origin.region_code == 'KR'" \
    --action=allow
```

---

## 📚 참고 자료

- [GCP App Engine 문서](https://cloud.google.com/appengine/docs)
- [Cloud Run 문서](https://cloud.google.com/run/docs)
- [Cloud Build 문서](https://cloud.google.com/build/docs)
- [Secret Manager 문서](https://cloud.google.com/secret-manager/docs)
- [GCP 비용 계산기](https://cloud.google.com/products/calculator)

---

## 🆘 지원

문제 발생 시:
1. GCP Console 로그 확인
2. `gcloud` 명령어로 상태 체크
3. Stack Overflow 검색
4. GCP 지원팀 문의

---

**☁️ GCP로 안정적이고 확장 가능한 서비스를 운영하세요!**