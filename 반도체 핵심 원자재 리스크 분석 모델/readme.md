# Semiconductor Fab Shutdown Risk Modeling

> **Integrating Process Networks with Global Supply Chain Risks using Graph Neural Networks (GNN)**

## 1\. Project Overview

### 1.1 Background

반도체 제조(Fab) 공정은 수백 개의 단위 공정과 소재가 정밀하게 연결된 복잡계(Complex Network)입니다. 이 네트워크는 단 하나의 소재 공급 중단이나 공정 병목만으로도 전체 생산 라인이 멈추는(Shutdown) 특성을 가집니다.

최근 지정학적 갈등, 수출 규제, 특정 국가 편중 생산 등 외부 환경의 불확실성이 커지며 **Supply Chain Shock**가 빈번해지고 있습니다. 그러나 기존 연구는 '외부 공급망(Supply Chain)'과 '내부 공정 구조(Fab Internal Logic)'를 분리하여 분석한다는 한계가 있었습니다.

### 1.2 Objective

본 프로젝트는 Fab 내부의 \*\*공정 네트워크(Process Sequence)\*\*와 외부의 \*\*공급망 리스크(Material Supply Chain)\*\*를 통합 모델링하여, **Fab Shutdown을 유발하는 핵심 소재(Chokepoint Material)를 정량적으로 식별**하는 것을 목표로 합니다.

-----

## 2\. Methodology Framework

본 연구는 데이터 전처리부터 딥러닝 기반의 리스크 예측까지 총 4단계의 파이프라인으로 구성됩니다.

### Phase 1: Data Construction & Graph Modeling

  * **Dataset:** CSET Advanced Semiconductor Supply Chain Dataset (2025), UN Comtrade API.
  * **Structure:** Fab 내의 모든 요소를 \*\*이종 그래프(Heterogeneous Graph)\*\*로 구축.
      * **Nodes:** Materials (10 types), Processes (7 stages).
      * **Edges:** `Material -> Process` (Input), `Process -> Process` (Sequence), `Process -> Material` (Feedback).

### Phase 2: Structural Vulnerability Analysis ($c_i$)

네트워크 이론을 적용하여 Fab 공정 구조상의 물리적 취약성을 산출합니다.

  * **Metrics:** Betweenness Centrality, PageRank, Out-degree, Downstream Reach.
  * **Process Bottleneck Index:** 공정의 구조적 중요도를 0\~1로 정량화.
  * **Shock Simulation:** 특정 노드 제거 시 Downstream 공정의 붕괴율을 측정하여 **Static Vulnerability ($c_i$)** 도출.

### Phase 3: Supply Risk Analysis ($s_i$)

  * **UN Comtrade Data:** HS Code 기반 국가별 수출 데이터 수집.
  * **HHI (Herfindahl-Hirschman Index):** 특정 국가/기업에 대한 공급 집중도를 계산하여 **Supply Risk ($s_i$)** 도출.

### Phase 4: GNN-based Importance Learning ($w_i$)

단순한 통계적 수치를 넘어, 그래프 구조 내의 잠재적 중요도를 학습하기 위해 \*\*Graph Neural Network (GNN)\*\*를 도입했습니다.

  * **Model:** Heterogeneous GNN (Link Prediction Task).
  * **Objective:** "이 소재가 네트워크에서 사라질 경우 전체 구조를 복원할 수 있는가?"를 학습하여 **Production Importance ($w_i$)** 추론.
  * **Advantage:** 시장 점유율 데이터가 부재한 소재에 대해서도 관계(Relation) 기반의 중요도 산출 가능.

-----

## 3\. Integrated Risk Scoring Model

Fab Shutdown을 유발하는 최종 리스크 스코어는 다음 세 가지 차원의 가중합으로 정의됩니다.

$$\text{Risk}_i = \alpha \cdot w_i + \beta \cdot c_i + \gamma \cdot s_i$$

| Component | Variable | Description | Weight |
| :--- | :---: | :--- | :---: |
| **Production Importance** | $w_i$ | GNN이 학습한 Fab 생산 유지의 필수성 (Uniqueness) | 0.4 |
| **Structural Vulnerability** | $c_i$ | 물리적 공정 병목 및 Shock Simulation 결과 | 0.3 |
| **Supply Risk** | $s_i$ | 국가/기업 편중도에 따른 외부 공급 충격 위험 | 0.3 |

