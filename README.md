[README.md](https://github.com/user-attachments/files/31404665/README.md)
# ZEISS DIC + Tensile All-in-One

file_list 촬영시간 → ZEISS raw distance smoothing → zeroing → 1초 fitting →
tensile (ULM-T5, N) 병합 → strain / stress 계산까지 브라우저에서 바로 처리하는 도구입니다.

서버나 설치 없이 정적 HTML 하나로 동작하며, 업로드한 파일은 모두 브라우저 안에서만
처리됩니다 (외부로 전송되지 않음).

## 온라인에서 바로 쓰기

GitHub Pages로 배포하면 아래 주소에서 바로 사용할 수 있습니다.

```
https://[사용자명].github.io/[저장소이름]/
```

## 사용법

1. **file_list** — 사진 촬영시간이 들어있는 CSV/Excel을 넣습니다.
   파일명이 `..._YYYYMMDD_HH_MM_SS_...` 형식(예: `WIN_20260824_10_12_55_Pro.jpg`)이면
   자동으로 촬영 간격과 누적 경과시간을 표로 보여주고, CSV로 저장할 수 있습니다.
2. **ZEISS raw data** — 사진번호 + LXY distance(mm) 데이터를 넣습니다.
3. **Tensile raw data** — TIME(Second) + ULM-T5(N) 데이터를 넣습니다.
4. 세 파일을 모두 넣으면 DIC 전처리 → smoothing → tensile 병합까지 자동 진행됩니다.

## 로컬에서 열기

인터넷 연결이 없다면 `index.html`을 더블클릭해서 브라우저로 바로 열어도 동일하게 동작합니다.
단, Plotly와 SheetJS(xlsx) 라이브러리를 CDN에서 불러오므로 그래프와 xlsx 저장 기능은
인터넷 연결이 필요합니다.
