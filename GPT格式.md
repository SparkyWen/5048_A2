# GPT格式

下面是 **Typora 友好** 的 Markdown（行内 `$...$`；块级 `$$...$$`；并保证公式块上下各留一行）。直接整段复制即可。

```markdown
## 2.2 Country-level marginal contribution via leave-one-out $\Delta\eta^2$

Group-level $\eta^2$ says which variable explains separation, but it does not tell which countries drive that separation. I therefore designed a leave-one-out (LOO) contribution.

For each panel (fixed year $t$ and indicator $v$) and each country $i$:

1. Compute $\eta^2_{\text{all}}$ from **all** countries in that panel.

2. Remove country $i$ (and adjust its group’s sample size and mean), recompute $\eta^2_{-i}$.

3. Define the **marginal contribution**
   $$
   \Delta\eta^2_i \;=\; \eta^2_{\text{all}} \;-\; \eta^2_{-i}.
   $$

**Interpretation.**

- $\Delta\eta^2_i>0$: country $i$ increases between-group separation for that indicator and year (it strengthens the grouping signal).
- $\Delta\eta^2_i<0$: country $i$ reduces separation (a “drag” or counter-pattern).

For attribution charts I normalise the positive deltas within each **(country, year)**:
$$
\text{share}_{i,v,t}
= \frac{\max\!\big(\Delta\eta^2_{i,v,t},\,0\big)}
       {\sum_{u} \max\!\big(\Delta\eta^2_{i,u,t},\,0\big)}
\times 100\%.
$$
These shares sum to $100\%$ per country–year (when at least one $\Delta\eta^2_{i,\cdot,t}>0$) and yield interpretable lines over time.

**Note. Causation vs correlation.** $\eta^2$ is an effect size for association, not proof of causality. The visual narrative and tooltips therefore avoid causal language. See the “correlation $\neq$ causation” explainer slide **[INSERT FIG REF]** that I prepared for the discussion section.

---

## 2.3 Algorithmic design and mapping to code

I implemented two core components.

### (A) Group-level $\eta^2$ per year × indicator

**Code mapping.** `eta_squared_for_year(dyear, col)` in your bar-chart notebook.

**Data need.** Only values and group labels for the year; we ignore missing values (no imputation as per task rule).

**Computation.** For each indicator $v$ in year $t$, compute the overall mean and group means, then sums of squares:
$$
\bar{x}=\frac{1}{N}\sum_c x_c,\qquad
\bar{x}_g=\frac{1}{n_g}\sum_{c\in g} x_c,
$$
$$
SS_{total}=\sum_c (x_c-\bar{x})^2,\qquad
SS_{between}=\sum_g n_g\,(\bar{x}_g-\bar{x})^2,\qquad
\eta^2=\frac{SS_{between}}{SS_{total}}.
$$

### (B) LOO contributions per country × year × indicator

**Code mapping.**

- `_eta2_from_stats(N, sum_x, sum_sq, mu, grp_n, grp_mu)` computes $\eta^2$ from sufficient statistics.
- `loo_eta2_contrib_one_panel(panel)` loops over countries in a given panel; for each country $i$ it updates only its group’s count and mean and recomputes $\eta^2_{-i}$ in $O(G)$ time (where $G$ is the number of groups), rather than re-scanning raw rows.

**Edge cases.** If removal empties a group ($n_g=1$ before removal), the group contributes zero to $SS_{between}$ in the LOO calculation; if $SS_{total}=0$ we return `NaN` (no dispersion to explain).

**Complexity.** For a panel with $N$ countries and $G$ groups, the LOO pass costs $O(N\cdot G)$ with tiny constants; this is comfortably real-time for our dataset.

---

## 2.4 Visual design decisions

### (i) Group-level bar chart (Task 2: attribute trends over time)

**Encoding.** Nominal indicators on the $x$-axis; height encodes $\eta^2$ **share** (normalised to percentage per year to facilitate across-year comparisons).

**Interaction.** A slider controls year; the legend allows filtering/isolation by indicator (Shneiderman’s mantra: overview → zoom & filter → details on demand).

**Affordances for reasoning.** A right-side annotation shows the full membership list for the selected group and year (hover to read), which grounds interpretation (e.g., a spike in Inflation (%) coinciding with membership shifts).

**Why bar, not line?** The year is a control rather than a plotted scale; bar heights maximise perceptual accuracy for part-to-whole comparisons (Cleveland & McGill, 1984).

### (ii) Country-level line chart (Task 2: membership dynamics by attribute)

**Encoding.** For a selected country, seven lines plot $\text{share}_{i,v,t}$ (the normalised positive LOO contributions) over time. Values sum to $100\%$ per year, giving a clear per-year “budget” of explanatory emphasis.

**Tooltips.** Each point displays *Year*, *Share* (only when $\Delta\eta^2>0$), *Raw value*, *Group*, and signed $\Delta\eta^2$ to reveal when a variable turns negative (drag).

**Why positive-part normalisation?** For visual attribution we prefer to distribute the helpful signal across variables; negative $\Delta\eta^2$ is still available in tooltips for diagnosis.

### (iii) Accessibility & consistency

Uniform axis ranges (0–100%). No imputation; missing data simply produces gaps, which is more honest and highlights data-sparsity years by design.

---

## 2.5 Evaluation of the visualisations

I evaluated both correctness and utility.

**Correctness checks.**

- **Mass balance.** For each year $t$, the indicator shares in the bar chart sum to $100\%$ ($\pm\varepsilon$). Observed mean absolute deviation: **[INSERT VALUE]** across **[INSERT #YEARS]** years.
- **Country-year normalisation.** For each selected country and year,
  $$
  \sum_v \text{share}_{i,v,t}=100\%
  $$
  (where at least one positive $\Delta\eta^2$ exists). Observed mean deviation: **[INSERT VALUE]**.
- **Sanity under removal.** When I manually remove an extreme-value country from a high-$\eta^2$ indicator, the recomputed $\eta^2$ drops:
  $$
  \eta^2_{\text{all}}=\text{[INSERT]},\qquad
  \eta^2_{-i}=\text{[INSERT]},\qquad
  \Delta\eta^2_i=\text{[INSERT]}>0.
  $$
- **Cross-year consistency.** Indicators known to shift importance during **[INSERT PERIOD, e.g., 2020–2022]** (e.g., Inflation (%) during post-pandemic shocks) show corresponding rises in the bar chart and in the LOO lines for a broad set of countries. See **[INSERT FIG REF(S)]**.

**Analyst utility (user testing).**  
Using the Brehmer–Munzner task typology, the visuals support:

- Discover which indicator explains groupings in a given year (bar chart).
- Identify countries whose membership is most sensitive to specific indicators (LOO lines).
- Compare years to characterise temporal shifts in defining attributes.

Feedback from two peers indicated they could articulate “what changed and why” for **[INSERT TWO EXAMPLE COUNTRIES]** within **[INSERT TIME]** after a short demo.

---

## 2.6 Findings enabled by the visuals (with placeholders)

*(Illustrative template—replace bracketed items with your values.)*

- In **2024**, GDP per Capita explains **[INSERT %]** of between-group separation, while Inflation (%) explains **[INSERT %]** (Fig. **[INSERT]**).
- For **[COUNTRY A]**, the positive LOO share in **[YEAR]** is dominated by **[INDICATOR]** at **[INSERT %]**, but by **[YEAR+Δ]** it shifts to **[INDICATOR]** at **[INSERT %]**, aligned with the country’s group change from **[GROUP X]** to **[GROUP Y]** (Fig. **[INSERT]**).
- For **[COUNTRY B]**, Unemployment (%) repeatedly exhibits negative $\Delta\eta^2$ in **[YEARS]**, suggesting it counteracts group separation for that country (visible in tooltips).

---

## 2.7 Limitations, ethics, and future work

- **Association, not causation.** $\eta^2$ quantifies association between group labels and indicator values; it does not imply that changing an indicator would cause a group change. I explicitly communicated this with an explainer panel (Fig. **[INSERT]**).
- **No imputation.** Per task constraints, we do not fill missing values. This makes attribution conservative; gaps indicate insufficient data rather than interpolated artefacts.
- **Aggregation sensitivity.** Group means and group sizes both affect $\eta^2$. In future work we could add robust effect sizes (e.g., ANOVA on ranks) or stratified analyses by region/income to test stability.
- **What-if tooling.** The LOO engine can be extended into an interactive counterfactual slider that perturbs a country’s indicator and recomputes $\eta^2$ to simulate minimal changes needed to maintain group membership, directly addressing the “critical moment/what-if” requirement.

---

## 2.8 Reproducibility and mapping to the group report

**Reproduce.** Run the two notebooks:  
(i) Group-level $\eta^2$ bar chart by year (code block starting at `eta_squared_for_year`);  
(ii) Country-level LOO lines (code block starting at `_eta2_from_stats` / `loo_eta2_contrib_one_panel`). No random seeds are involved; results are deterministic given the CSV.

**Where my work appears in the group report.**

- Section **[INSERT]**: Figures **[INSERT IDs]** (bar chart, yearly $\eta^2$ shares).
- Section **[INSERT]**: Figures **[INSERT IDs]** (country LOO line charts).
- Appendix **[INSERT]**: Algorithm notes and diagnostic tables (e.g., normalisation checks).
```

