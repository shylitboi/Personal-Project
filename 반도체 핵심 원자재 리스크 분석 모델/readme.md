
# FAB 병목 기반 반도체 핵심 원자재 리스크 모델링

*Fab Shutdown Risk Assessment via Structural Process Vulnerability and Supply Chain Concentration*

### PPT

  https://www.canva.com/design/DAG5yD5Fy9I/KzA_zkTA0-J8Nm_wqjMD0A/edit?utm_content=DAG5yD5Fy9I&utm_campaign=designshare&utm_medium=link2&utm_source=sharebutton

---
## 1. 개요

본 연구는 반도체 제조(Fab) 공정의 내부 구조적 병목과 글로벌 공급망의 외부 리스크를 통합하여,
Fab 가동 중단(Fab shutdown)을 야기할 수 있는 핵심 원자재(chokepoint materials)를 정량적으로 식별하는 것을 목적으로 한다.

Fab 생산 네트워크는 수많은 공정 단계와 원자재가 유향 그래프로 연결된 복잡계 시스템이며, 특정 공정 혹은 특정 소재의 중단은 전체 공정 흐름의 연쇄적인 마비를 초래할 수 있다.
따라서 공정 구조와 공급망 구조를 동시에 고려한 통합 분석이 필요하다.

---

## 2. 연구 배경

기존 공급망 위험 분석은 국가·기업 편중도(HHI 등)에 집중하여 **Fab 내부 구조적 취약성**을 반영하지 못하는 한계가 있다.
반대로 Fab 공정 병목 분석은 **외부 지정학·정책 충격에 대한 소재별 공급 위험**을 설명하지 못한다.

본 연구는 이 두 한계를 해결하기 위해 다음의 요소를 통합하였다.

1. 공정 병목을 반영하는 **Structural Vulnerability**
2. 국가 편중도를 반영하는 **Supply Chain Risk**
3. 병목 공정에 대한 소재의 의존도를 나타내는 **Process Exposure**

이 세 요소를 조합하여 **Material Bottleneck Index(MBI)** 를 정의하고, shutdown 위험이 높은 원자재를 계량적으로 도출한다.

---

## 3. 데이터셋

### 3.1 CSET Advanced Semiconductor Supply Chain Dataset (2025)
https://github.com/georgetown-cset/eto-chip-explorer/tree/main/data


| 파일명             | 내용               |
| --------------- | ---------------- |
| `inputs.csv`    | 공정·재료·장비 목록 및 분류 |
| `providers.csv` | 공급 국가·기업 정보      |
| `provision.csv` | 국가별 공급 비중 데이터    |
| `sequence.csv`  | Fab 공정 간 순차 흐름   |
| `stages.csv`    | 공정 단계(stage) 정보  |

Fab 구조를 구성하기 위한 전역적 네트워크 데이터를 포함한다.

### 3.2 UN Comtrade HS Code 데이터

소재별 국가 수출 데이터를 활용하여 공급망 집중도(HHI)를 산출하였다.

---

## 4. 전처리 절차

### 4.1 공급 비중 정제(Provision Cleaning)

* 최신 연도 데이터만 선택
* 모호한 공급자(`"Various"`) 제거
* 중복 국가 기록 정제

### 4.2 카테고리 및 공정 스테이지 분류

* 재료(materials) / 공정(processes) / 장비(tools) 자동 분류
* 공정(stage)을 Lithography, Etch, CMP, Deposition 등으로 매핑

### 4.3 Directed Process Graph 생성

`sequence.csv`를 이용하여 다음과 같은 형태의 유향 그래프 구축:

```
(input_id) → (goes_into_id)
```

### 4.4 master_df 구성

분석 통합 테이블에는 다음 항목을 포함한다.

* `input_id`, `input_name`
* `category`, `stage_name`
* 국가별 공급 비중(`share_USA`, `share_JPN` 등)
* 시장 규모 정보

---

## 5. Fab 공정 구조 취약성 분석

Fab 공정 병목을 정량화하기 위해 네 가지 지표를 사용하였다.

* Betweenness centrality
* PageRank
* Out-degree
* Downstream reach

### 5.1 공정 병목 지수 (Process Bottleneck Index)

각 공정 $p$에 대해 다음과 같이 정의한다.

$$
B_p = \frac{1}{4}
\left(
\tilde{BC}_p +
\tilde{PR}_p +
\tilde{OD}_p +
\tilde{DR}_p
\right)
$$

여기서 각 항목은 min–max 정규화된 값이다.

### 5.2 주요 병목 공정

분석 결과, Fab 구조에서 높은 병목도를 보이는 공정은 다음과 같다.

* Ion implantation
* Etch & clean
* CMP
* Photolithography

이들 공정은 downstream 공정에 대한 영향이 매우 커, 공정 단위의 chokepoint로 기능한다.

---

## 6. 원자재 위험도 모델링

원자재 위험도는 **공급망 리스크**와 **Fab 구조적 노출도(Process Exposure)** 를 결합하여 산출한다.

---

### 6.1 공급망 리스크(Supply Risk)

