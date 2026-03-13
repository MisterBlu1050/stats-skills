# Skill: Missing Data Analysis

**Version:** 2.0  
**Contexte original:** Statistiek IV — Multivariate Data-Analyse, VUB (Theuns et al., 2016)  
**Usage:** Généraliste — applicable à tout projet de recherche en sciences sociales

---

## Objectif
Guider une analyse complète des données manquantes selon un protocole en 4 étapes,
du diagnostic au traitement, avec code R et SPSS et recommandations de reporting APA.

---

## Entrées attendues
- Dataset (CSV, .sav, ou description des variables)
- Contexte de recherche (design, variables, question de recherche)
- Préférence logicielle : R ou SPSS

---

## ÉTAPE 1 — Classification : négligeable vs. non-négligeable

| Type | Exemples | Action |
|---|---|---|
| **Négligeable** | Skip-patterns, censored data | Ne rien faire |
| **Non-négligeable (connu)** | Erreurs de codage, questionnaires incomplets | Remédier |
| **Non-négligeable (inconnu)** | Refus items sensibles, abandon | Remédier + rapporter |

> Les données manquantes négligeables (attendues, structurelles) ne doivent
> JAMAIS être imputées.

---

## ÉTAPE 2 — Quantification

```r
library(naniar)
miss_var_summary(df)
vis_miss(df)
gg_miss_var(df)
```

```spss
* Analyze > Missing Value Analysis
```

**Seuils indicatifs (Hair et al., 2013)**

> ⚠️ Ces seuils sont des heuristiques opérationnelles, non des règles absolues.
> C'est le **mécanisme** (MCAR/MAR/MNAR) qui détermine la stratégie appropriée,
> pas le pourcentage seul (van Buuren, 2018 ; Madley-Dowd et al., 2019).
> Même 5% de missing sous MNAR peut biaiser sérieusement les résultats ;
> à l'inverse, 15% sous MCAR avec grand N reste gérable par listwise deletion
> (perte de puissance, pas de biais).

| % missing | Stratégie indicative |
|---|---|
| < 10% | Toutes méthodes applicables ; listwise acceptable sous MCAR |
| 10–20% | Listwise/régression sous MCAR ; modelgebaseerd sous MAR |
| > 20% | Régression sous MCAR ; MI ou EM sous MAR |

---

## ÉTAPE 3 — Mécanisme : MCAR / MAR / MNAR

### Définitions

| Mécanisme | Cause | Testable ? |
|---|---|---|
| **MCAR** | Entièrement aléatoire, indépendant de toutes les données | Oui (Little's test) |
| **MAR** | Dépend de variables *observées*, pas de la valeur manquante elle-même | Indirectement |
| **MNAR** | Dépend de la *valeur manquante elle-même* | Non directement |

### Tests diagnostiques

**a) Test de Little (MCAR)**
```r
library(naniar)
mcar_test(df)
```
```spss
* Analyze > Missing Value Analysis > EM
```
> ⚠️ **Double limite du test de Little :**
> - Grand N → surpuissance : p < .05 pour des écarts pratiquement négligeables
> - Petit N → sous-puissance : p > .05 **ne confirme pas MCAR** —
>   cela signifie uniquement l'absence d'évidence suffisante pour le rejeter
>   (erreur de type II possible)

**b) Corrélations avec indicateurs de missingness**
```r
library(dplyr)
df_ind <- df %>%
  mutate(across(everything(),
                ~as.integer(is.na(.)),
                .names = "{col}_missing"))
cor(df_ind, use = "pairwise.complete.obs")
```
> Corrélation significative entre l'indicateur de missingness de Y
> et une variable observée X → indice de MAR par rapport à X

**c) Test t sur groupes**  
Comparer les moyennes sur toutes les autres variables entre :
- cas avec missing sur Y
- cas complets sur Y

**d) Visualisation des patterns**
```r
gg_miss_upset(df)
gg_miss_fct(df, gender)
```

---

## ÉTAPE 4 — Remédiation

| Situation | Stratégie recommandée |
|---|---|
| Négligeable | Rien faire |
| MCAR + < 10% | Listwise acceptable ; MI ou EM préférables |
| MCAR + 10–20% | Imputation par régression ou MI |
| MAR | **Multiple Imputation (MICE)** ou Maximum Likelihood (EM) |
| MNAR | Pas d'imputation fiable — analyses de sensibilité |