## 2.2 Country-level marginal contribution via leave-one-out $\Delta\eta^2$

Group-level $\eta^2$ says which variable explains separation, but it does not tell which countries drive that separation. I therefore designed a leave-one-out (LOO) contribution.

For each panel (fixed year $t$ and indicator $v$) and each country $i$:

1. Compute $\eta^2_{\text{all}}$ from **all** countries in that panel.

2. Remove country $i$ (and adjust its group’s sample size and mean), recompute $\eta^2_{-i}$.

3. Define the **marginal contribution**
   $$
   \Delta\eta^2_i \;=\; \eta^2_{\text{all}} \;-\; \eta^2_{-i}.
   $$

**Interpretation.**

- $\Delta\eta^2_i>0$: country $i$ increases between-group separation for that indicator and year (it strengthens the grouping signal).
- $\Delta\eta^2_i<0$: country $i$ reduces separation (a “drag” or counter-pattern).

For attribution charts I normalise the positive deltas within each **(country, year)**:
$$
\text{share}_{i,v,t}
= \frac{\max\!\big(\Delta\eta^2_{i,v,t},\,0\big)}
       {\sum_{u} \max\!\big(\Delta\eta^2_{i,u,t},\,0\big)}
\times 100\%.
$$
These shares sum to $100\%$ per country–year (when at least one $\Delta\eta^2_{i,\cdot,t}>0$) and yield interpretable lines over time.

