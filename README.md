# ZEISS DIC + Tensile All-in-One

file_list 촬영시각, ZEISS DIC raw distance, tensile 시험 데이터를 하나로 병합하여 stress-strain 곡선과 처리 과정 전체가 담긴 Excel 워크북을 생성하는 도구입니다.

서버나 설치 과정 없이 정적 HTML 파일 하나로 동작하며, 업로드한 파일은 모두 브라우저 안에서만 처리됩니다. 외부 서버로 전송되지 않습니다.

## 실행 방법

**온라인**

GitHub Pages로 배포하면 다음 주소에서 바로 사용할 수 있습니다.

```
https://kmyune.github.io/tensile/
```

**로컬**

인터넷 연결 없이 쓰려면 HTML 파일을 PC에 저장한 뒤 더블클릭해서 Chrome 등으로 직접 엽니다. 단, 라이브러리를 CDN에서 불러오므로(xlsx.js, Plotly.js) 파일 읽기와 계산은 오프라인에서도 되지만 그래프 표시와 Excel 저장 기능은 인터넷 연결이 필요합니다.

ChatGPT 등 임베디드 미리보기(iframe)에서 열면 Excel 다운로드가 막힐 수 있다는 경고가 화면 하단에 자동으로 표시됩니다. 이 경우 HTML 파일을 PC에 저장한 뒤 Chrome에서 직접 열어야 합니다.

## 준비할 파일 3종

| 항목 | 내용 | 형식 |
|---|---|---|
| file_list | 사진 촬영시각이 들어있는 파일명 목록, 또는 촬영시각/경과시간 컬럼 | CSV / XLSX / XLS |
| ZEISS raw data | 사진번호 + LXY distance(mm) | CSV / XLSX / XLS |
| Tensile raw data | TIME(Second) + ULM-T5(N), displacement(mm)는 선택 | CSV / XLSX / XLS |

세 파일 중 file_list와 ZEISS raw만 있으면 ①DIC 전처리가 가능하고, 세 파일이 모두 있어야 ③최종 병합(stress-strain)까지 진행됩니다.

## ① file_list 처리

**파일명 촬영시각 인식**

file_list의 각 행에서 파일명(첫 번째 열)을 읽어 다음 정규식으로 타임스탬프를 찾습니다.

```
(\d{8})_(\d{2})_(\d{2})_(\d{2})   →  YYYYMMDD_HH_MM_SS
```

예: `WIN_20260824_10_12_55_Pro.jpg` → 2026-08-24 10:12:55

- 월/일/시/분/초 범위를 벗어나거나(월 1-12, 일 1-31, 시 0-23, 분·초 0-59) 패턴 자체가 없는 행은 자동으로 제외됩니다. 헤더 행("file_list.csv" 등 패턴 없는 첫 행)도 이 규칙으로 자연스럽게 걸러집니다.
- 시트를 순서대로 검사해서 타임스탬프가 2개 이상 인식되는 첫 시트를 사용합니다.
- 인식에 성공하면 "촬영시간 간격" 표가 화면에 표시됩니다. 컬럼은 파일명, 촬영시각, 이전 사진 대비 경과(초), 첫 사진 기준 누적 경과(초)이며, 초 단위는 반올림됩니다. 이 표는 `(file_list 이름)_elapsed.csv`로 저장할 수 있습니다.
- 이 표는 참고/기록용이며, 아래 시간 매칭 로직에서 직접 사용되지는 않습니다.

**DIC 시간 매칭용 시간 컬럼 탐색**

ZEISS raw에서 읽은 point 개수(N)에 맞춰 file_list의 각 시트에서 다음 순서로 시간축을 찾습니다.

1. **경과시간 컬럼 우선**: 어떤 열에 숫자값이 정확히 N개 있고, 첫 값이 0이며, 단조증가(monotonic)하면 그 열을 그대로 사용합니다.
2. **시:분:초 3열 조합**: 조건 1을 만족하는 열이 없으면, 연속된 세 열에서 정수 시(0-23)/분(0-59)/초(0-59) 조합을 찾습니다. 조건을 만족하는 행이 N개 이상이면 마지막 N개 행을 사용합니다. 자정을 넘어가 시각이 역행하면 하루(86400초)를 더해 자동으로 이어 붙이고, 최종적으로 단조증가해야 채택됩니다. 시간은 첫 값을 0으로 재조정합니다.

두 방법 모두 실패하면 오류로 중단됩니다. 사용된 방법(열 번호, "elapsed-time column" 또는 "H:M:S columns")은 전처리 완료 메시지와 저장되는 Excel의 Method 시트에 기록됩니다.

## ② ZEISS raw 처리 및 smoothing

**LXY distance 컬럼 자동 인식**

- 시트 상위 8행에서 "LXY"가 포함된 행을 헤더 행으로 우선 채택합니다.
- 헤더 중 이름에 "lxy"와 "mm"이 모두 포함된 열을 최우선으로 선택하고, 없으면 "lxy"만 포함된 열을 선택합니다.
- 그래도 못 찾으면 폴백으로, 값 5개 이상인 숫자열 중 사진번호처럼 1씩 증가하는 프레임 카운터(연속 차이의 90% 이상이 1에 가까움)를 감점 처리하고 나머지 중 점수가 가장 높은 열을 사용합니다.
- 값이 없는 행은 정렬을 유지한 채 null로 남겨 두어 시간축과의 위치가 어긋나지 않게 합니다.

**Baseline**

raw distance 중 첫 번째 유효값(null이 아닌 값)을 raw baseline으로 사용합니다. Raw displacement = raw distance − raw baseline입니다.

