# task2&3 and individual part

# Individual Component (4 pages)

# 1. Non-visualisation contributions (5 pts)

**Data engineering & provenance.** 

I kept every variable in datasets remain the same, didn't do any cleaning or handle the missing values. But established a reproducible data pipeline (Python/pandas). In that copied table, I enforced strict schema checks (assert on `Country Name`, `Year`, `Group`) and converted `Year` to integer types to prevent silent coercion errors. 

**Algorithmic implementation.**  

I implemented from first principles (no black-box stats packages) the between-group effect size $\eta^2$ for each indicator $(v)$ and year $(t)$, and then designed an efficient leave-one-out (LOO) recomputation that quantifies every **country–year–indicator**’s marginal contribution to group separability. The implementation uses **sufficient statistics**—counts, means, sum of squares—rather than re-computing from raw rows each time, reducing complexity to $O(N\cdot G)$ per panel (where a “panel” is a fixed year × indicator). See §2.2–§2.3.

**Quality assurance.**  

I wrote internal consistency checks and “sanity” diagnostics:  

(i) verify $\eta^2\in[0,1]$;  

(ii) verify, for a fixed country–year, that the positive parts of the LOO deltas normalise to $100\%$ (up to numerical tolerance);  

**Project management.**  

I maintained a change log for each visualisation iteration (from z-scores to $\eta^2$ and to LOO contributions) and coordinated parameter lock-down before the team compiled final screenshots and the demo video.

# 2. Visualisation contributions (15 pts)

## 2.1 Rationale for switching from z-scores to $\eta^2$

Early prototypes used z-scores to "highlight" unusual values. 
$$
B_{y,v}^{(\mathrm{L1})}
=\sum_{g} n_{g,y,v}\,\bigl|\bar{z}_{g,y,v}\bigr|
$$

$$
\mathrm{Share}^{(\mathrm{L1})}_{y,v}
=\frac{B_{y,v}^{(\mathrm{L1})}}{\sum_{v'\in\mathcal{V}} B_{y,v'}^{(\mathrm{L1})}}
\times 100\%.
$$

However, z-scores measure within-indicator **standardisation** and do not speak directly to **group separability**. For example, the absulate value can not speak the direction of each variable if speak to contribution to sparate each Group. Therefore,  to better understand how group membership and the attributes that define groups evolve over time. I chose $\eta^2$ as new indicator. The analytic question the relevant quantity is the **proportion of total variance explained by between-group differences**, i.e., $\eta^2$:
$$
SS_{total}=\sum_{c}\left(x_c-\bar{x}\right)^2,\qquad
SS_{between}=\sum_{g} n_g\left(\bar{x}_g-\bar{x}\right)^2,\qquad
\eta^2=\frac{SS_{between}}{SS_{total}}\in[0,1].
$$

- **Large $\eta^2$** (closer to 1) for indicator $v$ in year $t$ means group means are far apart relative to overall spread, so that indicator strongly "organises" countries into groups.  
- **Small $\eta^2$** means groups are indistinguishable on that indicator.

This directly addresses "which attributes define the groupings and how this changes over time," whereas z-scores do not. Hence, through this method, we got the results how each variable contributes to each Group in Group level. However, it raised another question that what if we wanna know how each country contributes to each Group given such amount missing values. So we create 2.2 for the country level to see the contributions. 

> **==补充说明这里== ：Numeric micro-example.** Insert your one-slide step-by-step example here as Figure **[INSERT FIG REF]**, showing

$$
SS_{between}=80,\; SS_{total}=100 \Rightarrow \eta^2=0.8.
$$

> **[INSERT FIG REF]**

## 2.2 Country-level marginal contribution via leave-one-out $\Delta\eta^2$

Simply, this equation illustrates how separate variable contributes to the Group, if remove this attribbutes, what gonna happen to the Group. 

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

