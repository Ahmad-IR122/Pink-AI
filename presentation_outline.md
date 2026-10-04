# Relax Team: Pink AI pitch deck (3 minutes, 9 slides)

Every number below is printed by `PinkAI_Solution.ipynb`.

---

## Slide 1: Title
**Can AI learn to spot malignancy?**
Breast-mass FNA classification. Pink AI 2026
Relax Team: Yousef Shabib · Ahmad Irshaid · Sarah Alzaro
*Speaker (10s):* "Our model is the least interesting part of our work. The data had a trap in almost every column."

## Slide 2: The data was lying
Four traps, each would quietly wreck a model:
1. A column that leaks the answer
2. Columns that contradict geometry
3. Patients entered twice, with opposite diagnoses
4. A test set drawn differently from train
*Speaker (15s):* set up the three stories that follow.

## Slide 3: Trap 1, the leak
`biopsy_followup_code`
- ONC-REF = **96.1%** malignant · ROUTINE = **2.1%**
- On its own: **AUC 0.971**
- Written *after* the diagnosis, so the model would learn the clerk, not the cells
**Dropped.**
*Speaker (25s)*

## Slide 4: Trap 2, a geometry lie detector
P ≈ 2πr · A ≈ πr². Constants learned from the data: **6.479** and **3.080**
- **158** rows: perimeter and area swapped
- **1378** cells: radius in cm instead of mm
- **40** negative areas, wrong even after abs()
- **2601** missing size cells rebuilt from geometry. **No rows deleted.**
- Rows with an impossible P/R ratio: **10.4% → 0.00%**
*Visual:* before/after histogram (`fig_geometry_before_after.png`)
*Speaker (35s)*

## Slide 5: Trap 3, duplicates and noisy labels
- **346** patients entered twice, **50** with opposite diagnoses
- Merged; **48** labels repaired with the follow-up code (**97.7%** agreement on uncontested rows)
- About **3–6%** of training labels look randomly flipped
*Speaker (25s)*

## Slide 6: Model and validation
- Logistic Regression + Gradient Boosting ensemble
- StratifiedGroupKFold, 5 × 3, **grouped by patient**: no patient appears in both train and validation
- CV AUC **0.9329** (against noisy labels)
- Cleaning alone: recall@P90 **0.5473 → 0.7613**
*Speaker (25s)*

## Slide 7: Under the hood (technical stack)
- 01 Clean: pandas + regex parsing; IDs, labels, centres and dates normalised
- 02 Repair: sister-column regressions (×100/×1000); geometry P = 6.479 R, A = 3.080 R²; duplicates merged; parameters learned on train only
- 03 Model: LogisticRegression (L2) + HistGradientBoosting; 30 log10 measurements + age; StratifiedGroupKFold 5 × 3 by patient
- 04 Decide: IsotonicRegression calibration, label-noise correction, 20% prevalence stress test, t = 0.84
- Stack: Python 3 · pandas · NumPy · scikit-learn · Matplotlib · Jupyter on DataCamp DataLab · Claude Code
*Speaker (20s)*

## Slide 8: The threshold
- **t = 0.84**, not 0.5
- Stress test: label noise removed + prevalence halved to **20%**
- Expected precision **0.921**, recall **0.641**
- Default t = 0.5 → precision **0.812**, which would fail the 0.90 floor
*Visual:* threshold curve (`fig_threshold.png`)
*Speaker (30s)*

## Slide 9: Limits and responsible use
- Weaker for women under 40 (CV AUC **0.888**)
- A new site or scanner needs re-calibration (Site C reads larger)
- **Never a stand-alone diagnosis.** It is a triage aid for pathologists.
**Thank you, questions?**
*Speaker (15s)*