**Smoothing (기본 켜짐, 끌 수 있음)**

ZEISS의 **absolute LXY distance 자체를 먼저 smoothing**한 뒤, smoothing된 첫 유효 거리값을 새 baseline으로 잡고 displacement를 계산합니다(원본이 아니라 smoothing된 값 기준). Raw distance/Raw displacement도 결과 파일에 그대로 함께 보존됩니다.

- 방식: 국소 다항 회귀(local polynomial regression, Savitzky-Golay류)로, 각 점 주변의 지정한 개수(window)만큼 유효 데이터를 모아 최소자승법으로 다항식을 맞추고, 그 다항식이 예측한 해당 시점 값으로 치환합니다.
- **Window**: 5 / 7(기본) / 9 / 11 포인트
- **Polynomial order**: 1차(선형) / 2차(quadratic, 기본값)
- window에 필요한 최소 포인트 수(order+1)를 채우지 못하는 경계 구간은 회귀를 적용하지 않고 원본값을 그대로 둡니다.
- smoothing을 끄면 raw distance를 그대로 사용하고 baseline도 raw baseline과 동일하게 계산됩니다.

**1초 fitting**

file_list 시간축(정수초가 아닐 수 있음)을 정수초 격자로 변환하는 단계입니다.

- 범위는 `ceil(첫 시간)`부터 `floor(마지막 시간)`까지, 1초 간격 정수입니다.
- 각 정수초에 대해 원래 시간축에 정확히 그 값이 있으면 "Actual_Data"로 표시하고, 없으면 앞뒤 유효한 smoothed displacement 두 점 사이를 **선형 보간(piecewise linear interpolation)** 하여 `fitted_mm`을 계산합니다. 구간을 벗어나는 시각은 보간하지 않습니다.

## ③ Tensile 병합 + Stress-Strain

**Tensile 원본 파싱**

각 시트 상위 35행에서 헤더를 찾습니다.

- 시간 열: 헤더에 "time"과 "second"가 모두 포함되거나, "second" 또는 "time s" 단독 표기
- 힘 열: 헤더에 "ulm"과 "t5"가 모두 포함
- (선택) displacement 열: 헤더에 "displacement"와 "mm"이 모두 포함

시간·힘 열을 모두 찾은 첫 헤더 행을 채택하고, 이후 행에서 시간과 힘이 둘 다 숫자인 행만 데이터로 모아 시간순 정렬합니다. displacement 열이 없으면 해당 값은 null로 둡니다.

**병합 로직**

1. DIC 1초 fitting 구간(t0~t1)과 tensile 시간 범위의 교집합(overlap)을 구합니다. 겹치는 구간이 없으면 오류입니다.
2. tensile의 각 시간점에서 DIC fitted displacement를 선형 보간으로 구합니다(위 1초 fitting 값들 사이를 다시 보간).
3. **DIC 최종 zero 기준**을 두 가지 중 선택합니다.
   - 첫 tensile 시점 기준(기본값): overlap 구간의 첫 tensile 시각에서의 DIC displacement를 0으로 재조정
   - 첫 사진 기준: 재조정 없이 DIC displacement(= smoothed distance − smoothed baseline) 값을 그대로 사용

**계산식**

```
Strain (%)  = DIC ΔDisplacement(mm) / Gauge length(mm) × 100
Stress (MPa) = ULM-T5(N) / Area(mm²)
```

Gauge length와 Area는 화면에서 직접 입력합니다(기본값 12.46 mm, 9 mm²로 채워져 있으며 시편에 맞게 수정 필요).

**출력 컬럼(최종 병합 테이블)**

TIME - Second, Tensile Displacement - mm, ULM-T5 - N, DIC fitted_mm, DIC ΔDisplacement - mm, Strain (%), Stress (MPa)

## 출력 파일

**Fitting data Excel 저장** (`(ZEISS 파일명)_Fitting_data.xlsx`)

DIC 전처리만 실행한 상태에서도 저장 가능합니다.

- `Fitting_Source`: Actual time, Actual length(smoothed), ZEISS raw/smoothed distance, Raw/Smoothed displacement
- `Processed_1s`: 1초 격자의 second, Actual_Data(실측치 있는 시점만), fitted_mm
- `Method`: 사용된 시트/열, 시간 인식 방식, baseline, smoothing 방식·window·order 등 처리 이력

**최종 Excel 다운로드** (`(Tensile 파일명)_ZEISS_DIC_final.xlsx`)

- `Final_Merged`: 병합된 최종 테이블
- `DIC_Preprocessing`: DIC 전처리 전체 행(raw/smoothed distance, displacement)
- `DIC_1s_Fitting`: 1초 fitting 결과
- `Tensile_Raw`: tensile 원본(TIME, Displacement, ULM-T5)
- `Method`: gauge length, area, zero 기준, smoothing 설정, 사용된 공식 등 처리 이력 전체

**Stress-Strain PNG**: 1600×900px, scale 2배로 저장됩니다.

## 참고 사항

- 다운로드할 때마다 저장 위치를 직접 고르려면 Chrome 설정 → 다운로드 → "다운로드 전에 각 파일의 저장 위치 확인"을 켜야 합니다.
- CSV는 UTF-8로 읽습니다.
- Excel/CSV 어느 파일이든 시트 여러 개가 있으면 규칙에 맞는 조건을 만족하는 첫 시트를 자동으로 선택합니다. 파일 내 시트 구성이나 열 위치가 바뀌어도 이름 기반 자동 인식이 우선 적용됩니다.
