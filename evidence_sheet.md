# Pink AI evidence sheet | team Relax Team

**Members:** Yousef Shabib, Ahmad Irshaid, Sarah Alzaro

**1. What we cleaned and why (top 5)**
1. Numbers stored as text: 9900 decimal commas, 1389 thousands separators, 3256 missing tokens (?, --, unknown); 4 column names and a 2-line banner.
2. Scale slips: 610 cells x100 and 613 x1000 (factor chosen by the sister columns); 409 float-junk 1e-11 values set to 0.
3. 40 negative areas: abs() still disagreed with the radius, so they were rebuilt, not sign-flipped.
4. 2601 missing radius/perimeter/area cells rebuilt from geometry (P=6.479R, A=3.080R^2, learned on train). Missing size cells went from 2654 to 99; no rows dropped.
5. Labels in 67 raw spellings mapped to M/B. 4328 IDs normalised (ID-, commas, .0, tabs), which is what reveals the duplicates. 9 ages and 79 dates set missing.

**2. Problems that were not obvious (between columns)**
- **Perimeter and area swapped** in 158 rows (P > A is impossible): swapped back. **Radius in cm** in 1378 cells (P/R = 65, not 6.5): x10. Rows with P/R outside [5, 8]: 10.4% -> 0.00%.
- **346 patients entered twice, 50 with opposite diagnoses.** Merged; 48 labels repaired by the follow-up code (97.7% agreement on uncontested rows), the rest soft-labelled.
- **Training labels are noisy:** 0.044 / 0.057 of the most confident 20% carry the opposite label. **Site C** reads larger even in benign cases (radius x1.073).

**3. Columns we dropped, and why**
- **biopsy_followup_code = target leakage:** ONC-REF is 96.1% malignant and ROUTINE 2.1%; on its own it gives AUC 0.971. It is written after the diagnosis, so it does not exist at prediction time.
- notes (malignant rate 34%-43% in every template), sample_ref (random lab string), IDs and dates: no signal. center: a calibration risk.

**4. How we validated**
- StratifiedGroupKFold, 5 folds x 3 repeats, grouped by the *normalised* patient_id after merging duplicates. The code asserts that no patient lands in both train and validation. Repair parameters are learned on train only.
- Ensemble of logistic regression and gradient boosting: CV AUC 0.9329 +/- 0.0004, scored against noisy labels. Repairs raise the linear model from AUC 0.9038 to 0.9229 and recall@P90 from 0.5473 to 0.7613.

**5. Our threshold, and why**
- **t = 0.84.** It is the lowest t where precision stays >= 0.92 after removing label noise *and* halving prevalence to 20% (the brief warns that test is drawn differently), and where precision on the labels as given is >= 0.90 (0.933; bootstrap 5th percentile 0.924).
- Expected precision 0.921 under that stress, recall 0.641. The default t = 0.5 would give only 0.812.
- Cost: dropping t by 0.10 costs 0.14 false alarms per extra cancer at the training mix (0.34 at 20% prevalence) and breaks the stressed floor (0.887).

**6. What would break this model**
- Younger women (<40: CV AUC 0.888 vs 0.9329 overall), a new site or scanner with its own calibration, or an unseen unit convention.
- A population with far fewer than 20% cancers (screening): precision would drop below 0.90, so the threshold must be re-set.
- **Never** a stand-alone diagnosis and never a reason to skip biopsy. It is a triage aid on top of a pathologist.

**AI assistants:** Claude Code helped with exploratory checks, drafting the cleaning and validation code, and stress-testing the threshold logic. The team reviewed and can explain every decision.
