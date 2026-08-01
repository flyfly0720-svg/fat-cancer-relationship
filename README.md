# 비만·운동과 암: Cox 회귀분석 + YAP/TAZ 미분방정식 모델링

## 폴더 구성
- `app.py` — Streamlit 앱 본체 (4개 탭)
- `hn24_cox_data.csv` — 국민건강영양조사(HN24, 2024) 원자료에서 분석에 필요한
  변수만 정제한 경량 데이터 (BMI, 유산소 신체활동 실천율, 공복혈당, 암 진단 시점 등)
- `requirements.txt` — 필요 패키지 목록

## 실행 방법
같은 폴더 안에 세 파일(`app.py`, `hn24_cox_data.csv`, `requirements.txt`)을
반드시 함께 두어야 합니다. `app.py`가 `hn24_cox_data.csv`를 상대경로로 불러옵니다.

```bash
pip install -r requirements.txt
streamlit run app.py
```

실행 후 브라우저에서 자동으로 열리는 주소(보통 http://localhost:8501)로 접속하면 됩니다.

## 4개 탭 구성
1. **데이터 & Cox 회귀분석** — 실제 관측자료 기반, 비만·운동이 암 진단까지 걸리는
   시간에 미치는 영향을 위험비(HR)로 추정
2. **YAP/TAZ 미분방정식 모형** — Akt(비만)-AMPK(운동)-YAP/TAZ-암세포 동역학을
   상미분방정식으로 시뮬레이션, 모든 파라미터를 슬라이더로 조절 가능
3. **모형 진단(디버깅)** — ODE 수치해 이상 여부를 자동 진단 + Cox 모형과 ODE 모형의
   예측이 어긋나는 지점을 역추적
4. **민감도 분석** — ODE 파라미터가 가정치임을 보완하기 위해, 파라미터를 체계적으로
   바꾸며 결과의 안정성을 검증

## 데이터 재생성이 필요한 경우
`hn24_cox_data.csv`는 질병관리청 국민건강영양조사 원시자료(hn24_all.sas7bdat)에서
아래 로직으로 파생시킨 것입니다. 원본 SAS 파일을 다시 가공하려면:

```python
import pyreadstat, pandas as pd, numpy as np

cols = ["ID","sex","age","DC01_dg","DC01_ag","HE_BMI","pa_aerobic","HE_glu"]
df, meta = pyreadstat.read_sas7bdat("hn24_all.sas7bdat", usecols=cols)

df_adult = df[df["age"] >= 19].copy()
df_adult["cancer"] = df_adult["DC01_dg"].replace({8: np.nan, 9: np.nan})
df_adult["female"] = df_adult["sex"].map({1: 0, 2: 1})
df_adult["event"] = df_adult["cancer"]
df_adult["time"] = np.where(df_adult["cancer"] == 1, df_adult["DC01_ag"], df_adult["age"])

keep = ["ID","time","event","HE_BMI","pa_aerobic","HE_glu","age","female"]
df_final = df_adult[keep].dropna().reset_index(drop=True)
df_final.columns = ["ID","time","event","BMI","aerobic_pa","fasting_glucose","age","female"]
df_final.to_csv("hn24_cox_data.csv", index=False, encoding="utf-8-sig")
```

## 방법론적 한계 (보고서 작성 시 반드시 명시)
- 원자료는 **단면조사**이므로 BMI·운동·혈당은 2024년 현재 시점 측정값이고,
  암 진단 시점은 과거이므로 엄밀한 인과관계를 증명하지 않습니다.
- YAP/TAZ 미분방정식의 파라미터(α, β, γ, r, K, δ)는 문헌에서 추정된 값이 아니라
  모형의 정성적 동역학을 보여주기 위한 **가정치**입니다. 4번 탭의 민감도 분석
  결과를 반드시 함께 제시해 이 한계를 보완하시길 권장합니다.
