# Skill: Missing Data Diagnosis and Imputation

**Version:** 2.2  
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

> **Reproducibility principle:** Always set a random seed before any stochastic
> operation (imputation, resampling). Always report exact model figures
> (χ², df, p, N, r, m) so results can be independently verified.

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

### R
```r
library(naniar)

# Summary table: n missing, % missing, n complete per variable
miss_var_summary(df)

# Visual overview
vis_miss(df)       # heatmap
gg_miss_var(df)    # bar chart per variable
```

### SPSS
```spss
* --- STEP 2: Quantify missing data per variable ---
MISSING VALUE ANALYSIS
  /VARIABLES = BMI Age Gender Sleep Smoke BP Chol
  /TTEST PERCENT=5
  /DESCRIBE UNIVARIATE MISMATCH
  /CROSSTAB PERCENT=5.
* Output: table with n missing, % missing, mean, SD per variable.
* Copy the descriptive table to report exact figures.
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

#### a) Little's MCAR test

**R**
```r
library(naniar)
result <- mcar_test(df)
print(result)
# Report: chi2 = result$statistic, df = result$df, p = result$p.value
```

**SPSS**
```spss
* --- STEP 3a: Little's MCAR test ---
MISSING VALUE ANALYSIS
  /VARIABLES = BMI Age Gender Sleep Smoke BP Chol
  /EM(TOLERANCE=0.001 CONVERGENCE=0.0001 ITERATIONS=25).
* Output: Little's MCAR test chi-square, df, and p-value.
* Report exact figures: chi2(df) = X.XX, p = .XXX
```

> ⚠️ **Two-sided limitation of Little's test:**
> - Large N → over-powered: p < .05 for practically negligible deviations
> - Small N → under-powered: p > .05 **does not confirm MCAR** —
>   it only means insufficient evidence to reject it (Type II error possible)

#### b) Missing-indicator correlations

**R**
```r
library(dplyr)

# Create binary missing indicators (0 = present, 1 = missing)
df_ind <- df %>%
  mutate(across(everything(),
                ~as.integer(is.na(.)),
                .names = "{col}_missing"))

# Correlation matrix: indicators vs. observed variables
cor_matrix <- cor(df_ind, use = "pairwise.complete.obs")
print(round(cor_matrix, 3))
# Report significant correlations: r = X.XX, p = .XXX
```

**SPSS**
```spss
* --- STEP 3b: Create missing indicators ---
COMPUTE BP_missing = MISSING(BP).
COMPUTE Chol_missing = MISSING(Chol).
COMPUTE Smoke_missing = MISSING(Smoke).
VARIABLE LABELS BP_missing 'Missing indicator BP'
                 Chol_missing 'Missing indicator Chol'
                 Smoke_missing 'Missing indicator Smoke'.
EXECUTE.

* --- Correlate indicators with all observed variables ---
CORRELATIONS
  /VARIABLES = Age Gender BMI Sleep BP Chol Smoke
                BP_missing Chol_missing Smoke_missing
  /PRINT = TWOTAIL SIG NOSIG
  /MISSING = PAIRWISE.
* Report: r(indicator_Y, X) = X.XX, p = .XXX
```

> Significant correlation between the missingness indicator of Y
> and an observed variable X → evidence of MAR with respect to X

#### c) Independent-samples t-test

**R**
```r
# Example: does age differ between cases with/without missing BP?
t.test(Age ~ is.na(BP), data = df)
# Report: t(df) = X.XX, p = .XXX, M_missing = X.XX, M_complete = X.XX
```

**SPSS**
```spss
* --- STEP 3c: t-test missing vs. complete cases ---
T-TEST GROUPS = BP_missing(0 1)
  /VARIABLES = Age Gender BMI Sleep Chol Smoke
  /CRITERIA = CI(0.95).
* Report exact t, df, p, and means for each group.
```

#### d) Visual pattern inspection

**R**
```r
gg_miss_upset(df)          # co-occurrence pattern matrix
gg_miss_fct(df, gender)    # missingness by subgroup
```

**SPSS**
```spss
* --- STEP 3d: Missing value pattern matrix ---
MISSING VALUE ANALYSIS
  /VARIABLES = BMI Age Gender Sleep Smoke BP Chol
  /TTEST PERCENT=5
  /DESCRIBE UNIVARIATE
  /PATTERN VARIABLES = BMI Age Gender Sleep Smoke BP Chol.
* The pattern table shows which combinations of variables are missing together.
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

