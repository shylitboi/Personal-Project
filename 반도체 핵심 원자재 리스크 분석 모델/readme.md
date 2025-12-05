# FAB Chokepoint–Driven Semiconductor Material Risk Modeling

> A Structural Vulnerability and Supply Chain–Integrated Model for Semiconductor Fab Shutdown Risk

---

## 1. Overview

본 프로젝트는 반도체 제조 공정(Fab) 내부의 구조적 병목과 글로벌 공급망 리스크를 통합적으로 분석하여,
Fab 가동 중단(Fab shutdown)을 유발할 ㄴㄴ수 있는 핵심 원자재(chokepoint materials)를 정량적으로 식별하는 것을 목표로 한다.

Fab 생산 네트워크는 수많은 공정 단계와 소재가 유향 그래프 형태로 연결된 복잡계 시스템이다.
이와 동시에, 각 소재는 특정 국가 및 소수 공급자에 편중된 공급망 구조를 갖는다.
따라서 공정 구조와 공급망 두 차원을 동시에 고려하지 않으면 실질적인 Fab shutdown 위험을 제대로 평가하기 어렵다.

본 프로젝트는 다음의 연구 질문에 답하고자 한다.

* RQ1. Fab 공정 네트워크 상에서 구조적으로 가장 취약한 공정(Structural chokepoints)은 무엇인가?
* RQ2. 이러한 공정에 강하게 결합된 원자재 중, 공급망 관점에서 가장 위험한 소재는 무엇인가?
* RQ3. 공정 병목과 공급망 리스크를 결합하여, Fab shutdown 위험 관점에서 핵심 원자재를 어떻게 정량화할 수 있는가?

---

## 2. Background and Motivation

반도체 산업은 지정학적 갈등, 수출 규제, 특정 국가·기업 의존 등 다양한 외부 충격 요인에 노출되어 있다.
기존 연구와 실무 분석은 다음과 같은 한계를 가진다.

1. 공급 측면(HHI, 국가 의존도 등)만을 고려하여,
   Fab 내부 공정 구조(어느 공정이 bottleneck인지)를 반영하지 못함.
2. 또는 Fab 내부 공정 효율·캐파 분석에만 집중하여,
   실제 공급망 충격 발생 시 어떤 소재가 공정 전체를 멈추게 할지를 평가하지 못함.

이에 본 프로젝트는 다음과 같은 통합적 분석 틀을 제안한다.

1. CSET Advanced Semiconductor Supply Chain Dataset을 활용하여 Fab 공정 및 원자재 네트워크를 구성한다.
2. 네트워크 중심성과 reachability 기반으로 공정 병목 구조를 분석한다.
3. UN Comtrade 기반 HS code 수출 데이터를 활용하여 소재별 공급 위험(Supply Risk)을 산출한다.
4. 두 결과를 결합하여 Material Bottleneck Index(MBI)를 정의하고, Fab shutdown 관점의 핵심 원자재를 식별한다.
5. 향후 연구 방향으로서 Graph Neural Network(GNN) 및 Temporal Graph Network(TGN)를 통한 중요도 학습 및 동태적 리스크 예측을 제안한다.

---

## 3. Data Description

### 3.1 Primary Dataset: CSET Advanced Semiconductor Supply Chain Dataset (2025)

CSET 데이터셋은 칩 생산에 사용되는 공정·재료·장비·국가·기업 정보를 구조적으로 정리한 공급망 데이터셋이다.
본 프로젝트에서는 다음 파일들을 사용하였다.

| 파일명             | 주요 내용                                                 |
| --------------- | ----------------------------------------------------- |
| `inputs.csv`    | 공정(process), 재료(material), 장비(tool) 목록 및 ID, 이름, 카테고리 |
| `providers.csv` | 소재별 공급 국가·기업 정보                                       |
| `provision.csv` | 연도별 소재–국가별 수출·공급 비중                                   |
| `sequence.csv`  | 공정 간 연결 관계 (전·후 공정 ID)                                |
| `stages.csv`    | 공정 stage 정보 (예: Lithography, Etch, Deposition 등)      |

### 3.2 External Trade Data: UN Comtrade

공급망 리스크 산출을 위해 HS Code 기반 국가별 수출 데이터를 추가로 수집하였다.

* 예시

  * Wafer: HS 3818
  * Photoresist: HS 370790 / 3707
  * CMP Slurry: HS 2834 / 2811 등 관련 코드
  * Wet Chemicals: HS 2812 / 2827 / 2829 등

