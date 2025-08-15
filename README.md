# KSPHM Data Challenge - Bearing Remaining Useful Life (RUL) Prediction

이 저장소는 **KSPHM 데이터 챌린지**에 참가하며 수행한 베어링 잔존수명(RUL, Remaining Useful Life) 예측 프로젝트의 코드와 시행착오 과정을 담고 있습니다.  
데이터 파일은 포함되어 있지 않으며, 모든 실험 코드, 전처리 파이프라인, 모델 구현 및 평가 스크립트를 공유합니다.

---

## 프로젝트 개요

본 과제의 목표는 베어링 진동 데이터를 기반으로 고장 시점을 예측하고, 이를 통해 유지보수 효율성을 높이는 예측 모델을 개발하는 것입니다.  
주어진 TDMS 포맷의 시계열 데이터를 분석하여 Remaining Useful Life를 추정합니다.

---

## 데이터 처리 및 특징 추출

### 1. 데이터 세트 구분

- **중단조건 포함 (고장 데이터)** : Train 1, 4, 5, 7, 8
- **중단조건 미포함 (정상 종료 데이터)** : Train 2, 3, 6

### 2. RUL 라벨링

- **중단조건 포함 세트**
  - 고장 시점 `T_failure` :
    - Torque ≤ -17 Nm
    - TC SP Front 또는 TC SP Rear ≥ 200℃
    - 위 두 조건 중 먼저 도달한 시점을 고장 시점으로 정의
- **중단조건 미포함 세트**
  - 온도 데이터 기반 선형 회귀로 200℃ 도달 시점 예측
  - 회귀 결정계수(R²) ≥ 0.85일 때만 유효

### 3. 특징 추출 (Feature Engineering)

- **진동 데이터 (CH1~CH4)**
  - 샘플링: 25.6 kHz
  - 윈도우 크기: 25,600 (1초 단위), 오버랩 50%
  - WPT (Wavelet Packet Transform, db4, level 3)
  - 각 노드 FFT → bandwidth=5로 주파수 대역 나눔 → 에너지 계산
  - 주요 고장 주파수: BPFI=140Hz, BPFO=93Hz, BSF=78Hz, Cage=6.7Hz
  - 에너지 기준 상위 Top 10 노드 선택
- **운전 데이터 (Torque, Temperature)**
  - Torque Slope / Temperature Slope
  - 복합 지표: Temp Slope × Torque Slope

---

## 모델 구조

**1D CNN + LSTM 기반 회귀 모델**

- **Conv1D**
  - 필터: 8, 커널 크기: 3, Padding: causal, Activation: ReLU
  - MaxPooling, Dropout(0.2)
- **LSTM**
  - 유닛: 16, L2 정규화 적용, Dropout(0.3)
- **Dense** : 1차원 출력 (RUL 예측, 선형 회귀)

---

## 학습 설정

- 손실함수: MSE
- 최적화: Adam (lr=0.0001)
- 배치 크기: 32, Epoch: 100
- EarlyStopping (patience=7)
- ReduceLROnPlateau (factor=0.5, min_lr=1e-5)

---

## 성능

- 최종 파라미터 수: 1,649
- val_loss: 초기 1.79 → 최종 0.0114
- MCNN 대비 단일 CNN 구조가 더 나은 일반화 성능을 보임

---

## 시행착오 & 개선 방향

- WPT + FFT 특징 조합이 주파수 기반 RUL 예측에 유용함을 확인
- MCNN 구조는 과적합 발생 → 단순 CNN이 더 안정적
- 향후 개선: 물리 기반 특징 + 통계적 신호 특징 융합, 추가 센서 데이터 활용
