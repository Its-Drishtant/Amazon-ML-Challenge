# Production candidate: matcher_v3.joblib  (PROVISIONAL)

v3 = v2 + ALIAS blocking.  The ONLY change from v2 is `w_alias_name_tok=0.9` in
candidate generation, which adds alias-canonicalized name tokens to the retrieval
index.  Model hyperparameters, seed, feature definitions (57), validation split and
evaluation code are byte-identical to v2.

## Independently verified (src/verify_model.py, metric recomputed from the formula)

                        v1            v2            v3
    macro F0.5      0.951895      0.954418      0.956248   (tau 0.70)
                                                0.956132   (tau 0.65)
    macro precision 0.977404      0.976751      0.979818
    macro recall    0.897231      0.907223      0.905135
    cand recall     0.957744      0.957744      0.961853
    full coverage   0.880906      0.880906      0.889887
    singleton acc   0.955449      0.950331      0.960867
    features              47            57            57
    threshold           0.70          0.65          0.70

## Paired comparisons, 60,000 identical validation entities

    v1 -> v2   +0.002523  t=+8.99  CI [+0.001982, +0.003080]
    v2 -> v3   +0.001714  t=+6.89  CI [+0.001235, +0.002212]   (tau fixed at 0.65)
    v2 -> v3   +0.001830  t=+7.30  CI [+0.001337, +0.002332]   (each at own best tau)

Both v2->v3 confidence intervals exclude zero.

## v3 configuration

Blocking (src/gen_candidates.py BEST_CFG + one explicit flag):
    w_name_tok=1.0 w_addr_tok=1.1 w_char_gram=0.35 char_n=4 cand_norm=sqrt
    df_cap=20000 df_cap_frac=0.012 df_cap_gram=3000 df_cap_gram_frac=0.0
    topk=20   --w-alias-name-tok 0.9      <-- the only difference from v2

Model: HistGradientBoostingClassifier, max_iter=400, learning_rate=0.10,
    max_leaf_nodes=63, min_samples_leaf=100, l2_regularization=1.0,
    early_stopping=True, validation_fraction=0.08, n_iter_no_change=25,
    random_state=20260925
Features: 57, use_aliases=True, addr_nan=True
Decision: threshold, tau=0.70 (stored).  tau=0.65 scores 0.956132, a
    noise-level 0.000116 lower; 0.70 is kept because it is also the
    higher-precision, better-singleton operating point.

## Reproduce

    python3 -m src.gen_candidates --split train --queries train --sample 250000 \
        --topk 20 --w-alias-name-tok 0.9 --tag trainsub_k20_alias
    python3 -m src.gen_candidates --split train --queries val \
        --topk 20 --w-alias-name-tok 0.9 --tag val_k20_alias
    python3 -m src.train --train-cand artifacts/cache/cand_train_trainsub_k20_alias.parquet \
        --val-cand artifacts/cache/cand_train_val_k20_alias.parquet --exp v3 \
        --chunk 1000000 --max-iter 400 --lr 0.10 --max-leaf 63 --min-leaf 100 --l2 1.0

## Fallbacks
    models/fallback_v2_LOCKED_matcher_v2.joblib  (read-only copy of v2)
    models/baseline_v1_LOCKED/matcher_v1.joblib (read-only, see PROVENANCE.md)

## Validation-hygiene note (not perfectly hermetic)
src/discover_tokens.py derives the 75-token legal/noise list from ground-truth pairs
sampled across ALL Source-1 entities, including validation ones, and that list feeds
name_core -> blocking, features and IDF.  Measured effect: restricting discovery to
training entities changes the list by exactly ONE token (`kanstrakshansa`), and that
token appears in the TRAIN-ONLY list, i.e. the difference is sampling noise from a
smaller edge pool rather than validation information leaking in.  Using all training
ground truth is legitimate for test inference (it is provided training data, not
external data); the caveat is only that the validation figures above are not
mathematically perfectly hermetic.  src/discover_aliases.py IS restricted to
training entities.
