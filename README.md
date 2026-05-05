# Hybrid-OPF: AC/DC Optimal Power Flow (NLP to SOC)

![Python](https://img.shields.io/badge/Python-3.x-blue?style=flat&logo=python)
![Julia](https://img.shields.io/badge/Julia-1.x-9558B2?style=flat&logo=julia)
![Optimization](https://img.shields.io/badge/Optimization-NLP%20%7C%20SOCP-orange)

## Description
본 프로젝트는 전력 계통 최적 조류 계산(OPF, Optimal Power Flow)의 핵심 알고리즘을 손코딩(Scratch Implementation)부터 시작하여 최신 최적화 기법까지 단계별로 분석하고 구현합니다. 비선형 방정식 기반의 기본 정식화(NLP)부터 전역 최적해(Global Optimum) 보장을 위한 볼록 완화(Convex Relaxation) 과정을 다루며, Python과 Julia를 활용한 다각도 코드 분석을 포함합니다.

---

## Implementation Roadmap

### Phase 1: 모델링 및 수식 정식화 (Modeling & Formulation)
전력 시스템의 비선형 특성을 반영한 NLP(Non-Linear Programming) 기반의 기초 방정식을 정립합니다.
- 사용 모델: BIM (Bus Injection Model), BFM (Branch Flow Model)
- 확장 구조: AC 망과 DC 망(HVDC), 그리고 이를 연결하는 컨버터(VSC, LCC) 통합 모델링
### Phase 2: 단계별 직접 구현 (Scratch Implementation)
- Step 2.1: NLP-BFM 손코딩
  - 선로 전류 중심 모델(BFM)을 사용하여 비선형 최적화 문제를 직접 구성하고 Ipopt 솔버를 통해 로컬 최적해를 도출합니다.
- Step 2.2: SOC Relaxation (Convexification)
  - 비선형 등식 제약을 Second-Order Cone(2차 원뿔) 부등식 제약으로 완화하여 문제를 볼록화(Convexify)합니다.
  - 치환 변수($W = U^2, i_{sq} = I^2$) 도입 및 Relaxation Gap 분석을 진행합니다.
- Step 2.3: 풀이 기법 확장
  - QC(Quadratic Convex), SDP(Semidefinite Programming), LP 근사 등 다양한 솔버 적용 및 성능 비교.

### Phase 3: 멀티 언어 교차 검증 (Code Deep Dive)
- Python (Pyomo/CVXPY): 수식의 직관적인 모델링 및 프로토타이핑 구현.
- Julia (`PowerModelsACDC.jl`): 줄리아 기반 연구용 패키지 내부 코드 구조 분석 및 Python 구현 결과와의 벤치마킹.

---

## 핵심 수식 (Core Formulation)
Ergun et al. (2019)의 연구를 기반으로 AC/DC 그리드 및 컨버터 스테이션을 통합 최적화합니다.

### 1. 전압 한계 및 DC 선로 모델 (BFM 기준)
- 전압 제약: $U_i^{mag,min} \le U_i^{mag} \le U_i^{mag,max}$
- DC 전력 조류 (P=VI): $P_{def}^{dc} = p_d \cdot U_e^{dc} \cdot I_{def}^{dc}$
- DC 선로 손실 ($I^2R$): $P_d^{dc,loss} = p_d \cdot r_d \cdot (I_{def}^{dc})^2$

### 2. 전력전자 컨버터 (AC-DC 연결)
컨버터는 AC측과 DC측이 옴의 법칙이 아닌 에너지 보존 법칙으로 연결됩니다.
- 에너지 보존: $P_c^{cv,ac} + P_c^{cv,dc} = P_c^{cv,loss}$
- 컨버터 손실 모델: $P_c^{cv,loss} = a_c^{cv} + b_c^{cv}I_c^{cv,mag} + c_c^{cv}(I_c^{cv,mag})^2$
- AC측 전력-전류 관계 (비선형): $(P_c^{cv,ac})^2 + (Q_c^{cv,ac})^2 = (U_c^{cv,mag})^2 \cdot (I_c^{cv,mag})^2$

### 3. 노달 밸런스 (KCL)
- DC 노드 KCL: $\sum P_c^{cv,dc} + \sum P_{def}^{dc} = -\sum P_m$
- AC 유효전력 KCL: $\sum P_{cie}^{tf} + \sum P_{lij}^{ac} = \sum P_g - \sum P_m - g_i^{sh}(U_i^{mag})^2$

---

## 모델 관점 및 Relaxation 계층

본 프로젝트는 수식을 바라보는 변수의 관점(축 1)과 수학적 풀이 방법(축 2)을 분리하여 분석합니다.

### 축 1: 모델 관점 (Model Perspectives)
| 약어 | 설명 | 핵심 변수 | 특징 |
| :--- | :--- | :--- | :--- |
| **BIM** | Bus Injection Model | $V, \theta$ | 망형(Meshed) 계통에 자연스러움|
| **BFM** | Branch Flow Model | $I$ (선로 전류) | 방사형(Radial) 계통 및 DC 표현에 매우 직관적 |
| **DC** | DC Linear 근사 | $\theta$, $U \approx 1$ | AC 선형화, 손실 무시 |
| **NF** | Network Flow | (전력 보존) | DC 선형화, 손실 완전 무시 |

### 축 2: 수학적 풀이 방법 (Solving Methods)
| 약어 | 풀이 방법 | 완화/근사 기법 | 특징 |
| :--- | :--- | :--- | :--- |
| **NLP** | Non-Linear Programming | 원본 비선형 수식 | 로컬 최적해 도출 (Ipopt) |
| **SDP** | Semidefinite Programming | $W \succeq 0$ (PSD Matrix) | 대규모 계통에서 수치 문제 발생 가능성 높음 |
| **QC** | Quadratic Convex | Convex Envelope | SDP < QC < SOC 강도, 확장성 한계 존재 |
| **SOC** | Second-Order Cone | $P^2+Q^2 \le W \cdot i_{sq}$ | 실용적 최선, DC 그리드에서 Exact 보장 |
| **LP** | Linear Programming | 선형 근사 | 속도는 가장 빠르나 오차 발생 |

---

## Tech Stack
- Languages: Python 3.x, Julia
- Optimization Modeling: Pyomo, CVXPY, JuMP
- Solvers: Ipopt (NLP), Mosek / Gurobi (SOC/SDP)
