# Source-label confounding code: fixes

Research code for a paper on source-label confounding in AI-phishing detection benchmarks. Scripts were run in stages on Colab and Kaggle; this covers the statistics, baselines and training scripts.

## Logic & data

### Confidence intervals treat near-duplicate emails as independent (High)

**Why it's a problem.** The stage-3 bootstrap resampled single emails. The project's own stage-1 analysis found clusters of near-duplicate emails, and its own helper could resample whole groups, but the main comparison did not use it. Emails in one cluster are not independent, so the intervals were too narrow and the evidence looked stronger than it is.

**AI-written code**

```python
def boot_diff(x0, x1, B=2000, seed=42):
    x0 = np.asarray(x0, float); x1 = np.asarray(x1, float); n = len(x0)
    rng = np.random.default_rng(seed); d = x1 - x0; out = np.empty(B)
    for s in range(0, B, 250):
        idx = rng.integers(0, n, size=(min(250, B - s), n))     # resamples single emails
        out[s:s + idx.shape[0]] = d[idx].mean(1)
    return d.mean(), np.percentile(out, 2.5), np.percentile(out, 97.5)
```

**Fixed code**

```diff
+# Near-duplicate groups found in stage 1. Emails without a group count as their own group.
+def group_of(email_ids): ...   # reads the stage-1 group file

-def boot_diff(x0, x1, B=2000, seed=42):
+def boot_diff(x0, x1, B=2000, seed=42, groups=None):
+    """with `groups`, whole groups are resampled (cluster bootstrap)"""
     ...
-    for s in range(0, B, 250):
-        idx = rng.integers(0, n, size=(min(250, B - s), n)); out[s:s + idx.shape[0]] = d[idx].mean(1)
+    if groups is None:
+        for s in range(0, B, 250): ...                 # unchanged single-email path
+    else:
+        codes, inv = np.unique(np.asarray(groups), return_inverse=True); G = len(codes)
+        gsum = np.bincount(inv, weights=d, minlength=G); gcnt = np.bincount(inv, minlength=G).astype(float)
+        for i in range(B):
+            pick = rng.integers(0, G, G); out[i] = gsum[pick].sum() / gcnt[pick].sum()
```

```diff
-    d, lo, hi = boot_diff(x0, x1)
+    grp = group_of(m.email_id.astype(str).values)
+    d, lo, hi = boot_diff(x0, x1, groups=grp)
-    return dict(comparison=label, fold=fold, set=what, n=len(m), rate_B0=...
+    return dict(comparison=label, fold=fold, set=what, n=len(m), n_groups=len(set(grp)), rate_B0=...
```

**Status.** Partially fixed. The confidence intervals now resample whole groups and the result tables report `n_groups`. **What remains:** the exact McNemar p-values (`mcnemar_exact`) still treat every email as an independent pair, so they are still too small; they need a cluster-aware replacement (for example a group-level permutation test). Emails that have no stage-1 group, such as the held-out and generated sets, are treated as their own group.

**Lesson.** If you know your data has clusters, resample the clusters, and apply the same care to every test, not only the intervals.

### A paper baseline was rebuilt with invented features (High)

**Why it's a problem.** The definition of the paper's "8-feature formatting model" could not be found, so the script used eight features of its own choosing (length, newlines, link count and so on) and logged "UNSURE whether identical to the paper". The result was written under the paper's baseline name, so a reader could take it for a reproduction of the published baseline.

**AI-written code**

```python
F = {'length_only': ...,
     'subject_present_only': subj_flag,
     'formatting_8_features': fmt_feats}     # own features, but named like the paper's model
```

**Fixed code**

```diff
-    F={'length_only':...,'subject_present_only':subj_flag,'formatting_8_features':fmt_feats}
+    # fmt_feats uses features chosen here, not the paper's 8-feature model (its definition was not
+    # found). Name it so in every table, never as if it were the paper's baseline.
+    fmt_name='own_8_formatting_features_NOT_paper_model'
+    F={'length_only':...,'subject_present_only':subj_flag,fmt_name:fmt_feats}
```

**Status.** Fixed (untested)

**Lesson.** A substitute for something you could not find must carry its own name.

### One "score" column holds two different kinds of numbers (Moderate)

**Why it's a problem.** For logistic regression the `score` column is a probability between 0 and 1 (cut at 0.5). For the linear SVM it is the raw decision value (any real number, cut at 0). Both went into the same prediction files, so any threshold, calibration or pooled curve computed on `score` across models mixes two scales.

**AI-written code**

```python
sc = m.predict_proba(Xte)[:, 1] if hasattr(m, 'predict_proba') else m.decision_function(Xte)
rows.append(pd.DataFrame(dict(..., y_pred=m.predict(Xte), score=sc, model=name, ...)))
```

**Fixed code**

```diff
                     sc = m.predict_proba(Xte)[:, 1] if hasattr(m, 'predict_proba') else m.decision_function(Xte)
+                    # probability (cut 0.5) or raw margin (cut 0): score_type says which
+                    stype = 'probability' if hasattr(m, 'predict_proba') else 'decision_function'
-                    rows.append(pd.DataFrame(dict(..., y_pred=m.predict(Xte), score=sc,
+                    rows.append(pd.DataFrame(dict(..., y_pred=m.predict(Xte), score=sc, score_type=stype,
```

**Status.** Fixed (untested). Existing result files written before this change have no `score_type` column.