각 소재에 대해 Material–HS Code 매핑을 수동·반자동으로 수행한 후,
국가별 점유율을 계산하여 공급 집중도를 측정하였다.

---

## 4. Preprocessing Pipeline

전처리 단계는 크게 네 가지 블록으로 구성된다.

### 4.1 Provision Cleaning 및 연도 선택

* `provision.csv`에서 최신 연도(가장 최근 연도)의 데이터만을 선택하였다.
* 동일 원자재–국가 조합에 대해 중복 또는 비정상 값이 존재하는 경우 제거 또는 평균값으로 통합하였다.
* 공급자가 "Various" 등으로 표기된 레코드는 국가별 리스크 분석이 불가능하므로 제외하였다.

### 4.2 Category 및 Stage 자동 분류

* `inputs.csv`의 `input_name`, `category` 정보를 기반으로 다음과 같이 상위 카테고리를 구성하였다.

  * Materials: Wafer, Photoresist, CMP materials, Wet chemicals, Deposition materials, Photomask, Electronic gases, Packaging materials 등
  * Processes: Lithography, Etch & Clean, Ion implantation, Deposition, CMP, Assembly & Packaging, Testing 등
* `stages.csv`를 활용해 각 공정에 stage 정보를 부여하고, 공정–stage 매핑 테이블을 구축하였다.

### 4.3 공정 시퀀스 그래프 구성

* `sequence.csv`에서 `input_id`–`goes_into_id` 쌍을 추출하여 Directed Graph를 생성하였다.
* Self-loop, cycle 등이 비정상적으로 표시된 경우 제거하거나 수동 수정하였다.
* 최종적으로 공정 노드를 기준으로 한 Directed Process Graph ( G_P = (V_P, E_P) )를 정의하였다.

### 4.4 master_df 구성

전처리 결과를 통합하여 분석에 사용되는 기본 데이터프레임 `master_df`를 생성하였다.

* 주요 컬럼

  * `input_id`: 원자재 또는 공정 고유 ID
  * `input_name`: 이름
  * `category`: 상위 카테고리 (예: Photo–Material, Etch–Process)
  * `stage_name`: Fab 단계 정보
  * `market_size_bn`: 시장 규모 추정치 (단위: billion USD)
  * `share_{country}`: 국가별 점유율 (예: `share_USA`, `share_JPN` 등)

---

## 5. Structural Vulnerability of Fab Processes

### 5.1 Process Bottleneck Index

Fab 공정 네트워크의 구조적 병목을 파악하기 위해,
다음 네 가지 네트워크 지표를 사용하였다.

* Betweenness centrality
* PageRank
* Out-degree
* Downstream reach (해당 공정에서 시작해 도달 가능한 후속 노드 수)

각 공정 ( p )에 대해 지표 값을 정규화한 후 평균하여
Process Bottleneck Index ( B_p )를 정의한다.

[
B_p = \frac{1}{4}\bigg( \tilde{BC}_p + \tilde{PR}_p + \tilde{OD}_p + \tilde{DR}_p \bigg)
]

여기서

* ( \tilde{BC}_p ): betweenness centrality의 min–max 정규화 값
* ( \tilde{PR}_p ): PageRank 정규화 값
* ( \tilde{OD}_p ): out-degree 정규화 값
* ( \tilde{DR}_p ): downstream reach 정규화 값

### 5.2 Process-Level Results

CSET 기준 공정 흐름(Deposition → Photolithography → Etch → Ion implantation → CMP → Assembly → Testing)에 대해 분석한 결과,
다음 공정들이 높은 병목 지수를 보였다.

* Ion implantation
* Etch & clean
* CMP
* Photolithography

특히 Ion implantation과 CMP는, 해당 공정 제거 시 downstream 공정의 상당 부분이 비가역적으로 차단되는 것으로 나타나
실질적인 구조적 chokepoint로 해석된다.

---

## 6. Material-Level Risk Modeling

원자재 리스크는 공정 병목에 대한 노출도와 공급망 리스크를 결합하여 평가하였다.

### 6.1 Supply Risk Score

특정 원자재 ( m )에 대해, 해당 소재의 글로벌 수출을 담당하는 국가 집합을 ( {1,\dots,N} ),
각 국가의 점유율을 ( share_i )라 할 때, Herfindahl–Hirschman Index(HHI)를 다음과 같이 정의한다.

