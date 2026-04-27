# 전국 미세먼지 농도 예측 모델

> 2025년 2학기 어프렌티스 프로젝트 / 팀 **튀김소보로**

회귀(Regression) 기법을 활용해 전국 측정소 단위의 시간별 미세먼지(PM10·PM2.5) 농도를 예측하는 모델을 구축한 프로젝트임. 공공데이터 크롤링부터 전처리, 모델 선정, 학습·평가, 결론에 이르는 데이터 분석 파이프라인 전 과정을 직접 수행하여 정리한 결과물임.

---

## 프로젝트 개요

- **주제** : 전국 단위 미세먼지(PM10, PM2.5) 시간별 농도 예측 회귀 모델 구축
- **수행 기간** : 2025년 2학기 (어프렌티스 프로젝트)
- **팀명** : 튀김소보로
- **목표**
  - 에어코리아(AirKorea) 실측 데이터를 직접 수집·전처리하여 분석 가능한 형태로 가공함
  - 다양한 선형 회귀 계열 모델을 비교하여 미세먼지 농도 예측에 적합한 모델을 선정함
  - 측정소·시간대별로 모델 성능을 검증하고, 실증 예측 데모까지 수행함

---

## 분석 파이프라인

데이터 수집 → 전처리 → 회귀 모델 선정 → 학습 / 평가 → 결론 순으로 진행함.

### 1. 데이터 수집

- 출처 : [에어코리아(AirKorea)](https://www.airkorea.or.kr/) 시간별 미세먼지 공시 데이터
- 수집 기간 : **2024-01-01 ~ 2025-11-26**
- 수집 항목 : PM10, PM2.5
- 수집 범위 : 전국 17개 시·도 측정소
- 수집 방식 : `requests` + `BeautifulSoup` 기반 자체 크롤러를 구현함
- 관련 파일 : [미세먼지데이터_크롤링.ipynb](미세먼지데이터_크롤링.ipynb)
- 결과 산출물 : [airkorea_result/](airkorea_result/) 하위에 PM10·PM2.5별 CSV 17개씩 저장됨

### 2. 데이터 전처리

원본 wide-format 데이터를 시계열 분석에 적합한 long-format으로 변환하고, lag/rolling 기반 피처를 생성함.

- 시간 컬럼(0~23시)을 melt하여 long-format으로 재구성함
- `-` 결측치를 NaN으로 치환 후 측정소별 선형 보간으로 처리함
- 시계열 피처 엔지니어링
  - 시간 기반 : `month`, `day`, `weekday`, `is_weekend`
  - 지연(lag) : `lag1` (1시간 전), `lag24` (24시간 전)
  - 이동평균(rolling) : `rolling3` (3시간), `rolling24` (24시간)
- `StandardScaler`로 수치형 피처를 정규화함
- 학습/검증 분리 : **2025-07-01** 기준으로 train/test 시계열 분할
- 관련 파일 : [preprocessing.ipynb](preprocessing.ipynb)
- 산출물 : `preprocessed_pm*.csv`, `train_pm*.csv`, `test_pm*.csv`, [scaler_pm10.pkl](scaler_pm10.pkl), [scaler_pm2.5.pkl](scaler_pm2.5.pkl)
- 전처리 결과 데이터 규모 : 약 **1,122만 건** (PM2.5 기준)

### 3. 회귀 모델 선정

선형 회귀 계열을 중심으로 후보 모델을 선정하고, 정규화 강도와 비선형성 보강 가능성을 함께 검토함.

- **LinearRegression** : 기본 최소제곱 회귀 (베이스라인)
- **Ridge (α=1.0)** : L2 정규화로 과적합을 완화함
- **Lasso (α=0.1)** : L1 정규화로 변수 선택 효과를 확인함
- **MLP (다층 퍼셉트론)** : 비선형성 확장을 위한 추가 비교 모델
- 비교 자료 : [선형회귀모델 비교.pptx](선형회귀모델%20비교.pptx), [선형회귀모델 비교_MLP추가.pptx](선형회귀모델%20비교_MLP추가.pptx)

### 4. 학습 및 평가

- 평가 지표 : R², MAE, RMSE
- 전체 테스트셋 성능 (PM10 기준)

| 모델               |     R² |   MAE |  RMSE |
| ------------------ | -----: | ----: | ----: |
| Ridge (α=1.0)      | 0.9151 | 2.569 | 4.360 |
| LinearRegression   | 0.9151 | 2.569 | 4.360 |
| Lasso (α=0.1)      | 0.9074 | 2.676 | 4.553 |

- 상위 두 모델(Ridge, LinearRegression)이 **R² 0.915** 수준의 안정적인 성능을 기록함
- 측정소별 RMSE 비교(대전 서구 월평동·둔산동, 충북 청주시 오창읍·봉명동 등)와 시계열 시각화로 실제 관측치 대비 예측 곡선의 적합도를 확인함
- 실증 예측 데모 : 특정 측정소·일시를 입력하면 실제값과 예측값을 비교하는 함수를 구현함
  - 예) 대전 서구 둔산동 / 2025-09-23 12시 → 실제 6.00 vs 예측 8.83 (오차 +2.83)
