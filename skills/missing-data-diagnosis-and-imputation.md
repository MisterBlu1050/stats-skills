# Skill: Missing Data Diagnosis and Imputation

**Version:** 2.1  
**Author:** Bruno Mineo (MisterBlu1050)  
**License:** MIT  
**Field:** Social sciences, psychology, quantitative research

> **Source disclaimer:** This skill is an original independent work.
> It draws on methodological concepts from the standard scientific literature
> (Hair et al., van Buuren, Rubin, Schafer & Graham).
> It was developed in the context of a university course
> (Statistiek IV, VUB) but reproduces no extract from Theuns et al. (2016),
> whose copyright belongs to the authors and VUB.
> Any structural resemblance reflects the common logic of the field,
> not a copy of pedagogical material.

---

## Purpose
Guide a complete missing data analysis through a 4-step protocol —
from diagnosis to remediation — with R and SPSS code and APA reporting templates.

---

## Expected inputs
- Dataset (CSV, .sav, or variable description)
- Research context (design, variables, research question)
- Software preference: R or SPSS

---

## STEP 1 — Classification: negligible vs. non-negligible

| Type | Examples | Action |
|---|---|---|
| **Negligible** | Skip-patterns, censored data | Do nothing |
| **Non-negligible (known)** | Coding errors, incomplete questionnaires | Remediate |
| **Non-negligible (unknown)** | Refusal of sensitive items, dropout | Remediate + report |

> Negligible missing data (expected, structural) must NEVER be imputed.

---

## STEP 2 — Quantification

```r
library(naniar)
miss_var_summary(df)
vis_miss(df)
gg_miss_var(df)
```

```spss
* Analyze > Missing Value Analysis
```

**Indicative thresholds (Hair et al., 2013)**

> ⚠️ These thresholds are operational heuristics, not absolute rules.
> The **mechanism** (MCAR/MAR/MNAR) determines the appropriate strategy,
> not the percentage alone (van Buuren, 2018; Madley-Dowd et al., 2019).
> Even 5% missing under MNAR can seriously bias results;
> conversely, 15% under MCAR with large N remains manageable via listwise deletion
> (loss of power, not of unbiasedness).

| % missing | Indicative strategy |
|---|---|
| < 10% | All methods applicable; listwise acceptable under MCAR |
| 10–20% | Listwise/regression under MCAR; MI or EM under MAR |
| > 20% | Regression under MCAR; MI or EM under MAR |

---

## STEP 3 — Mechanism: MCAR / MAR / MNAR

### Definitions

| Mechanism | Cause | Testable? |
|---|---|---|
| **MCAR** | Fully random, independent of all data | Yes (Little's test) |
| **MAR** | Depends on *observed* variables, not on the missing value itself | Indirectly |
| **MNAR** | Depends on the *missing value itself* | Not directly |

### Diagnostic tests

**a) Little's MCAR test**
```r
library(naniar)
mcar_test(df)
```
```spss
* Analyze > Missing Value Analysis > EM
```
> ⚠️ **Two-sided limitation of Little's test:**
> - Large N → over-powered: p < .05 for practically negligible deviations
> - Small N → under-powered: p > .05 **does not confirm MCAR** —
>   it only means insufficient evidence to reject it (Type II error possible)

**b) Missing-indicator correlations**
```r
library(dplyr)
df_ind <- df %>%
  mutate(across(everything(),
                ~as.integer(is.na(.)),
                .names = "{col}_missing"))
cor(df_ind, use = "pairwise.complete.obs")
```
> Significant correlation between the missingness indicator of Y
> and an observed variable X → evidence of MAR with respect to X

**c) Independent-samples t-test**  
Compare means on all other variables between:
- cases with missing on Y
- complete cases on Y

**d) Visual pattern inspection**
```r
gg_miss_upset(df)
gg_miss_fct(df, gender)
```

---

## STEP 4 — Remediation

| Situation | Recommended strategy |
|---|---|
| Negligible | Do nothing |
| MCAR + < 10% | Listwise acceptable; MI or EM preferable |
| MCAR + 10–20% | Regression imputation or MI |
| MAR | **Multiple Imputation (MICE)** or Maximum Likelihood (EM) |
| MNAR | No reliable imputation — sensitivity analyses |