[
HHI_m = \sum_{i=1}^{N} share_i^2
]

이 값을 공급망 집중도 지표로 사용하며, 정규화 후 Supply Risk Score ( S_m )로 사용한다.

[
S_m = \frac{HHI_m - \min(HHI)}{\max(HHI) - \min(HHI)}
]

PDF의 결과에 따르면, Wet chemicals, CMP materials, Photoresist 등이 높은 공급 집중도를 나타낸다.

### 6.2 Process Exposure Score

Material ( m )가 투입되는 공정 집합을 ( \mathcal{P}(m) )라고 할 때,
해당 원자재의 공정 노출도(Process Exposure Score)는 다음과 같이 정의한다.

[
PE_m = \frac{1}{|\mathcal{P}(m)|} \sum_{p \in \mathcal{P}(m)} f(B_p, B^{\max}_p, R_p)
]

실제 구현에서는 각 원자재가 연관된 공정의

* 평균 bottleneck index
* 최대 bottleneck index
* 평균 downstream reach

를 산술 평균하여 다음과 같이 계산하였다.

[
PE_m = \frac{1}{3}\big( \overline{B}_m + B^{\max}_m + \overline{DR}_m \big)
]

여기서

* ( \overline{B}_m ): 원자재 ( m )이 투입되는 공정들의 평균 병목 지수
* ( B^{\max}_m ): 해당 공정들 중 최대 병목 지수
* ( \overline{DR}_m ): 해당 공정들의 평균 downstream reach

### 6.3 Material Bottleneck Index (MBI)

최종적으로 각 원자재에 대한 Material Bottleneck Index ( MBI_m )는
공급망 리스크와 공정 노출도를 동일 가중으로 결합하여 정의하였다.

[
MBI_m = 0.5 \times S_m + 0.5 \times PE_m
]

이는

* 공급망 충격 발생 확률(혹은 용이성)과
* 충격 발생 시 Fab 구조에 미치는 영향도의

균형 잡힌 조합으로 해석할 수 있다.

---

## 7. Integrated Risk Scoring and Results

### 7.1 Material Risk Ranking

MBI를 기준으로 산출한 상위 위험 원자재는 다음과 같다(값은 예시이며 PDF 결과를 요약한 것이다).

| Rank | Material             | Supply Risk (S_m) | Process Exposure (PE_m) | MBI (MBI_m) | 해석                                             |
| :--: | -------------------- | :---------------: | :---------------------: | :---------: | ---------------------------------------------- |
|   1  | Deposition materials |         높음        |          매우 높음          |      최고     | 전 공정 초입부에서 Layer 형성에 필수, 다양한 공정과 연결된 구조적 병목    |
|   2  | Photoresist          |       중간~높음       |            높음           |    매우 높음    | Lithography 공정에 독점적으로 투입, 공급 집중과 공정 병목이 동시에 존재 |
|   3  | Photomask            |         중간        |            높음           |      높음     | 노광 공정 품질과 직결, 대체 어려움                           |
|   4  | CMP materials        |         높음        |          중간~높음          |      높음     | 평탄화 공정에 필수, 특정 공급자 의존도가 크며 공정 자체도 chokepoint   |
|   5  | Wafer                |         중간        |          매우 높음          |      높음     | 모든 공정의 시작점으로 Fab 전체를 좌우하는 기반 소재                |

위 결과에서, Wafer는 공급 집중도가 상대적으로 낮음에도 불구하고 공정 노출도가 매우 높아
Fab 전체의 "전 공정 기반 리스크"를 형성한다.
반면 Wet chemicals 등은 공급 리스크가 매우 높지만 공정 노출도 관점에서는 중간 수준으로 평가된다.

### 7.2 Process–Material Risk Coupling

공정 병목 분석과 원자재 MBI를 결합하면 다음과 같은 결론을 얻을 수 있다.

1. Ion implantation, Etch, CMP, Photolithography 등 구조적 병목 공정에 강하게 결합된 소재는
   본질적으로 Fab shutdown 위험이 높다.
2. 이 중에서도 공급망 집중도가 높은 소재(예: CMP slurry, 특정 Wet chemicals)는
   정책·투자 관점에서 최우선 관리 대상이다.
3. Wafer와 같이 공급망 리스크는 상대적으로 완화되어 있으나, 공정 구조상 절대적인 기반을 이루는 소재는
   Fab 가동성 관점에서 별도의 관리 프레임워크가 필요하다.

