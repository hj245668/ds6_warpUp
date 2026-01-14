# Docker 배포 가이드

배당주 퀀트 분석 시스템의 Docker 컨테이너화 및 배포 가이드

## 📋 목차

- [개요](#개요)
- [Dockerfile 구성](#dockerfile-구성)
- [Docker Compose 설정](#docker-compose-설정)
- [로컬 배포](#로컬-배포)
- [프로덕션 배포](#프로덕션-배포)
- [트러블슈팅](#트러블슈팅)

---

## 🎯 개요

Docker를 사용하면 다음과 같은 이점을 얻을 수 있습니다:
- ✅ 환경 일관성 보장
- ✅ 간편한 배포 및 확장
- ✅ 의존성 격리
- ✅ CI/CD 파이프라인 통합 용이

---

## 📦 Dockerfile 구성

### 기본 Dockerfile

프로젝트 루트에 `Dockerfile` 생성:

```dockerfile
# Python 3.10 slim 이미지 사용
FROM python:3.10-slim

# 작업 디렉토리 설정
WORKDIR /app

# 시스템 패키지 업데이트 및 필수 도구 설치
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    && rm -rf /var/lib/apt/lists/*

# 의존성 파일 복사 및 설치
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 애플리케이션 코드 복사
COPY . .

# Streamlit 포트 노출
EXPOSE 8501

# 헬스체크 설정
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl --fail http://localhost:8501/_stcore/health || exit 1

# Streamlit 실행
CMD ["streamlit", "run", "app4.py", "--server.port=8501", "--server.address=0.0.0.0"]
```

### 최적화된 Dockerfile (멀티스테이지 빌드)

더 작은 이미지 크기와 빠른 빌드를 위한 최적화 버전:

```dockerfile
# 빌드 스테이지
FROM python:3.10-slim as builder

WORKDIR /app

# 의존성 설치를 위한 가상환경 생성
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

COPY requirements.txt .
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# 런타임 스테이지
FROM python:3.10-slim

WORKDIR /app

# 빌드 스테이지에서 가상환경 복사
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# 애플리케이션 코드 복사
COPY app4.py .
COPY finance_targets.py .
COPY news_backtest.py .

# 비루트 사용자 생성 및 전환 (보안)
RUN useradd -m -u 1000 streamlit && \
    chown -R streamlit:streamlit /app
USER streamlit

EXPOSE 8501

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl --fail http://localhost:8501/_stcore/health || exit 1

CMD ["streamlit", "run", "app4.py", \
     "--server.port=8501", \
     "--server.address=0.0.0.0", \
     "--server.headless=true", \
     "--browser.gatherUsageStats=false"]
```

### .dockerignore 파일

불필요한 파일 제외:

```
# 가상환경
venv/
env/
.venv/

# Python 캐시
__pycache__/
*.pyc
*.pyo
*.pyd
.Python

# IDE
.vscode/
.idea/
*.swp
*.swo

# Git
.git/
.gitignore

# 환경변수 (보안)
.env
.env.local

# 테스트
tests/
*.pytest_cache/

# 문서
docs/
*.md
!README.md

# 기타
.DS_Store
*.log
```

---

## 🐳 Docker Compose 설정

### 기본 docker-compose.yml

개발 환경용 설정:

```yaml
version: '3.8'

services:
  dividend-analyzer:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: dividend-analyzer
    ports:
      - "8501:8501"
    environment:
      - GROQ_API_KEY=${GROQ_API_KEY}
    env_file:
      - .env
    volumes:
      # 개발 시 코드 변경 실시간 반영
      - ./app4.py:/app/app4.py
      - ./finance_targets.py:/app/finance_targets.py
      - ./news_backtest.py:/app/news_backtest.py
    restart: unless-stopped
    networks:
      - dividend-net

networks:
  dividend-net:
    driver: bridge
```

### 프로덕션용 docker-compose.yml

프록시 및 볼륨 관리 포함:

```yaml
version: '3.8'

services:
  dividend-analyzer:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: dividend-analyzer-prod
    expose:
      - "8501"
    environment:
      - GROQ_API_KEY=${GROQ_API_KEY}
      - PYTHONUNBUFFERED=1
    restart: always
    networks:
      - dividend-net
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 512M

  nginx:
    image: nginx:alpine
    container_name: dividend-nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - dividend-analyzer
    networks:
      - dividend-net
    restart: always

networks:
  dividend-net:
    driver: bridge
```

### Nginx 설정 (nginx.conf)

리버스 프록시 설정:

```nginx
events {
    worker_connections 1024;
}

http {
    upstream streamlit {
        server dividend-analyzer:8501;
    }

    server {
        listen 80;
        server_name your-domain.com;

        # HTTPS로 리다이렉트
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name your-domain.com;

        ssl_certificate /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;

        location / {
            proxy_pass http://streamlit;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # Streamlit WebSocket 지원
            proxy_read_timeout 86400;
        }
    }
}
```

---

## 🚀 로컬 배포

### 1. 이미지 빌드

```bash
# 기본 빌드
docker build -t dividend-analyzer:latest .

# 캐시 없이 빌드
docker build --no-cache -t dividend-analyzer:latest .

# 특정 플랫폼용 빌드 (M1 Mac 등)
docker build --platform linux/amd64 -t dividend-analyzer:latest .
```

### 2. 컨테이너 실행

#### 단독 실행
```bash
# 기본 실행
docker run -d \
  --name dividend-analyzer \
  -p 8501:8501 \
  -e GROQ_API_KEY=your_api_key_here \
  dividend-analyzer:latest

# .env 파일 사용
docker run -d \
  --name dividend-analyzer \
  -p 8501:8501 \
  --env-file .env \
  dividend-analyzer:latest

# 볼륨 마운트 (개발 모드)
docker run -d \
  --name dividend-analyzer \
  -p 8501:8501 \
  --env-file .env \
  -v $(pwd):/app \
  dividend-analyzer:latest
```

#### Docker Compose 사용
```bash
# 백그라운드 실행
docker-compose up -d

# 로그 확인
docker-compose logs -f

# 중지
docker-compose down

# 이미지 재빌드 후 실행
docker-compose up -d --build
```

### 3. 접속 확인

브라우저에서 `http://localhost:8501` 접속

### 4. 컨테이너 관리

```bash
# 실행 중인 컨테이너 확인
docker ps

# 로그 확인
docker logs dividend-analyzer

# 실시간 로그
docker logs -f dividend-analyzer

# 컨테이너 내부 접속
docker exec -it dividend-analyzer /bin/bash

# 컨테이너 중지
docker stop dividend-analyzer

# 컨테이너 삭제
docker rm dividend-analyzer

# 이미지 삭제
docker rmi dividend-analyzer:latest
```

---

## 🏭 프로덕션 배포

### AWS EC2 배포

#### 1. EC2 인스턴스 준비
```bash
# Docker 설치
sudo yum update -y
sudo yum install -y docker
sudo service docker start
sudo usermod -a -G docker ec2-user

# Docker Compose 설치
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

#### 2. 애플리케이션 배포
```bash
# 코드 클론
git clone https://github.com/yourusername/dividend-quant-analyzer.git
cd dividend-quant-analyzer

# 환경변수 설정
echo "GROQ_API_KEY=your_key" > .env

# 실행
docker-compose -f docker-compose.prod.yml up -d
```

#### 3. 보안 그룹 설정
- 인바운드 규칙: 
  - HTTP (80)
  - HTTPS (443)
  - Custom TCP (8501) - 필요시

### Docker Hub 배포

```bash
# Docker Hub 로그인
docker login

# 이미지 태그
docker tag dividend-analyzer:latest yourusername/dividend-analyzer:v1.0
docker tag dividend-analyzer:latest yourusername/dividend-analyzer:latest

# 푸시
docker push yourusername/dividend-analyzer:v1.0
docker push yourusername/dividend-analyzer:latest

# 다른 서버에서 pull 및 실행
docker pull yourusername/dividend-analyzer:latest
docker run -d -p 8501:8501 --env-file .env yourusername/dividend-analyzer:latest
```

### 자동 재시작 설정

```bash
# systemd 서비스 생성
sudo nano /etc/systemd/system/dividend-analyzer.service
```

```ini
[Unit]
Description=Dividend Analyzer Docker Container
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/ec2-user/dividend-quant-analyzer
ExecStart=/usr/local/bin/docker-compose up -d
ExecStop=/usr/local/bin/docker-compose down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```

```bash
# 서비스 활성화
sudo systemctl enable dividend-analyzer
sudo systemctl start dividend-analyzer
sudo systemctl status dividend-analyzer
```

---

## 🔧 트러블슈팅

### 문제 1: 포트 충돌

**증상**: `Bind for 0.0.0.0:8501 failed: port is already allocated`

**해결**:
```bash
# 포트 사용 확인
sudo lsof -i :8501

# 프로세스 종료
kill -9 <PID>

# 또는 다른 포트 사용
docker run -p 8502:8501 dividend-analyzer:latest
```

### 문제 2: 메모리 부족

**증상**: 컨테이너가 갑자기 종료됨

**해결**:
```bash
# 메모리 제한 설정
docker run -d \
  --memory="1g" \
  --memory-swap="2g" \
  dividend-analyzer:latest

# docker-compose.yml에서
services:
  dividend-analyzer:
    deploy:
      resources:
        limits:
          memory: 1G
```

### 문제 3: 빌드 시간 오래 걸림

**해결**:
```bash
# 레이어 캐싱 활용
# requirements.txt를 먼저 COPY하여 의존성 레이어 캐시

# BuildKit 사용
DOCKER_BUILDKIT=1 docker build -t dividend-analyzer:latest .
```

### 문제 4: 환경변수 로드 안됨

**해결**:
```bash
# .env 파일 확인
cat .env

# 명시적으로 환경변수 전달
docker run -d \
  -e GROQ_API_KEY=${GROQ_API_KEY} \
  dividend-analyzer:latest

# docker-compose에서 확인
docker-compose config
```

### 문제 5: 네트워크 연결 실패

**해결**:
```bash
# 네트워크 재생성
docker network rm dividend-net
docker network create dividend-net

# DNS 설정 확인
docker run --dns 8.8.8.8 dividend-analyzer:latest
```

---

## 📊 모니터링

### 리소스 사용량 확인

```bash
# 실시간 통계
docker stats dividend-analyzer

# 디스크 사용량
docker system df

# 미사용 리소스 정리
docker system prune -a
```

### 로그 관리

```bash
# 로그 크기 제한
docker run -d \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  dividend-analyzer:latest

# 로그 드라이버 변경
docker run -d \
  --log-driver=json-file \
  dividend-analyzer:latest
```

---

## 🔐 보안 권장사항

1. **비루트 사용자 실행**: Dockerfile에 USER 지시어 추가
2. **읽기 전용 파일시스템**: `--read-only` 옵션 사용
3. **시크릿 관리**: Docker secrets 또는 환경변수 암호화
4. **이미지 스캔**: `docker scan dividend-analyzer:latest`
5. **최소 권한 원칙**: 필요한 포트만 노출

---

## 📚 참고 자료

- [Docker 공식 문서](https://docs.docker.com/)
- [Streamlit Docker 가이드](https://docs.streamlit.io/knowledge-base/tutorials/deploy/docker)
- [Docker Compose 문서](https://docs.docker.com/compose/)
- [Docker 보안 베스트 프랙티스](https://docs.docker.com/develop/security-best-practices/)

---

**🐳 Docker로 어디서나 동일한 환경에서 실행하세요!**