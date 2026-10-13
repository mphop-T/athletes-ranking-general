# athletes-ranking-general

Code for *Data-Driven Athlete Ranking Using Learning-to-Rank and Regression Models*. The project ranks senior men's and women's 100 m sprint athletes within each event, using only information available before the event. It compares three learning-to-rank (LTR) models with a regression baseline:

| Model | Type | Library |
|---|---|---|
| LambdaRank | Pairwise-listwise hybrid | LightGBM (`LGBMRanker`) |
| ListNet | Listwise (probability-based) | allRank |
| ListMLE | Listwise (likelihood-based) | allRank |
| Random Forest Regression | Non-ranking baseline | scikit-learn |

Men and women are modelled separately. All models are trained on 2023 data and tested on 2024 data.

This repo covers the whole pipeline for LambdaRank and Random Forest, and the first and last stages for ListNet and ListMLE, which are spread over three repos:

| Stage | Step | Repo |
|---|---|---|
| 1 | Data preparation and LibSVM export | **athletes-ranking-general* (this repo) |
| 2 | Tuning and training of ListNet and ListMLE (Docker), producing a `model.pkl` and `used_config.json` per run | [listmle_listnet_train](https://github.com/mphop-T/listmle_listnet_train) |
| 3 | Scoring the 2024 test files with the selected checkpoints | [listnet-listmle-infer](https://github.com/mphop-T/listnet-listmle-infer) |
| 4 | Evaluation (NDCG, Spearman) and significance testing for all four models | **athletes-ranking-general* (this repo) |

## Pipeline

1. **Data preparation.** Event results are cleaned and turned into an event-grouped dataset with 30 pre-event features. History features are shifted chronologically to avoid look-ahead leakage. The target, `Relevance = Max_Rank − Normalized_Place + 1`, gives the best finisher in an event the highest value. `Position`, `Normalized_Place` and `Mark` are excluded from the features. The same features are exported in two formats: LibSVM (for allRank) and tabular (for LightGBM and scikit-learn).
2. **Splits.**

   | Partition | Period | Used for |
   |---|---|---|
   | Train | Jan–Sep 2023 | Fitting all models |
   | Validation | Oct–Dec 2023 | Tuning LambdaRank, ListNet and ListMLE |
   | Test | 2024 | Final scoring only (10,418 men's and 8,050 women's events) |

3. **Tuning** (all by random search, selected on NDCG@10, which scores the whole list because events have at most 10 athletes):
   - LambdaRank: 25 sampled configurations (`ParameterSampler`, seed 42), scored on the Oct–Dec 2023 validation partition.
   - Random Forest: 25 sampled configurations (`RandomizedSearchCV`), scored by 4-fold `GroupKFold` grouped by `Event_ID` within the 2023 data.
   - ListNet and ListMLE: 30 sampled configurations per model and gender with `tune_allrank.py` in [listmle_listnet_train](https://github.com/mphop-T/listmle_listnet_train/). Each is trained for up to 50 epochs with early stopping, and the best final validation NDCG@10 is kept.
4. **Scoring.** Each model's scores are grouped by `Event_ID` and turned into a ranking within the event. ListNet and ListMLE scores come from `listnet-listmle-infer`.
5. **Evaluation.** Mean NDCG@1–4, mean per-event Spearman correlation, and paired t-tests on per-event full-list NDCG for all six model pairs, by gender.

## Repository contents

**Notebooks**

| Notebook | Purpose |
|---|---|
| `men_women_senior_athletes_2023_2024.ipynb` | Data cleaning, weather and venue merging, feature engineering, target construction, train/validation/test split, LibSVM export |
| `LambdaRankGendersCombined.ipynb` | LambdaRank training, tuning, scoring and evaluation, men and women |
| `RandomForestGenderCombined.ipynb` | Random Forest tuning, scoring and evaluation, men and women |
| `allRank_models_combined.ipynb` | Evaluation of ListNet and ListMLE scores produced by `listnet-listmle-infer` |
| `all_models_ttest.ipynb` | Paired t-tests on per-event NDCG across all four models |

The cells in `LambdaRankGendersCombined.ipynb` that use `LambdaRank_pred` (without `_tuned`) belong to an earlier untuned men's-only run. The reported results use the `LambdaRank_pred_tuned` columns.

**Data and intermediate files**

| File | Contents |
|---|---|
| `WA_2023_2024_100m_Sprints.xlsx` | Source 100 m results for 2023 and 2024 |
| `unique_venues_coordinates.xlsx` | Venue coordinates used to attach weather |
| `merged_df_by_Key_combined.xlsx`, `merged_df_modified_Weather_by_Key_combined.xlsx` | Merged results, and the same with weather added |
| `X_train_merged_df_men.xlsx`, `X_train_merged_df_women.xlsx` | Training feature tables |
| `X_test_merged_df_men.xlsx`, `X_test_merged_df_women.xlsx` | 2024 test feature tables |
| `train_men.txt`, `train_women.txt`, `vali_men.txt`, `vali_women.txt` | LibSVM training and validation files for allRank |
| `X_test_merged_df_{men,women}_allRank.xlsx` | Test tables with ListNet and ListMLE scores attached |
| `X_test_merged_df_{men,women}_copy_LambdaRank.xlsx` | Test tables with LambdaRank scores attached |
| `X_test_merged_df_{men,women}_copy_Random_FR_results.xlsx` | Test tables with Random Forest scores attached |
| `allrank_{listmle,listnet}_{male,women}_combined_tensor_output.txt` | Raw ListNet and ListMLE scores for the test file, one event per line |

## Results (2024 test partition)

| Model | Gender | NDCG@1 | NDCG@2 | NDCG@3 | NDCG@4 | Spearman |
|---|---|---|---|---|---|---|
| Random Forest | Men | 0.87 | 0.88 | 0.88 | 0.89 | 0.57 |
| Random Forest | Women | 0.89 | 0.89 | 0.89 | 0.90 | 0.63 |
| LambdaRank | Men | 0.81 | 0.85 | 0.85 | 0.86 | 0.46 |
| LambdaRank | Women | 0.81 | 0.85 | 0.86 | 0.87 | 0.48 |
| ListMLE | Men | 0.68 | 0.74 | 0.75 | 0.77 | 0.13 |
| ListMLE | Women | 0.67 | 0.73 | 0.75 | 0.76 | 0.11 |
| ListNet | Men | 0.65 | 0.72 | 0.73 | 0.75 | 0.07 |
| ListNet | Women | 0.63 | 0.71 | 0.72 | 0.74 | 0.02 |

The model order is the same for both genders: Random Forest, LambdaRank, ListMLE, ListNet. All six pairwise per-event NDCG t-tests are significant at p < .001 for both genders. The ListMLE-versus-ListNet difference is small in size. Because the four models differ in model family as well as in loss function, and each was tuned separately, the results do not show that a ranking objective is inherently weaker than regression.

## Reproducing the results

1. Run the data-preparation notebook to produce the tabular files and the LibSVM files (`train.txt` and `vali.txt` per gender folder, plus the 2024 test file).
2. Run `LambdaRankGendersCombined.ipynb` and `RandomForestGenderCombined.ipynb`.
3. Train and tune ListNet and ListMLE in `listmle_listnet_train`, then generate test scores with `listnet-listmle-infer` using the selected `model.pkl` and `used_config.json` for each model and gender. The resulting scores are the `*_tensor_output.txt` files in this repo.
4. Run `allRank_models_combined.ipynb`, then `all_models_ttest.ipynb`.

## Requirements

- Python 3.13.5 with `pandas`, `numpy`, `scikit-learn`, `lightgbm` and `scipy`
- Docker (for `listmle_listnet_train` only)

The notebooks in this repo run in a Python 3.13.5 environment. [listnet-listmle-infer](https://github.com/mphop-T/listnet-listmle-infer) uses a separate Python 3.9.23 environment (and kernel), because it depends on the older PyTorch version that allRank requires. Use a separate environment for each repo and do not run the notebooks of one in the kernel of the other.

## Notes and limitations

- One discipline and a single 2023/2024 split. Events share athletes, so events are not fully independent.
- The models were tuned with different procedures and search sizes (25 to 30 configurations), so gaps between them can reflect the tuning outcome as well as the model.
- allRank computes NDCG with exponential gain, whereas the reported NDCG values use scikit-learn's `ndcg_score` (linear gain). The two are not directly comparable.