**Note. Causation vs correlation.** $\eta^2$ is an effect size for association, not proof of causality. The visual narrative and tooltips therefore avoid causal language. See the “correlation $\neq$ causation” explainer slide **[INSERT FIG REF]** that I prepared for the discussion section.

---

## 2.3 Algorithmic design and mapping to code

I implemented two core components.

### (A) Group-level $\eta^2$ per year × indicator

**Code mapping.** `eta_squared_for_year(dyear, col)` in your bar-chart notebook.

**Data need.** Only values and group labels for the year; we ignore missing values (no imputation as per task rule).

**Computation.** For each indicator $v$ in year $t$, compute the overall mean and group means, then sums of squares:
$$
\bar{x}=\frac{1}{N}\sum_c x_c,\qquad
\bar{x}_g=\frac{1}{n_g}\sum_{c\in g} x_c,
$$
$$
SS_{total}=\sum_c (x_c-\bar{x})^2,\qquad
SS_{between}=\sum_g n_g\,(\bar{x}_g-\bar{x})^2,\qquad
\eta^2=\frac{SS_{between}}{SS_{total}}.
$$

### (B) LOO contributions per country × year × indicator

**Code mapping.**

- `_eta2_from_stats(N, sum_x, sum_sq, mu, grp_n, grp_mu)` computes $\eta^2$ from sufficient statistics.
- `loo_eta2_contrib_one_panel(panel)` loops over countries in a given panel; for each country $i$ it updates only its group’s count and mean and recomputes $\eta^2_{-i}$ in $O(G)$ time (where $G$ is the number of groups), rather than re-scanning raw rows.

**Edge cases.** If removal empties a group ($n_g=1$ before removal), the group contributes zero to $SS_{between}$ in the LOO calculation; if $SS_{total}=0$ we return `NaN` (no dispersion to explain).

**Complexity.** For a panel with $N$ countries and $G$ groups, the LOO pass costs $O(N\cdot G)$ with tiny constants; this is comfortably real-time for our dataset.