-----

## 4\. Key Results & Insights

### 4.1 Material Risk Ranking

분석 결과, Fab을 멈추게 할 가능성이 가장 높은 상위 5개 소재는 다음과 같습니다.

| Rank | Material | $w_i$ (Imp.) | $c_i$ (Vuln.) | $s_i$ (Supply) | **Final Risk** | Interpretation |
| :---: | :--- | :---: | :---: | :---: | :---: | :--- |
| **1** | **Wafer** | **0.444** | **0.375** | 0.432 | **0.735** | **Global Base Risk:** 모든 공정의 시작점이자 구조적 절대 병목. |
| **2** | **CMP Materials** | 0.440 | 0.153 | **1.460** | **0.631** | **Process/Supply Double Choke:** 공정 중요도와 공급 집중도가 모두 높음. |
| **3** | **Wet Chemicals** | 0.433 | 0.253 | **1.900** | **0.615** | **Supply Driven Risk:** 구조적 중요도는 중간이나, 공급 편중이 극심함. |
| **4** | **Deposition Mat.** | 0.438 | 0.233 | 0.334 | **0.451** | **Broad Coverage:** 광범위한 공정 영향력을 가진 전공정 허브. |
| **5** | **Photoresists** | 0.430 | 0.213 | 1.117 | **0.349** | **Litho-Specific Bottleneck:** Lithography 공정의 독점적 지위로 인한 고위험. |

### 4.2 Risk Propagation Mechanism

본 연구는 리스크가 독립적으로 존재하지 않고 \*\*전파(Propagate)\*\*됨을 규명했습니다.

1.  **Source:** Wafer, PR, CMP 등 핵심 소재에서 공급 충격 발생.
2.  **Primary Bottleneck:** Lithography, CMP 등 1차 병목 공정이 즉각 중단.
3.  **Secondary Diffusion:** Etch, Implant, Assembly 등 후속 공정으로 도미노 효과 확산.

-----

## 5\. Significance & Future Work

### 5.1 Research Significance

  * **Integrated Approach:** 최초로 Fab 공정 시퀀스와 글로벌 공급망 데이터를 결합한 통합 그래프 모델 구축.
  * **Data-Driven Discovery:** HHI 등 정형 데이터가 없는 소재에 대해서도 GNN을 통해 구조적 중요도를 산출하는 방법론 제시.
  * **Quantifiable Metric:** 직관적인 Risk Score를 통해 공급망 다변화 및 국산화 우선순위 결정에 실질적 가이드라인 제공.

### 5.2 Future Research: Temporal Dynamics

현재의 모델은 정적(Static) 리스크를 측정합니다. 향후 연구는 \*\*Temporal Graph Neural Networks (TGN)\*\*을 도입하여 다음을 수행할 예정입니다.

  * **Trend Prediction:** 시간에 따라 중요도가 상승하는 소재의 조기 탐지.
  * **Dynamic Simulation:** 지정학적 이벤트 발생 시 실시간 리스크 전파 경로 예측.

-----

## 6\. Project Structure

```bash
├── data
│   ├── raw                 # CSET dataset, UN Comtrade raw data
│   └── processed           # Normalized master_df, Graph structures (HeteroData)
├── notebooks
│   ├── 01_Data_Preprocessing.ipynb    # Market size norm, Categorization
│   ├── 02_Structural_Analysis.ipynb   # NetworkX based Bottleneck/Shock Sim
│   ├── 03_Supply_Risk_Analysis.ipynb  # UN Comtrade API & HHI Calc
│   ├── 04_GNN_Modeling.ipynb          # Heterogeneous GNN for Link Prediction
│   └── 05_Risk_Scoring_Vis.ipynb      # Final Scoring & Visualization
├── src
│   ├── graph_utils.py      # Graph construction helper functions
│   └── models.py           # PyTorch Geometric GNN model definitions
└── README.md
```

## 7\. Tech Stack

  * **Language:** Python 3.10+
  * **Graph Analysis:** NetworkX, PyTorch Geometric (PyG)
  * **Data Processing:** Pandas, NumPy
  * **Visualization:** Matplotlib, Seaborn, Plotly
