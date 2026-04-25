# AdaptiMed: Guideline-Aware Adaptive Agentic Workflow for Medical Diagnosis

> A faithful reimplementation of MedAgent-Pro (ICLR 2026) extended with three contributions: guideline-grounded tool selection, calibrated iterative depth, and parallel tier orchestration.

---

## What this is

Medical agentic systems have a structural problem: they run every tool on every case, regardless of how obvious or complex the presentation is. A clear-cut glaucoma case with IOP = 28 mmHg and CDR = 0.90 goes through the exact same pipeline as an ambiguous normal-tension case where IOP is normal but structural damage is present. That is wasteful, and in a real clinical setting, unnecessary.

This repo contains two things:

**Run 1** is a faithful reimplementation of MedAgent-Pro, evaluated on n=50 REFUGE2 fundus cases. It exists as an honest baseline, reproducing the original RAG-Planner-CodingAgent-ProDecider pipeline under identical LLM and tool conditions.

**Run 2 (AdaptiMed)** adds three contributions on top of that baseline.

There is also a research proposal for **CAGED** (Calibrated Agentic Guideline-grounded Early-exit Diagnostics), the principled Bayesian successor to AdaptiMed, which will be added to this repo once implemented.

---

## The three contributions

**1. EvidenceTierPrioritiser**
Structures EGS 2020 glaucoma diagnostic criteria into three priority tiers before the planner generates any plan. Tier 1 covers primary quantitative tools (CDR, IOP, VF-MD, RNFL, ONH), Tier 2 covers demographic risk profile, and Tier 3 reserves the expensive VLM analysis for genuinely ambiguous cases only. Each tool selection is now traceable to a named guideline criterion rather than an implicit LLM decision.

**2. CalibratedQuickDecider**
After each tier, the system queries accumulated evidence k=3 times at varied temperatures and computes mean confidence and coefficient of variation. The exit policy is:

```
exit after tier k  <=>  mean_conf >= theta_k  AND  CV <= delta
                        theta_1 = 0.72, theta_2 = 0.90, delta = 0.30
```

The CV gate is the key part. It blocks early exit when confidence estimates are noisy, ensuring expansion reflects genuine evidence accumulation rather than LLM temperature variance.

**3. Parallel Tier Orchestration**
Tools without data dependencies within a tier execute simultaneously via ThreadPoolExecutor. Tools with sequential dependencies (ONH feeds Tier-3 VLM analysis) run in order. A threading.Lock protects concurrent writes to `diagnosis.json`.

---

## Results

Evaluated on n=50 REFUGE2 fundus cases. CDR was computed from real optic disc/cup segmentation masks for 23 of 50 cases; IOP, VF-MD, and RNFL were supplemented from EGS-2020-calibrated clinical distributions throughout. These scope constraints are disclosed. Results validate the agentic orchestration and iterative-depth logic and are not benchmark-comparable to end-to-end image pipelines on labelled REFUGE2 test sets.

| Metric | MedAgent-Pro | AdaptiMed | Change |
|---|---|---|---|
| Category accuracy | 28.0% | 52.0% | +24 pp |
| Adjacent-category accuracy | 92.0% | 96.0% | +4 pp |
| Broad accuracy | 54.0% | 80.0% | +26 pp |
| Avg tool calls / case | 7.0 | 5.2 | -25.7% |
| Cases with early exit | 0 / 50 | 46 / 50 | 92% |
| Cases requiring full depth | 50 / 50 | 4 / 50 | 8% |

**Four-configuration ablation:**

| Config | Cat. Acc. | Broad Acc. | Avg Tools | Early Exit |
|---|---|---|---|---|
| A: MedAgent-Pro baseline | 28.0% | 54.0% | 7.0 | 0% |
| B: +EvidenceTierPrioritiser | 28.0% | 54.0% | 7.0 | 0% |
| C: +IterativeDepth (single-vote) | 52.0% | 80.0% | 5.2 | 92% |
| D: Full AdaptiMed | 52.0% | 80.0% | 5.2 | 92% |

Config B being iso-accurate with baseline is expected and honest. The EvidenceTierPrioritiser is a static interpretability artifact; its contribution is auditability, not accuracy. The accuracy gain comes from the iterative depth mechanism.

---

## Repo contents

```
AdaptiMed/
├── ADEPTIMED_and_MEDAGENTPRO_REIMPLEMENTATION.ipynb   # Full implementation: Run 1 + Run 2 + ablation
└── Proposal.pdf                                        # CAGED research proposal (Bayesian successor)
```

CAGED implementation will be added here once complete.

---

## How to run

**Requirements:** Kaggle or any environment with a T4 GPU. All experiments were run on Kaggle free tier + Groq free API.

```bash
pip install langchain langchain-text-splitters langchain-community \
    groq faiss-cpu sentence-transformers \
    beautifulsoup4 scikit-learn opencv-python-headless openpyxl tqdm
```

Set your Groq API key, then run the notebook top to bottom. Run 1 and Run 2 are clearly marked. The ablation follows in the final section.

**LLM used:** `llama-3.1-8b-instant` via Groq API.

---

## What comes next (CAGED)

AdaptiMed works but its confidence mechanism is brittle. The CV gate thresholds (theta=0.72, delta=0.30) were hand-tuned for glaucoma under EGS 2020. They do not transfer to heart failure staging under ACC/AHA or rare disease without manual recalibration.

CAGED replaces the heuristic gate with a Dirichlet posterior updated incrementally per evidence tier. The exit criterion becomes:

```
H(P(Y | E_1:k)) <= tau_k^G  AND  E[delta_KL_{k+1}] <= epsilon
```

Where `tau_k^G` is retrieved dynamically from the relevant clinical guideline at inference time (EGS 2020, ACC/AHA, WHO ICD-11), not fixed during training. The same trained system transfers across glaucoma, cardiology, and rare disease without retraining.

The proposal is in `Proposal.pdf`. Implementation is in progress.

The main goal is to find out **when has the system seen enough to commit?**

---

## Contact

Ammar Bhilwarawala · [ammarbhilwarawala@gmail.com](mailto:ammarbhilwarawala@gmail.com) · [Hugging Face](https://huggingface.co/Ammar-ss) · [LinkedIn](https://linkedin.com/in/ammar-bww)