### Imputation methods

---

#### ⛔ Mean/Median imputation — avoid in most cases
Underestimates variance, produces artificially narrow CIs,
biases correlations. Use only as a **comparative benchmark**,
never as a final method in a paper or thesis.

```r
df$BP[is.na(df$BP)] <- mean(df$BP, na.rm = TRUE)
```

---

#### ⚠️ Deterministic regression imputation — limited version
Predicts missing values without a residual error term →
also underestimates variance. Prefer the stochastic version
or MICE/PMM directly.

```r
# Deterministic version (not recommended as final method)
model <- lm(BP ~ Age + Gender + BMI + Sleep + Chol + Smoke, data = df)
df$BP[is.na(df$BP)] <- predict(model, newdata = df[is.na(df$BP), ])

# Stochastic version (recommended for simple regression)
df$BP[is.na(df$BP)] <- predict(model, newdata = df[is.na(df$BP), ]) +
  sample(residuals(model), sum(is.na(df$BP)), replace = TRUE)
```

---

#### ✅ Multiple Imputation by Chained Equations (MICE) — reference method
```r
library(mice)
set.seed(12345)
imp <- mice(df, m = 20, method = "pmm")  # PMM = Predictive Mean Matching
fit <- with(imp, lm(BP ~ Age + Gender + BMI))
pooled <- pool(fit)
summary(pooled)  # Rubin's pooling rules
```
```spss
* Analyze > Multiple Imputation > Impute Missing Data Values
* Number of imputations: 20
* SPSS pools automatically via Rubin's rules
```

---

#### ✅ Maximum Likelihood / EM
```spss
* Analyze > Missing Value Analysis > EM
```

---

#### MNAR — Sensitivity analyses (advanced level)
When MNAR is suspected, no imputation can be validated
without untestable assumptions. Recommended approaches:
- **δ-adjustment**: introduce a systematic shift in imputed values
  to test the robustness of conclusions
- **Pattern-mixture models**: model subgroups separately
  according to their missingness pattern
- **Selection models**: jointly model the missingness mechanism
  and the substantive model

> Even without these advanced methods, explicitly reporting
> the limitations related to suspected MNAR is an ethical and scientific requirement.

---

## Reporting (APA style)

Always include:
1. **N total**, **N complete** (before treatment), **N after imputation**
2. % missing per variable
3. Diagnosed mechanism + test used + its limitations
4. Chosen method + justification

**Example:**
> "Of the 350 observations, 312 had complete data on all variables
> (N complete = 312, 89.1%). Missingness was analysed via Little's MCAR test,
> χ²(df) = X.XX, p = .XXX, and correlations between missingness indicators
> and observed variables. A significant association between cholesterol missingness
> and age (r = .XX, p < .05) suggested a MAR mechanism.
> Multiple imputation was performed (m = 20, MICE, PMM method)
> following Rubin's (1987) pooling rules."

---

## References
- Hair, J.F. et al. (2019). *Multivariate Data Analysis* (8th ed.). Cengage.
- Field, A. (2024). *Discovering Statistics Using IBM SPSS Statistics* (6th ed.). Sage.
- van Buuren, S. (2018). *Flexible Imputation of Missing Data* (2nd ed.).
  CRC Press. https://stefvanbuuren.name/fimd/
- van Buuren, S. & Groothuis-Oudshoorn, K. (2011). mice: Multivariate Imputation
  by Chained Equations in R. *Journal of Statistical Software*, 45(3).
- Madley-Dowd, P. et al. (2019). The proportion of missing data should not be
  used to guide decisions on multiple imputation. *Journal of Clinical
  Epidemiology*, 110, 63–73.
- Rubin, D.B. (1987). *Multiple Imputation for Nonresponse in Surveys*. Wiley.
- Schafer, J.L. & Graham, J.W. (2002). Missing data: Our view of the state of
  the art. *Psychological Methods*, 7(2), 147–177.

---

*Independently developed by Bruno Mineo. Feedback welcome via GitHub Issues.*