**==这里是解释因果和相关是有区别的==Note. Causation vs correlation.** $\eta^2$ is an effect size for association, not proof of causality. The visual narrative and tooltips therefore avoid causal language. See the “correlation $\neq$ causation” explainer slide **[INSERT FIG REF]** that I prepared for the discussion section. 

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

I evaluated both correctness and utility. Like I mentioned before, the contributes can not use the z-score contribution because for some variables the contribution can be negative. So the direction can not simply use the abosute value to represent. The second reflection is that when i implement the visualization, i found for most time the GDP per Capita always remain the big part, i realized there is one fault i made is i didn't normalize each variables, some of the measurement is % but others are values. So i normalize the data and fixed the problem. There are other refinement like color- friendly, remove overlapping also implemented during optimization. So here are the results. 

# Group level

![](./results/visual1.png)

### Visual encoding

- **Bar chart** (one bar per indicator).
- **X-axis:** indicator names.
- **Y-axis:** contribution to A/B/C separation (**%**, 0–100).
- **Data labels** on bars show exact percentages.
- **Color** encodes the indicator; colors are consistent with the legend.

### Interactions

- **Legend (top-right):** click to toggle indicators; **double-click** an item to isolate it.
- **Year slider (bottom):** scrubs the timeline; each position animates to that year’s distribution.
- **Hover tooltip:** shows the year and the indicator’s share; also lists the **full set of Group B members** for that year and their **count**.
- **Dynamic annotation (right panel):** updates with the **member count** and quick interaction tips.

### Reading the chart

1. Pick a **year** from the slider at the bottom.
2. Compare **bar heights** to see which indicators most explain the separation that year (e.g., a tall GDP-per-capita bar means it dominates explanation that year).
3. Use the legend to focus on a subset (e.g., only prices-related indicators).
4. Hover any bar to verify the exact % and check who is in **Group B** for that year.



# Country level

![](./results/visual2.png)

### Visual encoding

- **One polyline per indicator** (legend colors).
- **X-axis:** Year; **Y-axis:** Share from positive Δη² in that year (%).
- **Gaps** in a line = indicator missing or no positive contribution that year (no imputation)



### Interactions

- **Country dropdown (top-left):** switch the profiled country.
- **Legend (top-right):** click to hide/show indicators; double-click to isolate one.
- **Hover a point:** shows Year, the **normalized share %**, the country’s **raw indicator value**, its **group label** that year, and the **signed Δη²** (so you can see if the unnormalized effect was negative).



## How to use it (quick workflow)

1. **Pick a country** from the dropdown.
2. **Scan peaks**: the tallest line(s) in a year are the **dominant drivers** of this country’s group separation then.
3. **Trace trajectories** to see how the **driver mix changes over time** (e.g., Imports ↓ while Population Growth ↑).
4. **Toggle lines** in the legend to focus on a subset (prices, trade, labor).
5. **Hover** to verify exact % share, the **raw value**, and whether the underlying Δη² was positive/negative.

# Here's one wrong visualization using the Z-score:

![](./results/falsevisual1.png)

as we can see, the country American Samoa in 2015, the Imports contributes 26.92% using the Z-score contribution, however, the correct one is 68.75%. Which is a huge discrepancy. So i made this algorithm change to the contribution with direction, using the Country-level marginal contribution via leave-one-out $\eta^2$. 

Here are more details:

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

## 2.6 Findings enabled by the visuals (==补充with placeholders==)

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



# task2

# 2. Temporal Analysis

## 2.0 Aim and Method (short recap for both groups)

**Aim.** To describe how (i) **group membership** and (ii) the **attribute combination that defines each group** evolve over time.

**Metrics & visuals.** We quantify attribute importance by **η²** (share of between-group variance). We examine individual mechanisms with **leave-one-out (LOO) Δη²** shares per country. The analysis uses:

