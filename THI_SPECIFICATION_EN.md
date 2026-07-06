# THI Specification
## Typhoon Hazard Index

**Version:** 1.0 Draft
**Author:** YNGMT
**Date:** 2026-07-04

---

# Abstract

This specification defines the "Typhoon Hazard Index (THI)", a composite typhoon hazard indicator that can be calculated in real time using only Japan Meteorological Agency (JMA) XML bulletins.

THI is not a measure of typhoon "intensity" in the traditional sense. It integrates wind, heavy rainfall, translation speed, rapid intensification, storm-radius expansion, and warning information in order to evaluate the potential societal hazard posed by a typhoon on a five-level scale: Minimal / Minor / Moderate / Significant / Extreme.

This specification also defines an extensible architecture that accounts for explainable AI, self-learning, statistical parameter optimization, forecast projection, historical-similarity search, and uncertainty evaluation.

---

# 1. Research Background

Conventional typhoon evaluation relies on single factors such as central pressure, maximum wind speed, or storm size (large / very large). In practice, however, there are many cases that cannot be assessed with a single factor alone — for example, heavy-rainfall disasters caused by small typhoons, linear precipitation bands triggered despite a small storm radius, damage amplified by slow translation speed, and cases of rapid intensification.

THI evaluates "disaster potential" rather than the typhoon itself.

---

# 2. Design Philosophy

THI is designed to satisfy the following goals:

1. Real-time computability using JMA XML alone
2. A mathematically consistent model
3. Statistical optimizability
4. Support for Explainable AI
5. Support for future self-learning
6. Applicability to typhoons worldwide
7. Prioritization of disaster-prevention utility

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

Let $X$ denote the explanatory variable vector used by THI.

$$
X = [P_{min},\ V_{max},\ R_{25},\ R_{15},\ Rain_{24},\ V_{trans},\ \Delta P_{24},\ \Delta R_{25},\ WarningLevel]
$$

| Variable | Definition |
|---|---|
| $P_{min}$ | Minimum central pressure [hPa] |
| $V_{max}$ | Maximum wind speed [m/s] |
| $R_{25}$ | Storm (gale-force) radius [km] |
| $R_{15}$ | Gale (strong-wind) radius [km] |
| $Rain_{24}$ | 24-hour precipitation [mm] |
| $V_{trans}$ | Translation speed [km/h] |
| $\Delta P_{24}$ | 24-hour central-pressure change [hPa] |
| $\Delta R_{25}$ | 24-hour storm-radius change [km] |
| $WarningLevel$ | Warning index |

---

# 5. Hazard Functions

Each input variable is transformed into a hazard function $I_i(x)$. The general form is:

$$
I_i(x) = \frac{1}{1+\exp\left(-d_i\dfrac{x-\mu_i}{\sigma_i}\right)}
$$

where

- $\mu$: the center value at which hazard reaches 50%
- $\sigma$: the width of the rising slope
- $d$: the hazard direction ($+1$: hazard increases with larger values, $-1$: hazard increases with smaller values)

The hazard direction $d$ for each variable is as follows:

| Variable | $d$ |
|---|---|
| Central pressure | $-1$ |
| Maximum wind speed | $+1$ |
| Precipitation | $+1$ |
| Translation speed | $-1$ |
| Storm radius | $+1$ |
| Gale radius | $+1$ |
| Rapid intensification | $+1$ |
| Storm-radius expansion | $+1$ |
| Warning | $+1$ |

These $\mu$ and $\sigma$ values are not fixed; they are re-learned every year from historical typhoon data and actual damage data. This makes THI a statistically optimized index rather than a rule-of-thumb heuristic.

---

# 6. THI Integration Model

## 6.1 Overview

Individual hazard functions are not simply summed. The conventional weighted linear sum

$$
THI=\sum w_iI_i
$$

fails to adequately evaluate cases where only a single element is extremely hazardous, or conversely where multiple elements are simultaneously hazardous. This study adopts the **geometric mean** for integration.

## 6.2 Base Hazard

Given the hazard vector

$$
\mathbf I= [I_1,I_2,\cdots,I_n]
$$

the base hazard is defined as

$$
H = \left( \prod_{i=1}^{n} I_i \right)^{1/n}
$$

### Rationale for This Choice

The geometric mean has the following properties:

- It naturally integrates the hazard of all elements
- The overall value drops if any single element is extremely low
- The value rises sharply when multiple elements become hazardous simultaneously

These properties are well suited to expressing the compound nature of disaster phenomena.

## 6.3 Maximum Hazard

On the other hand, there are cases where a single extreme element — extreme rainfall alone, or extreme wind alone — is sufficient to cause a severe disaster. For this reason we define

$$
M = \max(I_i)
$$

which is used as an Extreme correction term.

## 6.4 Final THI

The final hazard value is defined as

$$
THI = (1-\beta)H + \beta M
$$

where $\beta$ is the Extreme correction coefficient, satisfying

$$
0\le\beta\le1
$$

### Determining $\beta$

$\beta$ is not a fixed value. It is re-optimized every year using historical typhoon data together with human casualties, property damage, power outages, and flooding as training data.

## 6.5 Normalization

THI is always clipped so that

$$
0\le THI\le1
$$

$$
THI = \min(1,\max(0,THI))
$$

---

# 7. Forecast Engine

## 7.1 Purpose

Current conditions alone are insufficient for disaster prevention. For this reason, forecast data is evaluated using the same algorithm.

## 7.2 Forecast Time

THI is calculated independently for each of the following times:

- Current conditions
- +6 hours
- +12 hours
- +24 hours
- +48 hours
- +72 hours
- +96 hours
- +120 hours

## 7.3 Future Hazard

For each time,

$$
THI(t)
$$

is computed, and the forecast hazard is defined as

$$
THI_{forecast} = \max_t THI(t)
$$

## 7.4 Forecast Trend

The rate of change of hazard,

$$
\frac{dTHI}{dt}
$$

is calculated. A larger value indicates that hazard is increasing more rapidly.

### Trend Classification

Rapidly Increasing, Increasing, Stable, Weakening, Rapidly Weakening

---

# 8. Confidence Engine

## 8.1 Purpose

The amount of information available in JMA XML bulletins varies by message. Missing information must be accounted for.

## 8.2 Data Mask

For each explanatory variable,

$$
M_i = \begin{cases} 1 & \text{available}\\ 0 & \text{missing} \end{cases}
$$

is defined.

## 8.3 Confidence

$$
Confidence = \frac {\sum w_iM_i} {\sum w_i}
$$

Confidence is expressed on a 0–1 scale and converted to a percentage for display. Example: 99%, 95%, 82%.

### Confidence Rank

| Confidence | Rank |
|------------|------|
|95% or above|Excellent|
|90–95|Very High|
|80–90|High|
|70–80|Moderate|
|Below 70|Low|

---

# 9. Data Quality Engine

Independently of Confidence, input data quality is also evaluated. The Data Quality Index (DQI) is defined as

$$
DQI = \frac {\sum M_iw_i} {\sum w_i}
$$

DQI accounts for XML retrieval failures, missing fields, and bulletin delays. DQI is displayed alongside THI at calculation time.

Example: THI 0.82 (Extreme), Confidence 96%, Data Quality 98% (Excellent)

---

# 10. Hazard Engine

## 10.1 Overview

The Hazard Engine is the core computation module of THI. It converts each meteorological element into an independent hazard function and passes the results to the THI Integration Model. Because each module is implemented independently, future refinement, addition, or removal is straightforward.

## 10.2 Pressure Hazard Module

### Input

Minimum central pressure

$$
P_{min}
$$

### Hazard Function

$$
I_P= \frac 1 {1+\exp \left( \frac{P-\mu_P} {\sigma_P} \right)}
$$

where

- $\mu_P$: the learned center value
- $\sigma_P$: the learned slope

### Characteristics

- Hazard increases as pressure decreases
- Does not easily saturate even for extremely large typhoons
- Can adapt to regional characteristics through learning

## 10.3 Wind Hazard Module

Input: maximum wind speed

$$
V_{max}
$$

Hazard:

$$
I_V= \frac1 {1+\exp \left( -\frac {V-\mu_V} {\sigma_V} \right)}
$$

Characteristics: represents

- Storm damage
- Wind-borne debris
- Structural damage

## 10.4 Storm Radius Module

Input: storm (gale-force) radius

$$
R_{25}
$$

Hazard:

$$
I_B= \frac1 {1+\exp \left( -\frac {R_{25}-\mu_B} {\sigma_B} \right)}
$$

A wider storm radius increases the area affected by damage.

## 10.5 Gale Radius Module

Input: gale (strong-wind) radius

$$
R_{15}
$$

Hazard:

$$
I_G= \frac1 {1+\exp \left( -\frac {R_{15}-\mu_G} {\sigma_G} \right)}
$$

The gale radius reflects widespread power outages, transportation disruption, and agricultural damage.

## 10.6 Rainfall Hazard Module

### Input

24-hour precipitation

$$
Rain_{24}
$$

Hazard:

$$
I_R= \frac1 {1+\exp \left( -\frac {Rain-\mu_R} {\sigma_R} \right)}
$$

### Characteristics

This model evaluates heavy rainfall on equal footing with wind. It accounts for cases where rainfall disasters occur even when the storm radius is small.

## 10.7 Translation Speed Module

Input: translation speed

$$
V_{trans}
$$

Hazard:

$$
I_T= \frac1 {1+\exp \left( \frac {V_{trans}-\mu_T} {\sigma_T} \right)}
$$

