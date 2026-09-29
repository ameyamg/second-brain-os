# Gate Practice

The discipline in one sentence: before evaluating any vendor, label a few hundred real examples and train a basic classifier. It takes thirty minutes, and it is the baseline every claim must beat on your data before it earns a place in your pipeline.


> This audit-and-baseline routine is packaged as a skill: install [the course plugin](../../plugins/README.md) and run `/agents-course:gate-check` on your pipeline.

## Label two hundred real examples

Pull the last 200 items from the stream your agent actually processes — not synthetic examples, not someone's public dataset. Use three labels:

- `file` — pure reference: newsletters, receipts, digests. Goes straight to storage, no model needed.
- `act` — creates a concrete task: an invoice to pay, a form to sign, a meeting to confirm.
- `judge` — needs actual judgement: an ambiguous request, a proposal, anything with stakes.

Three rules while labelling. Label what the item is, not what you wish your categories were. When torn between two labels, choose `judge` — hesitation in you predicts hesitation in the gate. Do not invent new categories mid-run; note candidates and finish the pass. At five seconds an item this is about fifteen minutes. Save as `labels.csv`:

```csv
text,label
"Weekly digest: ten AI papers you missed",file
"Invoice 2214 attached, due Friday, please confirm receipt",act
"Fwd: any thoughts on this partnership proposal?",judge
```

## Train the baseline

Install once with `pip install scikit-learn pandas joblib`, then:

```python
# train_gate.py
import joblib
import pandas as pd
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import classification_report
from sklearn.model_selection import train_test_split
from sklearn.pipeline import Pipeline

df = pd.read_csv("labels.csv")

X_train, X_test, y_train, y_test = train_test_split(
    df["text"], df["label"], test_size=0.25,
    stratify=df["label"], random_state=42,
)

gate = Pipeline([
    ("tfidf", TfidfVectorizer(ngram_range=(1, 2), min_df=2, sublinear_tf=True)),
    ("clf", LogisticRegression(max_iter=1000, class_weight="balanced")),
])
gate.fit(X_train, y_train)

print(classification_report(y_test, gate.predict(X_test), digits=3))
joblib.dump(gate, "gate.joblib")
```

## Measure, then decide

Read the per-class precision and recall, not just the headline accuracy. These numbers are your baseline. Only now is the vendor question well-posed: does anything fancier beat this, on this data, by enough to justify a network call, a bill, and a dependency? On the independent evidence so far — see [gates in practice](gates-in-practice.md) — the answer for a fixed, labelled task is usually no.

## Wire the winner in

```python
# gate.py
import joblib

gate = joblib.load("gate.joblib")
THRESHOLD = 0.85
HANDLED = {"file", "act"}

def route(text: str) -> tuple[str, float]:
    probs = gate.predict_proba([text])[0]
    best = probs.argmax()
    label = gate.classes_[best]
    confidence = float(probs[best])
    if label not in HANDLED or confidence < THRESHOLD:
        return "escalate", confidence
    return label, confidence
```

Everything returning `escalate` goes to the main model — fail closed. The full routing scaffold, with logging and the LLM fall-through, is in [the confidence-gated router](../track-jev/build-router.md); this pipeline drops into its fast path unchanged.

## Tips

- Log every decision with its confidence, routed or not. The log is your future evaluation set.
- Retrain weekly. Escalations the main model resolved are free labels — append them to `labels.csv`.
- Keep `judge` forever. A gate with no escape hatch is a gate that guesses.
- Start the threshold at 0.85 and move it only after banding accuracy by confidence on held-out data.

## Prove it to yourself

1. Band your held-out predictions by confidence (0–0.5, 0.5–0.85, 0.85–1.0) and report accuracy per band. If the top band is not near-perfect, raise the threshold.
2. Rerun the report at thresholds 0.7, 0.85, and 0.95, and record escalation rate against fast-path error rate for each. Pick the trade-off deliberately.
3. Shadow-run for a week: the gate decides but the main model still handles everything. Count the disagreements before trusting the fast path.

With the gate sorting the stream, the expensive model finally has room to do its real work — the subject of the next module: [the office](../course-4-harness/the-office.md).

Go deeper → the [Jev engineering handbook](../track-jev/system-one-models.md): System One models, what they are good for, and a confidence-gated router build.