- 관련 파일 : [modeling.ipynb](modeling.ipynb)

### 5. 결론

- 시계열 lag/rolling 피처를 활용한 **선형 회귀 모델만으로도 R² 0.91 이상의 예측력**을 확보할 수 있음을 확인함
- Ridge와 LinearRegression의 성능이 거의 동일하게 나타났으며, Lasso는 일부 피처 가중치가 0으로 수렴하면서 성능이 소폭 하락함
- 측정소별 RMSE 편차가 존재하므로, 지역 특성에 따른 모델 세분화 또는 비선형 모델(MLP, Tree 계열) 도입 시 추가 개선 여지가 있음
- 본 프로젝트를 통해 **공공 데이터 수집 → 전처리 → 모델링 → 평가**로 이어지는 데이터 분석 사이클을 end-to-end로 경험함

---

## 프로젝트 산출물

### 문서

- [어프렌티스_프로젝트계획_튀김소보로.pdf](어프렌티스_프로젝트계획_튀김소보로.pdf) — 기획 단계 발표 자료
- [어프렌티스_프로젝트_튀김소보로_최종발표.pdf](어프렌티스_프로젝트_튀김소보로_최종발표.pdf) — 최종 발표 자료
- [선형회귀모델 비교.pptx](선형회귀모델%20비교.pptx) — 모델 비교 1차본
- [선형회귀모델 비교_MLP추가.pptx](선형회귀모델%20비교_MLP추가.pptx) — MLP 포함 확장본

### 코드 / 노트북

- [미세먼지데이터_크롤링.ipynb](미세먼지데이터_크롤링.ipynb) — 에어코리아 크롤러
- [preprocessing.ipynb](preprocessing.ipynb) — 데이터 전처리 및 피처 엔지니어링
- [modeling.ipynb](modeling.ipynb) — 모델 학습·평가·시각화·실증 데모

### 데이터 / 산출물

- [airkorea_result/PM10/](airkorea_result/PM10/) — PM10 측정소별 원본 CSV
- [airkorea_result/PM2.5/](airkorea_result/PM2.5/) — PM2.5 측정소별 원본 CSV
- [scaler_pm10.pkl](scaler_pm10.pkl), [scaler_pm2.5.pkl](scaler_pm2.5.pkl) — 학습된 StandardScaler

---

## 사용 기술 스택

- **언어** : Python 3.10
- **데이터 수집** : `requests`, `BeautifulSoup`, `pandas.read_html`
- **데이터 처리** : `pandas`, `numpy`
- **모델링** : `scikit-learn` (LinearRegression, Ridge, Lasso), MLP
- **시각화** : `matplotlib` (한글 폰트 [malgun.ttf](malgun.ttf) 사용)
- **저장/직렬화** : `joblib`

---

## 실행 방법

```bash
# 1) 데이터 수집 (이미 수집된 CSV가 airkorea_result/에 포함되어 있음)
jupyter notebook 미세먼지데이터_크롤링.ipynb

# 2) 전처리
jupyter notebook preprocessing.ipynb

# 3) 모델 학습 및 평가
jupyter notebook modeling.ipynb
```

> ※ `preprocessed_pm*.csv`, `train_pm*.csv`, `test_pm*.csv`는 [.gitignore](.gitignore)로 관리되므로, 전처리 노트북 실행 시 재생성됨.