Characteristics: the slower the translation speed, the longer the duration of rainfall over the same area, and hazard therefore increases.

## 10.8 Rapid Intensification Module

24-hour central-pressure change

$$
\Delta P_{24}
$$

Hazard:

$$
I_D= \frac1 {1+\exp \left( -\frac {\Delta P-\mu_D} {\sigma_D} \right)}
$$

Typhoons undergoing rapid intensification are rated more highly.

## 10.9 Storm Expansion Module

Input: 24-hour storm-radius expansion

$$
\Delta R_{25}
$$

Hazard:

$$
I_E= \frac1 {1+\exp \left( -\frac {\Delta R-\mu_E} {\sigma_E} \right)}
$$

This evaluates typhoons that are growing in size.

## 10.10 Warning Module

JMA-issued warning information is converted into an index.

|Information|Index|
|----|----|
|None|0.00|
|Advisory|0.25|
|Warning|0.50|
|Landslide Warning|0.75|
|Special Warning|1.00|

This is defined as

$$
I_W
$$

Note: this module is valid only within Japan. For sea areas outside Japan it is treated as a missing value and reflected in Confidence.

## 10.11 Compound Hazard Module

Disasters worsen sharply when multiple elements occur simultaneously. For this reason, interaction terms are introduced.

### Heavy Rain × Stagnation

$$
C_1 = I_R \times I_T
$$

Represents prolonged rainfall.

### Wind × Storm Radius

$$
C_2 = I_V \times I_B
$$

Represents widespread wind damage.

### Rapid Intensification × Heavy Rain

$$
C_3 = I_D \times I_R
$$

Emphasizes rainfall risk immediately following intensification.

### Rapid Intensification × Wind

$$
C_4 = I_D \times I_V
$$

Accounts for cases where hazard increases sharply within a short time.

### Heavy Rain × Warning

$$
C_5 = I_R \times I_W
$$

Corrects for actual disaster-prevention hazard.

## 10.12 Composite Hazard Score

Compound hazard:

$$
S = \sum_{k=1}^{m} \alpha_k C_k
$$

where $\alpha_k$ is subject to learning.

## 10.13 Total Hazard

The base hazard

$$
H
$$

and the compound hazard

$$
S
$$

are integrated, and the final output of the Hazard Engine is defined as

$$
H_{total} = H+S
$$

This value is passed to the THI Integration Model described in the previous chapter to compute the final THI.

---

# 11. Classification Engine

## 11.1 Purpose

THI is a continuous value. However, since disaster-prevention operations require an intuitive hazard display, THI is classified into five levels.

## 11.2 Hazard Classes

| THI | Category |
|------|----------|
|0.00–0.20|Minimal|
|0.20–0.40|Minor|
|0.40–0.60|Moderate|
|0.60–0.80|Significant|
|0.80–1.00|Extreme|

## 11.3 Threshold Optimization

The thresholds T₁, T₂, T₃, T₄ are not fixed values. They are re-optimized every year using historical typhoon data and actual disaster records as training data, maximizing ROC analysis, F1 Score, Recall, and Precision.

## 11.4 Dynamic Classification

Extreme is not determined by a simple threshold test alone. A typhoon is promoted to Extreme if any of the following hold:

- THI ≥ T₄
- Maximum Hazard is 0.98 or above
- Forecast THI is Extreme
- The compound hazard $S$ exceeds a learned threshold

---

# 12. Dominant Hazard Engine

## Purpose

THI alone does not reveal the type of hazard involved. For this reason, the most dominant hazard element is displayed alongside THI.

## Definition

For the hazard functions

$$
I_i
$$

we define

$$
Dominant = \arg\max(I_i)
$$

## Output Example

Wind Dominant, Rain Dominant, Large Storm Dominant, Slow Translation Dominant, Rapid Intensification Dominant, Compound Hazard

## Compound Hazard

If the difference between the top two elements is within a set threshold, the result is displayed as Compound Hazard. Example: Rain 0.91, Wind 0.89 → Compound Hazard

---

# 13. Hazard Contribution Analysis

To improve the explainability of THI, the contribution rate of each element is calculated.

## Contribution

Contribution rate:

$$
C_i = \frac{I_i} {\sum I_i}
$$

Display example:

| Hazard | Contribution |
| Hazard | Contribution |
|---------|-------------:|
|Rain|39%|
|Wind|24%|
|Storm Radius|14%|
|Pressure|10%|
|Translation|7%|
|Rapid Intensification|6%|

## Purpose

The purpose of Explainable AI here is:

- Explanation for disaster-prevention officials
- Research use
- Training-data analysis

---

# 14. Forecast Stability Engine

## Purpose

This engine evaluates the stability of the forecast. The spread of forecast values is as important as the current-condition value itself.

## Definition

For the forecast THI series

$$
THI(t)
$$

the standard deviation is defined as

$$
FSI = Std ( THI(t) )
$$

## Interpretation

A small FSI indicates a stable forecast, while a large FSI indicates high uncertainty in the future projection.

## Stability Rank

|FSI|Rank|
|---|----|
|0.00–0.05|Very Stable|
|0.05–0.10|Stable|
|0.10–0.20|Moderate|
|0.20 or above|Unstable|

---

# 15. Lifetime Hazard Engine

## Purpose

Hazard cannot be evaluated from the peak value alone. Typhoons that remain in a hazardous state for a long duration tend to cause greater damage.

## Definition

$$
LHI = \int THI(t)\,dt
$$

In discrete time this becomes

$$
LHI = \sum_i THI_i \Delta t
$$

## Interpretation

Even for the same Extreme classification, LHI differs greatly between a typhoon that lasts half a day and one that lasts five days.

---

# 16. Trend Analysis Engine

## Purpose

This evaluates not only the current hazard level but also its rate of change.

## Definition

$$
Trend = \frac {THI(t+\Delta t)-THI(t)} {\Delta t}
$$

## Classification

Classification: Rapidly Increasing, Increasing, Stable, Weakening, Rapidly Weakening. Trend works together with the Forecast Engine to express future hazard change.

---

# 17. Historical Similarity Engine

## Purpose

This evaluates similarity to past typhoons.

## Feature Vector

The comparison target is

$$
X= [ P, V, R_{25}, R_{15}, Rain, V_{trans}, \Delta P, \Delta R ]
$$

## Similarity

After standardizing the feature values, the Euclidean distance

$$
d = \sqrt{ \sum (x_i-y_i)^2 }
$$

is computed as the baseline metric. Future extensions to Mahalanobis distance or cosine similarity are also possible.

## Output Example

Most Similar Typhoon: Typhoon Hagibis (2019), Similarity 94%; Second: Typhoon Nanmadol, 89%; Third: Typhoon Talas, 86%. The Historical Similarity Engine allows the current typhoon to be objectively compared against past cases to show "which disaster pattern it most closely resembles." This is useful not only for disaster-prevention decision-making but also for model verification and improved explainability.

---

# 18. Uncertainty Engine

## 18.1 Purpose

Typhoon analysis involves both observation error and forecast error. THI presents an estimated error alongside the single hazard value, rather than the value alone.

## 18.2 Sources of Uncertainty

Major sources of uncertainty:

- Center-position error
- Central-pressure estimation error
- Maximum-wind-speed estimation error
- Storm-radius estimation error
- Precipitation forecast error
- Translation-speed forecast error
- Expansion of the forecast (probability) circle

## 18.3 Monte Carlo Simulation

For each input variable

$$
x_i
$$

an observation error

$$
\varepsilon_i
$$

is applied, generating

$$
x_i' = x_i+\varepsilon_i
$$

This operation is repeated N times. Recommended:

$$
N=10000
$$

## 18.4 THI Distribution

THI is calculated for each sample, yielding a distribution

$$
THI_1, THI_2, ... THI_N
$$

## 18.5 Confidence Interval

The 95% confidence interval is defined as

$$
CI_{95} = [ P_{2.5}, P_{97.5} ]
$$

Display example: THI 0.84 (95% CI 0.80–0.88)

---

# 19. Pseudo Integrated Kinetic Energy (PIKE)

## 19.1 Purpose

Integrated Kinetic Energy (IKE) is a representative indicator of the kinetic energy possessed by a typhoon. However, a rigorous calculation of IKE requires the full spatial distribution of the wind field, which cannot be fully reconstructed from JMA XML alone. This specification therefore introduces **Pseudo Integrated Kinetic Energy (PIKE)** as an approximate indicator computable from JMA XML alone. PIKE is not an independent hazard of THI; it is used as a physical indicator that corrects the wind-related hazard.

## 19.2 Physical Background

Kinetic energy is given by

$$
E=\frac12 mV^2
$$

Since typhoon mass cannot be obtained directly, the affected area is treated as a proxy for mass. The affected area can be approximated as

$$
Area \propto R^2
$$

and therefore

$$
PIKE \propto V^2R^2
$$

## 19.3 Basic Definition

In Version 1.0, PIKE is defined as

$$
PIKE = V_{max}^{2} R_{25}^{2}
$$

where

- $V_{max}$: maximum wind speed
- $R_{25}$: storm (gale-force) radius

## 19.4 Extended Definition

When the gale (strong-wind) radius is also considered,

$$
PIKE = V_{max}^{2} (R_{25}+kR_{15})^{2}
$$

where $k$ is subject to learning. An initial value may be set, but in formal operation it is estimated through learning.

## 19.5 Normalization

The value range of PIKE becomes very large, so a logarithmic transform is applied:

$$
PIKE' = \log(1+PIKE)
$$