---

## 2.4 Visual design decisions

### (i) Group-level bar chart (Task 2: attribute trends over time)

**Encoding.** Nominal indicators on the $x$-axis; height encodes $\eta^2$ **share** (normalised to percentage per year to facilitate across-year comparisons).

**Interaction.** A slider controls year; the legend allows filtering/isolation by indicator (Shneiderman’s mantra: overview → zoom & filter → details on demand).

**Affordances for reasoning.** A right-side annotation shows the full membership list for the selected group and year (hover to read), which grounds interpretation (e.g., a spike in Inflation (%) coinciding with membership shifts).

**Why bar, not line?** The year is a control rather than a plotted scale; bar heights maximise perceptual accuracy for part-to-whole comparisons (Cleveland & McGill, 1984).

### (ii) Country-level line chart (Task 2: membership dynamics by attribute)

**Encoding.** For a selected country, seven lines plot $\text{share}_{i,v,t}$ (the normalised positive LOO contributions) over time. Values sum to $100\%$ per year, giving a clear per-year “budget” of explanatory emphasis.

**Tooltips.** Each point displays *Year*, *Share* (only when $\Delta\eta^2>0$), *Raw value*, *Group*, and signed $\Delta\eta^2$ to reveal when a variable turns negative (drag).

**Why positive-part normalisation?** For visual attribution we prefer to distribute the helpful signal across variables; negative $\Delta\eta^2$ is still available in tooltips for diagnosis.

### (iii) Accessibility & consistency

Uniform axis ranges (0–100%). No imputation; missing data simply produces gaps, which is more honest and highlights data-sparsity years by design.

---

## 2.5 Evaluation of the visualisations

I evaluated both correctness and utility.

**Correctness checks.**