국가별 공급 비중을 이용해 다음과 같은 HHI 지수를 계산한다.

$$
HHI_m = \sum_{i=1}^{N} share_i^2
$$

정규화된 Supply Risk Score는 다음과 같다.

$$
S_m =
\frac{HHI_m - \min(HHI)}
{\max(HHI) - \min(HHI)}
$$

높을수록 특정 국가·기업에 대한 공급 편중이 심함을 의미한다.

---

### 6.2 공정 노출도(Process Exposure Score)

소재 $m$이 투입되는 공정 집합을 $\mathcal{P}(m)$이라 하면:

* $\overline{B}_m$ : 평균 병목 지수
* $B^{\max}_m$ : 최대 병목 지수
* $\overline{DR}_m$ : 평균 downstream reach

Process Exposure Score는 다음과 같다.

$$
PE_m = \frac{1}{3}
\left(
\overline{B}_m +
B^{\max}_m +
\overline{DR}_m
\right)
$$

이는 Fab 구조 내에서 해당 소재가 어느 정도의 병목성 공정에 의존하는지를 나타낸다.

---

### 6.3 소재 병목 지수(Material Bottleneck Index, MBI)

최종 리스크 스코어는 다음과 같이 정의한다.

$$
MBI_m = 0.5 \cdot S_m + 0.5 \cdot PE_m
$$

MBI가 높을수록 **공급 충격에 취약하며 동시에 구조적 병목 공정에 깊이 연결된 소재**임을 의미한다.

---

## 7. 통합 결과

분석 결과 MBI 상위 위험 소재는 다음과 같다.

1. **Deposition materials**
2. **Photoresists**
3. **Photomasks**
4. **CMP materials**
5. **Wafer**

해석:

* Deposition materials는 초기 layer 형성 공정에 폭넓게 활용되며 구조적 영향이 크다.
* Photoresist는 Lithography 공정의 필수 요소로 대체성이 낮고 공급 집중도도 높다.
* Wafer는 공급 집중도가 상대적으로 낮아도, Fab 전체의 기반 구조를 형성하므로 구조적 리스크가 매우 높다.

---

## 8. Fab 공정–소재 리스크 결합 해석

분석을 종합하면 다음의 결론을 도출할 수 있다.

1. 공정 병목도가 높은 공정(Ion implantation, Etch, CMP 등)에 연계된 소재는 내재적 고리스크 구조를 가진다.
2. 공급망 집중도가 높은 소재(Wet chemicals, CMP slurry)는 외부 충격에 즉각적으로 취약하다.
3. Wafer처럼 공급 집중도는 낮더라도 Fab 구조 전체를 지탱하는 소재는 “전 공정 기반 리스크”를 형성한다.
4. 따라서 Fab shutdown 위험은 단일 지표로 설명할 수 없으며, 구조적 노출도와 공급망 리스크의 상호작용으로 결정된다.

---

## 9. GNN 및 Temporal Analysis 확장

현재 분석은 정적 통계 기반이지만, 향후 연구에서는 GNN 및 TGN을 적용할 수 있다.

### 9.1 이종 그래프 기반 GNN

노드 유형

* Material, Process, Provider, Country

엣지 유형

* Material → Process
* Process → Process
* Provider → Material
* Country → Provider

GNN을 이용하여 소재 또는 공정 노드의 구조적 중요도 $w_i$를 학습할 수 있으며,
이는 다음 질문에 답하는 데 유용하다.

* 특정 노드가 제거될 때 네트워크는 얼마나 복원 가능한가?
* 공급망 변화가 Fab 구조에 미치는 영향을 어떻게 예측할 수 있는가?

### 9.2 Temporal Graph Network(TGN) 확장

* 연도별 HHI 변화 반영
* 공급 규제·전쟁 등의 이벤트 삽입
* 리스크 전파 경로의 시간적 역학 모델링

---

## 10. 연구 의의

1. Fab 공정 구조와 외부 공급망을 통합적으로 고려한 리스크 평가 모델을 제시하였다.
2. MBI를 통해 Fab shutdown을 유발할 수 있는 소재를 계량적으로 식별하였다.
3. 종래의 공급망 분석(HHI 기반)에서 포착되지 않는 구조적 취약성을 반영함으로써 정책·투자 의사결정에 기여할 수 있다.
4. 향후 GNN·TGN 기반 확장성을 명확히 제시하여 연구적 응용 가능성을 확보하였다.

---

## 11. 리포지토리 구조

```bash
.
├── data
│   ├── raw/
│   └── external/
├── notebooks
│   ├── 01_preprocessing.ipynb
│   ├── 02_process_bottleneck.ipynb
│   ├── 03_supply_risk.ipynb
│   ├── 04_process_exposure.ipynb
│   └── 05_mbi_scoring.ipynb
├── src
│   ├── build_master_df.py
│   ├── process_graph.py
│   ├── metrics_process.py
│   ├── metrics_supply.py
│   ├── metrics_exposure.py
│   └── mbi_model.py
└── README.md
```

---