- **Fig. T-1**: Yearly η² composition (stacked bars or ranked bars).
- **Fig. T-2**: Membership flows (Alluvial/Sankey or year-to-year transition matrix).
- **Fig. T-3**: Country-level LOO share trends (lines by variable, dropdown by country).

**Change thresholds (define once).**
 A year is flagged as a **combination change** if any of the following holds:

1. Top-k indicators (k = **[INSERT k]**) replace ≥ **[INSERT m]** items relative to last year;
2. Cumulative η² of Top-k changes by ≥ **[INSERT %]**;
3. The leading indicator’s η² shifts by ≥ **[INSERT %]** (relative).

> *Note*: η² is an **effect size** (association), not causation.

## 2.A Group A — Temporal Analysis

### 2.A.1 Membership dynamics over time

**What we see.**
 From **[INSERT start year]** to **[INSERT end year]**, Group A ranges between **[INSERT min]** and **[INSERT max]** members (mean **[INSERT]**). The largest year-to-year change occurs in **[INSERT year→year]**, when **[INSERT +N/−N]** countries **[joined/left]** Group A.

- **Key flows.** Major transitions include:
  - **[INSERT country1]**: **A→[B/C]** in **[INSERT year]**.
  - **[INSERT country2]**: **[B/C]→A** in **[INSERT year]**.
  - **[Optional]** Persistent stayers: **[INSERT 3–5 countries]** remained in A throughout the period.

**Evidence.**

- **Fig. A-1 (Membership flows)**: Alluvial diagram highlighting A↔B/C transitions.
- **Table A-1 (Transition counts)**: Row = origin year group, column = next-year group, entries show **[INSERT]** moves into A.

### 2.A.2 Attribute-combination dynamics (what defines Group A each year)

**Headline pattern.**
 In **[INSERT year]**, Group A is primarily defined by **[IND1]** and **[IND2]** (η² shares **[INSERT %]**, **[INSERT %]**). By **[INSERT later year]**, the combination shifts to **[IND3]**/**[IND4]** (η² **[INSERT %]**, **[INSERT %]**), exceeding the change threshold **[INSERT rule reference]**.

- **Top-k over time.**
  - **[INSERT year1]**: Top-k = **[IND-list]** (cum η² = **[INSERT %]**).
  - **[INSERT year2]**: Top-k = **[IND-list]** (cum η² = **[INSERT %]**).
  - **[INSERT year3]**: … (highlight any replacement).

**Evidence.**

- **Fig. A-2 (Yearly η² composition)**: Stacked/ranked bars per year.
- **Table A-2 (Top-k by year)**: Indicators and cumulative η²; replacements bolded.

### 2.A.3 Linking member changes to attribute changes (country-level mechanism)

Select **2–3** representative cases that moved into/out of A.

**Case A-α — [INSERT country], [INSERT year t→t+1].**

- **Membership change.** **[Country]** moved **[A→B / B→A]** in **[INSERT]**.
- **Group-level shift.** Over the same interval, η² of **[INDX]** **[rose/fell]** from **[INSERT %]** to **[INSERT %]** (Fig. A-2).
- **LOO mechanism.** The country’s LOO share for **[INDX]** **[increased/decreased]** from **[INSERT %]** to **[INSERT %]**, while **[INDY]** turned **[negative/less positive]** (Signed Δη² = **[INSERT]**).
- **Interpretation.** This aligns with Group A’s definition shifting toward **[INDX]**, making **[country]** **[less/more]** aligned with A.

**Evidence.**

- **Fig. A-3a (LOO lines, country = [INSERT])**: Seven-line share plot with hover values.
- **Fig. A-3b (Before/after focus)**: Insets at **[t]** and **[t+1]**.

> *Add 1–2 more cases in the same format.*

### 2.A.4 Summary for Group A

- **Stable vs. volatile phases.** Group A is **[stable/volatile]** in **[INSERT years]**, with **[INSERT count]** net inflows/outflows.
- **Defining attributes.** Over the full period, **[IND list]** contributes **[INSERT % range]** η²; the **most decisive pivot** occurs in **[INSERT year segment]**.
- **Mechanism.** Country-level LOO shows that moves **into** A are typically driven by **[IND…]**, while exits are associated with **[IND…]**.

------

## 2.B Group B — Temporal Analysis

### 2.B.1 Membership dynamics over time

**What we see.**
 Group B size spans **[INSERT min–max]** (mean **[INSERT]**). The sharpest change is **[INSERT year→year]** with **[INSERT +N/−N]** net **[gain/loss]**.

- **Key flows.**
  - **[INSERT country]**: **B→[A/C]** in **[INSERT year]**.
  - **[INSERT country]**: **[A/C]→B** in **[INSERT year]**.
  - **Long-tenure members**: **[INSERT]**.

**Evidence.**

- **Fig. B-1 (Membership flows)** and **Table B-1 (Transition counts)**.

### 2.B.2 Attribute-combination dynamics

**Headline pattern.**
 In **[INSERT year]**, Group B’s separation is dominated by **[IND1]**/**[IND2]** (η² **[INSERT %]**, **[INSERT %]**). After **[INSERT]**, the dominance shifts to **[IND3]** (**[INSERT %]**) with cumulative Top-k η² changing by **[INSERT %]**, passing our threshold.

