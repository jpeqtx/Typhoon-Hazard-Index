# THI Specification
## Typhoon Hazard Index

**Version:** 1.0 Draft
**Author:** YNGMT
**Date:** 2026-07-04

---

# Abstract

本仕様書は、気象庁XML電文のみを用いてリアルタイムに算出可能な台風総合危険度指標「Typhoon Hazard Index（THI）」を定義する。

THIは従来の「勢力」を表す指標ではない。暴風・豪雨・移動速度・急速発達・暴風域拡大・警報情報を統合し、台風が社会へ与える潜在的危険度をMinimal / Minor / Moderate / Significant / Extremeの5段階で評価することを目的とする。

また本仕様は、説明可能AI・自己学習・統計的パラメータ最適化・将来予報・類似台風検索・不確実性評価を考慮した拡張可能アーキテクチャである。

---

# 1. Research Background

従来の台風評価は、中心気圧・最大風速・大型/超大型といった単一要素に依存している。しかし実際には、小型でも豪雨災害が発生する事例、暴風域は小さいが線状降水帯を誘発する事例、移動速度が遅く被害が拡大する事例、急速発達する事例など、単一要素では評価できないケースが多数存在する。

THIは「台風そのもの」ではなく「災害ポテンシャル」を評価する。

---

# 2. Design Philosophy

THIは以下を満たすことを目標とする。

1. 気象庁XMLのみでリアルタイム計算可能であること
2. 数学的に一貫したモデルであること
3. 統計学的に最適化可能であること
4. Explainable AIに対応すること
5. 将来的な自己学習に対応すること
6. 世界中の台風へ適用可能であること
7. 防災利用を最優先とすること

---

# 3. System Architecture

```text
                 +----------------------+
                 |   JMA XML Feed       |
                 +----------+-----------+
                            |
                            v
                 +----------------------+
                 | XML Parser           |
                 +----------+-----------+
                            |
                            v
                 +----------------------+
                 | Feature Generator    |
                 +----------+-----------+
                            |
                            v
                 +----------------------+
                 | Hazard Engine        |
                 +----------+-----------+
                            |
             +--------------+--------------+
             |                             |
             v                             v
        THI Engine                 Confidence Engine
             |                             |
             +--------------+--------------+
                            |
                            v
                  Classification Engine
                            |
                            v
       Minimal / Minor / Moderate / Significant / Extreme
```

---

# 4. Input Variables

THIで使用する説明変数ベクトルをXと定義する。

$$
X = [P_{min},\ V_{max},\ R_{25},\ R_{15},\ Rain_{24},\ V_{trans},\ \Delta P_{24},\ \Delta R_{25},\ WarningLevel]
$$

| 変数 | 定義 |
|---|---|
| $P_{min}$ | 最低中心気圧 [hPa] |
| $V_{max}$ | 最大風速 [m/s] |
| $R_{25}$ | 暴風域半径 [km] |
| $R_{15}$ | 強風域半径 [km] |
| $Rain_{24}$ | 24時間降水量 [mm] |
| $V_{trans}$ | 移動速度 [km/h] |
| $\Delta P_{24}$ | 24時間中心気圧変化 [hPa] |
| $\Delta R_{25}$ | 24時間暴風域変化 [km] |
| $WarningLevel$ | 警報指数 |

---

# 5. Hazard Functions

各入力変数は危険度関数$I_i(x)$へ変換される。一般式は次のとおりである。

$$
I_i(x) = \frac{1}{1+\exp\left(-d_i\dfrac{x-\mu_i}{\sigma_i}\right)}
$$

ここで、

- $mu$：危険度50%となる中心値
- $sigma$：立ち上がり幅
- $d$：危険方向（$+1$：値が大きいほど危険、$-1$：値が小さいほど危険）

各変数の危険方向$d$は以下のとおりである。

| 変数 | $d$ |
|---|---|
| 中心気圧 | $-1$ |
| 最大風速 | $+1$ |
| 降水量 | $+1$ |
| 移動速度 | $-1$ |
| 暴風域 | $+1$ |
| 強風域 | $+1$ |
| 急速発達 | $+1$ |
| 暴風域拡大 | $+1$ |
| 警報 | $+1$ |

これらの$\mu$、$\sigma$は固定値ではなく、過去台風データおよび実被害データから毎年再学習する。これによりTHIは経験則ではなく、統計学的に最適化された指標となる。

---

# 6. THI Integration Model

## 6.1 Overview

各危険度関数は単純加算を行わない。従来の重み付き線形和

$$
THI=\sum w_iI_i
$$

では、一要素のみ極端に危険な場合や、逆に複数要素が同時に危険な場合の評価が不十分となる。本研究では**幾何平均（Geometric Mean）**を採用する。

## 6.2 Base Hazard

危険度ベクトル

$$
\mathbf I= [I_1,I_2,\cdots,I_n]
$$

とすると、基礎危険度

$$
H = \left( \prod_{i=1}^{n} I_i \right)^{1/n}
$$

と定義する。

### 採用理由

幾何平均には

- 全要素の危険性を自然に統合できる
- どれか一つが極端に低い場合は全体も低下する
- 複数要素が同時に危険になると急激に増加する

という特徴がある。これは災害現象の複合性を表現するのに適している。

## 6.3 Maximum Hazard

一方、豪雨のみ極端あるいは暴風のみ極端など単一要素だけでも重大災害となるケースが存在する。そのため

$$
M = \max(I_i)
$$

を定義する。これはExtreme補正として使用する。

## 6.4 Final THI

最終危険度は

$$
THI = (1-\beta)H + \beta M
$$

と定義する。ここでβはExtreme補正係数であり

$$
0\le\beta\le1
$$

を満たす。

### βの決定

βは固定値ではない。過去台風データ、人的被害、物的被害、停電、浸水などを教師データとして毎年最適化する。

## 6.5 Normalization

THIは必ず

$$
0\le THI\le1
$$

となるようクリッピングを行う。

$$
THI = \min(1,\max(0,THI))
$$

---

# 7. Forecast Engine

## 7.1 Purpose

実況だけでは防災には不十分である。そのため将来予報も同じアルゴリズムで評価する。

## 7.2 Forecast Time

以下の時刻について独立にTHIを計算する。

- 実況
- 6時間後
- 12時間後
- 24時間後
- 48時間後
- 72時間後
- 96時間後
- 120時間後

## 7.3 Future Hazard

各時刻について

$$
THI(t)
$$

を求め、将来危険度を

$$
THI_{forecast} = \max_t THI(t)
$$

と定義する。

## 7.4 Forecast Trend

危険度変化率

$$
\frac{dTHI}{dt}
$$

を計算する。この値が大きいほど急速に危険度が増加していることを示す。

### Trend分類

Rapidly Increasing、Increasing、Stable、Weakening、Rapidly Weakening

---

# 8. Confidence Engine

## 8.1 Purpose

気象庁XMLでは取得できる情報量が電文によって異なる。欠損情報を考慮する必要がある。

## 8.2 Data Mask

各説明変数について

$$
M_i = \begin{cases} 1 & 利用可能\\ 0 & 欠損 \end{cases}
$$

と定義する。

## 8.3 Confidence

$$
Confidence = \frac {\sum w_iM_i} {\sum w_i}
$$

Confidenceは0〜1で表される。表示時には百分率へ変換する。例、99%、95%、82%

### Confidence Rank

| Confidence | Rank |
|------------|------|
|95%以上|Excellent|
|90〜95|Very High|
|80〜90|High|
|70〜80|Moderate|
|70未満|Low|

---

# 9. Data Quality Engine

Confidenceとは別に入力データ品質を評価する。Data Quality Index、(DQI)を

$$
DQI = \frac {\sum M_iw_i} {\sum w_i}
$$

と定義する。DQIはXML取得失敗、欠損、電文遅延などを考慮する。THI計算時にはDQIも同時に表示する。

例：THI 0.82（Extreme）、Confidence 96%、Data Quality 98%（Excellent）

---

# 10. Hazard Engine

## 10.1 Overview

Hazard Engine は THI の中核となる計算モジュールである。各気象要素を独立した危険度関数へ変換し、その後 THI Integration Model に渡す。各モジュールは独立して実装されるため、将来的な改良・追加・削除が容易である。

## 10.2 Pressure Hazard Module

### 入力

最低中心気圧

$$
P_{min}
$$

### 危険度関数

$$
I_P= \frac 1 {1+\exp \left( \frac{P-\mu_P} {\sigma_P} \right)}
$$

ここで

- $\mu_P$：学習された中心値
- $\sigma_P$：学習された傾き

### 特徴

- 気圧が低いほど危険
- 極端な超大型台風でも飽和しにくい
- 学習により地域特性へ適応可能

## 10.3 Wind Hazard Module

入力、最大風速

$$
V_{max}
$$

危険度

$$
I_V= \frac1 {1+\exp \left( -\frac {V-\mu_V} {\sigma_V} \right)}
$$

特徴

- 暴風被害
- 飛散物
- 構造物損傷

などを代表する。

## 10.4 Storm Radius Module

入力、暴風域半径

$$
R_{25}
$$

危険度

$$
I_B= \frac1 {1+\exp \left( -\frac {R_{25}-\mu_B} {\sigma_B} \right)}
$$

暴風域が広いほど被害範囲が増加する。

## 10.5 Gale Radius Module

入力、強風域半径

$$
R_{15}
$$

危険度

$$
I_G= \frac1 {1+\exp \left( -\frac {R_{15}-\mu_G} {\sigma_G} \right)}
$$

強風域は広域停電、交通障害、農業被害を反映する。

## 10.6 Rainfall Hazard Module

### 入力

24時間降水量

$$
Rain_{24}
$$

危険度

$$
I_R= \frac1 {1+\exp \left( -\frac {Rain-\mu_R} {\sigma_R} \right)}
$$

### 特徴

本モデルでは豪雨を暴風と同等に評価する。暴風域が小さくても豪雨災害が発生する事例へ対応する。

## 10.7 Translation Speed Module

入力、移動速度

$$
V_{trans}
$$

危険度

$$
I_T= \frac1 {1+\exp \left( \frac {V_{trans}-\mu_T} {\sigma_T} \right)}
$$

特徴：移動速度が遅いほど同一地域への降水継続時間が増加する。そのため危険度は上昇する。

## 10.8 Rapid Intensification Module

24時間中心気圧変化

$$
\Delta P_{24}
$$

危険度

$$
I_D= \frac1 {1+\exp \left( -\frac {\Delta P-\mu_D} {\sigma_D} \right)}
$$

急速発達中の台風を高評価する。

## 10.9 Storm Expansion Module

入力、24時間暴風域拡大量

$$
\Delta R_{25}
$$

危険度

$$
I_E= \frac1 {1+\exp \left( -\frac {\Delta R-\mu_E} {\sigma_E} \right)}
$$

