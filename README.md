# ds6_warpUp
warp up pjt 

[ds6]
--------------------------------------------------------------------------------------
■ 주제
- AI를 활용한 금융 데이터 분석
- 딥러닝을 이용한 주가 예측, 환율 예측, 수익률·리스크 기반 포트폴리오 최적화(Monte Carlo Simulation, 효율적 프론티어 구현)

■ 목표
- AI을 활용한 데이터 분석 능력 습득
- 파이썬 환경 설정, 데이터 전처리, 시계열 모델링(ARIMA, Facebook Prophet 실습)  등등
- colab/Kaggle/VScode 활용, 금융 기반 실시간 데이터 분석 및 AI 모델링
- 최종 나만의 주식 분석기 생성
- 꾸준히 수익을 주면서 장기적으로 끌고 갈 수 있는 주식을 발굴하고 시스템화함
  
■ 방식 및 방법
- 금융 데이터 학습 : 엔씨소프트 외 게임주식을 시작으로 주식의 생리와 매커니즘 이해
- 각종 시계열 데이터 분석법 실습을 통한 가치 확인
- Colab의 GPU 환경에서 실행

■ 진행 안
1회차 - inflearn학습, github학습(1. 주식데이터이해)
        202510 - 주식진행안 확정 (워렌버핏따라잡기)
2회차 - github학습 (2. 데이터 수집 및 정제)
        20251031 - 분석 섹터 확정, 데이터사이언스 진행 방향 확정, test modeling 성공(NCsoft.ipynb)
                 - next : SHAP vs XBooter BUG해결 or 다른 솔루션 검색 적용
                            data 범주확장, 가중치 범주 확장, 모델링 정교화 + 1031버젼 Update, 소규모소외주 모델링 
3회차 - My github  (2. 데이터 수집 및 정제)
        20251117 - test modeling, visualization 
4회차 - github학습 (3. 분석 시스템 구현 학습)
        20251125 - 시계열 주가 수익율, 상승하락 방향 예측(엔씨소프트, 이동평균,RSI,MACD 후 로지스틱회귀와 랜덤포레스트 적용
5회차 - My github (3. 분석 시스템 구현 학습)
        20251216 - 엔씨소프트  이벤트-driven 변동성을 분석
                  일별 주가 데이터를 수집, 이동평균 등 기본 전처리 수행, 
                  Isolation Forest 비정상 가격·거래량 변동 날짜를 탐지
6회차 - github학습 (4. RNN 학습 및 구현: AI분석)
        20251231 - Python, Streamlit, Pandas(Time-series), GPT-4o(RAG/LLM), SQLite
                   현재 시점의 기술적 상태를 조건 변수(feature)로 정의하고,
                   과거 유사 상태 기반 통계 추론(conditional empirical analysis), 
                   예측값 하나를 출력하는 대신, 수익률 분포, 승률, 평균·중앙값 등 
                   경험적 통계량을 제공

7회차 - My github 완성 및 공유
        20260109 - 'AI 기반 배당주 가치투자 플랫폼'
                  멀티 팩터 퀀트 모델과 LLM을 결합,개인별 투자 목표에 최적화된 배당주 포트폴리오를 제안
                  금융 데이터 스크래핑, 퀀트 모델링, 그리고 생성형 AI 분석을 결합한 지능형 배당주 투자 분석 솔루션
                    분류                사용 기술              역할
                    Data Collection     "Requests, Pandas"    네이버 증권의 배당주 데이터를 실시간 스크래핑 및 가공
                    Quant Modeling      Numeric Python,"PER, ROE, 배당수익률 기반의 다변수 가중 적정가 알고리즘 구현"
                    Visualization       "Plotly, Streamlit"  투자 매력도 산점도(Scatter Plot) 및 대화형 대시보드 구축
                    AI Integration      Groq API (Llama-3.3)  퀀트 데이터와 사용자 투자 아이디어를 결합한 전략 생성
                    Finance Math        CAGR/ROI Logic        목표 수익률 달성을 위한 연복리 및 월간 필요 수익 정량화

■ 참고 site

(1) inflearn학습
: [(무료) 파이썬으로 하는 주가 데이터 분석 입문(금융/퀀트) 강의 | 대시보드 - 인프런](https://www.inflearn.com/course/%ED%8C%8C%EC%9D%B4%EC%8D%AC-%EC%B4%88%EB%B3%B4%EC%9E%90-%ED%80%80%ED%8A%B8%ED%88%AC%EC%9E%90-%EC%89%BD%EA%B2%8C%EC%8B%9C%EC%9E%91%ED%95%98%EA%B8%B0/dashboard)

(2) github학습
 1. [Isanghada/Stock_Analysis: 주식 분석 및 종가 예측 프로젝트](https://github.com/Isanghada/Stock_Analysis)
 2. [teddylee777/machine-learning: 머신러닝 입문자 혹은 스터디를 준비하시는 분들에게 도움이 되고자 만든 repository입니다. (This repository is intented for helping whom are interested in machine learning study)](https://github.com/teddylee777/machine-learning)
 3. [LSTM과 FinanceDataReader API를 활용한 삼성전자 주가 예측 - 테디노트](https://teddylee777.github.io/tensorflow/lstm-stock-forecast/)
(3) 그외
 1. [**https://hyun-tori.tistory.com/112**](https://hyun-tori.tistory.com/112)

■ book - 통계 101 x 데이터 분석 