- **Top-k sequence.**
  - **[year]**: **[list]** (cum η² **[INSERT %]**).
  - **[year]**: **[list]** (cum η² **[INSERT %]**).
  - **[year]**: …

**Evidence.**

- **Fig. B-2 (Yearly η² composition)** and **Table B-2 (Top-k by year)**.

### 2.B.3 Linking member changes to attribute changes (country-level mechanism)

**Case B-β — [INSERT country], [INSERT year t→t+1].**

- **Membership change.** **[B→A / A→B / B→C]**.
- **Group-level shift.** η² for **[INDX]** **[rises/falls]** **[INSERT %→%]**.
- **LOO mechanism.** Country LOO share for **[INDX]** **[INSERT %→%]**; **[INDY]** Signed Δη² **[INSERT]**.
- **Interpretation.** Indicates **[country]** became **[more/less]** consistent with B’s new attribute mix.

**Evidence.**

- **Fig. B-3a (LOO lines)** and **Fig. B-3b (before/after insets)**.

> *Add 1–2 more Group B cases here.*

### 2.B.4 Summary for Group B

- **Stability.** **[Stable/volatile]** periods: **[INSERT]**; net flows: **[INSERT]**.
- **Defining attributes.** Persistent drivers: **[IND…]** with η² **[range]**; notable shift in **[INSERT]**.
- **Mechanism.** Entrants typically align on **[IND…]**; exits are associated with **[IND…]**.

------

## 2.C (Optional but high-impact) Cross-group comparison (A vs B)

- **Different stability regimes.** Group A is **[more/less]** stable than B (**std. of size = [INSERT] vs [INSERT]**).
- **Contrasting attribute mixes.** A is dominated by **[IND…]**, whereas B relies more on **[IND…]** in **[INSERT year range]**.
- **Synchronized pivots or divergences.** In **[INSERT year]**, A and B **[converge/diverge]** as **[IND…]** rises in A but **[falls/rises]** in B.
- **Implication.** These contrasts explain **[INSERT short business-facing takeaway]**.

------

## Figures and Tables you should include

- **Fig. A-1/B-1**: Membership flows (Alluvial) with labeled inflow/outflow counts.
- **Fig. A-2/B-2**: Yearly η² composition (Top-k emphasis).
- **Fig. A-3/B-3**: LOO share lines for selected countries (hover shows Value, Group, Signed Δη²).
- **Table A-1/B-1**: Transition matrices (t→t+1).
- **Table A-2/B-2**: Top-k indicators per year with cumulative η² and replacements in **bold**.