巨大化する台風を評価する。

## 10.10 Warning Module

気象庁発表情報を指数化する。

|情報|指数|
|----|----|
|なし|0.00|
|注意報|0.25|
|警報|0.50|
|土砂災害警戒情報|0.75|
|特別警報|1.00|

これを

$$
I_W
$$

と定義する。注意：このモジュールは日本国内でのみ有効である。海外海域では欠損値として扱い、Confidence に反映する。

## 10.11 Compound Hazard Module

災害は複数要素が同時発生することで急激に悪化する。そのため相互作用項を導入する。

### 豪雨 × 停滞

$$
C_1 = I_R \times I_T
$$

長時間降雨を表現する。

### 暴風 × 暴風域

$$
C_2 = I_V \times I_B
$$

広範囲暴風被害を表現する。

### 急速発達 × 豪雨

$$
C_3 = I_D \times I_R
$$

発達直後の豪雨リスクを強調する。

### 急速発達 × 暴風

$$
C_4 = I_D \times I_V
$$

短時間で危険度が増す事例へ対応する。

### 豪雨 × 警報

$$
C_5 = I_R \times I_W
$$

実際の防災上の危険度を補正する。

## 10.12 Composite Hazard Score

複合危険度

$$
S = \sum_{k=1}^{m} \alpha_k C_k
$$

ここで$\alpha_k$は学習対象である。

## 10.13 Total Hazard

基礎危険度

$$
H
$$

と複合危険度

$$
S
$$

を統合し、最終的な Hazard Engine の出力を

$$
H_{total} = H+S
$$

と定義する。この値は次章の THI Integration Model に入力され、最終的な THI を算出する。

---

# 11. Classification Engine

## 11.1 Purpose

THIは連続値である。しかし、防災現場では直感的な危険度表示が求められるため、THIを5段階へ分類する。

## 11.2 Hazard Classes

| THI | Category |
|------|----------|
|0.00–0.20|Minimal|
|0.20–0.40|Minor|
|0.40–0.60|Moderate|
|0.60–0.80|Significant|
|0.80–1.00|Extreme|

## 11.3 Threshold Optimization

各閾値T₁T₂T₃T₄は固定値ではない。過去台風データおよび実際の災害記録を教師データとして、ROC解析、F1 Score、Recall、Precisionを最大化するよう毎年最適化する。

## 11.4 Dynamic Classification

Extreme は単なる閾値判定ではない。以下のいずれかを満たす場合、Extreme へ昇格する。

- THI ≥ T₄
- Maximum Hazard が0.98以上
- Forecast THI がExtreme
- 複合危険度、Sが学習された閾値を超える

---

# 12. Dominant Hazard Engine

## Purpose

THIだけでは危険の種類が分からない。そのため最も支配的な危険要素を同時に表示する。

## Definition

危険度関数

$$
I_i
$$

について

$$
Dominant = \arg\max(I_i)
$$

と定義する。

## Output Example

Wind Dominant、Rain Dominant、Large Storm Dominant、Slow Translation Dominant、Rapid Intensification Dominant、Compound Hazard

## Compound Hazard

上位2要素の差が設定値以内ならCompound Hazardと表示する。例：Rain 0.91、Wind 0.89 → Compound Hazard

---

# 13. Hazard Contribution Analysis

THIの説明可能性を向上させるため各要素の寄与率を算出する。

## Contribution

寄与率

$$
C_i = \frac{I_i} {\sum I_i}
$$

とする。表示例

| Hazard | Contribution |
|---------|-------------:|
|Rain|39%|
|Wind|24%|
|Storm Radius|14%|
|Pressure|10%|
|Translation|7%|
|Rapid Intensification|6%|

## Purpose

Explainable AIの目的：

- 防災担当者への説明
- 研究利用
- 学習データ解析

---

# 14. Forecast Stability Engine

## Purpose

予報の安定性を評価する。実況だけではなく予報値のばらつきも重要である。

## Definition

予報THI

$$
THI(t)
$$

について標準偏差

$$
FSI = Std ( THI(t) )
$$

と定義する。

## Interpretation

FSIが小さい場合は予報が安定していることを示し、FSIが大きい場合は将来予測の不確実性が高いことを示す。

## Stability Rank

|FSI|Rank|
|---|----|
|0.00〜0.05|Very Stable|
|0.05〜0.10|Stable|
|0.10〜0.20|Moderate|
|0.20以上|Unstable|

---

# 15. Lifetime Hazard Engine

## Purpose

危険度はピーク値だけでは評価できない。危険状態が長時間継続する台風は被害が大きくなる傾向がある。

## Definition

$$
LHI = \int THI(t)\,dt
$$

離散時間では

$$
LHI = \sum_i THI_i \Delta t
$$

となる。

## Interpretation

同じExtremeでも半日継続、5日継続ではLHIが大きく異なる。

---

# 16. Trend Analysis Engine

## Purpose

現在の危険度だけでなく変化速度も評価する。

## Definition

$$
Trend = \frac {THI(t+\Delta t)-THI(t)} {\Delta t}
$$

## Classification

分類：Rapidly Increasing、Increasing、Stable、Weakening、Rapidly Weakening。Trend はForecast Engineと連携し今後の危険度変化を表現する。

---

# 17. Historical Similarity Engine

## Purpose

過去台風との類似性を評価する。

## Feature Vector

比較対象は

$$
X= [ P, V, R_{25}, R_{15}, Rain, V_{trans}, \Delta P, \Delta R ]
$$

とする。

## Similarity

ユークリッド距離を基本とし、特徴量の標準化後に

$$
d = \sqrt{ \sum (x_i-y_i)^2 }
$$

を計算する。将来的にはマハラノビス距離やコサイン類似度への拡張も可能である。

## Output Example

Most Similar Typhoon、Typhoon Hagibis (2019)、Similarity、94%、Second、Typhoon Nanmadol、89%、Third、Typhoon Talas、86%。この Historical Similarity Engine により、現在の台風を過去事例と比較し、「どのような災害パターンに近いか」を客観的に提示できる。これは防災判断だけでなく、モデル検証や説明可能性の向上にも有用である。

---

# 18. Uncertainty Engine

## 18.1 Purpose

台風解析には観測誤差および予報誤差が存在する。THI は単一値だけではなく、推定誤差も同時に提示する。

## 18.2 Sources of Uncertainty

主な不確実性要因

- 中心位置誤差
- 中心気圧推定誤差
- 最大風速推定誤差
- 暴風域半径推定誤差
- 降水予測誤差
- 移動速度予測誤差
- 予報円拡大

## 18.3 Monte Carlo Simulation

各入力変数

$$
x_i
$$

について観測誤差

$$
\varepsilon_i
$$

を与え、

$$
x_i' = x_i+\varepsilon_i
$$

を生成する。この操作をN回繰り返す。推奨：

$$
N=10000
$$

## 18.4 THI Distribution

各サンプルについてTHI を計算する。その結果

$$
THI_1, THI_2, ... THI_N
$$

という分布が得られる。

## 18.5 Confidence Interval

95%信頼区間を

$$
CI_{95} = [ P_{2.5}, P_{97.5} ]
$$

と定義する。表示例：THI 0.84（95% CI 0.80〜0.88）

---

# 19. Pseudo Integrated Kinetic Energy (PIKE)

## 19.1 Purpose

Integrated Kinetic Energy（IKE）は、台風が保有する運動エネルギーを表す代表的な指標である。しかし、厳密なIKEの算出には風速場全体の空間分布が必要であり、気象庁XMLのみでは完全な再現はできない。そこで本仕様では、気象庁XMLのみで算出可能な近似指標として **Pseudo Integrated Kinetic Energy（PIKE）** を導入する。PIKEはTHIの独立した危険度ではなく、風に関する危険度を補正する物理指標として利用する。

## 19.2 Physical Background

運動エネルギーは

$$
E=\frac12 mV^2
$$

で与えられる。一方、台風では質量を直接取得できないため、影響面積を質量の代理変数とみなす。影響面積は

$$
Area \propto R^2
$$

で近似できる。したがって

$$
PIKE \propto V^2R^2
$$

となる。

## 19.3 Basic Definition

Version 1.0では

$$
PIKE = V_{max}^{2} R_{25}^{2}
$$

と定義する。ここで

- $V_{max}$：最大風速
- $R_{25}$：暴風域半径

である。

## 19.4 Extended Definition

強風域も考慮する場合

$$
PIKE = V_{max}^{2} (R_{25}+kR_{15})^{2}
$$

とする。ここで$k$は学習対象である。初期値は設定してもよいが、正式運用では学習により推定する。

## 19.5 Normalization

PIKEは値のレンジが非常に大きくなる。そのため対数変換を行う。

$$
PIKE' = \log(1+PIKE)
$$

さらにシグモイド変換する。

