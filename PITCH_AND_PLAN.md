# Pink AI: work plan, pitch and judge Q&A

## 1. خطة العمل (3 ساعات، فريق من 4)

| الوقت | العضو 1 (Data Cleaning) | العضو 2 (Structural / Leakage) | العضو 3 (Modeling) | العضو 4 (Evidence + Pitch) |
|---|---|---|---|---|
| 0:00–0:20 | تحميل البيانات، توحيد أسماء الأعمدة، إرسال `sample_submission` كتجربة | جدول التسريب: `biopsy_followup_code` × diagnosis | تجهيز CV مُجمّع حسب المريض | قراءة الـ rubric وتجهيز قالب الـ evidence sheet |
| 0:20–1:10 | parsing الأرقام (فواصل عشرية، آلاف، `?`/`--`) + إصلاح ×100/×1000 | المشاكل البنيوية: تبديل perimeter↔area، الـ radius بالسنتيمتر، القيم السالبة، الـ duplicates المتعارضة | baseline: LR + HGB | تتبّع كل رقم يُطبع في الـ notebook |
| 1:10–2:00 | مراجعة عدد الخلايا المُصلحة لكل نوع | تحليل Site C وضوضاء الـ labels | الـ ensemble + اختيار الـ threshold (stress test) | كتابة الـ pitch والإجابات المتوقعة |
| 2:00–2:40 | تشغيل الـ notebook من البداية للنهاية | مراجعة أن الـ test لم يُستخدم لأي fit | توليد ملف الـ submission والتحقق منه | تجربة الـ pitch بالتوقيت (3 دقائق) |
| 2:40–3:00 | تسليم | تسليم | تسليم | تسليم |

**الأدوات:** pandas, numpy, scikit-learn (`HistGradientBoostingClassifier`, `LogisticRegression`, `StratifiedGroupKFold`, `IsotonicRegression`), matplotlib. لا يوجد deep learning (لا يعطي نقاطًا إضافية حسب القوانين)، ولا xgboost/lightgbm حتى يعمل الـ notebook على أي بيئة.

**الخوارزمية:** متوسط Logistic Regression (ثابت عند تغيّر توزيع البيانات) و Gradient Boosting (يلتقط العلاقات غير الخطية). الميزات هي الـ 30 قياسًا بعد الإصلاح على log scale + العمر.

---

## 2. Pitch (English, ~3 minutes)

**[0:00, the hook]**
"Our model is the least interesting part of our work. The data had a trap in almost every column, and three of them would have quietly wrecked any model trained on it."

**[0:20, the leak]**
"First trap: `biopsy_followup_code`. ONC-REF is 96% malignant, and on its own it scores AUC 0.971. It's an admin code written *after* the diagnosis, so a model using it learns the clerk, not the cells. We dropped it."

**[0:45, structural problems]**
"Second, the columns contradicted each other. A nucleus obeys geometry: perimeter is about 2πr and area is about πr². We learned those constants from the data, 6.48 and 3.08, and used them as a lie detector.
- In 158 rows perimeter and area were swapped.
- In 1,378 cells the radius was in centimetres.
- 40 'negative areas' were not sign errors: even their absolute value disagreed with the radius.

We repaired all of these jointly and rebuilt 2,601 missing size cells from geometry. We deleted no rows. Out-of-range P/R ratios went from 10.4% to zero."

**[1:25, duplicates and labels]**
"Third, 346 patients were entered twice, and 50 of those pairs had opposite diagnoses. We merged them and adjudicated the label with the follow-up code, which agrees with the diagnosis 97.7% of the time. That's the code's only legitimate use. We also found the training labels are noisy: even where our model is most certain, about 3–5% of labels say the opposite."

**[1:55, validation]**
"Validation is grouped by normalised patient ID, so the same patient is never in both train and validation, and the code asserts it. Every cleaning parameter is learned on train only. CV AUC is 0.933 against noisy labels. Our repairs alone lift the linear model's recall at 90% precision from 0.55 to 0.76."

**[2:15, under the hood]**
"Under the hood it's all plain Python. pandas and regex do the parsing. Sister-column regressions and geometry do the repairs. scikit-learn provides logistic regression plus HistGradientBoosting, and isotonic calibration with a 20% prevalence stress test sets the threshold. No deep learning, no external labels, and the test set is never used for fitting."

**[2:30, threshold]**
"The threshold is where we were most careful. The brief says the test is drawn differently, and the shift that kills precision is fewer cancers. So we stress-tested: we removed label noise and halved prevalence to 20%. We picked t = 0.84, the lowest threshold that still keeps precision above 0.92 under that stress. The default 0.5 would have collapsed to 0.81."

**[2:45, limits]**
"It's weaker for women under 40, and it would need re-calibration for a new site, as we saw with Site C. It's a triage aid. It must never replace a pathologist or be a reason to skip a biopsy."

---

## 3. Likely judge questions and short answers

**Q: Why did you drop `biopsy_followup_code` when it's so predictive?**
It's predictive *because* it's written after the diagnosis (ONC-REF = oncology referral). At prediction time it either doesn't exist or already contains the answer. Its single-feature AUC of 0.971 is the red flag, not a feature.

**Q: Isn't using the code to fix the conflicting labels also leakage?**
No. Leakage means using it as an input for new patients. We use it only to decide which of two contradictory *training* labels to trust, and only for 48 patients. It never reaches the model or the test set.

**Q: Why 0.84 and not 0.5?**
At 0.5, precision falls to 0.81 once prevalence drops to 20%. That would be zero points and, clinically, too many false alarms. 0.84 is the lowest threshold that keeps precision ≥ 0.92 under that stress *and* ≥ 0.90 on the labels as given. Lowering it by 0.10 buys more cancers at 0.14 false alarms each, but breaks the 20% stress floor (0.887).

**Q: Did you use the test set to choose the threshold?**
No. The threshold comes from out-of-fold training predictions. The 20% stress prevalence is a design choice: half the training prevalence, because the brief warned of a shift. The test set is only transformed with train-learned parameters and then scored.

**Q: How do you know the labels are noisy?**
Two independent signs. First, 50 duplicate pairs with identical measurements but opposite diagnoses. Second, among the 5% of patients the model is most certain about, 2.8–3.5% are labelled the other way, and that rate levels off instead of going to zero. We took the *smallest* estimate, which gives the strictest threshold.

**Q: Why not just delete messy rows?**
Only 1,778 of 9,000 rows have all 30 measurements, so dropping incomplete rows would throw away 80% of the data. Geometry lets us repair instead: of 2,654 missing size cells, only 99 remain.

**Q: What about Site C?**
Its size measurements run about 7% larger in *both* benign and malignant cases, so it's calibration, not case mix. We tested harmonising it and adding a site flag; neither improved CV, so the model stays site-agnostic. A new site would need the same check.

---

## 4. Notes for the team (read before submitting)

1. **Team name:** already set to `Relax Team` in the second cell. The submission file is `RelaxTeam_submission_1.csv`, without the space so the file name stays safe.
2. **Run top to bottom:** the notebook runs in about one minute and needs only pandas / numpy / scikit-learn / matplotlib. The CSV files must be in the same folder.
3. **Transparency:** while exploring, we looked at the raw test file for format only (column names, ID formats, duplicates), plus one descriptive check: in test, `biopsy_followup_code` has no relationship with the measurements. None of this entered the notebook or the threshold choice.
4. **Additional submissions:** we have 3. Don't adjust the threshold based on leaderboard results, because that counts as using the test to pick the threshold. If you want a second submission, decide its variant *before* seeing the first score and justify it from train alone.