**R**
```r
df$BP[is.na(df$BP)] <- mean(df$BP, na.rm = TRUE)
# Report: M_BP = X.XX, SD_BP = X.XX (n = XXX after imputation)
```

**SPSS**
```spss
* --- Mean imputation (benchmark only) ---
COMPUTE BP_mean_imp = BP.
IF (MISSING(BP)) BP_mean_imp = MEAN(BP).
EXECUTE.
* Report: M = X.XX, SD = X.XX before and after imputation.
```

---

#### ⚠️ Deterministic regression imputation — limited version
Predicts missing values without a residual error term →
also underestimates variance. Prefer the stochastic version
or MICE/PMM directly.

**R**
```r
# Deterministic (not recommended as final method)
model <- lm(BP ~ Age + Gender + BMI + Sleep + Chol + Smoke, data = df)
df$BP[is.na(df$BP)] <- predict(model, newdata = df[is.na(df$BP), ])

# Stochastic (recommended if using simple regression)
set.seed(12345)
df$BP[is.na(df$BP)] <- predict(model, newdata = df[is.na(df$BP), ]) +
  sample(residuals(model), sum(is.na(df$BP)), replace = TRUE)
# Report: M = X.XX, SD = X.XX, model R2 = X.XX
```

**SPSS**
```spss
* --- Regression imputation ---
REGRESSION
  /MISSING LISTWISE
  /STATISTICS COEFF R ANOVA
  /DEPENDENT BP
  /METHOD ENTER Age Gender BMI Sleep Chol Smoke
  /SAVE PRED(BP_reg_pred).

* Replace missing BP with predicted value
COMPUTE BP_reg_imp = BP.
IF (MISSING(BP)) BP_reg_imp = BP_reg_pred.
EXECUTE.
* Report: M = X.XX, SD = X.XX; regression R2 = X.XX
```

---

#### ✅ Multiple Imputation by Chained Equations (MICE) — reference method

**R**
```r
library(mice)
set.seed(12345)  # for reproducibility
imp <- mice(df, m = 20, method = "pmm", printFlag = FALSE)
# PMM = Predictive Mean Matching

# Fit model on each imputed dataset and pool results (Rubin's rules)
fit    <- with(imp, lm(BP ~ Age + Gender + BMI))
pooled <- pool(fit)
summary(pooled)
# Report: b = X.XX, SE = X.XX, t = X.XX, p = .XXX (pooled across m=20)
```

**SPSS**
```spss
* --- STEP 4: Multiple Imputation (MICE) ---
SET SEED 12345.

MULTIPLE IMPUTATION
  BMI Age Gender Sleep Smoke BP Chol
  /IMPUTE METHOD = AUTO
         NIMPUTATIONS = 20
  /OUTFILE IMPUTATIONS = 'nhanes_imputed.sav'
  /MISSINGSUMMARY = NONE.

* --- Pool results with Rubin's rules ---
* Open nhanes_imputed.sav, then:
DATASET ACTIVATE nhanes_imputed.
ANALYZE > Compare Means > Means
  /DEPENDENT BP Chol Smoke
  /LAYER Imputation_.
* SPSS pools M and SE automatically via Rubin's rules.
* Report: M = X.XX, SE = X.XX (pooled, m = 20)
```

---

#### ✅ Maximum Likelihood / EM

**SPSS**
```spss
* --- EM imputation ---
MISSING VALUE ANALYSIS
  /VARIABLES = BMI Age Gender Sleep Smoke BP Chol
  /EM(TOLERANCE=0.001 CONVERGENCE=0.0001 ITERATIONS=25
      OUTFILE='nhanes_em.sav').
* Report: log-likelihood = X.XX at convergence (iteration k)
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
2. % missing per variable (exact figures from output)
3. Diagnosed mechanism + exact test statistics (χ², df, p, r)
4. Chosen method + justification + reproducibility seed

**Example:**
> "Of the 350 observations, 312 had complete data on all variables
> (N complete = 312, 89.1%). Missingness was analysed via Little's MCAR test,
> χ²(24) = 38.14, p = .031, and correlations between missingness indicators
> and observed variables. A significant association between cholesterol missingness
> and age (r = .18, p = .042) suggested a MAR mechanism.
> Multiple imputation was performed (m = 20, MICE, PMM method, seed = 12345)
> following Rubin's (1987) pooling rules.
> Pooled estimates: M_BP = 122.4, SE = 1.3."

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