followed by a sigmoid transform:

$$
I_{PIKE} = \frac 1 { 1+ \exp \left( - \frac {PIKE'-\mu} {\sigma} \right) }
$$

$\mu$ and $\sigma$ are subject to learning.

## 19.6 Integration

PIKE corrects the Wind Hazard:

$$
I_{Wind}^{*} = I_{Wind} \left( 1+ \alpha I_{PIKE} \right)
$$

where $\alpha$ is subject to learning. This allows widespread windstorms to be rated more highly than Wind Hazard alone would indicate.

## 19.7 Relationship with TSI

TSI evaluates the geometric size of a typhoon, while PIKE evaluates its kinetic energy — the two play different roles. TSI is therefore used as a size correction and PIKE as an energy correction.

## 19.8 XML Requirements

All data required for PIKE can be obtained from JMA XML. Required items:

- Maximum wind speed
- Storm radius
- Gale radius (extended version)

No additional data source is required.

## 19.9 Advantages

Introducing PIKE allows the following to be evaluated within a single unified framework:

- Large, strong typhoons
- Small but violent typhoons
- Widespread windstorm types
- Compact, high-wind-speed types

This improves the physical consistency of THI.

## 19.10 Final Hazard Vector

After introducing PIKE, the hazard vector handled by THI is defined as

$$
\mathbf{I} = \left[ I_P, I_V, I_{PIKE}, I_B, I_G, I_R, I_T, I_D, I_E, I_W \right]^T
$$

PIKE is not an independent final score; it is used as a correction term for Wind Hazard.

---

# 20. Typhoon Size Index (TSI)

## 20.1 Purpose

The storm radius and gale radius are each input to THI as independent features. However, evaluating the typhoon's spatial scale as a single index makes it possible to quantify the impact the typhoon has over a wide area. This specification therefore introduces the Typhoon Size Index (TSI).

## 20.2 Definition

TSI is defined as

$$
TSI = w_BR_{25} + w_GR_{15}
$$

where

- $R_{25}$: storm radius
- $R_{15}$: gale radius
- $w_B, w_G$: parameters subject to learning

## 20.3 Normalization

TSI is converted to a hazard value via a sigmoid transform:

$$
I_{TSI} = \frac 1 {1+\exp \left( -\frac {TSI-\mu_{TSI}} {\sigma_{TSI}} \right)}
$$

## 20.4 Purpose (Use Cases)

TSI is used to represent:

- Widespread windstorms
- Widespread power outages
- Prolonged storm conditions
- An approximation of IKE

---

# 21. Development Potential Index (DPI)

## 21.1 Purpose

Current-condition values alone cannot adequately express future increases in hazard. DPI evaluates the potential for future rapid intensification.

## 21.2 Definition

DPI is defined as

$$
DPI = f ( \Delta P, \Delta R, V_{trans} )
$$

where

- $\Delta P$: 24-hour pressure change
- $\Delta R$: 24-hour storm-radius change
- $V_{trans}$: translation speed

## 21.3 Practical Model

In Version 1,

$$
DPI = \alpha I_D + \beta I_E + \gamma I_T
$$

where

- $I_D$: rapid-intensification hazard
- $I_E$: storm-radius-expansion hazard
- $I_T$: stagnation hazard

The coefficients $\alpha, \beta, \gamma$ are subject to learning.

## 21.4 Future Expansion

Future versions could add sea-surface temperature, ocean heat content, vertical wind shear, and similar factors. However, since THI v1.0 Core uses JMA XML only, these are not adopted.

---

# 22. Integration into THI

TSI and DPI are treated as independent hazards. The hazard vector becomes

$$
\mathbf I = [ I_P, I_V, I_B, I_G, I_R, I_T, I_D, I_E, I_{TSI}, I_{DPI}, I_W ]
$$

The subsequent geometric mean, compound hazard, and Extreme correction are all carried out using this extended vector as input.

---

# 23. Forecast Hazard Modifier (FHM)

## 23.1 Purpose

THI is an indicator that evaluates "current hazard." However, in disaster-prevention operations, "whether the hazard is about to increase" is critically important. This specification therefore defines a Forecast Hazard Modifier (FHM) that is independent of the current-condition THI. FHM does not rewrite THI; it is a module that corrects the future hazard.

## 23.2 Definition

FHM is defined as

$$
FHM=f(DPI,THI_{forecast})
$$

which integrates DPI and Forecast THI to evaluate future hazard.

## 23.3 Final Forecast Score

The future hazard is defined as

$$
THI_F = THI + \omega FHM
$$

where

$$
0\le\omega\le1
$$

is subject to learning.

---

# 24. Extreme Override System

## Purpose

Averaging can potentially underestimate extreme disasters. An Override System is therefore introduced.

## Rule A

A typhoon becomes an Extreme candidate if any of the following hold:

- Maximum hazard is 0.98 or above
- Forecast Extreme
- The compound hazard exceeds its threshold
- The rainfall hazard exceeds its threshold

## Rule B

A typhoon is promoted to Extreme if multiple hazard elements are simultaneously high — for example, Rain 0.91, Wind 0.92, Pressure 0.88.

## Rule C

If the forecast indicates Extreme, the result can be displayed as "Early Extreme" even while current conditions are only Significant. This is intended to support evacuation decisions.

---

# 25. Explainable THI

THI must never be a black box. It always outputs the basis for its computation.

## Example

| Item | Value |
|---|---|
| THI | 0.84 (Extreme) |
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

These are output in both JSON and human-readable report form.

---

# 26. Final THI Equation

Based on the above, the final THI equation is defined as follows. First, each hazard function

$$
I_i
$$

is computed. Next, the geometric mean

$$
H = \left( \prod I_i \right)^{1/n}
$$

is calculated. The compound hazard

$$
S = \sum \alpha_kC_k
$$

is computed. The Extreme correction

$$
M = \max(I_i)
$$

is computed. Finally,

$$
THI = (1-\beta) (H+S) + \beta M
$$

is defined. THI is normalized to the range 0–1.

---

# 27. Category Definition

|THI|Category|
|---|--------|
|0.00–0.20|Minimal|
|0.20–0.40|Minor|
|0.40–0.60|Moderate|
|0.60–0.80|Significant|
|0.80–1.00|Extreme|

Note that the classification boundaries are updated every year through learning.

---

# 28. Summary

THI Version 1.0 is a composite typhoon hazard index based on the following seven design principles:

- Computable from JMA XML alone
- Computable in real time
- Supports forecast projection
- Explainable
- Capable of self-learning
- Statistically optimizable
- Maintains consistency with physical laws

This specification is defined as the THI Version 1.0 Core Specification.

---

# 29. Self Calibration Engine

## Purpose

THI is not a fixed indicator. It is automatically updated every year using new typhoon data.

## Learning Dataset

Inputs:

- JMA XML
- Typhoon best-track data
- Disaster statistics
- Human casualties
- Property damage
- Precipitation observations

## Training Flow

Past typhoons → feature extraction → THI calculation → comparison with actual damage → parameter update → validation → config update

## Parameters to Learn

- Pressure μ, σ
- Wind μ, σ
- Rain μ, σ
- Storm Radius μ, σ
- Translation μ, σ
- Rapid Intensification μ, σ
- Expansion μ, σ
- β
- Classification thresholds
- Compound-hazard coefficient α

---

# 30. Loss Function

## Purpose

An ordinary mean-squared-error loss would underestimate hazardous typhoons. THI gives top priority to disaster-prevention utility.

## Asymmetric Loss

$$
L = \alpha (y-\hat y)^2
$$

where

$$
\alpha= \begin{cases} 5,&\hat y<y\\ 1,&\hat y\ge y \end{cases}
$$

In other words, errors that underestimate hazard are penalized five times as heavily.

---

# 31. Parameter Optimization

## Optimization Target

Parameters to be optimized:

- Sigmoid center values
- Sigmoid slopes
- β
- Hazard thresholds
- Compound-hazard coefficients

## Candidate Algorithms

- Bayesian Optimization
- Tree-structured Parzen Estimator
- Grid Search
- Random Search
- Optuna

## Evaluation Metrics

Among ROC-AUC, F1 Score, Recall, Precision, Balanced Accuracy, Matthews Correlation Coefficient, and Brier Score, Recall is treated as the most important evaluation metric. The reason is that the model's purpose is to never miss a hazardous typhoon.

---

# 32. Version Management

THI never changes its formula structure. Only parameters are changed.

Version example: THI 1.0.0 → THI 1.1.0 → THI 1.2.0 → THI 2.0.0

| Category | Content |
|---|---|
| Major version | Formula change |
| Minor version | Parameter update |
| Patch | Bug fix |

## Reproducibility

For each version, the Config used, training data, evaluation metrics, and Git commit ID are saved. This guarantees complete reproducibility.

---

# 33. Core Design Principles

THI, as proposed in this study, is designed to satisfy the following:

- Computable using JMA XML alone
- Independent of external GIS data
- Independent of population data
- Independent of infrastructure data
- Applicable to typhoons worldwide
- Fixed formula structure
- Learning limited to parameters only
- Support for Explainable AI
- Real-time computability
- Fully automatable operation

This defines the THI v1.0 Core Mathematical Specification. The next chapter describes the Python implementation specification, module structure, XML parsing specification, class design, and API design.

---

# 34. Python Software Architecture

## 34.1 Overview

THI is not merely a Python script. It is designed as a complete modular structure, taking maintainability, reusability, and future extension toward machine learning into account.

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

The XML Parser reads JMA XML and converts it into standardized data usable by THI.

### Input

- Typhoon analysis information
- Typhoon forecast information
- Weather warnings / advisories
- Precipitation-related XML

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

The Feature Generator converts the data obtained by the Parser into feature values. It is responsible for the transformation from physical quantities to hazard values (Feature → Sigmoid → Normalized Hazard).

## Hazard Engine

Each feature is evaluated independently — for example, by PressureModel, WindModel, RainModel, TranslationModel, ExpansionModel, and RapidDevelopmentModel.

Every module shares a common interface:

```python
calculate(feature)
```

## THI Engine

The THI Engine receives all hazard values and applies the base hazard, compound hazard, Extreme correction, and Forecast correction. Its output is

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

Input: THI. Output: Minimal, Minor, Moderate, Significant, Extreme. The decision is read from the Threshold Manager.

## Configuration System

All learned parameters are saved as JSON. Example:

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

Forecast XML → each forecast time → THI → maximum-value extraction → Forecast THI

Trend, Lifetime, and Forecast Stability are also computed at the same time.

## Similarity Engine

Input: current typhoon → feature vector → database → distance calculation → similarity ranking → return top 10

The initial implementation uses Euclidean distance; Version 2 can use Mahalanobis distance, and Version 3 can extend to machine-learning embeddings.

## Explainable Engine

After THI is computed, the contribution rate of each hazard is calculated. Output example:

| Element | Contribution |
|---|---|
| Rain | 41% |
| Wind | 26% |
| Pressure | 11% |
| Storm Radius | 10% |
| Translation | 8% |
| Rapid Intensification | 4% |

The Dominant Hazard is determined at the same time.

## Uncertainty Engine

Monte Carlo → THI → distribution → 95% confidence interval → display. This process can also be applied to forecast values.

## Annual Learning Engine

Every year, all typhoons up to the previous year are added to the training set. The learning results are saved to `parameters.json`. The formula structure is never changed — only μ, σ, β, and the thresholds are updated.

Through the above, THI is implemented as a consistent software architecture that takes only JMA XML as input and provides real-time computation, forecast evaluation, explainability, and self-learning.

---

# 35. Japan Meteorological Agency XML Mapping Specification

## 35.1 Purpose

This chapter defines the correspondence between all input variables used by THI (Typhoon Hazard Index) and JMA XML. The design goal of this specification is **to complete all calculations using JMA XML alone, wherever possible**.

## 35.2 XML Sources

The XML used by THI is classified into the following categories:

|Category|Use|
|----|----|
|Typhoon Analysis XML|Retrieval of current-condition values|
|Typhoon Forecast XML|Retrieval of future projections|
|Weather Warning / Advisory XML|Warning Index|
|Precipitation-related XML|Rainfall Hazard|
|Weather Overview XML|Supplementary information|

## 35.3 Feature Mapping

|Feature|XML|Required|Notes|
|--------|---|----|----|
|Minimum central pressure|Typhoon Analysis XML|Required|Current value|
|Maximum wind speed|Typhoon Analysis XML|Required|Current value|
|Storm radius|Typhoon Analysis XML|Required|Current value|
|Gale radius|Typhoon Analysis XML|Required|Current value|
|Typhoon center coordinates|Typhoon Analysis XML|Required|Used to compute translation speed|
|Analysis time|Typhoon Analysis XML|Required|Used for difference calculation|
|Forecast central pressure|Typhoon Forecast XML|Optional|Forecast|
|Forecast maximum wind speed|Typhoon Forecast XML|Optional|Forecast|
|Forecast storm radius|Typhoon Forecast XML|Optional|Forecast|
|24-hour precipitation|Precipitation-related XML|Optional|Rainfall evaluation|
|Warnings / advisories|Warning XML|Optional|Japan only|

## 35.4 Derived Features

THI computes values that cannot be obtained directly from XML using time-series differences.

### Translation Speed

Input: time, latitude, longitude → great-circle distance → translation speed

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

THI is computed for each forecast time, and the rate of change is calculated.

## 35.5 Missing Data Policy

When a value is not present in the XML, the following applies:

### Mandatory Feature

Retrieval failure → THI calculation halts, Confidence 0%

### Optional Feature

Missing → the corresponding hazard is excluded → Confidence decreases → DQI decreases

## 35.6 International Typhoon Support

Warning XML is available around Japan, but no warning information exists for Southeast Asia or the western Pacific. Warning Hazard is therefore treated as missing in those cases, only Confidence is reduced, and THI itself continues to be calculated.

## 35.7 Rainfall Policy

Heavy rainfall can cause severe disasters even when the typhoon itself is small. Rainfall Hazard is therefore treated as an independent, primary hazard element. Normal calculation is performed when precipitation data is available; Confidence is reduced when it is not.

## 35.8 Forecast Policy

When Forecast XML is available, the following are computed:

- Forecast THI
- Forecast Trend
- Forecast Stability
- Lifetime Hazard

When Forecast XML is not available, calculation uses current conditions only.

## 35.9 XML Validation

After XML retrieval, the following are validated:

- XML schema
- Required tags
- Time consistency
- Units
- Missing values

Anomalies are logged as XML Parse Errors.

## 35.10 Unit Standardization

All values are standardized to the following units:

|Item|Unit|
|----|----|
|Pressure|hPa|
|Wind|m/s|
|Rainfall|mm|
|Radius|km|
|Translation Speed|km/h|
|Latitude|degree|
|Longitude|degree|

## 35.11 Feature Cache

To avoid re-parsing the same XML, feature values are cached. The cache key consists of:

- Announcement time
- Bulletin ID
- Typhoon number

## 35.12 XML Processing Pipeline

JMA XML → Schema Validation → Parser → Feature Generator → Derived Features → Hazard Engine → THI Engine → Classification → Output

## 35.13 Reproducibility

Every THI result stores the following:

- XML announcement time
- XML bulletin ID
- THI version
- Config version
- Calculation timestamp
- List of features used

This guarantees complete reproducibility.

## 35.14 Core Philosophy

By making JMA XML the sole data source, this specification achieves:

- Simplified implementation
- Guaranteed reproducibility
- Long-term operability
- Academic verifiability

Future parameter optimization is performed, but **the input data source itself is never changed** — this is treated as a governing principle.

## 35.7 Rainfall Hazard Policy (Revised)

### Purpose

THI treats rainfall disasters as a primary hazard on equal footing with wind. However, because the precipitation information provided in JMA XML varies by bulletin type and coverage area, this specification adopts an abstraction that does not depend on any particular XML format.

### Rainfall Feature

The rainfall indicator is

$$
Rain_{obs}
$$

which represents the current or analyzed precipitation obtainable from JMA XML. For future forecasts,

$$
Rain_{forecast}
$$

is defined.

### XML Source Priority

When multiple applicable XML sources are available, a priority order is established:

1. Analyzed precipitation
2. Observed (current) precipitation
3. Forecast precipitation
4. Other precipitation-related XML

The XML type retrieved is saved as metadata.

### Metadata

THI output includes: Rain Source, Rain Timestamp, Rain Quality, and XML ID. This ensures reproducibility.

### Missing Policy

If precipitation information cannot be retrieved, Rain Hazard is treated as missing and THI calculation continues. Only Confidence and the Data Quality Index are reduced.

### Hazard Function

The rainfall hazard is defined as

$$
I_R = \frac 1 {1+\exp \left( -\frac {Rain-\mu_R} {\sigma_R} \right)}
$$

$\mu$ and $\sigma$ are not fixed values; they are updated every year through learning.

### Heavy Rain Override

Heavy rainfall can cause a severe disaster even without accompanying wind. For this reason, if Rain Hazard exceeds a learned threshold, THI can be promoted to Significant or Extreme regardless of the size of the storm radius. This decision threshold is also subject to learning.

---

# 36. Model Governance

The formula structure of THI is fixed; updates are limited to the following:

- Sigmoid center values (μ)
- Sigmoid slopes (σ)
- Weighting coefficients (w)
- Compound-hazard coefficient (α)
- Extreme decision threshold
- Classification thresholds

These are statistically optimized using historical typhoon data and actual damage data. This specification allows THI to simultaneously satisfy three requirements: "reproducible using JMA XML alone," "capable of self-learning," and "physically interpretable."

---

# 37. JSON Output Specification

This defines the standard output format of THI.

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

JSON alone is difficult to use in disaster-prevention operations, so a natural-language report is also generated. Example: "The current THI is 0.84, Extreme. The dominant hazard is heavy rain. Hazard is expected to increase further within 24 hours. Forecast confidence is 97%."

---

# 39. API Specification

This defines an API usable as a Python library.

```python
from thi import THI

engine = THI()

result = engine.calculate(xml_files)

print(result.score)
print(result.category)
```

Forecast:

```python
forecast = engine.forecast(xml_files)

print(forecast.max_score)
```

Similarity:

```python
similar = engine.similarity()

for typhoon in similar:
    print(typhoon)
```

Training:

```python
engine.train(dataset)
```

Optimization:

```python
engine.optimize(dataset)
```

---

# 40. Standard Calculation Flow

THI calculation is carried out in the following order:

```text
JMA XML Retrieval
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

THI is designed for real-time processing. Letting $n$ denote the number of features, the basic computational complexity is

$$
O(n)
$$

For Monte Carlo simulation alone, it becomes

$$
O(Nn)
$$

Even with $N=10000$, real-time computation on a CPU remains fully feasible.

---

# 42. Implementation Policy

The THI implementation adheres to the following:

- Python 3.12 or later
- NumPy
- SciPy
- Pandas
- scikit-learn
- Optuna (for training)

Learning libraries are not required for real-time computation. Up to this chapter, the mathematical model, software structure, input/output specification, explainability, and operational policy of THI have been defined. The next chapter defines the learning and validation methods based on historical typhoon data, evaluation metrics, and methods for verifying statistical validity.

---

# 43. Statistical Learning and Validation

## 43.1 Purpose

This chapter defines the learning and validation methods used to guarantee the statistical validity of THI (Typhoon Hazard Index). THI is not positioned as a fixed, rule-of-thumb model; rather, it is a **semi-physical, semi-statistical model (Physics-informed Statistical Model) in which the formula structure is fixed and only the parameters are estimated from data**.

## 43.2 Learning Dataset

Learning uses JMA data over as long a period as possible. Recommended sources:

- JMA best-track data
- JMA Typhoon Analysis XML
- JMA Typhoon Forecast XML
- JMA Warning / Advisory XML
- JMA precipitation-related XML

The target variable (training label) combines one or more of the following:

- Human casualties (deaths, missing persons)
- Housing damage (total/partial destruction, above-floor flooding, etc.)
- Total damage amount
- Insurance payouts (where available)

When multiple damage indicators are used, each is normalized and treated as a composite damage index.

## 43.3 Target Variable

For variables with highly skewed distributions, such as damage amount, a logarithmic transform is applied:

$$
Y = \log(1 + Damage)
$$

This suppresses the influence of outliers.

## 43.4 Parameter Estimation

The following are subject to learning:

- Sigmoid center values (μ)
- Sigmoid slopes (σ)
- Geometric-mean correction coefficient
- Extreme correction coefficient (β)
- Compound Hazard coefficients (α)
- TSI weights
- FHM weight
- Classification thresholds

The formula structure itself is never changed.

## 43.5 Cross Validation

To prevent overfitting, k-fold cross validation is used. Recommended:

$$
k = 5
$$

or

$$
k = 10
$$

## 43.6 Time-series Validation

Ordinary random splitting is not used; the training data preserves its time-series order. Example: 2010–2020 for Training, 2021 for Validation, 2022 for Test.

## 43.7 Performance Metrics

The following are used as the primary evaluation metrics.

Classification performance:

- ROC-AUC
- Precision
- Recall
- F1 Score
- Balanced Accuracy
- Matthews Correlation Coefficient

Regression performance:

- RMSE
- MAE
- R²
- Brier Score

## 43.8 Priority of Evaluation

Since THI is intended for disaster-prevention use, the evaluation priority is:

1. Recall
2. F1 Score
3. ROC-AUC
4. Precision

Top priority is given to never missing a hazardous typhoon.

## 43.9 Hyperparameter Optimization

Recommended methods:

- Bayesian Optimization
- Optuna
- Tree-structured Parzen Estimator (TPE)

Grid Search is used only as a baseline comparison.

## 43.10 Annual Retraining

Every year, learning is performed with newly added typhoon cases. The parameters updated are:

- μ
- σ
- β
- α
- Thresholds

only. The core formula of THI is preserved.

---

# 44. Benchmark Strategy

THI is compared against existing indicators. Example comparison targets:

- Central pressure alone
- Maximum wind speed alone
- Storm radius alone
- Saffir–Simpson Hurricane Wind Scale (for reference)
- Integrated Kinetic Energy (IKE, where comparable)

The evaluation determines whether THI shows higher predictive performance than these.

---

# 45. Sensitivity Analysis

Sensitivity analysis is performed for each input variable. Varying a single variable at a time, the amount of change in THI,

$$
\frac{\partial THI}{\partial x_i}
$$

is evaluated. This quantifies the degree of influence of each element.

---

# 46. Robustness Test

Observation error is applied to the input values and a Monte Carlo simulation is performed. This confirms that the range of variation in THI remains within an acceptable tolerance.

---

# 47. Reproducibility

At training time, the following are saved:

- THI version
- Parameter file
- Training data period
- Number of training samples
- Training date/time
- Python version
- Library versions
- Git commit ID (optional)

This allows the same result to be reproduced from the same data.

---

# 48. Scope of THI

THI is an indicator that evaluates "the natural hazard posed by a typhoon." The following are outside its scope:

- Population density
- Building seismic resistance
- Socioeconomic conditions
- Infrastructure vulnerability
- Evacuation behavior

These are important for disaster-risk assessment in general, but they are not included in this specification. THI is strictly defined as **an indicator that uniformly evaluates the hazard of the typhoon itself, using only meteorological information obtainable from JMA XML**.

---

# 49. Mathematical Consistency and Variable Independence

## 49.1 Purpose

This chapter defines the consistency of the mathematical model adopted by THI. Many highly correlated variables exist among typhoon physical phenomena. For example:

- Lower central pressure is associated with higher maximum wind speed
- A wider storm radius is associated with a wider gale radius
- The storm radius tends to expand during rapid intensification

Simply summing all of these as-is would double-count the same underlying physical phenomenon. THI is designed to prevent this.

## 49.2 Variable Classification

THI classifies explanatory variables into four types.

### Primary Variables

Independent physical quantities:

- Minimum central pressure
- Maximum wind speed
- Storm radius
- Gale radius
- Precipitation
- Translation speed

### Dynamic Variables

Quantities of change over time:

- 24-hour pressure change
- 24-hour storm-radius change

### Composite Variables

Derived quantities computed from multiple variables:

- TSI
- FHM
- Lifetime Hazard

### Meta Variables

Used for model evaluation:

- Confidence
- DQI
- Forecast Stability

## 49.3 Dependency Rule

THI feeds only Primary Variables directly into the hazard functions. Composite Variables are used only as correction terms. This prevents double-counting.

## 49.4 Typhoon Size Index Revision

TSI is computed from $R_{25}$ and $R_{15}$, and is therefore not added directly to THI. Instead, it corrects the wind and gale hazards:

$$
I_{Wind} ' = I_{Wind} \times (1+\alpha I_{TSI})
$$

where $\alpha$ is subject to learning. This naturally increases Wind Hazard for typhoons with a wider storm radius.

## 49.5 Forecast Hazard Modifier Revision

FHM is a Forecast-only module and is not used for current-condition THI.

$$
THI_{Forecast} = THI + \omega FHM
$$

Current-condition THI is always calculated from current-condition values only.

## 49.6 Compound Hazard Revision

Compound Hazard does not use every possible combination. Only terms found significant through learning are adopted.

For example, the following combinations are adopted:

- Rain × Translation
- Wind × Radius
- Rain × Warning

On the other hand, combinations whose physical meaning overlaps are not adopted:

- Pressure × Radius
- Pressure × Wind

## 49.7 Automatic Feature Selection

At training time, L1 regularization, Elastic Net, or sequential feature selection is used to remove unnecessary features. This prevents overfitting of the model.

## 49.8 Model Stability

The rate of coefficient change is recorded for each year's training results. A sudden large change triggers retraining or review.

---

# 50. Design Constraints

THI Version 1.0 does not change the following:

- Formula structure
- Feature definitions
- Output specification
- Category names

Only the following may be changed:

- Sigmoid parameters
- Weighting coefficients
- Compound coefficients
- Extreme threshold
- Category boundary values

---

# 51. Operational Principles

THI satisfies the following:

- Uses JMA XML as its only input
- Operates in real time
- Is applicable to typhoons outside Japan (expressed via Confidence when warning information is unavailable)
- Continuously learns its parameters

- Maintains the formula structure, ensuring academic reproducibility
- Provides output in both JSON and human-readable report form

---

# 52. Future Research

The following are positioned as research topics for Version 1.0:

- Evaluating the correlation between the PIKE approximation and rigorous IKE
- Verifying the statistical relationship between THI and actual damage
- Searching for an optimal asymmetric loss function
- Optimizing the coefficients of TSI and PIKE
- Evaluating generalization performance using long-term best-track data
- Evaluating applicability to tropical cyclones outside Japan

---

# 53. Conclusion

Typhoon Hazard Index (THI) Version 1.0 is a mathematical model whose purpose is to comprehensively evaluate the physical hazard of a typhoon using JMA XML alone. This model is defined as a hybrid indicator, fusing physical and statistical modeling, that integrates:

- Nonlinear hazard functions
- Integration via the geometric mean
- Compound-hazard evaluation
- Forecast Hazard Modifier
- Pseudo Integrated Kinetic Energy
- Typhoon Size Index
- Explainable AI
- Confidence evaluation
- Self-learning parameter estimation

This specification serves as the foundational document of the THI Version 1.0 Core Specification.

The THI Version 1.0 Core Specification is a comprehensive design specification integrating:

- The mathematical model
- The statistical model
- The XML specification
- The software design
- The learning method
- The validation method
- The API specification
- The implementation standard

In future versions, parameter updates will continue while the formula structure is preserved, developing THI further as a unified typhoon hazard index operable using JMA XML alone.

---

# References

## R.1 Purpose

This chapter presents representative research, statistical methods, and meteorological knowledge that form the theoretical background of THI (Typhoon Hazard Index). THI does not simply reproduce existing research; it is designed as a new composite hazard index that integrates and extends it.

## R.2 Tropical Cyclone Dynamics

### Emanuel, K. A.

**Emanuel, K. A.** *Atmospheric Convection.* Oxford University Press. Energetics and intensification theory of tropical cyclones.

### Emanuel, K. A.

**Emanuel, K. A.** *Divine Wind: The History and Science of Hurricanes.* Oxford University Press. Mechanisms of hurricane development.

### Holland, G.

**Holland, G. J.** *An Analytic Model of the Wind and Pressure Profiles in Hurricanes.* Monthly Weather Review. A typhoon wind-profile model.

## R.3 Integrated Kinetic Energy

### Powell et al.

**Powell, M. D., Reinhold, T. A.** *Tropical Cyclone Destructive Potential by Integrated Kinetic Energy.*

Proposed Integrated Kinetic Energy (IKE). Theoretical background for the PIKE design in THI.

## R.4 Best Track Data

### Japan Meteorological Agency

JMA best-track data. Base data for THI training.

### IBTrACS

International Best Track Archive for Climate Stewardship. World-standard best-track data, used for comparative validation.

## R.5 Statistical Learning

### Hastie, Tibshirani, Friedman

**The Elements of Statistical Learning.** Springer. Machine learning in general.

### Bishop

**Pattern Recognition and Machine Learning.** Springer. Probabilistic models.

### Murphy

**Machine Learning.** MIT Press. Loss functions, probabilistic prediction.

## R.6 Explainable AI

### Lundberg

SHAP, Feature Contribution

### Ribeiro

LIME, Local Explainability

## R.7 Generalized Additive Models

### Hastie

*Generalized Additive Models.* GAM theory, used in the THI nonlinear function design.

## R.8 Extreme Value Theory

### Coles

*An Introduction to Statistical Modeling of Extreme Values.* Extreme-value theory, the theoretical background for the Extreme decision.

## R.9 Copula Theory

### Nelsen

*Introduction to Copulas.* Dependency structure among multiple hazard elements; background for Compound Hazard.

## R.10 Bayesian Optimization

### Snoek

*Practical Bayesian Optimization of Machine Learning Algorithms.* Parameter optimization.

## R.11 Physics-informed Machine Learning

Karniadakis et al., *Physics-informed Machine Learning.* Fusion of physical models and learning; the design philosophy of THI.

## R.12 Meteorological Data

Japan Meteorological Agency, Disaster Prevention Information XML. Japan Meteorological Agency, Guide to the Use of Meteorological Data. Japan Meteorological Agency, Typhoon Analysis and Forecast Materials.

## R.13 Hazard Assessment

UNDRR, Sendai Framework. Disaster risk assessment. WMO (World Meteorological Organization), Tropical Cyclone Programme. Tropical cyclone assessment.

## R.14 Software Engineering

Python Software Foundation, Python Documentation, NumPy Documentation, SciPy Documentation, Pandas Documentation, Optuna Documentation, scikit-learn Documentation

## R.15 Scientific Position of THI

THI is a model that fuses the following academic fields:

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

The originality of THI lies in the following:

1. A composite hazard index computable from JMA XML alone.
2. A nonlinear model integrating central pressure, wind speed, storm radius, precipitation, and translation speed.
3. IKE approximation via PIKE.
4. Spatial-scale evaluation via TSI.
5. Future-hazard estimation via the Forecast Hazard Modifier.
6. Hazard decomposition premised on Explainable AI.
7. Parameter updates through annual self-learning.
8. Adoption of a Physics-informed Statistical Hazard Model in which the formula structure is fixed and learning is limited to parameters.

---

# Appendix A. Mathematical Foundations

## A.1 Design Philosophy

THI (Typhoon Hazard Index) is defined as a nonlinear index integrating multiple physical quantities to express a typhoon's overall hazard — something that cannot be expressed by conventional single indicators (central pressure, maximum wind speed, etc.) alone. The design policy consists of the following five points:

1. **Physical Interpretability**
2. **Statistical Optimizability**
3. **Explainability**
4. **Reproducibility**
5. **Real-time Computability**

## A.2 Hazard Function

All primary physical quantities are mapped to a common hazard function:

$$
I(x) = \frac 1 { 1+ \exp \left( -\frac{x-\mu}{\sigma} \right) }
$$

where

- $x$: input physical quantity
- $\mu$: hazard center value
- $\sigma$: transition width

In Version 1.0, all inputs are unified into this form.

## A.3 Base Hazard

Given each hazard

$$
I_i
$$

the base hazard is defined as

$$
H = \left( \prod_{i=1}^{n} I_i \right) ^{1/n}
$$

The geometric mean is adopted because it suppresses the overall hazard when other elements are low, even if a single element is extremely high.

## A.4 Compound Hazard

When multiple elements are simultaneously high, synergistic effects are taken into account:

$$
S = \sum_{k} \alpha_k C_k
$$

where $C_k$ are the compound terms determined to be significant. Examples:

- Rain × Translation
- Wind × Radius
- Rain × Warning

## A.5 Maximum Hazard

The maximum hazard element is defined as

$$
M = \max(I_i)
$$

$M$ is always used in the Extreme decision.

## A.6 Final Equation

The final THI is defined by

$$
THI = (1-\beta) (H+S) + \beta M
$$

where $\beta$ is subject to learning.

---

# Appendix B. Data Quality Index (DQI)

DQI evaluates the quality of the input data.

## Required

- Central pressure
- Maximum wind speed
- Typhoon position
- Storm radius
- Gale radius

If these are absent, DQI drops significantly.

## Optional

- Precipitation
- Warning
- Forecast

Calculation continues even if these are missing. DQI is expressed on a 0–1 scale.

---

# Appendix C. Confidence Score

Confidence indicates the reliability of the THI estimate. It is calculated from:

- XML completeness
- Missing-data rate
- Forecast consistency
- Monte Carlo variance

---

# Appendix D. Explainable Output

The standard output returns:

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

THI adopts Semantic Versioning.

## Major

Formula change, e.g. 1.x → 2.x

## Minor

Learning-parameter update, e.g. 1.0 → 1.1

## Patch

Implementation fix, e.g. 1.0.0 → 1.0.1

---

# Appendix F. Recommended Software Stack

Python 3.12+, NumPy, SciPy, Pandas, lxml, scikit-learn, Optuna, Matplotlib, pytest

---

# Appendix G. Expected Outputs

Standard JSON, detailed JSON, CSV, Markdown report, PDF report, REST API, CLI. These output formats are all supported.

---

# Appendix H. Reference Datasets

Datasets recommended for training and evaluation under this specification.

## Meteorological

- JMA best-track data
- JMA XML
- AMeDAS precipitation observations
- JMA warning / advisory history

## Damage

- Cabinet Office disaster records
- Fire and Disaster Management Agency disaster information
- MLIT flood-damage statistics

## Academic Comparison

- JTWC Best Track
- IBTrACS (for comparative evaluation)

Note: **THI's input is JMA XML only**; the above datasets are used exclusively for training and validation.

## End of THI Version 1.0 Core Specification

This specification is the foundational document defining the mathematical model, software structure, learning strategy, validation methods, and operational policy of THI. Implementation follows this specification, and by continuously updating only the learning parameters while preserving the formula structure, it achieves both long-term accuracy improvement and academic reproducibility.

---

# Appendix I. Japan Meteorological Agency XML Implementation Specification

## I.1 Purpose

This appendix defines the implementation specification of the JMA XML used in THI Version 1.0. The purpose is to:

- Clarify the source of all input data
- Eliminate differences in interpretation among implementers
- Guarantee an algorithm that is fully reproducible using JMA XML alone

## I.2 Supported XML Products

Version 1.0 targets the following bulletins:

|Bulletin|Use|Required|
|----|----|----|
|Typhoon Analysis Information|Current-condition analysis|Required|
|Typhoon Forecast Information|Forecast THI|Recommended|
|Warnings / Advisories|Warning Hazard|Optional|
|Precipitation-related XML|Rain Hazard|Optional|

## I.3 Feature Extraction Table

|THI Feature|XML Source|Retrieval Method|If Missing|
|------------|---------|---------|-------|
|Minimum central pressure|Typhoon Analysis|Direct retrieval|Calculation halts|
|Maximum wind speed|Typhoon Analysis|Direct retrieval|Calculation halts|
|Storm radius|Typhoon Analysis|Direct retrieval|Calculation halts|
|Gale radius|Typhoon Analysis|Direct retrieval|Calculation halts|
|Typhoon center coordinates|Typhoon Analysis|Direct retrieval|Calculation halts|
|Analysis time|Typhoon Analysis|Direct retrieval|Calculation halts|
|Precipitation|Precipitation XML|Latest available value|May be missing|
|Warning Index|Warning XML|Aggregated|May be missing|
|Forecast|Typhoon Forecast|Each forecast time|Current conditions only|

## I.4 Derived Variables

Items not obtained directly from XML.

### Translation Speed

Computed from the change in position between consecutive analysis times.

### Pressure Change

24-hour difference.

### Radius Expansion

24-hour difference.

### PIKE

Computed from maximum wind speed and storm radius.

### TSI

Computed from the storm radius and gale radius.

### Compound Hazard

Computed from the individual hazards.

## I.5 XML Validation Rules

The following are validated before parsing:

- XML schema
- XML version
- Namespace
- Announcement time
- Typhoon number
- Units
- Numeric range
- Missing values

A ParseError is returned on anomaly.

## I.6 XML Cache

The same bulletin is not re-parsed. Cache key:

```
Typhoon number
+
Announcement time
+
Bulletin ID
```

## I.7 Forecast Handling

When Forecast XML is present, THI is recalculated for each forecast time:

```
0h

6h

12h

24h

48h

72h
```

(following whichever forecast times are available)

## I.8 Missing Policy

Required-value missing → THI halts. Optional-value missing → Confidence decreases, DQI decreases, calculation continues.

## I.9 Logging

All processing is logged. Recorded content:

- XML retrieval date/time
- XML announcement time
- XML type
- XML version
- THI version
- Config version
- Parser version

## I.10 Output Metadata

Added to the JSON output:

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

The implementation is fully object-oriented. Each hazard is an independent class.

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

Each module is tested independently. Minimum tests to perform:

- XML Parser
- Feature Generator
- Hazard Function
- PIKE
- TSI
- THI
- Forecast
- JSON Export

## J.7 Continuous Integration

Recommended: GitHub Actions. Tasks performed:

- Lint
- Unit Test
- Type Check
- Coverage
- Documentation Build

## J.8 Configuration

Everything is managed as JSON:

- Learned coefficients
- Sigmoid parameters
- Thresholds
- Version

Hard-coding into source code is prohibited.

## J.9 Documentation

Generated automatically from source code. Recommended:

- Sphinx
- MkDocs

---

# Appendix K. Theoretical Basis of the THI Mathematical Model

## K.1 Purpose

This appendix presents the physical, meteorological, and statistical basis for each formula adopted by THI (Typhoon Hazard Index). THI is not an indicator based purely on rules of thumb; it is designed as a Physics-informed Statistical Model that integrates existing meteorological and statistical knowledge.

## K.2 Pressure Hazard

### Physical Basis

Minimum central pressure is a representative physical quantity expressing the degree of typhoon development. In general, the lower the central pressure, the larger the pressure gradient, creating an environment more conducive to strong winds. However, the actual hazard can differ even at the same pressure depending on typhoon size and the surrounding environmental field, so THI does not determine hazard from central pressure alone.

### Mathematical Basis

The relationship between central pressure and damage is not linear. A sigmoid function is therefore used to express the property that "hazard increases sharply once a certain threshold is exceeded."

## K.3 Wind Hazard

### Physical Basis

The energy of wind damage is roughly proportional to the square of wind speed. Building damage, tree falls, wind-borne debris, and storm surge are all strongly related to maximum wind speed.

### Mathematical Basis

THI does not use wind speed directly; it normalizes it into a hazard value. This makes optimization through learning easier.

## K.4 Rainfall Hazard

### Physical Basis

In recent years in Japan, the proportion of rainfall-related disasters relative to wind damage has increased. Linear precipitation bands, landslides, and river flooding are all strongly related to precipitation amount. Rain Hazard is therefore treated as a primary hazard element.

## K.5 Translation Hazard

### Physical Basis

Typhoons with slower translation speed bring prolonged rainfall to the same area. Total precipitation is closely related to residence time. THI uses the reciprocal of translation speed.

## K.6 Storm Size Hazard

### Physical Basis

Large typhoons increase the duration of storm conditions and the extent of power outages and storm surge. The storm radius and gale radius represent the typhoon's spatial scale.

## K.7 PIKE

### Physical Basis

Rigorous IKE integrates the entire wind field. However, JMA XML does not provide the wind-speed distribution. THI therefore introduces PIKE as a simplified approximation of kinetic energy — this is an IKE proxy, not IKE itself.

## K.8 TSI

### Physical Basis

Even at the same maximum wind speed, the range of societal impact differs between a large typhoon and a small one. TSI is introduced to correct for this spatial scale.

## K.9 Compound Hazard

### Statistical Basis

In actual disasters, the simultaneous occurrence of multiple factors is more hazardous than any single factor — for example, heavy rain × stagnation, or heavy rain × widespread wind. THI adopts only those combinations found significant through learning.

## K.10 Geometric Mean

### Statistical Basis

An arithmetic mean averages out some extreme values. The geometric mean yields a high hazard value only when all elements are high. THI adopts it as the base hazard.

## K.11 Maximum Hazard

### Statistical Basis

In extreme disasters, a single dominant factor can govern the damage. Maximum Hazard is therefore introduced to correct the Extreme decision.

## K.12 Sigmoid Function

### Statistical Basis

The sigmoid function is widely used in probability theory, machine learning, and survival analysis. It normalizes hazard to the range 0–1, and its μ and σ can be learned.

## K.13 Parameter Learning

THI fixes its formula structure and learns only its parameters. This is based on the concept of Physics-informed Machine Learning, aiming to reconcile physical laws with data-driven learning.

## K.14 Scope

THI is a natural-hazard indicator. Population, economics, infrastructure, and social vulnerability are outside its scope; these are handled by separate risk models.

## K.15 Scientific Position

THI is positioned as a Physics-informed Statistical Hazard Model integrating:

- Meteorology
- Fluid dynamics
- Disaster science
- Mathematical statistics
- Machine learning

Its purpose is to **objectively and reproducibly evaluate the hazard of the typhoon itself, using JMA XML alone**.

---

# Appendix L. Limitations, Assumptions, and Future Development

## L.1 Purpose

This appendix defines the scope of application, assumptions, known limitations, and future development plans of THI Version 1.0. This chapter is provided to ensure appropriate use of this index and academic transparency.

## L.2 Scope of Application

THI targets the following phenomena:

- Typhoons
- Tropical cyclones
- Extratropical cyclones that developed from a typhoon (when traceable via JMA XML)

THI is an indicator that evaluates "the hazard of the typhoon as a meteorological phenomenon itself"; social vulnerability and regional characteristics are outside its scope.

## L.3 Fundamental Assumptions

Version 1.0 assumes the following:

### 1. JMA XML is the sole input

No additional observational data is required.

### 2. The formula structure is fixed

The only items subject to change are:

- Parameters
- Weights
- Thresholds

### 3. Hazard is not a probability

THI is a Hazard Index, not a Probability.

### 4. It is not a damage-prediction model

THI is not a model that directly predicts damage amount or death toll; it evaluates natural hazard.

## L.4 Known Limitations

Version 1.0 has the following constraints:

### Sea-surface temperature

Not used directly.

### Ocean heat content

Not used.

### Vertical wind shear

Not used.

### Terrain effects

Not directly modeled.

### Tide level

A storm-surge model is not included.

### Wave action

Outside scope.

### Population density

Outside scope.

### Infrastructure

Outside scope.

### Evacuation rate

Outside scope.

### Economic damage

Usable for training, but not used as an input.

## L.5 Advantages

Advantages of Version 1.0:

- Self-contained using JMA XML alone
- Applicable worldwide
- Capable of real-time processing
- Supports Explainable AI
- Capable of self-learning
- High reproducibility
- Relatively easy to implement

## L.6 Future Versions

### Version 2

Candidate additions:

- Sea-surface temperature
- Ocean heat content
- Vertical wind shear
- Satellite analysis
- Radar analysis

### Version 3

Candidate additions:

- AI ensembles
- Deep Learning
- PINNs (Physics-Informed Neural Networks)
- Graph Neural Networks

### Version 4

Candidate additions:

- Real-time global analysis
- Multi-model integration
- Automated report generation
- Fully automatic parameter updating

## L.7 Compatibility

Within the Version 1 series, THI maintains API compatibility, JSON compatibility, and formula compatibility.

## L.8 Validation Policy

The following are always performed at each version update:

- Re-evaluation of all past typhoons
- ROC evaluation
- Recall evaluation
- Comparison against actual damage
- Parameter comparison
- Regression testing

## L.9 Ethical Considerations

THI is a supplementary indicator intended to support disaster prevention. It does not replace evacuation information from government agencies or official JMA announcements. Users are expected to use it alongside official disaster-prevention information.

## L.10 Open Science Policy

THI recommends:

- Publishing the formulas
- Publishing the parameters
- Publishing the version
- Publishing validation results
- Publishing the learning method

It is published as reproducible research to the greatest extent possible.

## L.11 Final Definition

THI (Typhoon Hazard Index) Version 1.0 is **a hazard index, equipped with reproducibility, explainability, and self-learning capability, that comprehensively evaluates the natural hazard of a typhoon through the fusion of physics, meteorology, statistics, and machine learning, using JMA XML alone as input**. This specification establishes the THI Version 1.0 Core Specification as the formal specification.

---

# Appendix M. Benchmark and Verification Protocol

## M.1 Purpose

This appendix defines an evaluation protocol for verifying the performance of THI (Typhoon Hazard Index) in an objective and reproducible manner. The purpose is to standardize:

- Reproducible verification
- Comparison with other indicators
- Evaluation of statistical significance
- Annual performance monitoring

## M.2 Benchmark Objectives

THI verifies the following:

1. Whether hazardous typhoons are correctly rated highly
2. Whether the correlation with damage is higher than existing indicators
3. Whether underestimation can be reduced
4. Whether performance is maintained across different years

## M.3 Benchmark Dataset

The validation data is kept independent of the training data. Recommended periods:

- Training: 1991–2018
- Validation: 2019–2022

- Test: 2023 onward

(adjusted according to available data volume)

## M.4 Baseline Models

THI is compared against the following.

### Single Variable

- Minimum central pressure
- Maximum wind speed
- Storm radius
- Gale radius
- Precipitation

### Existing Hazard Metrics

- Saffir–Simpson Hurricane Wind Scale
- Integrated Kinetic Energy (IKE)
- Accumulated Cyclone Energy (ACE)
- Power Dissipation Index (PDI)

Note: IKE, ACE, and PDI are comparison targets only; they are not inputs to THI.

## M.5 Regression Evaluation

When the target variable is treated as continuous, the following metrics are used:

- RMSE
- MAE
- R²
- Spearman rank correlation
- Pearson correlation

## M.6 Classification Evaluation

For the five-level classification, the following are used:

- Accuracy
- Precision
- Recall
- F1 Score
- ROC-AUC
- PR-AUC
- Balanced Accuracy
- Matthews Correlation Coefficient

## M.7 Calibration Evaluation

This confirms whether THI is internally consistent as a hazard indicator. Metrics used:

- Reliability Curve
- Calibration Error
- Brier Score

## M.8 Robustness Test

Observation error is applied to the input values. Example:

- Pressure ±2 hPa
- Maximum wind speed ±2 m/s
- Radius ±10 km

The stability of THI is evaluated via Monte Carlo simulation.

## M.9 Sensitivity Analysis

For each feature, a single variable is varied at a time, and

$$
S_i = \frac{\partial THI}{\partial x_i}
$$

is evaluated. This identifies the dominant factors.

## M.10 Ablation Study

Performance is compared with each module removed. Example:

- Without PIKE
- Without Rain Hazard
- Without Compound Hazard
- Without Forecast

The amount of performance degradation is measured to quantify the effectiveness of each element.

## M.11 Statistical Significance

Statistical tests are performed on the difference between THI and comparison models. Recommended methods:

- Wilcoxon signed-rank test
- McNemar test
- Bootstrap confidence intervals

The test adopted is selected according to the evaluation target.

## M.12 Annual Monitoring

The following are compared every year:

- ROC-AUC
- Recall
- RMSE
- Amount of parameter change
- DQI distribution

If performance degradation is observed, parameters are retrained.

## M.13 Reproducibility Checklist

Every validation result must record:

- THI version
- Config version
- Training data period
- Validation data period
- XML retrieval date/time
- Python version
- NumPy version
- SciPy version
- scikit-learn version
- Optuna version
- Git commit ID (optional)

## M.14 Acceptance Criteria

To bring THI Version 1.0 into formal operation, the following goals must be met:

- Classification performance at or above baseline indicators
- Reduced underestimation rate
- Stable performance across years
- Explainability of the model
- Reproducibility for identical inputs

## M.15 Conclusion

This protocol is the standard procedure for evaluating THI's performance objectively and reproducibly. All training, validation, and version updates shall be carried out in accordance with this protocol.

---

# Appendix N. Learning and Parameter Optimization Specification

## N.1 Purpose

This appendix defines the learning method, parameter estimation approach, and model update procedure for THI Version 1.0. In THI, **the formula structure is fixed**, and only parameters are subject to learning.

## N.2 Learning Targets

The parameters subject to learning are as follows.

### Hazard Function

- μ (center value)
- σ (slope)

### Weight

- β (Maximum Hazard coefficient)
- α (Compound coefficient)
- PIKE correction coefficient
- TSI correction coefficient

### Category

- Minimal
- Minor
- Moderate
- Significant
- Extreme

and their respective boundary values.

## N.3 Fixed Components

Not changed in Version 1.0:

- Formula structure
- Input variables
- PIKE definition
- TSI definition
- Geometric mean
- Compound structure
- Explainable Output

## N.4 Objective Variable

Recommended objective variables.

### First Choice

Damage amount (log-transformed)

### Second Choice

Number of housing damage cases

### Third Choice

Human casualties

### Fourth Choice

Warning duration. Multi-task learning using multiple objective variables is also possible.

## N.5 Loss Function

Version 1.0 adopts an asymmetric loss function that heavily penalizes underestimation. Conceptual form:

$$
L= \begin{cases} \lambda (y-\hat y)^2 & (y>\hat y) \\ (y-\hat y)^2 & (y\le\hat y) \end{cases}
$$

where

$$
\lambda>1
$$

$\lambda$ is also subject to learning.

## N.6 Optimization Algorithms

Recommended order of preference:

1. Optuna
2. Bayesian Optimization
3. CMA-ES
4. Grid Search

Grid Search is used only for baseline evaluation.

## N.7 Cross Validation

Recommended: Time Series Cross Validation. Ordinary K-fold is not used.

## N.8 Regularization

Recommended:

- L1
- Elastic Net

These suppress unnecessary features.

## N.9 Early Stopping

Training stops when validation loss no longer improves. Patience is managed via the configuration file.

## N.10 Hyperparameter Storage

Everything is saved as JSON. Example:

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

For each training run, the following are saved:

- Version
- Training date
- Data period
- Hash value

## N.12 Model Comparison

The new and old models are compared. Update conditions:

- Improved Recall
- Improved AUC
- Improved RMSE
- Reduced underestimation rate

The model is updated only if all conditions are satisfied.

## N.13 Explainability Verification

After training, the contribution of each feature is checked using SHAP or similar methods. If the contributions of PIKE, TSI, Rain, Wind, etc. are extremely unnatural, the training result is not adopted.

## N.14 Parameter Drift

Every year, the difference from the previous year is evaluated. A sudden change raises suspicion of climate change or a data anomaly.

## N.15 Final Learning Policy

THI is operated as a **Physics-informed Machine Learning model in which the formula is fixed and only the parameters are continuously optimized**. This ensures:

- Physical consistency
- Explainability
- Reproducibility

- Reproducibility
- Long-term performance improvement

all at the same time.

### Recommended Directory Structure

```
thi/
│
├── main.py                  # Execution entry point
├── config.py                # Parameter management
├── constants.py              # Constants
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

### Implementation Order

### STEP 1 (Highest Priority)

JMA XML → Feature extraction → THI calculation → JSON output

### STEP 2

Forecast, PIKE, TSI, Confidence

### STEP 3

Learning functionality, Optuna, SHAP, asymmetric loss

### STEP 4

GUI, REST API, CLI

### Feature Generator

First, from the XML,

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

are generated. Then,

```
translation_speed

pressure_drop

radius_growth

PIKE

TSI
```

are calculated.

### THI Calculation

Finally,

```
Pressure Hazard → Wind Hazard → Rain Hazard → Translation Hazard → PIKE → TSI
→ Compound Hazard → Geometric Mean → Maximum Hazard → THI → Category → Confidence → JSON
```

### JSON Output Example

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

### Implementation Policy

Since this project will eventually grow to several thousand lines, **implementing it as a modular structure rather than a single file** is recommended. Completing

- The JMA XML parser
- The Feature Generator
- The THI calculation engine
- JSON output

as **Version 1.0 Core** first, and then adding learning functionality and Optuna-based parameter optimization afterward, is the structure best suited to maintainability and extensibility.

Beyond that, the most reliable approach is to **build from the foundation upward rather than all at once**. The first thing to implement is the **common data model**. Building this first allows the XML parser, THI calculation, and trainer that follow to all share the same data structure.

### 1. Common Data Classes (`models.py`)

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

### 2. THI Result Class

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

### 3. Category Determination

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

### 4. Sigmoid Function

In Version 1.0, every hazard is unified through this function.

```
import math

def sigmoid(x, mu, sigma):

    return 1.0 / (

        1.0 +

        math.exp(-(x - mu) / sigma)

    )
```

For indicators such as central pressure, where "smaller is more hazardous,"

```
def inverse_sigmoid(x, mu, sigma):
    return 1.0 - sigmoid(x, mu, sigma)
```

is used.

### 5. PIKE

```
import math

def calc_pike(vmax, storm_radius):

    raw = vmax ** 2 * storm_radius ** 2

    return math.log1p(raw)
```

### 6. TSI

```
def calc_tsi(storm_radius, gale_radius, alpha=0.5):

    return storm_radius + alpha * gale_radius
```

### 7. Feature Generator

Using the values obtained from the XML, this module generates, in a single pass, all derived features needed by THI, including:

- PIKE
- TSI
- Translation speed
- 24-hour pressure change
- Storm-radius expansion rate

This completes the **foundation of the THI Engine**. **The next module to implement is `jma_typhoon.py` (the JMA XML parser)**. Once this module can generate a `TyphoonFeature` from XML, it can be fed directly into the THI calculation engine.

---

# Appendix O. Mathematical Symbols and Variable Definitions

## O.1 Purpose

This chapter defines every symbol, variable, and function used by THI (Typhoon Hazard Index). Throughout this specification, the same symbol always carries the same meaning.

## O.2 Physical Variables

|Symbol|Definition|Unit|
|-------|----------|----|
|$P_{min}$|Minimum central pressure|hPa|
|$V_{max}$|Maximum wind speed|m/s|
|$R_{25}$|Storm radius|km|
|$R_{15}$|Gale radius|km|
|$Rain$|Precipitation|mm|
|$V_{trans}$|Translation speed|km/h|
|$\Delta P$|24-hour pressure change|hPa|
|$\Delta R$|24-hour storm-radius change|km|

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
|Confidence|Completeness of input information|
|DQI|Data Quality Index|
|Version|THI specification version|

---

# Appendix P. Recommended Initial Parameters

In Version 1.0, all sigmoid parameters are set as **initial values** and are subsequently updated through learning.

|Parameter|Description|
|----------|-----------|
|$\mu_P$|Center value of central pressure|
|$\sigma_P$|Slope of central pressure|
|$\mu_V$|Center value of maximum wind speed|
|$\sigma_V$|Slope of maximum wind speed|
|$\mu_R$|Center value of precipitation|
|$\sigma_R$|Slope of precipitation|
|$\mu_T$|Center value of translation speed|
|$\sigma_T$|Slope of translation speed|
|$\mu_{PIKE}$|Center value of PIKE|
|$\sigma_{PIKE}$|Slope of PIKE|

These are not fixed values; they are updated through annual retraining.