### Méthodes d'imputation

---

#### ⛔ Mean/Median imputation — à éviter dans la majorité des cas
Sous-estime la variance, produit des IC artificiellement étroits,
biaise les corrélations. À utiliser uniquement comme **benchmark comparatif**,
jamais comme méthode finale dans un article ou mémoire.

```r
df$BP[is.na(df$BP)] <- mean(df$BP, na.rm = TRUE)
```

---

#### ⚠️ Régression déterministe — version limitée
Prédit les valeurs manquantes sans terme d'erreur résiduel →
sous-estime également la variance. Préférer la version stochastique
ou directement MICE/PMM.

```r
# Version déterministe (déconseillée comme méthode finale)
model <- lm(BP ~ Age + Gender + BMI + Sleep + Chol + Smoke, data = df)
df$BP[is.na(df$BP)] <- predict(model, newdata = df[is.na(df$BP), ])

# Version stochastique (recommandée si régression simple)
df$BP[is.na(df$BP)] <- predict(model, newdata = df[is.na(df$BP), ]) +
  sample(residuals(model), sum(is.na(df$BP)), replace = TRUE)
```

---

#### ✅ Multiple Imputation by Chained Equations (MICE) — méthode de référence
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
* SPSS poolt automatiquement via les règles de Rubin
```

---

#### ✅ Maximum Likelihood / EM
```spss
* Analyze > Missing Value Analysis > EM
```

---

#### MNAR — Analyses de sensibilité (niveau avancé)
Quand MNAR est suspecté, aucune imputation ne peut être validée
sans hypothèses non testables. Les approches recommandées sont :
- **δ-adjustment** : introduire un décalage systématique dans les valeurs imputées
  pour tester la robustesse des conclusions
- **Pattern-mixture models** : modéliser séparément les sous-groupes
  selon leur pattern de missingness
- **Selection models** : modéliser conjointement le mécanisme de missingness
  et le modèle substantiel

> Même sans ces méthodes avancées, rapporter explicitement
> les limites liées au MNAR suspect est une exigence éthique et scientifique.

---

## Reporting (style APA)

Inclure systématiquement :
1. **N total**, **N complets** (avant traitement), **N après imputation**
2. % missing par variable
3. Mécanisme diagnostiqué + test utilisé + ses limites
4. Méthode choisie + justification

**Exemple :**
> "Sur les 350 observations, 312 présentaient des données complètes sur toutes
> les variables (N complet = 312, 89.1%). L'analyse du mécanisme de missingness
> via le test de Little, χ²(df) = X.XX, p = .XXX, et les corrélations entre
> indicateurs de missingness et variables observées, a révélé une association
> significative entre le missingness du cholestérol et l'âge (r = .XX, p < .05),
> suggérant un mécanisme MAR. Une imputation multiple a été réalisée
> (m = 20, MICE, méthode PMM) selon les règles de pooling de Rubin (1987)."

---

## Références
- Theuns, P., Van Den Bussche, E., Isaac, C., & Muscarella, F. (2016).
  *Multivariate Data-analyse*. VUB Uitgaven.
- Hair, J.F. et al. (2019). *Multivariate Data Analysis* (8e ed.). Cengage.
- Field, A. (2024). *Discovering Statistics Using IBM SPSS Statistics* (6e ed.). Sage.
- van Buuren, S. (2018). *Flexible Imputation of Missing Data* (2e ed.).
  CRC Press. https://stefvanbuuren.name/fimd/
- van Buuren, S. & Groothuis-Oudshoorn, K. (2011). mice: Multivariate Imputation
  by Chained Equations in R. *Journal of Statistical Software*, 45(3).
- Madley-Dowd, P. et al. (2019). The proportion of missing data should not be
  used to guide decisions on multiple imputation. *Journal of Clinical
  Epidemiology*, 110, 63–73.
- Rubin, D.B. (1987). *Multiple Imputation for Nonresponse in Surveys*. Wiley.
- Schafer, J.L. & Graham, J.W. (2002). Missing data: Our view of the state of
  the art. *Psychological Methods*, 7(2), 147–177.