$$
I_{PIKE} = \frac 1 { 1+ \exp \left( - \frac {PIKE'-\mu} {\sigma} \right) }
$$

μσは学習対象である。

## 19.6 Integration

PIKEはWind Hazardを補正する。

$$
I_{Wind}^{*} = I_{Wind} \left( 1+ \alpha I_{PIKE} \right)
$$

ここでαは学習対象とする。Wind Hazard単独より広域暴風を高く評価できる。

## 19.7 Relationship with TSI

TSIは台風の幾何学的大きさを評価する。PIKEは運動エネルギーを評価する。両者は役割が異なる。そのためTSIはサイズ補正PIKEはエネルギー補正として利用する。

## 19.8 XML Requirements

PIKEに必要なデータはすべて気象庁XMLから取得できる。必要項目

- 最大風速
- 暴風域半径
- 強風域半径（拡張版）

追加データソースは不要である。

## 19.9 Advantages

PIKEを導入することで

- 大型で強い台風
- 小型でも猛烈な台風
- 広域暴風型
- コンパクトで高風速型

を同一フレームワークで評価できる。これはTHIの物理的整合性を向上させる。

## 19.10 Final Hazard Vector

PIKE導入後、THIで扱う危険度ベクトルは

$$
\mathbf{I} = \left[ I_P, I_V, I_{PIKE}, I_B, I_G, I_R, I_T, I_D, I_E, I_W \right]^T
$$

と定義する。PIKEは独立した最終スコアではなく、Wind Hazardの補正項として使用する。

---

# 20. Typhoon Size Index (TSI)

## 20.1 Purpose

暴風域半径および強風域半径は、それぞれ独立した特徴量としてTHIへ入力される。しかし、台風の空間的規模を一つの指標として評価することで、台風が広範囲へ及ぼす影響を定量化できる。そのため、本仕様では Typhoon Size Index（TSI）を導入する。

## 20.2 Definition

TSI は

$$
TSI = w_BR_{25} + w_GR_{15}
$$

と定義する。ここで

- $R_{25}$：暴風域半径
- $R_{15}$：強風域半径
- $w_B,w_G$：学習対象パラメータ

## 20.3 Normalization

TSIはシグモイド変換により危険度へ変換する。

$$
I_{TSI} = \frac 1 {1+\exp \left( -\frac {TSI-\mu_{TSI}} {\sigma_{TSI}} \right)}
$$

## 20.4 Purpose

TSIは

- 広域暴風
- 広域停電
- 長時間暴風
- IKEの近似指標

として利用される。

---

# 21. Development Potential Index (DPI)

## 21.1 Purpose

実況値だけでは今後の危険度上昇を十分に表現できない。DPIは将来の急速発達可能性を評価する。

## 21.2 Definition

DPIは

$$
DPI = f ( \Delta P, \Delta R, V_{trans} )
$$

と定義する。ここで

- $\Delta P$：24時間気圧変化
- $\Delta R$：24時間暴風域変化
- $V_{trans}$：移動速度

である。

## 21.3 Practical Model

Version 1では

$$
DPI = \alpha I_D + \beta I_E + \gamma I_T
$$

とする。ここで

- $I_D$：急速発達危険度
- $I_E$：暴風域拡大危険度
- $I_T$：停滞危険度

係数$\alpha,\beta,\gamma$は学習対象である。

## 21.4 Future Expansion

将来Versionでは海面水温、海洋熱含量、鉛直風シアなどを追加可能である。ただし、THI v1.0 Coreでは気象庁XMLのみを使用するため、これらは採用しない。

---

# 22. Integration into THI

TSIおよびDPIは独立した危険度として扱う。危険度ベクトルは

$$
\mathbf I = [ I_P, I_V, I_B, I_G, I_R, I_T, I_D, I_E, I_{TSI}, I_{DPI}, I_W ]
$$

となる。以降の幾何平均・複合危険度・Extreme補正は、この拡張ベクトルを入力として実施する。

---

# 23. Forecast Hazard Modifier (FHM)

## 23.1 Purpose

THIは「現在の危険度」を評価する指標である。しかし、防災運用では「これから危険になるか」が極めて重要である。そのため、本仕様では実況THIとは独立したForecast Hazard Modifier（FHM）を定義する。FHMはTHIを書き換えるものではなく、将来危険度を補正するモジュールである。

## 23.2 Definition

FHMは

$$
FHM=f(DPI,THI_{forecast})
$$

と定義する。ここでDPI、Forecast THIを統合して将来危険度を評価する。

## 23.3 Final Forecast Score

将来危険度を

$$
THI_F = THI + \omega FHM
$$

と定義する。ここで

$$
0\le\omega\le1
$$

は学習対象とする。

---

# 24. Extreme Override System

## Purpose

平均化は極端な災害を過小評価する可能性がある。そのためOverride Systemを導入する。

## Rule A

以下のいずれかを満たした場合Extreme候補となる。

- 最大危険度が0.98以上
- Forecast Extreme
- 複合危険度が閾値超過
- 豪雨危険度が閾値超過

## Rule B

複数危険要素が同時に高い場合Extremeへ昇格する。例えばRain 0.91、Wind 0.92、Pressure 0.88などである。

## Rule C

ForecastでExtremeになる場合実況がSignificantでもEarly Extremeとして表示できる。これは避難判断支援を目的とする。

---

# 25. Explainable THI

THIはブラックボックスであってはならない。必ず計算根拠を出力する。

## Example

| 項目 | 値 |
|---|---|
| THI | 0.84（Extreme） |
| Pressure | 17% |
| Wind | 29% |
| Rain | 31% |
| Storm Radius | 11% |
| Translation | 5% |
| Rapid Intensification | 7% |
| Dominant Hazard | Heavy Rain |
| Forecast | Extreme within 24h |
| Confidence | 97% |
| DQI | 99% |

これらはJSONおよび人間向けレポート双方で出力する。

---

# 26. Final THI Equation

以上を踏まえ、THIの最終式を以下のように定義する。まず、各危険度関数

$$
I_i
$$

を求める。次に幾何平均

$$
H = \left( \prod I_i \right)^{1/n}
$$

を計算する。複合危険度

$$
S = \sum \alpha_kC_k
$$

を求める。Extreme補正

$$
M = \max(I_i)
$$

を求める。最後に

$$
THI = (1-\beta) (H+S) + \beta M
$$

と定義する。THIは0〜1へ正規化される。

---

# 27. Category Definition

|THI|Category|
|---|--------|
|0.00–0.20|Minimal|
|0.20–0.40|Minor|
|0.40–0.60|Moderate|
|0.60–0.80|Significant|
|0.80–1.00|Extreme|

なお、分類境界は毎年学習により更新される。

---

# 28. Summary

THI Version 1.0 は

- 気象庁XMLのみで算出可能
- リアルタイム計算可能
- 将来予測対応
- 説明可能
- 自己学習可能
- 統計学的最適化可能
- 物理法則との整合性を維持

という7つの設計原則に基づく統合台風危険度指標である。本仕様をTHI Version 1.0 Core Specificationと定義する。

---

# 29. Self Calibration Engine

## Purpose

THI は固定指標ではない。新しい台風データを利用し毎年自動的に更新される。

## Learning Dataset

入力

- 気象庁XML
- 台風ベストトラック
- 災害統計
- 人的被害
- 住宅被害
- 降水観測

## Training Flow

過去台風 → 特徴量抽出 → THI計算 → 実被害比較 → パラメータ更新 → Validation → Config更新

## Parameters to Learn

- Pressure μ, σ
- Wind μ, σ
- Rain μ, σ
- Storm Radius μ, σ
- Translation μ, σ
- Rapid Intensification μ, σ
- Expansion μ, σ
- β
- 分類閾値
- 複合危険係数 α

---

# 30. Loss Function

## Purpose

一般的な平均二乗誤差では危険な台風を過小評価してしまう。THIでは防災利用を最優先とする。

## Asymmetric Loss

$$
L = \alpha (y-\hat y)^2
$$

ここで

$$
\alpha= \begin{cases} 5,&\hat y<y\\ 1,&\hat y\ge y \end{cases}
$$

つまり危険を低く見積もる誤差に5倍のペナルティを与える。

---

# 31. Parameter Optimization

## Optimization Target

最適化対象

- シグモイド中心値
- シグモイド傾き
- β
- 危険度閾値
- 複合危険係数

## Candidate Algorithms

- Bayesian Optimization
- Tree-structured Parzen Estimator
- Grid Search
- Random Search
- Optuna

## Evaluation Metrics

ROC-AUC、F1 Score、Recall、Precision、Balanced Accuracy、Matthews Correlation Coefficient、Brier Scoreのうち、Recallを最重要評価指標とする。理由：危険な台風を見逃さないことが本モデルの目的だからである。

---

# 32. Version Management

THI は数式を変更しない。変更されるのはパラメータのみである。

Version例：THI 1.0.0 → THI 1.1.0 → THI 1.2.0 → THI 2.0.0

| 区分 | 内容 |
|---|---|
| メジャーバージョン | 数式変更 |
| マイナーバージョン | パラメータ更新 |
| パッチ | バグ修正 |

## Reproducibility

各Versionでは使用したConfig、学習データ、評価指標、Git Commit IDを保存する。これにより完全な再現性を保証する。

---

# 33. Core Design Principles

本研究で提案する THI は以下を満たすことを設計目標とする。

- 気象庁XMLのみで計算可能
- 外部GISに依存しない
- 人口データに依存しない
- インフラ情報に依存しない
- 世界中の台風へ適用可能
- 数式は固定
- パラメータのみ学習
- Explainable AI対応
- リアルタイム計算可能
- 完全自動運用可能

以上をもって THI v1.0 Core Mathematical Specification を定義する。次章では、Python実装仕様、モジュール構成、XML解析仕様、クラス設計、およびAPI設計について記述する。

---

# 34. Python Software Architecture

## 34.1 Overview

THIは単なるPythonスクリプトではない。保守性・再利用性・将来的な機械学習への拡張を考慮し、完全なモジュール構造として設計する。

## 34.2 Directory Structure

```text
thi/

├── main.py
├── config.py
├── constants.py
├── parser/
│   ├── typhoon_xml.py
│   ├── warning_xml.py
│   └── forecast_xml.py
│
├── features/
│   ├── pressure.py
│   ├── wind.py
│   ├── rainfall.py
│   ├── radius.py
│   ├── translation.py
│   ├── rapid_intensification.py
│   └── expansion.py
│
├── models/
│   ├── sigmoid.py
│   ├── geometric_mean.py
│   ├── compound.py
│   ├── thi.py
│   ├── classification.py
│   └── confidence.py
│
├── forecast/
│   ├── forecast_engine.py
│   ├── trend.py
│   ├── stability.py
│   └── lifetime.py
│
├── explain/
│   ├── contribution.py
│   ├── dominant.py
│   ├── similarity.py
│   └── uncertainty.py
│
├── learning/
│   ├── trainer.py
│   ├── optimizer.py
│   ├── validation.py
│   └── loss.py
│
├── config/
│   ├── parameters.json
│   └── thresholds.json
│
└── tests/
```

## XML Parser

### Purpose

XML Parser は気象庁XMLを読み込み、THIが利用できる標準化データへ変換する。

### Input

- 台風解析情報
- 台風予報情報
- 気象警報・注意報
- 降水関連XML

### Output

```python
TyphoonFeature(
    pressure=945,
    wind=45,
    storm_radius=280,
    gale_radius=650,
    rainfall=410,
    translation_speed=18,
    pressure_change=-32,
    radius_change=45,
    warning_level=0.75
)
```

## Feature Generator

Parser が取得したデータを特徴量へ変換する。Feature Generator は物理量から危険度への変換を担当する（Feature → Sigmoid → Normalized Hazard）。

## Hazard Engine

各Featureを独立に評価する。例：PressureModel、WindModel、RainModel、TranslationModel、ExpansionModel、RapidDevelopmentModel。

各モジュールは共通インターフェース

```python
calculate(feature)
```

を持つ。

## THI Engine

THI Engine は全危険度を受け取り基礎危険度、複合危険度、Extreme補正、Forecast補正を適用する。出力

```python
THIResult
```

### THIResult

```python
score

category

confidence

forecast

dominant_hazard

forecast_trend

forecast_stability

lifetime_hazard
```

## Classification Engine

入力：THI。出力：Minimal、Minor、Moderate、Significant、Extreme。判定はThreshold Managerから読み込まれる。

## Configuration System

すべての学習済みパラメータはJSONで保存する。例：

```json
{
  "pressure":{
      "mu":948.3,
      "sigma":7.5
  },

  "wind":{
      "mu":41.2,
      "sigma":6.8
  },

  "rain":{
      "mu":325,
      "sigma":81
  },

  "translation":{
      "mu":22,
      "sigma":5.2
  },

  "beta":0.31,

  "thresholds":[
      0.20,
      0.40,
      0.60,
      0.80
  ]
}
```

## Forecast Engine

Forecast XML → 各予報時刻 → THI → 最大値取得 → Forecast THI

さらにTrend、Lifetime、Forecast Stabilityを同時に計算する。

## Similarity Engine

入力：現在台風 → 特徴量ベクトル → Database → 距離計算 → 類似順位 → Top10返却

初期実装ではユークリッド距離、Version 2ではマハラノビス距離、Version 3では機械学習Embeddingへ発展可能とする。

## Explainable Engine

THI算出後、各危険度の寄与率を計算する。出力例：

| 要素 | 寄与率 |
|---|---|
| Rain | 41% |
| Wind | 26% |
| Pressure | 11% |
| Storm Radius | 10% |
| Translation | 8% |
| Rapid Intensification | 4% |

同時にDominant Hazardも判定する。

## Uncertainty Engine

Monte Carlo → THI → 分布 → 95%信頼区間 → 表示、という処理でこの処理はForecastにも適用可能である。

## Annual Learning Engine

毎年、前年までの全台風を追加学習する。学習結果は`parameters.json`へ保存する。数式は変更しない。変更されるのはμ、σ、β、閾値のみである。

以上により、THIは気象庁XMLのみを入力として、リアルタイム計算・予報評価・説明可能性・自己学習を備えた一貫したソフトウェアアーキテクチャとして実装される。

---

# 35. Japan Meteorological Agency XML Mapping Specification

## 35.1 Purpose

本章では、THI（Typhoon Hazard Index）で利用する全入力変数と、気象庁XMLとの対応関係を定義する。本仕様の設計目標は、**可能な限り気象庁XMLのみで全計算を完結させること**である。

## 35.2 XML Sources

THIで利用するXMLは次のカテゴリに分類する。

|区分|用途|
|----|----|
|台風解析XML|実況値取得|
|台風予報XML|将来予測取得|
|気象警報・注意報XML|Warning Index|
|降水関連XML|Rainfall Hazard|
|天気概況XML|補助情報|

## 35.3 Feature Mapping

|Feature|XML|必須|備考|
|--------|---|----|----|
|最低中心気圧|台風解析XML|必須|実況値|
|最大風速|台風解析XML|必須|実況値|
|暴風域半径|台風解析XML|必須|実況値|
|強風域半径|台風解析XML|必須|実況値|
|台風中心座標|台風解析XML|必須|移動速度算出|
|解析時刻|台風解析XML|必須|差分計算|
|予報中心気圧|台風予報XML|任意|Forecast|
|予報最大風速|台風予報XML|任意|Forecast|
|予報暴風域|台風予報XML|任意|Forecast|
|24時間降水量|降水関連XML|任意|豪雨評価|
|警報・注意報|警報XML|任意|日本国内のみ|

## 35.4 Derived Features

THIではXMLから直接取得できない値を時系列差分から計算する。

### Translation Speed

入力：時刻、緯度、経度 → 球面距離 → 移動速度

$$
V_{trans} = \frac{Distance} {\Delta t}
$$

### Pressure Change

$$
\Delta P = P_t - P_{t-24}
$$

### Radius Expansion

$$
\Delta R = R_t - R_{t-24}
$$

### Forecast Trend

各予報時刻についてTHIを算出し変化率を計算する。

## 35.5 Missing Data Policy

XMLに値が存在しない場合以下を適用する。

### Mandatory Feature

取得失敗 → THI計算停止、Confidence 0%

### Optional Feature

欠損 → 該当危険度を除外 → Confidence低下 → DQI低下

## 35.6 International Typhoon Support

日本周辺では警報XMLを利用できる。しかし東南アジア、西太平洋では警報情報は存在しない。そのためWarning Hazardは欠損扱いとする。Confidenceのみ低下させる。THI本体は継続計算する。

## 35.7 Rainfall Policy

豪雨は台風本体が小規模でも重大災害を引き起こす。そのためRainfall Hazardは独立した主要危険要素として扱う。降水量が取得できる場合通常計算を行う。取得できない場合Confidenceを低下させる。

## 35.8 Forecast Policy

Forecast XMLが取得できる場合以下を計算する。

- Forecast THI
- Forecast Trend
- Forecast Stability
- Lifetime Hazard

Forecast XMLが存在しない場合実況のみで計算する。

## 35.9 XML Validation

XML取得後、以下を検証する。

- XML Schema
- 必須タグ
- 時刻整合性
- 単位
- 欠損値

異常時はXML Parse Errorとしてログへ記録する。

## 35.10 Unit Standardization

すべての値は以下へ統一する。

|項目|単位|
|----|----|
|Pressure|hPa|
|Wind|m/s|
|Rainfall|mm|
|Radius|km|
|Translation Speed|km/h|
|Latitude|degree|
|Longitude|degree|

## 35.11 Feature Cache

同一XMLの再解析を避けるため特徴量をキャッシュする。キャッシュキー

- 発表時刻
- 電文ID
- 台風番号

とする。

## 35.12 XML Processing Pipeline

JMA XML → Schema Validation → Parser → Feature Generator → Derived Features → Hazard Engine → THI Engine → Classification → Output

## 35.13 Reproducibility

すべてのTHI結果には以下を保存する。

- XML発表時刻
- XML電文ID
- THI Version
- Config Version
- 計算日時
- 使用Feature一覧

これにより完全な再現性を保証する。

## 35.14 Core Philosophy

本仕様では、気象庁XMLを唯一のデータソースとすることで、

- 実装の単純化
- 再現性の確保
- 長期運用性
- 学術的検証可能性

を実現する。将来的なパラメータ最適化は行うが、**入力データソースそのものは変更しない**ことを原則とする。

## 35.7 Rainfall Hazard Policy（Revised）

### Purpose

THIでは降水災害を暴風と同等の主要ハザードとして扱う。ただし、気象庁XMLでは提供される降水情報が電文の種類や対象領域によって異なるため、本仕様では特定のXML形式に依存しない抽象化を採用する。

### Rainfall Feature

降水指標を

$$
Rain_{obs}
$$

と定義する。これは気象庁XMLから取得可能な実況または解析降水量を表す。将来予報については

$$
Rain_{forecast}
$$

と定義する。

### XML Source Priority

利用可能なXMLが複数存在する場合は優先順位を設ける。

1. 解析降水量
2. 実況降水量
3. 予測降水量
4. その他の降水関連XML

取得したXML種別はメタデータとして保存する。

### Metadata

THI出力には以下を含める。Rain Source、Rain Timestamp、Rain Quality、XML ID。これにより再現性を確保する。

### Missing Policy

降水情報が取得できない場合Rain Hazard を欠損値とし、THI計算は継続する。ConfidenceおよびData Quality Indexのみ低下させる。

### Hazard Function

降水危険度は

$$
I_R = \frac 1 {1+\exp \left( -\frac {Rain-\mu_R} {\sigma_R} \right)}
$$

と定義する。μおよびσは固定値ではなく、毎年学習により更新される。

### Heavy Rain Override

豪雨は暴風を伴わなくても重大災害を引き起こす。そのためRain Hazard が学習済み閾値を超えた場合、THIは暴風域の大小に関係なくSignificantまたはExtremeへ昇格できる。この判定閾値も学習対象とする。

---

# 36. Model Governance

THIの数式構造は固定し、更新対象は以下に限定する。

- シグモイド中心値（μ）
- シグモイド傾き（σ）
- 重み係数（w）
- 複合危険係数（α）
- Extreme判定閾値
- 分類閾値

これらは過去台風データと実被害データを用いて統計的に最適化する。本仕様により、THIは「気象庁XMLのみで再現可能」「自己学習可能」「物理的解釈が可能」という三つの要件を同時に満たす。

---

# 37. JSON Output Specification

THIの標準出力形式を定義する。

```json
{
  "version":"1.0",

  "score":0.84,

  "category":"Extreme",

  "confidence":0.97,

  "dqi":0.99,

  "dominant_hazard":"Rain",

  "forecast":"Extreme",

  "forecast_trend":"Increasing",

  "forecast_stability":"Stable",

  "lifetime_hazard":18.7,

  "similarity":[
      {
        "name":"Hagibis",
        "year":2019,
        "score":0.94
      }
  ]
}
```

---

# 38. Human Readable Report

JSONだけでは防災現場では使いにくい。そのため文章生成を行う。例：「現在のTHIは0.84でExtremeです。最大の危険要因は豪雨です。24時間以内にさらに危険度が増加する見込みです。予報の信頼性は97%です。」

---

# 39. API Specification

Pythonライブラリとして利用できるAPIを定義する。

```python
from thi import THI

engine = THI()

result = engine.calculate(xml_files)

print(result.score)
print(result.category)
```

Forecast

```python
forecast = engine.forecast(xml_files)

print(forecast.max_score)
```

Similarity

```python
similar = engine.similarity()

for typhoon in similar:
    print(typhoon)
```

Training

```python
engine.train(dataset)
```

Optimization

```python
engine.optimize(dataset)
```

---

# 40. Standard Calculation Flow

THIの計算は以下の順序で実施する。

```text
気象庁XML取得
        │
        ▼
XML Validation
        │
        ▼
Feature Extraction
        │
        ▼
Derived Feature Generation
        │
        ▼
Sigmoid Transformation
        │
        ▼
PIKE Calculation
        │
        ▼
TSI Calculation
        │
        ▼
Compound Hazard
        │
        ▼
Base Hazard
        │
        ▼
Maximum Hazard
        │
        ▼
THI Integration
        │
        ▼
Classification
        │
        ▼
Forecast Calculation
        │
        ▼
Confidence / DQI
        │
        ▼
Explainable Report
        │
        ▼
JSON Output
```

---

# 41. Computational Complexity

THIはリアルタイム処理を目的とする。Feature数をnとすると基本計算量は

$$
O(n)
$$

となる。Monte Carloのみ

$$
O(Nn)
$$

となる。N=10000でもCPU上で十分実時間計算可能である。

---

# 42. Implementation Policy

THI実装では以下を遵守する。

- Python 3.12以上
- NumPy
- SciPy
- Pandas
- scikit-learn
- Optuna（学習時）

リアルタイム計算時には学習ライブラリを必要としない。本章までで、THIの数理モデル・ソフトウェア構造・入出力仕様・説明可能性・運用方針を定義した。次章では、過去台風データによる学習・検証手法、評価指標、および統計的妥当性の検証方法を定義する。

---

# 43. Statistical Learning and Validation

## 43.1 Purpose

本章では、THI（Typhoon Hazard Index）の統計学的妥当性を保証するための学習・検証方法を定義する。THIは経験則による固定モデルではなく、**数式構造を固定し、パラメータのみをデータから推定する半物理・半統計モデル（Physics-informed Statistical Model）**として位置付ける。

## 43.2 Learning Dataset

学習には、可能な限り長期間の気象庁データを使用する。推奨対象：

- 気象庁ベストトラック
- 気象庁台風解析XML
- 気象庁台風予報XML
- 気象庁警報・注意報XML
- 気象庁降水関連XML

目的変数（教師データ）は以下のいずれか、または複数を組み合わせる。

- 人的被害（死者・行方不明者数）
- 住家被害（全壊・半壊・床上浸水等）
- 被害総額
- 保険支払額（利用可能な場合）

複数の被害指標を用いる場合は、各指標を正規化し、総合被害指数として扱う。

## 43.3 Target Variable

被害額など分布の歪みが大きい変数については対数変換を適用する。

$$
Y = \log(1 + Damage)
$$

これにより外れ値の影響を抑制する。

## 43.4 Parameter Estimation

学習対象は以下とする。

- シグモイド中心値（μ）
- シグモイド傾き（σ）
- 幾何平均補正係数
- Extreme補正係数（β）
- Compound Hazard係数（α）
- TSI重み
- FHM重み
- 分類閾値

数式そのものは変更しない。

## 43.5 Cross Validation

過学習を防ぐため、k-fold Cross Validationを採用する。推奨

$$
k = 5
$$

または

$$
k = 10
$$

## 43.6 Time-series Validation

通常のランダム分割は採用しない。学習データは時系列を保持する。例：2010〜2020をTraining、2021をValidation、2022をTestとする。

## 43.7 Performance Metrics

以下を主要評価指標とする。分類性能

- ROC-AUC
- Precision
- Recall
- F1 Score
- Balanced Accuracy
- Matthews Correlation Coefficient

回帰性能

- RMSE
- MAE
- R²
- Brier Score

## 43.8 Priority of Evaluation

THIは防災利用を目的とするため、評価順位は以下とする。

1. Recall
2. F1 Score
3. ROC-AUC
4. Precision

危険な台風を見逃さないことを最優先とする。

## 43.9 Hyperparameter Optimization

推奨手法

- Bayesian Optimization
- Optuna
- Tree-structured Parzen Estimator (TPE)

Grid Searchはベースライン比較用とする。

## 43.10 Annual Retraining

毎年、新たな台風事例を追加して学習を行う。更新対象は

- μ
- σ
- β
- α
- 閾値

のみである。THIの基本数式は維持する。

---

# 44. Benchmark Strategy

THIは既存指標との比較を行う。比較対象例

- 中心気圧単独
- 最大風速単独
- 暴風域半径単独
- Saffir–Simpson Hurricane Wind Scale（参考）
- Integrated Kinetic Energy（IKE、比較可能な場合）

THIがこれらより高い予測性能を示すかを評価する。

---

# 45. Sensitivity Analysis

各入力変数について感度解析を実施する。一変数のみを変化させ、THIの変動量

$$
\frac{\partial THI}{\partial x_i}
$$

を評価する。これにより、各要素の影響度を定量化する。

---

# 46. Robustness Test

入力値に観測誤差を与え、Monte Carlo Simulation を実施する。THIの変動幅が許容範囲内であることを確認する。

---

# 47. Reproducibility

学習時には以下を保存する。

- THI Version
- パラメータファイル
- 学習データ期間
- 学習データ件数
- 学習日時
- Python Version
- ライブラリVersion
- Git Commit ID（任意）

これにより、同一データから同一結果を再現できる。

---

# 48. Scope of THI

THIは「台風が持つ自然ハザードの危険度」を評価する指標である。以下は評価対象外とする。

- 人口密度
- 建物耐震性
- 社会経済条件
- インフラ脆弱性
- 避難行動

これらは災害リスク評価では重要であるが、本仕様では含めない。THIはあくまで、**気象庁XMLから取得可能な気象学的情報のみを用いて、台風そのものの危険度を一元的に評価する指標**として定義する。

---

# 49. Mathematical Consistency and Variable Independence

## 49.1 Purpose

本章では、THIで採用する数理モデルの整合性を定義する。台風の物理現象には相関の高い変数が多数存在する。例えば

- 中心気圧が低いほど最大風速は大きくなる
- 暴風域が広いほど強風域も広くなる
- 急速発達中は暴風域が拡大しやすい

これらをそのまま全て加算すると、同一の物理現象を二重評価してしまう。THIではこれを防ぐ。

## 49.2 Variable Classification

THIでは説明変数を4種類へ分類する。

### Primary Variables

独立した物理量

- 最低中心気圧
- 最大風速
- 暴風域半径
- 強風域半径
- 降水量
- 移動速度

### Dynamic Variables

時間変化量

- 24時間気圧変化
- 24時間暴風域変化

### Composite Variables

複数変数から計算される派生量

- TSI
- FHM
- Lifetime Hazard

### Meta Variables

モデル評価用

- Confidence
- DQI
- Forecast Stability

## 49.3 Dependency Rule

THIではPrimary Variablesのみを直接危険度へ入力する。Composite Variables は補正項としてのみ利用する。これにより二重評価を防止する。

## 49.4 Typhoon Size Index Revision

TSIはR25、R15から計算される。そのためTHIへ直接加算しない。代わりに暴風・強風危険度を補正する。

$$
I_{Wind} ' = I_{Wind} \times (1+\alpha I_{TSI})
$$

ここでαは学習対象とする。これにより暴風域の広い台風ほどWind Hazardが自然に増強される。

## 49.5 Forecast Hazard Modifier Revision

FHMはForecast専用モジュールである。実況THIには利用しない。

$$
THI_{Forecast} = THI + \omega FHM
$$

実況THIは常に実況値のみから算出する。

## 49.6 Compound Hazard Revision

Compound Hazardはすべての組み合わせを使用しない。学習により有意性の認められた項のみ採用する。

例えば、次のような組み合わせは採用する。

- Rain × Translation
- Wind × Radius
- Rain × Warning

一方、次のように物理的意味が重複する組み合わせは採用しない。

- Pressure × Radius
- Pressure × Wind

## 49.7 Automatic Feature Selection

学習時にはL1正則化、Elastic Netまたは逐次特徴選択を利用し、不要な特徴量を除外する。これによりモデルの過学習を防止する。

## 49.8 Model Stability

各年度の学習結果について係数の変動率を記録する。急激な変動があれば再学習またはレビュー対象とする。

---

# 50. Design Constraints

THI Version 1.0では以下を変更しない。

- 数式構造
- Feature定義
- 出力仕様
- Category名称

変更可能なのは以下のみ。

- シグモイドパラメータ
- 重み係数
- Compound係数
- Extreme閾値
- Category境界値

---

# 51. Operational Principles

THIは以下を満たす。

- 気象庁XMLのみを入力とする。
- リアルタイムで動作する。
- 海外の台風にも適用可能（警報情報がない場合はConfidenceで表現）。
- パラメータは継続的に学習する。
- 数式構造は維持し、学術的再現性を確保する。
- 出力はJSONおよび人間向けレポートの両方を提供する。

---

# 52. Future Research

Version 1.0の研究課題として、以下を位置付ける。

- PIKE近似式と厳密IKEとの相関評価
- THIと実被害との統計的関連性の検証
- 最適な非対称損失関数の探索
- TSIおよびPIKEの係数最適化
- 長期ベストトラックデータによる汎化性能評価
- 海外熱帯低気圧への適用性評価

---

# 53. Conclusion

Typhoon Hazard Index (THI) Version 1.0は、気象庁XMLのみを用いて台風の物理的危険度を統合評価することを目的とした数理モデルである。本モデルは、

- 非線形危険度関数
- 幾何平均による統合
- 複合ハザード評価
- Forecast Hazard Modifier
- Pseudo Integrated Kinetic Energy
- Typhoon Size Index
- Explainable AI
- Confidence評価
- 自己学習可能なパラメータ推定

を統合した、物理モデルと統計モデルを融合したハイブリッド指標として定義される。本仕様はTHI Version 1.0 Core Specificationの基礎文書とする。

THI Version 1.0 Core Specificationは、

- 数理モデル
- 統計モデル
- XML仕様
- ソフトウェア設計
- 学習方法
- 検証方法
- API仕様
- 実装標準

を統合した包括的な設計仕様である。以後のVersionでは、数式構造を維持したままパラメータ更新を継続し、気象庁XMLのみで運用可能な統一台風危険度指標として発展させる。

---

# References

## R.1 Purpose

本章では、THI（Typhoon Hazard Index）の理論的背景となる代表的な研究、統計手法、および気象学的知見を示す。THIは既存研究をそのまま再現するものではなく、それらを統合・発展させた新しい統合危険度指標として設計されている。

## R.2 Tropical Cyclone Dynamics

### Emanuel, K. A.

**Emanuel, K. A.** *Atmospheric Convection.* Oxford University Press. 熱帯低気圧のエネルギー学・発達理論。

### Emanuel, K. A.

**Emanuel, K. A.** *Divine Wind: The History and Science of Hurricanes.* Oxford University Press. ハリケーンの発達メカニズム。

### Holland, G.

**Holland, G. J.** *An Analytic Model of the Wind and Pressure Profiles in Hurricanes.* Monthly Weather Review. 台風風速分布モデル。

## R.3 Integrated Kinetic Energy

### Powell et al.

**Powell, M. D., Reinhold, T. A.** *Tropical Cyclone Destructive Potential by Integrated Kinetic Energy.*

Integrated Kinetic Energy（IKE）の提案。THIのPIKE設計の理論的背景。

## R.4 Best Track Data

### Japan Meteorological Agency

気象庁ベストトラックデータ。THI学習用基礎データ。

### IBTrACS

International Best Track Archive for Climate Stewardship。世界標準ベストトラックデータ。比較検証用途。

## R.5 Statistical Learning

### Hastie, Tibshirani, Friedman

**The Elements of Statistical Learning.** Springer. 機械学習全般。

### Bishop

**Pattern Recognition and Machine Learning.** Springer. 確率モデル。

### Murphy

**Machine Learning.** MIT Press. 損失関数、確率予測。

## R.6 Explainable AI

### Lundberg

SHAP、Feature Contribution

### Ribeiro

LIME、Local Explainability

## R.7 Generalized Additive Models

### Hastie

*Generalized Additive Models.* GAM理論。THI非線形関数設計。

## R.8 Extreme Value Theory

### Coles

*An Introduction to Statistical Modeling of Extreme Values.* 極値理論。Extreme判定の理論背景。

## R.9 Copula Theory

### Nelsen

*Introduction to Copulas.* 複数危険要素の依存構造。Compound Hazardの背景。

## R.10 Bayesian Optimization

### Snoek

*Practical Bayesian Optimization of Machine Learning Algorithms.* パラメータ最適化。

## R.11 Physics-informed Machine Learning

Karniadakis et al., *Physics-informed Machine Learning.* 物理モデルと学習の融合。THI設計思想。

## R.12 Meteorological Data

気象庁、防災情報XML。気象庁、気象データ利用ガイド。気象庁、台風解析・予報資料。

## R.13 Hazard Assessment

UNDRR、Sendai Framework。災害リスク評価。WMO、Tropical Cyclone Programme（世界気象機関）。熱帯低気圧評価。

## R.14 Software Engineering

Python Software Foundation、Python Documentation、NumPy Documentation、SciPy Documentation、Pandas Documentation、Optuna Documentation、scikit-learn Documentation

## R.15 Scientific Position of THI

THIは以下の学術分野を融合したモデルである。

- Tropical Meteorology
- Boundary Layer Meteorology
- Synoptic Meteorology
- Hydrology
- Fluid Dynamics
- Disaster Science
- Mathematical Statistics
- Machine Learning
- Explainable AI
- Physics-informed Machine Learning

## R.16 Original Contributions of THI

THIの独自性は以下にある。

1. 気象庁XMLのみで算出可能な統合危険度指標。
2. 中心気圧・風速・暴風域・降水・移動速度を統合した非線形モデル。
3. PIKEによるIKE近似。
4. TSIによる空間規模評価。
5. Forecast Hazard Modifierによる将来危険度推定。
6. Explainable AIを前提とした危険度分解。
7. 年次自己学習によるパラメータ更新。
8. 数式構造を固定し、学習対象をパラメータに限定した Physics-informed Statistical Hazard Model の採用。

---

# Appendix A. Mathematical Foundations

## A.1 Design Philosophy

THI（Typhoon Hazard Index）は、従来の単一指標（中心気圧・最大風速など）では表現できない台風の総合的危険度を、複数の物理量を統合した非線形指数として定義する。設計方針は以下の5点である。

1. **Physical Interpretability**（物理的解釈可能性）
2. **Statistical Optimizability**（統計的最適化可能性）
3. **Explainability**（説明可能性）
4. **Reproducibility**（再現性）
5. **Real-time Computability**（リアルタイム計算可能性）

## A.2 Hazard Function

すべての一次物理量は共通の危険度関数へ写像する。

$$
I(x) = \frac 1 { 1+ \exp \left( -\frac{x-\mu}{\sigma} \right) }
$$

ここで、

- $x$：入力物理量
- $\mu$：危険度中心値
- $\sigma$：遷移幅

である。Version 1.0では全入力をこの形式に統一する。

## A.3 Base Hazard

各危険度を

$$
I_i
$$

とすると、基礎危険度

$$
H = \left( \prod_{i=1}^{n} I_i \right) ^{1/n}
$$

と定義する。幾何平均を採用する理由は、一部の要素だけが極端に高くても、他の要素が低ければ総合危険度が抑制されるためである。

## A.4 Compound Hazard

複数要素が同時に高い場合、相乗効果を考慮する。

$$
S = \sum_{k} \alpha_k C_k
$$

ここで$C_k$は有意と判定された複合項である。例

- Rain × Translation
- Wind × Radius
- Rain × Warning

## A.5 Maximum Hazard

最大危険要素を

$$
M = \max(I_i)
$$

と定義する。Extreme判定ではMを必ず利用する。

## A.6 Final Equation

最終的なTHIは

$$
THI = (1-\beta) (H+S) + \beta M
$$

で定義される。βは学習対象である。

---

# Appendix B. Data Quality Index (DQI)

DQIは入力データの品質を評価する。

## Required

- 中心気圧
- 最大風速
- 台風位置
- 暴風域
- 強風域

これらが存在しない場合、DQIは著しく低下する。

## Optional

- 降水量
- Warning
- Forecast

欠損しても計算は継続する。DQIは0〜1で表す。

---

# Appendix C. Confidence Score

ConfidenceはTHI推定値の信頼性を示す。Confidenceは

- XML完全性
- 欠損率
- Forecast整合性
- Monte Carlo分散

から算出する。

---

# Appendix D. Explainable Output

標準出力では以下を返す。

```text
THI
Category
Dominant Hazard
Contribution
Forecast
Confidence
DQI
Similar Typhoon
```

---

# Appendix E. Versioning Policy

THIはSemantic Versioningを採用する。

## Major

数式変更、例：1.x → 2.x

## Minor

学習パラメータ更新、例：1.0 → 1.1

## Patch

実装修正、例：1.0.0 → 1.0.1

---

# Appendix F. Recommended Software Stack

Python 3.12+、NumPy、SciPy、Pandas、lxml、scikit-learn、Optuna、Matplotlib、pytest

---

# Appendix G. Expected Outputs

標準JSON、詳細JSON、CSV、Markdown Report、PDF Report、REST API、CLI。以上の出力形式をサポートする。

---

# Appendix H. Reference Datasets

本仕様で学習・評価に利用を推奨するデータセット。

## 気象

- 気象庁ベストトラック
- 気象庁XML
- アメダス降水観測
- 気象庁警報・注意報履歴

## 被害

- 内閣府災害資料
- 消防庁災害情報
- 国土交通省水害統計

## 学術比較

- JTWC Best Track
- IBTrACS（比較評価用）

※ **THIの入力は気象庁XMLのみ**とし、これらは学習・検証専用データとして利用する。

## End of THI Version 1.0 Core Specification

本仕様書は、THIの数理モデル、ソフトウェア構成、学習戦略、検証手法および運用方針を定義する基礎仕様書である。実装は本仕様書に従って行い、数式構造を維持したまま、学習パラメータのみを継続的に更新することで、長期的な精度向上と学術的再現性を両立する。

---

# Appendix I. Japan Meteorological Agency XML Implementation Specification

## I.1 Purpose

本付録では、THI Version 1.0 において使用する気象庁XMLの実装仕様を定義する。目的は、

- 全入力データの取得元を明確化する
- 実装者間で解釈の差異をなくす
- 気象庁XMLのみで完全に再現可能なアルゴリズムを保証する

ことである。

## I.2 Supported XML Products

Version 1.0では以下の電文を対象とする。

|電文|用途|必須|
|----|----|----|
|台風解析情報|実況解析|必須|
|台風予報情報|予測THI|推奨|
|警報・注意報|Warning Hazard|任意|
|降水関連XML|Rain Hazard|任意|

## I.3 Feature Extraction Table

|THI Feature|XML取得元|取得方法|欠損時|
|------------|---------|---------|-------|
|最低中心気圧|台風解析|直接取得|計算停止|
|最大風速|台風解析|直接取得|計算停止|
|暴風域半径|台風解析|直接取得|計算停止|
|強風域半径|台風解析|直接取得|計算停止|
|台風中心座標|台風解析|直接取得|計算停止|
|解析時刻|台風解析|直接取得|計算停止|
|降水量|降水XML|利用可能な最新値|欠損可|
|Warning Index|警報XML|集計|欠損可|
|Forecast|台風予報|各予報時刻|実況のみ|

## I.4 Derived Variables

XMLから直接取得しない項目。

### Translation Speed

連続する解析時刻の位置変化から算出。

### Pressure Change

24時間差分。

### Radius Expansion

24時間差分。

### PIKE

最大風速と暴風域から算出。

### TSI

暴風域と強風域から算出。

### Compound Hazard

各危険度から算出。

## I.5 XML Validation Rules

解析前に以下を検証する。

- XML Schema
- XML Version
- Namespace
- 発表時刻
- 台風番号
- 単位
- 数値範囲
- 欠損値

異常時は ParseError を返す。

## I.6 XML Cache

同一電文を再解析しない。キャッシュキー

```
台風番号
+
発表時刻
+
電文ID
```

## I.7 Forecast Handling

Forecast XMLが存在する場合各予報時刻についてTHIを再計算する。

```
0h

6h

12h

24h

48h

72h
```

（取得可能な予報時刻に従う）

## I.8 Missing Policy

必須値欠損 → THI停止、任意値欠損 → Confidence低下・DQI低下・計算継続

## I.9 Logging

全処理を記録する。保存内容

- XML取得日時
- XML発表時刻
- XML種別
- XML Version
- THI Version
- Config Version
- Parser Version

## I.10 Output Metadata

JSONへ追加する。

```json
{
  "xml":{
      "issue_time":"",
      "report_id":"",
      "typhoon_no":"",
      "parser_version":"",
      "xml_version":""
  }
}
```

---

# Appendix J. Python Implementation Standard

## J.1 Design Principles

実装は完全なオブジェクト指向とする。各危険度は独立クラスとする。

## J.2 Core Classes

```text
TyphoonParser

FeatureGenerator

HazardModel

PIKEModel

TSIModel

CompoundModel

THIModel

ForecastModel

SimilarityModel

Trainer

Optimizer

Reporter
```

## J.3 Class Dependency

```text
Parser
    │
    ▼
FeatureGenerator
    │
    ▼
HazardModels
    │
    ▼
CompoundModel
    │
    ▼
THIModel
    │
    ▼
Reporter
```

## J.4 Standard API

```python
engine = THI()

result = engine.calculate(xml_path)

forecast = engine.forecast(xml_path)

engine.train(dataset)

engine.optimize(dataset)

engine.export_json(result)

engine.export_markdown(result)

engine.export_pdf(result)
```

## J.5 Internal Data Model

```
TyphoonFeature → HazardVector → THIResult
```

## J.6 Unit Testing

各モジュールは独立して試験する。最低限実施する試験。

- XML Parser
- Feature Generator
- Hazard Function
- PIKE
- TSI
- THI
- Forecast
- JSON Export

## J.7 Continuous Integration

推奨。GitHub Actions。実施内容。

- Lint
- Unit Test
- Type Check
- Coverage
- Documentation Build

## J.8 Configuration

すべてJSON管理。

- 学習済み係数
- シグモイドパラメータ
- 閾値
- Version

コードへのハードコーディングは禁止。

## J.9 Documentation

ソースコードから自動生成する。推奨。

- Sphinx
- MkDocs

---

# Appendix K. Theoretical Basis of the THI Mathematical Model

## K.1 Purpose

本付録では、THI（Typhoon Hazard Index）で採用した各数式について、その物理学的・気象学的・統計学的根拠を示す。THIは経験則のみに基づく指標ではなく、既存の気象学および統計学の知見を統合した Physics-informed Statistical Model として設計されている。

## K.2 Pressure Hazard

### Physical Basis

最低中心気圧は台風の発達度を表す代表的な物理量である。一般に中心気圧が低いほど気圧傾度が大きくなり、強風の発生しやすい環境となる。ただし、同一気圧であっても台風サイズや周囲の環境場によって実際の危険度は異なるため、THIでは中心気圧のみで危険度を決定しない。

### Mathematical Basis

中心気圧と被害との関係は線形ではない。そのためシグモイド関数により「ある閾値を超えると急激に危険度が増加する」という性質を表現する。

## K.3 Wind Hazard

### Physical Basis

風による被害エネルギーは概ね風速の二乗に比例する。建築物被害、倒木、飛散物、高潮などは最大風速と強く関係する。

### Mathematical Basis

THIでは風速を直接利用するのではなく危険度へ正規化する。これにより学習による最適化が容易になる。

## K.4 Rainfall Hazard

### Physical Basis

近年の日本では風害より豪雨災害の割合が増加している。線状降水帯、土砂災害、河川氾濫は降水量と強い関係を持つ。そのためRain Hazardを主要危険要素として扱う。

## K.5 Translation Hazard

### Physical Basis

移動速度が遅い台風ほど同一地域へ長時間降雨をもたらす。総降水量は滞留時間と密接に関係する。THIでは移動速度の逆数を利用する。

## K.6 Storm Size Hazard

### Physical Basis

大型台風は暴風継続時間、停電範囲、高潮範囲などを増大させる。暴風域半径、強風域半径は台風の空間スケールを表す。

## K.7 PIKE

### Physical Basis

厳密IKEは風速場全体を積分する。しかし気象庁XMLでは風速分布が得られない。そのためTHIではPIKEを導入し簡易的に運動エネルギーを近似する。これはIKEそのものではなくIKE Proxyである。

## K.8 TSI

### Physical Basis

同じ最大風速でも大型台風と小型台風では社会への影響範囲が異なる。TSIは空間スケールを補正する目的で導入する。

## K.9 Compound Hazard

### Statistical Basis

実災害では単一要因より複数要因の同時発生が危険となる。例えば豪雨×停滞、豪雨×広域暴風などである。THIでは学習により有意な組み合わせのみ採用する。

## K.10 Geometric Mean

### Statistical Basis

算術平均では一部の極端値が平均化される。幾何平均は全要素が高い場合のみ高い危険度となる。THIでは基礎危険度として採用する。

## K.11 Maximum Hazard

### Statistical Basis

極端災害では最大要因が被害を支配する場合がある。そのためMaximum Hazardを導入しExtreme判定を補正する。

## K.12 Sigmoid Function

### Statistical Basis

シグモイド関数は確率論、機械学習、生存時間解析。などで広く用いられる。危険度を0〜1へ正規化できる。またμσを学習可能である。

## K.13 Parameter Learning

THIは数式を固定しパラメータのみ学習する。これはPhysics-informed Machine Learningの考え方に基づく。物理法則とデータ駆動型学習の両立を目的とする。

## K.14 Scope

THIは自然ハザード指標である。人口、経済、インフラ、社会脆弱性は評価対象外とする。これらは別のRisk Modelで扱う。

## K.15 Scientific Position

THIは

- 気象学
- 流体力学
- 災害科学
- 数理統計学
- 機械学習

を統合したPhysics-informed Statistical Hazard Modelとして位置付けられる。その目的は、**気象庁XMLのみから、台風そのものの危険度を客観的かつ再現性高く評価すること**である。

---

# Appendix L. Limitations, Assumptions, and Future Development

## L.1 Purpose

本付録では、THI Version 1.0 の適用範囲、前提条件、既知の制約、および今後の発展計画を定義する。本章は、本指標の適切な利用と学術的な透明性を確保するために設ける。

## L.2 Scope of Application

THIは以下の現象を対象とする。

- 台風
- 熱帯低気圧
- 台風から変化した温帯低気圧（気象庁XMLで追跡可能な場合）

THIは「台風という気象現象そのものの危険度」を評価する指標であり、社会的脆弱性や地域特性は評価対象外とする。

## L.3 Fundamental Assumptions

Version 1.0では以下を前提とする。

### 1. 気象庁XMLが唯一の入力である

追加観測データを必要としない。

### 2. 数式は固定する

変更対象は

- パラメータ
- 重み
- 閾値

のみである。

### 3. 危険度は確率ではない

THIはProbabilityではなくHazard Indexである。

### 4. 被害予測モデルではない

THIは被害額、死者数を直接予測するモデルではない。自然ハザードを評価する。

## L.4 Known Limitations

Version 1.0には以下の制約が存在する。

### 海面水温

直接利用しない。

### 海洋熱含量

利用しない。

### 鉛直風シア

利用しない。

### 地形効果

直接モデル化しない。

### 潮位

高潮モデルは含まない。

### 波浪

評価対象外。

### 人口密度

評価対象外。

### インフラ

評価対象外。

### 避難率

評価対象外。

### 経済被害

学習には利用できるが入力には利用しない。

## L.5 Advantages

Version 1.0の利点。

- 気象庁XMLのみで完結
- 世界中へ適用可能
- リアルタイム処理可能
- Explainable AI対応
- 自己学習可能
- 高い再現性
- 実装が比較的容易

## L.6 Future Versions

### Version 2

追加候補

- 海面水温
- 海洋熱含量
- 鉛直風シア
- 衛星解析
- レーダ解析

### Version 3

追加候補

- AIアンサンブル
- Deep Learning
- PINNs
- Graph Neural Network

### Version 4

追加候補

- 全球リアルタイム解析
- マルチモデル統合
- 自動論文生成
- 完全自動パラメータ更新

## L.7 Compatibility

THIはVersion 1系ではAPI互換性、JSON互換性、数式互換性を維持する。

## L.8 Validation Policy

Version更新時には必ず以下を実施する。

- 全過去台風再評価
- ROC評価
- Recall評価
- 被害との比較
- パラメータ比較
- 回帰試験

## L.9 Ethical Considerations

THIは防災支援を目的とした補助指標である。行政機関の避難情報や気象庁の公式発表を代替するものではない。利用者は公式な防災情報と併せて利用することを前提とする。

## L.10 Open Science Policy

THIは以下を推奨する。

- 数式公開
- パラメータ公開
- バージョン公開
- 検証結果公開
- 学習方法公開

可能な限り再現可能な研究として公開する。

## L.11 Final Definition

THI（Typhoon Hazard Index）Version 1.0 は、**気象庁XMLのみを入力として、台風の自然ハザードを物理学・気象学・統計学・機械学習の融合により統合評価する、再現性・説明可能性・自己学習性を備えた危険度指標**である。本仕様書をもって、THI Version 1.0 Core Specification を正式仕様と定義する。

---

# Appendix M. Benchmark and Verification Protocol

## M.1 Purpose

本付録では、THI（Typhoon Hazard Index）の性能を客観的かつ再現可能な方法で検証するための評価プロトコルを定義する。目的は、

- 再現可能な検証
- 他指標との比較
- 統計学的有意性の評価
- 年次性能監視

を標準化することである。

## M.2 Benchmark Objectives

THIは以下を検証する。

1. 危険な台風を正しく高評価できるか
2. 被害との相関が既存指標より高いか
3. 過小評価を減らせるか
4. 年度が変わっても性能を維持できるか

## M.3 Benchmark Dataset

検証データは学習データとは独立させる。推奨期間

- 学習：1991–2018
- 検証：2019–2022
- テスト：2023以降

（利用可能なデータ量に応じて調整）

## M.4 Baseline Models

THIは以下と比較する。

### Single Variable

- 最低中心気圧
- 最大風速
- 暴風域半径
- 強風域半径
- 降水量

### Existing Hazard Metrics

- Saffir–Simpson Hurricane Wind Scale
- Integrated Kinetic Energy（IKE）
- Accumulated Cyclone Energy（ACE）
- Power Dissipation Index（PDI）

※ IKE・ACE・PDIは比較対象であり、THIの入力ではない。

## M.5 Regression Evaluation

目的変数を連続値とする場合、評価指標は以下とする。

- RMSE
- MAE
- R²
- Spearman順位相関
- Pearson相関

## M.6 Classification Evaluation

5段階分類では以下を用いる。

- Accuracy
- Precision
- Recall
- F1 Score
- ROC-AUC
- PR-AUC
- Balanced Accuracy
- Matthews Correlation Coefficient

## M.7 Calibration Evaluation

THIが危険度指標として一貫性を持つか確認する。使用指標

- Reliability Curve
- Calibration Error
- Brier Score

## M.8 Robustness Test

入力値へ観測誤差を付与する。例

- 気圧 ±2 hPa
- 最大風速 ±2 m/s
- 半径 ±10 km

Monte Carlo Simulation により、THIの安定性を評価する。

## M.9 Sensitivity Analysis

各特徴量について一変数のみ変化させる。

$$
S_i = \frac{\partial THI}{\partial x_i}
$$

を評価する。これにより支配的要因を確認する。

## M.10 Ablation Study

各モジュールを除外して性能を比較する。例

- PIKEなし
- Rain Hazardなし
- Compound Hazardなし
- Forecastなし

性能低下量を測定し、各要素の有効性を定量化する。

## M.11 Statistical Significance

THIと比較モデルとの差について、統計学的検定を行う。推奨手法

- Wilcoxon signed-rank test
- McNemar test
- Bootstrap法による信頼区間

採用する検定は評価対象に応じて選択する。

## M.12 Annual Monitoring

毎年以下を比較する。

- ROC-AUC
- Recall
- RMSE
- パラメータ変化量
- DQI分布

性能低下が認められた場合は、パラメータ再学習を実施する。

## M.13 Reproducibility Checklist

検証結果には以下を必ず記録する。

- THI Version
- Config Version
- 学習データ期間
- 検証データ期間
- XML取得日時
- Python Version
- NumPy Version
- SciPy Version
- scikit-learn Version
- Optuna Version
- Git Commit ID（任意）

## M.14 Acceptance Criteria

THI Version 1.0を正式運用するためには、以下を満たすことを目標とする。

- ベースライン指標以上の分類性能
- 過小評価率の低減
- 年度をまたいだ性能の安定性
- モデルの説明可能性
- 同一入力に対する再現性

## M.15 Conclusion

本プロトコルは、THIの性能を客観的・再現可能に評価するための標準手順である。すべての学習・検証・バージョン更新は、本プロトコルに従って実施するものとする。

---

# Appendix N. Learning and Parameter Optimization Specification

## N.1 Purpose

本付録では、THI Version 1.0 の学習方法、パラメータ推定手法、およびモデル更新手順を定義する。THIでは**数式構造は固定**し、学習対象はパラメータのみとする。

## N.2 Learning Targets

学習対象は以下のパラメータである。

### Hazard Function

- μ（中心値）
- σ（傾き）

### Weight

- β（Maximum Hazard係数）
- α（Compound係数）
- PIKE補正係数
- TSI補正係数

### Category

- Minimal
- Minor
- Moderate
- Significant
- Extreme

各境界値

## N.3 Fixed Components

Version 1.0では変更しない。

- 数式構造
- 入力変数
- PIKE定義
- TSI定義
- 幾何平均
- Compound構造
- Explainable Output

## N.4 Objective Variable

推奨目的変数。

### 第一候補

被害額（対数変換）

### 第二候補

住家被害件数

### 第三候補

人的被害

### 第四候補

警報継続時間。複数目的変数によるマルチタスク学習も可能とする。

## N.5 Loss Function

Version 1.0では、過小評価を重く罰する非対称損失関数を採用する。概念式

$$
L= \begin{cases} \lambda (y-\hat y)^2 & (y>\hat y) \\ (y-\hat y)^2 & (y\le\hat y) \end{cases}
$$

ここで

$$
\lambda>1
$$

とする。λも学習対象とする。

## N.6 Optimization Algorithms

推奨順位。

1. Optuna
2. Bayesian Optimization
3. CMA-ES
4. Grid Search

Grid Searchはベースライン評価のみ。

## N.7 Cross Validation

推奨。Time Series Cross Validation。通常のK-foldは使用しない。

## N.8 Regularization

推奨。

- L1
- Elastic Net

不要特徴量を抑制する。

## N.9 Early Stopping

Validation Lossが改善しない場合、学習終了。Patienceは設定ファイルで管理する。

## N.10 Hyperparameter Storage

すべてJSON保存。例

```json
{
  "mu_pressure":945.3,
  "sigma_pressure":10.2,
  "mu_wind":42.5,
  "sigma_wind":7.4,
  "beta":0.31
}
```

## N.11 Version Control

学習ごとに

- Version
- 学習日
- データ期間
- ハッシュ値

を保存する。

## N.12 Model Comparison

新旧モデルを比較する。更新条件。

- Recall改善
- AUC改善
- RMSE改善
- 過小評価率減少

すべて満たす場合のみ更新。

## N.13 Explainability Verification

学習後、SHAP等を用いて各特徴量の寄与を確認する。PIKE、TSI、Rain、Windなどの寄与が極端に不自然な場合、学習結果を採用しない。

## N.14 Parameter Drift

毎年、前年度との差を評価する。急激な変化は気候変動またはデータ異常を疑う。

## N.15 Final Learning Policy

THIは**数式は固定し、パラメータのみを継続的に最適化する Physics-informed Machine Learning モデル**として運用する。これにより、

- 物理的整合性
- 説明可能性
- 再現性
- 長期的な性能向上

を同時に実現する。

### 推奨ディレクトリ構成

```
thi/
│
├── main.py                  # 実行エントリ
├── config.py                # パラメータ管理
├── constants.py             # 定数
│
├── parser/
│   ├── jma_typhoon.py
│   ├── jma_warning.py
│   └── rainfall.py
│
├── feature/
│   ├── generator.py
│   ├── movement.py
│   ├── pressure.py
│   ├── radius.py
│   ├── rainfall.py
│   ├── pike.py
│   └── tsi.py
│
├── model/
│   ├── sigmoid.py
│   ├── hazards.py
│   ├── compound.py
│   ├── thi.py
│   ├── forecast.py
│   └── confidence.py
│
├── learning/
│   ├── dataset.py
│   ├── objective.py
│   ├── trainer.py
│   ├── optimizer.py
│   └── evaluate.py
│
├── report/
│   ├── json_export.py
│   ├── markdown.py
│   └── explain.py
│
├── utils/
│   ├── geo.py
│   ├── math.py
│   ├── logger.py
│   └── cache.py
│
└── tests/
```

### 実装順序

### STEP1（最優先）

気象庁XML → Feature抽出 → THI算出 → JSON出力

### STEP2

Forecast、PIKE、TSI、Confidence

### STEP3

学習機能、Optuna、SHAP、非対称損失

### STEP4

GUI、REST API、CLI

### Feature Generator

まずXMLから

```
pressure

wind

storm_radius

gale_radius

latitude

longitude

time

rainfall

warning
```

を生成します。その後

```
translation_speed

pressure_drop

radius_growth

PIKE

TSI
```

を計算します。

### THI計算

最後に

```
Pressure Hazard → Wind Hazard → Rain Hazard → Translation Hazard → PIKE → TSI
→ Compound Hazard → Geometric Mean → Maximum Hazard → THI → Category → Confidence → JSON
```

### JSON出力例

```
{
  "typhoon": {
    "number": 12,
    "name": "MEKKHALA"
  },
  "time": "2026-07-04T09:00:00+09:00",
  "thi": 0.74,
  "category": "Significant",
  "hazards": {
    "pressure": 0.88,
    "wind": 0.76,
    "rain": 0.53,
    "translation": 0.61,
    "pike": 0.72,
    "tsi": 0.58
  },
  "confidence": 0.98,
  "dqi": 1.00
}
```

### 実装方針

このプロジェクトは最終的に数千行規模になるため、**単一ファイルではなくモジュール構成で実装**することを推奨します。まずは **Version 1.0 Core** として、

- 気象庁XMLパーサ
- Feature Generator
- THI計算エンジン
- JSON出力

までを完成させ、その後に学習機能やOptunaによるパラメータ最適化を追加する構成が最も保守性と拡張性に優れています。

次に、実装は**一括ではなく土台から順番に構築**するのが最も確実です。最初に実装すべきなのは **共通データモデル** です。これを先に作ることで、その後のXMLパーサ・THI計算・学習器がすべて同じデータ構造を利用できます。

### 1\. 共通データクラス（`models.py`）

```
from dataclasses import dataclass
from datetime import datetime
from typing import Optional

@dataclass
class TyphoonFeature:
    number: int
    name: str

    report_time: datetime

    latitude: float
    longitude: float

    pressure: float          # hPa
    wind: float              # m/s

    storm_radius: float      # km
    gale_radius: float       # km

    rainfall: Optional[float] = None

    translation_speed: Optional[float] = None

    pressure_change_24h: Optional[float] = None

    radius_growth_24h: Optional[float] = None

    warning_index: Optional[float] = None
```

### 2\. THI結果クラス

```
from dataclasses import dataclass

@dataclass
class THIResult:

    thi: float

    category: str

    pressure_hazard: float

    wind_hazard: float

    rain_hazard: float

    translation_hazard: float

    pike_hazard: float

    tsi_hazard: float

    compound_hazard: float

    confidence: float

    dqi: float
```

### 3\. カテゴリー判定

```
CATEGORY = [

    (0.20, "Minimal"),

    (0.40, "Minor"),

    (0.60, "Moderate"),

    (0.80, "Significant"),

    (1.01, "Extreme")

]

def classify(value):

    for limit, name in CATEGORY:

        if value < limit:

            return name

    return "Extreme"
```

### 4\. シグモイド関数

Version 1.0では、すべての危険度はこの関数に統一します。

```
import math

def sigmoid(x, mu, sigma):

    return 1.0 / (

        1.0 +

        math.exp(-(x - mu) / sigma)

    )
```

中心気圧のように「小さいほど危険」な指標は、

```
def inverse_sigmoid(x, mu, sigma):
    return 1.0 - sigmoid(x, mu, sigma)
```

を使用します。

### 5\. PIKE

```
import math

def calc_pike(vmax, storm_radius):

    raw = vmax ** 2 * storm_radius ** 2

    return math.log1p(raw)
```

### 6\. TSI

```
def calc_tsi(storm_radius, gale_radius, alpha=0.5):

    return storm_radius + alpha * gale_radius
```

### 7\. Feature Generator

このモジュールでXMLから取得した値を基に、

- PIKE
- TSI
- 移動速度
- 24時間気圧変化
- 暴風域拡大率

など、THIで必要な派生特徴量を一括生成します。ここまでが **THI Engineの基盤** です。**次に実装すべきは `jma_typhoon.py`（気象庁XMLパーサ）** です。このモジュールでXMLから `TyphoonFeature` を生成できれば、THI計算エンジンへ直接入力できるようになります。

---

# Appendix O. Mathematical Symbols and Variable Definitions

## O.1 Purpose

本章では、THI（Typhoon Hazard Index）で使用するすべての記号・変数・関数を定義する。本仕様書では同一記号は常に同じ意味を持つものとする。

## O.2 Physical Variables

|Symbol|Definition|Unit|
|-------|----------|----|
|$P_{min}$|最低中心気圧|hPa|
|$V_{max}$|最大風速|m/s|
|$R_{25}$|暴風域半径|km|
|$R_{15}$|強風域半径|km|
|$Rain$|降水量|mm|
|$V_{trans}$|移動速度|km/h|
|$\Delta P$|24時間気圧変化|hPa|
|$\Delta R$|24時間暴風域変化|km|

## O.3 Hazard Functions

|Symbol|Meaning|
|--------|-------|
|$I_P$|Pressure Hazard|
|$I_V$|Wind Hazard|
|$I_B$|Storm Radius Hazard|
|$I_G$|Gale Radius Hazard|
|$I_R$|Rainfall Hazard|
|$I_T$|Translation Hazard|
|$I_D$|Rapid Intensification Hazard|
|$I_E$|Storm Expansion Hazard|
|$I_W$|Warning Hazard|
|$I_{PIKE}$|Pseudo IKE Hazard|
|$I_{TSI}$|Typhoon Size Hazard|

## O.4 Composite Variables

|Symbol|Meaning|
|--------|-------|
|$H$|Base Hazard|
|$S$|Compound Hazard|
|$M$|Maximum Hazard|
|$THI$|Typhoon Hazard Index|
|$THI_F$|Forecast THI|
|$TSI$|Typhoon Size Index|
|$PIKE$|Pseudo Integrated Kinetic Energy|
|$FHM$|Forecast Hazard Modifier|
|$LHI$|Lifetime Hazard Index|
|$FSI$|Forecast Stability Index|

## O.5 Quality Indicators

|Symbol|Meaning|
|--------|-------|
|Confidence|入力情報の完全性|
|DQI|Data Quality Index|
|Version|THI仕様バージョン|

---

# Appendix P. Recommended Initial Parameters

Version 1.0では、すべてのシグモイドパラメータは**初期値**として設定し、学習によって更新する。

|Parameter|Description|
|----------|-----------|
|$\mu_P$|中心気圧の中心値|
|$\sigma_P$|中心気圧の傾き|
|$\mu_V$|最大風速の中心値|
|$\sigma_V$|最大風速の傾き|
|$\mu_R$|降水量の中心値|
|$\sigma_R$|降水量の傾き|
|$\mu_T$|移動速度の中心値|
|$\sigma_T$|移動速度の傾き|
|$\mu_{PIKE}$|PIKE中心値|
|$\sigma_{PIKE}$|PIKE傾き|

これらは固定値ではなく、年次再学習により更新する。