---

## 8. Role of Graph Neural Networks (GNN) and Temporal Extensions

슬라이드 후반부 및 초기 설계안에는 GNN과 Temporal Graph Network(TGN)에 대한 확장 방향이 제시되어 있다.
본 README에서는 실제 구현된 정적 분석 파이프라인을 우선 기술하고,
GNN 기반 접근은 향후 연구 계획으로 정리한다.

### 8.1 GNN-based Structural Importance Learning

향후에는 공정–소재–공급자–국가를 포함하는 이종 그래프를 구성하고,
Heterogeneous GNN(HeteroConv, RGCN 등)을 통해 노드 중요도 ( w_i )를 학습하는 방안을 고려한다.

* 노드 유형: Material, Process, Provider, Country
* 엣지 유형:

  * Material → Process (투입 관계)
  * Process → Process (공정 시퀀스)
  * Provider → Material (공급 관계)
  * Country → Provider (소속 관계 등)

GNN 학습을 통해 다음과 같은 질문에 답할 수 있다.

* 특정 소재가 제거되었을 때, 공정 네트워크가 얼마나 쉽게 복원 가능한가?
* 구조적으로 대체 불가능한 소재는 무엇인가?

### 8.2 Temporal Graph Networks (TGN)

UN Comtrade 및 공급망 데이터는 연도별로 누적되므로,
시간 축을 포함한 Temporal Graph로 확장할 수 있다.

* 시간에 따라 증가하는 HHI 추세를 반영하여, 취약성이 성장하는 소재를 조기에 탐지한다.
* 공급 규제, 전쟁, 팬데믹 등 특정 이벤트를 노드·엣지 특성으로 주입하고,
  TGN을 통해 리스크 전파 경로를 시뮬레이션하는 방향을 고려한다.

이러한 확장 연구에서는, 정적 MBI를 기반으로 한 현재 모델이
초기 베이스라인 및 피처 생성기 역할을 하게 된다.

---

## 9. Significance

본 연구·프로젝트의 의의는 다음과 같이 정리할 수 있다.

1. Fab 내부 공정 구조와 외부 공급망 데이터를 통합한 리스크 분석 프레임워크를 제시하였다.
2. 공정 병목 지수와 공급망 집중도(HHI)를 결합한 Material Bottleneck Index(MBI)를 정의하여,
   반도체 Fab shutdown 위험 관점에서 원자재 우선순위를 정량화하였다.
3. 기존 공급망 분석(예: HHI 기반 국가 의존도)만으로는 포착할 수 없었던
   공정 구조 기반의 취약성을 반영함으로써, 실제 Fab 운영 및 투자 의사결정에 더 직접적인 시사점을 제공한다.
4. GNN/TGN을 활용한 동태적·관계적 리스크 학습 방향을 제안하여,
   향후 연구 및 실무 응용을 위한 확장 가능성을 명시하였다.

---

## 10. Repository Structure

아래 구조는 실제 코드 및 노트북 구성을 기준으로 한 권장 구조 예시이다.

```bash
.
├── data
│   ├── raw
│   │   ├── inputs.csv
│   │   ├── providers.csv
│   │   ├── provision.csv
│   │   ├── sequence.csv
│   │   └── stages.csv
│   └── external
│       └── un_comtrade_*.csv      # HS code별 국가 수출 데이터
│
├── notebooks
│   ├── 01_preprocessing.ipynb     # master_df 생성, 카테고리/스테이지 매핑
│   ├── 02_process_bottleneck.ipynb# 공정 네트워크 구축 및 병목 지수 계산
│   ├── 03_supply_risk.ipynb       # UN Comtrade 기반 HHI 및 SupplyRisk 산출
│   ├── 04_process_exposure.ipynb  # 원자재별 Process Exposure 계산
│   └── 05_mbi_scoring.ipynb       # Material Bottleneck Index 계산 및 시각화
│
├── src
│   ├── build_master_df.py
│   ├── process_graph.py
│   ├── metrics_process.py
│   ├── metrics_supply.py
│   ├── metrics_exposure.py
│   └── mbi_model.py
│
├── figures
│   ├── process_bottleneck_bar.png
│   ├── process_risk_scatter.png
│   ├── supply_risk_bar.png
│   └── mbi_bar.png
│
└── README.md
```

---