**Lesson.** A column that can hold two scales needs a second column that says which one it is.

## Reliability & quality

### Failed pipeline steps are swallowed and the run carries on (High)

**Why it's a problem.** The `@step` wrapper caught every exception, logged "FAIL" and returned `None`, and the next steps and the final summary still ran. Anything a failed step had written before it crashed stayed on disk, so the summary could be built from partial or stale tables without stopping. Two such failed first runs were recorded in the project's own notes.

**AI-written code**

```python
try: r = fn(*a, **k)
except Exception as e:
    log('FAIL', name, traceback.format_exc()); progress(name, 'FAIL ' + repr(e)[:300]); return None
```

**Fixed code**

```diff
             except Exception as e:
-                log('FAIL',name,traceback.format_exc()); progress(name,'FAIL '+repr(e)[:300]); return None
+                # log, then stop the run: continuing would let later steps read missing or half-written
+                # outputs. No DONE marker is written, so a rerun retries this step.
+                log('FAIL',name,traceback.format_exc()); progress(name,'FAIL '+repr(e)[:300]); raise
```

**Status.** Fixed (untested)

**Lesson.** Log a failure, then stop; a pipeline that continues after a failed step hides it.

### Dependencies are not pinned (Moderate)

**Why it's a problem.** `requirements.txt` listed only minimum versions (`transformers>=5.16`, `torch>=2.5`, ...). The project's own notes record that a library version difference already changed re-scoring results. The exact versions were available in the freeze file but not used.

**AI-written code**

```text
numpy>=2.1
scikit-learn>=1.6
torch>=2.5
transformers>=5.16
accelerate
```

**Fixed code**

```diff
-numpy>=2.1
-pandas>=2.2
-scipy>=1.13
-scikit-learn>=1.6
-torch>=2.5
-transformers>=5.16
-accelerate
-sentencepiece
-matplotlib
-tqdm
+numpy==2.1.3
+pandas==2.2.3
+scipy==1.16.3
+scikit-learn==1.6.1
+torch==2.11.0
+transformers==5.20.1
+tokenizers==0.23.3
+accelerate==1.15.0
+sentencepiece==0.2.2
+matplotlib==3.10.0
+tqdm==4.67.3
```

**Status.** Fixed (untested)

**Lesson.** Reproducible research needs exact versions, taken from the environment that produced the numbers.

### Hard-coded Colab and Kaggle paths, and a module that fails on import (Moderate)

**Why it's a problem.** Paths such as `/content/drive/MyDrive/...` were written into the module, and importing it read a `START_TIME.txt` file. On any other machine, or the first time the log folder was new, the import itself crashed before a single step ran.

**AI-written code**

```python
MD = '/content/drive/MyDrive'; R = MD + '/_revision'
V3 = MD + '/YOUR_STUDY_FOLDER'
...
START = datetime.datetime.fromisoformat(open(LG + '/START_TIME.txt').read().strip())   # runs on import
```

**Fixed code**

```diff
-MD='/content/drive/MyDrive'; R=MD+'/_revision'
-V3=MD+'/YOUR_STUDY_FOLDER'
+# Paths default to the original Colab layout; set these environment variables to run anywhere else.
+MD=os.environ.get('SLC_DRIVE','/content/drive/MyDrive'); R=os.environ.get('SLC_REVISION',MD+'/_revision')
+V3=os.environ.get('SLC_V3',MD+'/YOUR_STUDY_FOLDER')
```

```diff
-START=datetime.datetime.fromisoformat(open(LG+'/START_TIME.txt').read().strip())
-def elapsed(): return (datetime.datetime.now()-START).total_seconds()/60
+_START=None
+def elapsed():
+    # read lazily (not on import): importing this module must not crash when the log folder is new
+    global _START
+    if _START is None:
+        p=LG+'/START_TIME.txt'
+        if not os.path.exists(p):
+            os.makedirs(LG,exist_ok=True); open(p,'w').write(datetime.datetime.now().isoformat())
+        _START=datetime.datetime.fromisoformat(open(p).read().strip())
+    return (datetime.datetime.now()-_START).total_seconds()/60
```

**Status.** Partially fixed. **What remains:** only the stage-1 `common.py` was changed. The path modules of the other stages (their own `common.py` files, `rg_core.py`, `r2common.py`) still hard-code the Colab and Kaggle paths and need the same environment-variable treatment.

**Lesson.** Make paths configurable, and keep module imports free of side effects.

### A model-reload check is computed but never enforced (Moderate)

**Why it's a problem.** After saving each checkpoint the script reloaded it and compared predictions, but a mismatch was only written into a `reload_verified` column. Training went on and the unverified checkpoint was used for results.

**AI-written code**

```python
m2 = AutoModelForSequenceClassification.from_pretrained(out).to(dev)
ok = bool(np.allclose(predict(model, te[:32], tcond), predict(m2, te[:32], tcond), atol=1e-2)); del m2
...
reload_verified=ok     # recorded, never acted on
```

**Fixed code**

```diff
         ok = bool(np.allclose(predict(model, te[:32], tcond), predict(m2, te[:32], tcond), atol=1e-2)); del m2
+        if not ok:  # a saved checkpoint that does not reproduce the model must not be used for results
+            raise RuntimeError('%s: reloaded checkpoint gives different predictions than the trained model' % tag)
```

**Status.** Fixed (untested)

**Lesson.** A check that cannot stop the run is only a log line.