- **Mass balance.** For each year $t$, the indicator shares in the bar chart sum to $100\%$ ($\pm\varepsilon$). Observed mean absolute deviation: **[INSERT VALUE]** across **[INSERT #YEARS]** years.
- **Country-year normalisation.** For each selected country and year,
  $$
  \sum_v \text{share}_{i,v,t}=100\%
  $$
  (where at least one positive $\Delta\eta^2$ exists). Observed mean deviation: **[INSERT VALUE]**.
- **Sanity under removal.** When I manually remove an extreme-value country from a high-$\eta^2$ indicator, the recomputed $\eta^2$ drops:
  $$
  \eta^2_{\text{all}}=\text{[INSERT]},\qquad
  \eta^2_{-i}=\text{[INSERT]},\qquad
  \Delta\eta^2_i=\text{[INSERT]}>0.
  $$
- **Cross-year consistency.** Indicators known to shift importance during **[INSERT PERIOD, e.g., 2020–2022]** (e.g., Inflation (%) during post-pandemic shocks) show corresponding rises in the bar chart and in the LOO lines for a broad set of countries. See **[INSERT FIG REF(S)]**.

**Analyst utility (user testing).**  
Using the Brehmer–Munzner task typology, the visuals support:

- Discover which indicator explains groupings in a given year (bar chart).
- Identify countries whose membership is most sensitive to specific indicators (LOO lines).
- Compare years to characterise temporal shifts in defining attributes.

Feedback from two peers indicated they could articulate “what changed and why” for **[INSERT TWO EXAMPLE COUNTRIES]** within **[INSERT TIME]** after a short demo.

---

## 2.6 Findings enabled by the visuals (with placeholders)

*(Illustrative template—replace bracketed items with your values.)*

- In **2024**, GDP per Capita explains **[INSERT %]** of between-group separation, while Inflation (%) explains **[INSERT %]** (Fig. **[INSERT]**).
- For **[COUNTRY A]**, the positive LOO share in **[YEAR]** is dominated by **[INDICATOR]** at **[INSERT %]**, but by **[YEAR+Δ]** it shifts to **[INDICATOR]** at **[INSERT %]**, aligned with the country’s group change from **[GROUP X]** to **[GROUP Y]** (Fig. **[INSERT]**).
- For **[COUNTRY B]**, Unemployment (%) repeatedly exhibits negative $\Delta\eta^2$ in **[YEARS]**, suggesting it counteracts group separation for that country (visible in tooltips).

---

## 2.7 Limitations, ethics, and future work

- **Association, not causation.** $\eta^2$ quantifies association between group labels and indicator values; it does not imply that changing an indicator would cause a group change. I explicitly communicated this with an explainer panel (Fig. **[INSERT]**).
- **No imputation.** Per task constraints, we do not fill missing values. This makes attribution conservative; gaps indicate insufficient data rather than interpolated artefacts.
- **Aggregation sensitivity.** Group means and group sizes both affect $\eta^2$. In future work we could add robust effect sizes (e.g., ANOVA on ranks) or stratified analyses by region/income to test stability.
- **What-if tooling.** The LOO engine can be extended into an interactive counterfactual slider that perturbs a country’s indicator and recomputes $\eta^2$ to simulate minimal changes needed to maintain group membership, directly addressing the “critical moment/what-if” requirement.

---

## 2.8 Reproducibility and mapping to the group report

**Reproduce.** Run the two notebooks:  
(i) Group-level $\eta^2$ bar chart by year (code block starting at `eta_squared_for_year`);  
(ii) Country-level LOO lines (code block starting at `_eta2_from_stats` / `loo_eta2_contrib_one_panel`). No random seeds are involved; results are deterministic given the CSV.

**Where my work appears in the group report.**

- Section **[INSERT]**: Figures **[INSERT IDs]** (bar chart, yearly $\eta^2$ shares).
- Section **[INSERT]**: Figures **[INSERT IDs]** (country LOO line charts).
- Appendix **[INSERT]**: Algorithm notes and diagnostic tables (e.g., normalisation checks).







```markdown
# 2. Visualisation contributions (15 pts)

## 2.1 Rationale for switching from z-scores to $\eta^2$

Early prototypes used z-scores to "highlight" unusual values. However, z-scores measure within-indicator **standardisation** and do not speak directly to **group separability**. The assignment's Task 2 requires understanding how group membership and the attributes that define groups evolve over time. For that analytic question the relevant quantity is the **proportion of total variance explained by between-group differences**, i.e., $\eta^2$:

$$
SS_{total}=\sum_{c}\left(x_c-\bar{x}\right)^2,\qquad
SS_{between}=\sum_{g} n_g\left(\bar{x}_g-\bar{x}\right)^2,\qquad
\eta^2=\frac{SS_{between}}{SS_{total}}\in[0,1].
$$

- **Large $\eta^2$** (closer to 1) for indicator $v$ in year $t$ means group means are far apart relative to overall spread, so that indicator strongly "organises" countries into groups.  
- **Small $\eta^2$** means groups are indistinguishable on that indicator.

This directly addresses "which attributes define the groupings and how this changes over time," whereas z-scores do not.

> **Numeric micro-example.** Insert your one-slide step-by-step example here as Figure **[INSERT FIG REF]**, showing

$$
SS_{between}=80,\; SS_{total}=100 \Rightarrow \eta^2=0.8.
$$

> **[INSERT FIG REF]**

```

# 2. Visualisation contributions (15 pts)

## 2.1 Rationale for switching from z-scores to $\eta^2$

Early prototypes used z-scores to "highlight" unusual values. However, z-scores measure within-indicator **standardisation** and do not speak directly to **group separability**. The assignment's Task 2 requires understanding how group membership and the attributes that define groups evolve over time. For that analytic question the relevant quantity is the **proportion of total variance explained by between-group differences**, i.e., $\eta^2$:

$$
SS_{total}=\sum_{c}\left(x_c-\bar{x}\right)^2,\qquad
SS_{between}=\sum_{g} n_g\left(\bar{x}_g-\bar{x}\right)^2,\qquad
\eta^2=\frac{SS_{between}}{SS_{total}}\in[0,1].
$$

- **Large $\eta^2$** (closer to 1) for indicator $v$ in year $t$ means group means are far apart relative to overall spread, so that indicator strongly "organises" countries into groups.  
- **Small $\eta^2$** means groups are indistinguishable on that indicator.

This directly addresses "which attributes define the groupings and how this changes over time," whereas z-scores do not.

> **Numeric micro-example.** Insert your one-slide step-by-step example here as Figure **[INSERT FIG REF]**, showing

$$
SS_{between}=80,\; SS_{total}=100 \Rightarrow \eta^2=0.8.
$$

> **[INSERT FIG REF]**