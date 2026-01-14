# 배당주 퀀트 분석 시스템 v3

AI 기반 배당주 투자 전략 분석 및 포트폴리오 최적화 도구

## 📋 목차

- [프로젝트 개요](#프로젝트-개요)
- [주요 기능](#주요-기능)
- [기술 스택](#기술-스택)
- [설치 및 실행](#설치-및-실행)
- [사용 방법](#사용-방법)
- [프로젝트 구조](#프로젝트-구조)
- [주요 알고리즘](#주요-알고리즘)
- [배포 가이드](#배포-가이드)
- [라이선스](#라이선스)

---

## 🎯 프로젝트 개요

**배당주 퀀트 분석 시스템**은 데이터 기반 배당주 투자를 위한 종합 분석 플랫폼입니다.

### 핵심 가치
- 📊 **실시간 데이터**: 네이버 증권에서 배당주 데이터 자동 수집
- 🤖 **AI 분석**: Groq API 기반 뉴스 감성 분석 및 투자 전략 생성
- 📈 **퀀트 모델링**: PER, ROE, 배당수익률 기반 적정가 계산
- 🎯 **목표 수익률 관리**: 60% 목표 수익률 기반 포트폴리오 최적화
- 📰 **뉴스 크롤링**: 종목별 최신 뉴스 자동 수집 및 감성 분석

### 주요 특징
- 현실적인 진입가/목표가 계산 (5~40% 상승여력 범위)
- 재무지표 기반 자동 필터링 (ROE > 0, PER < 25)
- 매크로 테마별 종목 분류 (AI 인프라, 차세대 원자력 등)
- 백테스팅 시뮬레이션 기능

---

## ✨ 주요 기능

### 1. 📊 대시보드
- **진입 시그널 TOP 15**: 상승여력 10% 이상 종목 실시간 추천
- **투자 매력도 맵**: PER vs 배당수익률 버블 차트
- **목표 수익 계산기**: 투자금액 대비 필요 CAGR 자동 계산

### 2. 📰 뉴스 분석
- 종목별 최근 7~14일 뉴스 자동 크롤링
- AI 기반 감성 분석 (긍정/부정/중립, 0-100점)
- 실시간 호재/악재 탐지

### 3. 📬 백테스팅
- 현재 재무데이터 기반 1년 보유 시뮬레이션
- KOSPI 대비 초과수익률 분석
- 종목별 예상 수익 상세 리포트

### 4. 🤖 AI 전략 생성
- Groq LLaMA 3.3 70B 모델 활용
- 구체적인 종목명 + 진입가 + 목표가 제시
- 매크로 테마 연계 포트폴리오 구성
- 리스크 관리 시나리오 제공

---

## 🛠 기술 스택

### Frontend
- **Streamlit**: 인터랙티브 웹 인터페이스
- **Plotly**: 데이터 시각화

### Backend
- **Python 3.10+**
- **Pandas**: 데이터 처리
- **Requests**: 웹 크롤링
- **BeautifulSoup4**: HTML 파싱

### AI/ML
- **Groq API**: LLaMA 3.3 70B 기반 자연어 처리
- **퀀트 모델**: 자체 개발 적정가 계산 알고리즘

### Data Sources
- **네이버 증권 API**: 실시간 배당주 데이터
- **네이버 금융 뉴스**: 종목별 뉴스 크롤링
- **yfinance** (선택): 과거 주가 데이터 백테스팅

---

## 🚀 설치 및 실행

### 1. 필수 요구사항
```bash
Python 3.10 이상
pip (Python 패키지 관리자)
```

### 2. 저장소 클론
```bash
git clone https://github.com/yourusername/dividend-quant-analyzer.git
cd dividend-quant-analyzer
```

### 3. 가상환경 생성 (권장)
```bash
python -m venv venv

# Windows
venv\Scripts\activate

# Linux/Mac
source venv/bin/activate
```

### 4. 패키지 설치
```bash
pip install -r requirements.txt
```

### 5. 환경변수 설정
`.env` 파일 생성:
```env
GROQ_API_KEY=your_groq_api_key_here
```

### 6. 애플리케이션 실행
```bash
streamlit run app4.py
```

브라우저에서 `http://localhost:8501` 접속

---

## 📖 사용 방법

### 기본 워크플로우

#### 1. 투자 설정
- 사이드바에서 투자금액 입력 (기본 1억원)
- 목표 회수기간 설정 (기본 12개월)
- 관심 매크로 테마 선택

#### 2. 종목 분석
- **대시보드 탭**: 상승여력 상위 종목 확인
- **투자 매력도 맵**: 저PER/고배당 종목 시각적 식별

#### 3. 뉴스 분석 (선택)
- 관심 종목 선택
- "뉴스 분석 실행" 버튼 클릭
- AI 감성 분석 결과 확인

#### 4. 백테스팅
- 투자 종목수 설정 (5~20개)
- "시뮬레이션 실행"으로 예상 수익률 확인

#### 5. AI 전략 생성
- "구체적 종목 + 진입가 전략 생성" 버튼
- 추천 포트폴리오 및 진입 시나리오 확인

---

## 📁 프로젝트 구조

```
dividend-quant-analyzer/
│
├── app4.py                 # 메인 Streamlit 애플리케이션
├── finance_targets.py      # 금융 계산 함수 (CAGR, 목표수익 등)
├── news_backtest.py        # 뉴스 크롤링 & 백테스팅 모듈
├── requirements.txt        # Python 패키지 의존성
├── .env                    # 환경변수 (API 키)
├── README.md               # 프로젝트 문서
│
├── deployment/             # 배포 관련 파일
│   ├── Dockerfile
│   ├── docker-compose.yml
│   └── gcp/
│       ├── app.yaml        # GCP App Engine 설정
│       └── cloudbuild.yaml # Cloud Build 설정
│
└── tests/                  # 테스트 코드 (추후 추가)
    └── test_calculations.py
```

### 주요 파일 설명

#### `app4.py`
- Streamlit 기반 웹 인터페이스
- 4개 탭 구성: 대시보드, 뉴스 분석, 백테스팅, AI 전략
- 네이버 증권 데이터 실시간 수집 및 처리

#### `finance_targets.py`
핵심 금융 계산 함수:
- `target_profit()`: 목표 총 이익 계산
- `cagr_from_target()`: 필요 연복리 수익률
- `monthly_rate_from_target()`: 월복리 수익률

#### `news_backtest.py`
뉴스 분석 및 백테스팅:
- `crawl_stock_news()`: 네이버 금융 뉴스 크롤링
- `analyze_news_sentiment()`: Groq AI 감성 분석
- `simulate_dividend_strategy()`: 배당 전략 시뮬레이션
- `backtest_with_yfinance()`: 실제 과거 데이터 백테스팅

---

## 🧮 주요 알고리즘

### 적정가 계산 모델

```python
def calculate_fair_value(row, market_avg_per=12.0, market_avg_div_yield=3.0):
    """
    3가지 요소 기반 적정가 계산:
    
    1. PER 밴드 분석 (40% 가중치)
       - 시장 평균 대비 저평가 시 상승여력 부여
       - 최대 30% 상승여력
    
    2. 배당수익률 비교 (40% 가중치)
       - 시장 평균 대비 높은 배당률 시 프리미엄
       - 최대 20% 상승여력
    
    3. ROE 성장성 (20% 가중치)
       - ROE 10% 초과 시 성장성 보너스
       - 최대 15% 상승여력
    
    최종 상승여력: 5~40% 범위로 제한
    """
```

### 투자 등급 시스템

| 상승여력 | 등급 | 설명 |
|---------|------|------|
| 30% 이상 | ⭐⭐⭐ 강력매수 | 저평가 + 고배당 + 고ROE |
| 20~30% | ⭐⭐ 매수 | 2가지 이상 긍정 요소 |
| 10~20% | ⭐ 보유 | 1가지 긍정 요소 |
| 10% 미만 | ⚪ 관망 | 적정 평가 수준 |

---

## 🔧 배포 가이드

자세한 배포 방법은 별도 문서 참조:
- [Docker 배포 가이드](./docs/DOCKER_DEPLOYMENT.md)
- [GCP 배포 가이드](./docs/GCP_DEPLOYMENT.md)

### 빠른 시작

#### Docker로 로컬 실행
```bash
docker build -t dividend-analyzer .
docker run -p 8501:8501 --env-file .env dividend-analyzer
```

#### GCP App Engine 배포
```bash
gcloud app deploy
```

---

## ⚠️ 주의사항

### 투자 관련
- **본 시스템은 투자 참고용 도구이며, 투자 손실에 대한 책임은 사용자에게 있습니다.**
- 실제 투자 전 반드시 추가 분석 및 전문가 상담을 권장합니다.
- 과거 데이터 기반 시뮬레이션은 미래 수익을 보장하지 않습니다.

### 기술적 제약
- 네이버 증권 크롤링은 웹사이트 구조 변경 시 작동하지 않을 수 있습니다.
- Groq API 사용량 제한에 유의하세요 (무료 티어: 분당 30회).
- 백테스팅 결과는 단순화된 모델이며 실제 시장과 차이가 있을 수 있습니다.

---

## 🤝 기여하기

프로젝트 개선 아이디어나 버그 리포트를 환영합니다!

### 기여 방법
1. Fork the repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

---

## 📝 라이선스

MIT License - 자유롭게 사용, 수정, 배포 가능합니다.

---

## 📞 문의

- **이슈 트래커**: [GitHub Issues](https://github.com/yourusername/dividend-quant-analyzer/issues)
- **이메일**: your.email@example.com

---

## 🙏 감사의 글

- **네이버 금융**: 배당주 데이터 제공
- **Groq**: 고속 AI 추론 API
- **Streamlit**: 빠른 프로토타이핑 프레임워크

---

**⚡️ 데이터 기반 투자로 안정적인 수익을 추구하세요!**