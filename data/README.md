# 데이터

## 원본 데이터

이 저장소는 **CAERS 원본 파일을 포함하지 않습니다.** 아래 안내에 따라 직접 내려받아 이 디렉터리에 두면 노트북이 그대로 실행됩니다.

| 항목 | 값 |
|---|---|
| 이름 | CAERS — CFSAN Adverse Event Reporting System |
| 제공 | 미국 식품의약국(FDA) |
| 파일명 | `CAERS_ASCII_2004_2017Q2.csv` |
| 크기 | 약 19MB |
| 행 수 | 90,786 |
| 컬럼 수 | 12 |
| 인코딩 | UTF-8 |
| 집계 기간 | 2004-01-01 ~ 2017-06-30 (2017년은 Q2까지) |

### 내려받는 방법

FDA가 CAERS 데이터를 공개하는 페이지에서 해당 분기 파일을 받습니다.

> CFSAN Adverse Event Reporting System (CAERS)
> https://www.fda.gov/food/compliance-enforcement-food/cfsan-adverse-event-reporting-system-caers

FDA가 배포 경로와 파일 구성을 갱신하는 경우가 있어, 위 페이지에서 `2004 - 2017Q2` 구간 파일을 확인하시기 바랍니다. 컬럼 구성은 [`sample_schema.csv`](sample_schema.csv)와 대조해 확인할 수 있습니다.

### 경로 지정

기본값은 이 디렉터리입니다.

```
data/CAERS_ASCII_2004_2017Q2.csv
```

다른 위치에 두려면 노트북 첫 셀이나 셸에서 지정합니다.

```python
os.environ["ER_CAERS_CSV"] = "/path/to/CAERS_ASCII_2004_2017Q2.csv"
```

```bash
export ER_CAERS_CSV=/path/to/CAERS_ASCII_2004_2017Q2.csv
```

상대경로를 쓰면 프로젝트 루트를 기준으로 해석합니다. `.gitignore` 가 `data/*.csv` 를 제외하므로(단 `sample_schema.csv` 는 예외) 실수로 커밋될 위험은 없습니다.

---

## 스키마

[`sample_schema.csv`](sample_schema.csv) 에 컬럼별 자료형·결측 수·고유값 수·예시값을 정리했습니다.

| 컬럼 | 의미 | 결측 | 고유값 |
|---|---|---:|---:|
| `RA_Report #` | 신고 번호 | 0 | 64,517 |
| `RA_CAERS Created Date` | 신고 접수일 | 0 | 4,020 |
| `AEC_Event Start Date` | 부작용 발생일 | 37,133 | 5,174 |
| `PRI_Product Role` | 제품 역할 (Suspect / Concomitant) | 0 | 2 |
| `PRI_Reported Brand/Product Name` | 신고된 브랜드·제품명 | 0 | 45,685 |
| `PRI_FDA Industry Code` | 산업군 코드 | 0 | 44 |
| `PRI_FDA Industry Name` | 산업군 이름 | 0 | 41 |
| `CI_Age at Adverse Event` | 발생 시점 연령 | 37,860 | 115 |
| `CI_Age Unit` | 연령 단위 | 0 | 6 |
| `CI_Gender` | 성별 | 0 | 5 |
| `AEC_One Row Outcomes` | 부작용 결과 (쉼표 구분 다중값) | 0 | 298 |
| `SYM_One Row Coded Symptoms` | 증상 코드 (쉼표 구분 다중값) | 5 | 33,546 |

`RA_Report #` 의 예시값은 마스킹했습니다. 신고 번호 자체는 개인 식별 정보가 아니지만 스키마 문서에 실제 값을 남길 이유가 없습니다.

---

## 데이터를 읽을 때 주의할 점

이 저장소의 분석에서 확인한 구조적 특성입니다. 자세한 내용은 [`docs/reproduction_audit.md`](../docs/reproduction_audit.md) 를 참고하세요.

### 1. 행과 신고는 다릅니다

전체 90,786행이지만 고유 신고 번호는 **64,517건**입니다. 한 신고에 여러 제품이 등록되면 그 수만큼 행이 늘어납니다.

- 다중 행 신고: 10,817건 (37,086행)
- 다중 행 신고 중 제품명이 2종 이상인 비율: **99.7%**
- 다중 행 신고 중 **부작용 결과가 2종 이상인 비율: 0.0%**

**`AEC_One Row Outcomes` 는 신고 단위로 부여되어 제품 행마다 복제됩니다.** 따라서 행 단위 집계는 제품을 여러 개 신고한 사례를 그만큼 중복 반영합니다.

### 2. 부작용 결과는 다중값입니다

한 행이 여러 결과를 동시에 가집니다. 쉼표로 구분된 문자열이며, 실제 등장하는 항목은 12종입니다.

```
OTHER SERIOUS (IMPORTANT MEDICAL EVENTS)   36,837
NON-SERIOUS INJURIES/ ILLNESS              27,835
VISITED A HEALTH CARE PROVIDER             20,158
HOSPITALIZATION                            16,908
VISITED AN ER                              12,890
LIFE THREATENING                            5,274
SERIOUS INJURIES/ ILLNESS                   3,606
REQ. INTERVENTION TO PRVNT PERM. IMPRMNT.   2,944
DISABILITY                                  2,890
DEATH                                       2,028
NONE                                          145
CONGENITAL ANOMALY                             77
```

항목별 보유율을 단순 합산하면 100%를 넘습니다. **비율의 분모로 사용할 수 없습니다.**

> 응급실 방문의 실제 표기는 `VISITED AN ER` 입니다. `EMERGENCY` 로 검색하면 0건입니다.

### 3. 연령은 단위가 섞여 있습니다

`CI_Age Unit` 이 `Year(s)` · `Month(s)` · `Week(s)` · `Day(s)` · `Decade(s)` · `Not Available` 로 나뉩니다. 연령 결측·미상이 37,880행(41.7%)이므로 **연령 관련 분석의 분모는 52,906행**입니다.

### 4. 제품명이 표준화돼 있지 않습니다

고유 제품명 45,685종. 같은 브랜드가 여러 표기로 흩어져 있습니다. 예를 들어 `HYDROXYCUT` 을 포함하는 제품명은 245종 1,300행이지만, 정확히 `HYDROXYCUT` 인 것은 171행뿐입니다.

### 5. 제품 역할을 구분해야 합니다

`PRI_Product Role` 이 `Suspect`(부작용 원인으로 지목된 제품, 74,558행)와 `Concomitant`(함께 섭취한 제품, 16,228행)로 나뉩니다. 특정 브랜드의 위험도를 말할 때 Concomitant를 포함하면 인과 오해를 부를 수 있습니다.

### 6. 자발신고 데이터입니다

신고 건수는 실제 발생 건수가 아닙니다. 신고할 동기가 있는 사람만 신고하며, 언론 보도 이후 신고가 늘어나는 보고 편향이 존재합니다. 브랜드별 판매량(노출량)도 반영되지 않습니다.

---

## 라이선스

CAERS는 FDA가 공개하는 미국 연방정부 자료입니다. 원본에 개인 식별 정보는 포함돼 있지 않으며, 제품명 중 일부는 이미 `REDACTED` 로 처리돼 있습니다. 재배포 조건은 FDA 배포 페이지의 고지를 확인하시기 바랍니다.
