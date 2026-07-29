---
title: "Train & package the model"
date: 2026-07-13
weight: 3
chapter: false
pre: " <b> 5.3. </b> "
---

This section walks through the full pipeline from raw data to a deployable inference Docker image: dataset preparation → training → evaluation → ONNX INT8 export → Lambda container packaging → local testing → pushing to ECR.

#### 1. Dataset and preprocessing

The team uses **ViHSD** (Vietnamese Hate Speech Detection, ~33,000 comments) with 3 labels: `0 = CLEAN`, `1 = OFFENSIVE`, `2 = HATE`, using the original train/val/test split.

The dataset's most important property is **severe label imbalance** — roughly 82% CLEAN:

![Label distribution](/images/5-Workshop/5.3/label-distribution.png)

Preprocessing: **NFC** unicode normalization, URLs replaced with a `<URL>` token, mentions with `<USER>`, whitespace collapapsed. One counter-intuitive but important decision: **teencode and profanity are kept as-is** ("vcl", "đm"...) — they are exactly the classification signal; filtering them out destroys information.

```python
def clean_text(t):
    t = unicodedata.normalize("NFC", str(t))
    t = URL_RE.sub(" <URL> ", t)        # URL -> token
    t = MENTION_RE.sub(" <USER> ", t)   # mention -> token
    return re.sub(r"\s+", " ", t).strip()
```

#### 2. Training on Google Colab (T4 GPU)

The `train_text_classifier.ipynb` notebook runs end-to-end on a free Colab T4 GPU (~45–90 minutes), seed 42 for reproducibility:

![Colab GPU](/images/5-Workshop/5.3/colab-gpu.png)

**Baseline** first: TF-IDF (1–2 grams) + Logistic Regression with `class_weight="balanced"` — a cheap benchmark that the main model must beat.

**Main model**: fine-tuned **XLM-RoBERTa-base**. The team originally proposed PhoBERT (Vietnamese) + DistilBERT (English), but switched to XLM-R because **a single model handles multiple languages** (zero-shot cross-lingual capability) and simplifies deployment — only one model to package and serve.

Training configuration: `max_length=128` (based on the p95 sentence-length statistic), `batch=32`, `lr=2e-5`, 10% warmup, 3 epochs, fp16, early stopping on **macro-F1**. To fight label imbalance, `compute_loss` is overridden with a **weighted CrossEntropy** using inverse label-frequency weights:

```python
class WeightedTrainer(Trainer):
    def compute_loss(self, model, inputs, return_outputs=False, **kwargs):
        labels = inputs.pop("labels")
        outputs = model(**inputs)
        loss_fct = torch.nn.CrossEntropyLoss(weight=class_weights.to(model.device))
        loss = loss_fct(outputs.logits, labels)
        return (loss, outputs) if return_outputs else loss
```

![Training log](/images/5-Workshop/5.3/training-log.png)

#### 3. Evaluation results

Measured on the ViHSD test split, read from the evaluation cell of `train_text_classifier.ipynb`:

| Model | Accuracy | Macro-F1 | F1 CLEAN | F1 OFFENSIVE | F1 HATE |
|---|---|---|---|---|---|
| TF-IDF + Logistic Regression (baseline) | 0.8250 | 0.6230 | 0.9084 | 0.4181 | 0.5427 |
| XLM-RoBERTa-base, fine-tuned (deployed) | 0.7898 | 0.6065 | 0.8828 | 0.3866 | 0.5502 |

Against the proposal's target of macro-F1 ≥ 0.85, the fine-tuned model **did not meet the target** — and, being fully transparent, it also came in **below the TF-IDF baseline** on macro-F1 (0.6065 vs 0.6230). We report this as measured rather than presenting a more flattering number, because the gap is the most instructive result of the whole project.

![Confusion matrix](/images/5-Workshop/5.3/confusion-matrix.png)

![Baseline vs XLM-R comparison](/images/5-Workshop/5.3/compare-baseline.png)

**Why the fine-tuned model underperformed.** The root cause was compute, not method. No team member had a local GPU, so all fine-tuning ran on Google Colab's free tier, where the session is reclaimed after a few hours and the GPU is not guaranteed. In practice this capped us at **3 epochs on XLM-RoBERTa-base** with no hyper-parameter search — the training log shows validation loss already turning back up at epoch 3 (0.7407 → 0.7647) while macro-F1 had flattened at ~0.606, i.e. the run was stopped in the region where a longer, better-tuned schedule would normally start to pay off. A 270M-parameter transformer fine-tuned for 3 epochs on a heavily imbalanced 24k-sample set is simply not given enough budget to beat a well-regularised linear model on the majority class.

**What the aggregate numbers hide.** Macro-F1 alone is misleading for a moderation system. Reading the confusion matrix by class, the fine-tuned model is markedly better at *catching* harmful content:

| Class | Recall — baseline | Recall — XLM-R |
|---|---|---|
| OFFENSIVE | 0.4482 | **0.5315** |
| HATE | 0.6977 | **0.7645** |

XLM-R misses fewer OFFENSIVE and HATE messages; what it loses is precision (456 CLEAN messages flagged as OFFENSIVE), which drags macro-F1 down. For content moderation a false positive costs a second look, whereas a false negative means abusive text reaches the user — so the team deliberately deployed XLM-R and **designed the architecture around its weakness**: any prediction with confidence < 0.7 is escalated to Amazon Bedrock (Claude Haiku) for a second opinion, which is precisely the low-precision band. The cascade in section 5.4 is therefore not decoration, it is the mitigation for the number in this table.

**What we would do differently.** Train on a paid GPU runtime for 8–10 epochs with early stopping on macro-F1, sweep learning rate and warm-up, try `xlm-roberta-large` or a Vietnamese-pretrained encoder (PhoBERT) as a comparison, and use focal loss instead of plain class weighting for the 82/7/11 imbalance.

Finally, one property the baseline cannot match at any budget: although trained only on Vietnamese, XLM-R classifies **English** toxic sentences correctly thanks to its zero-shot cross-lingual capability — a TF-IDF model keyed to Vietnamese tokens scores these at chance:

![Prediction demo](/images/5-Workshop/5.3/demo-predict.png)

#### 4. ONNX export + INT8 quantization

The original PyTorch model weighs >1GB and needs torch at runtime — unsuitable for Lambda. Using `optimum`, the team exports to **ONNX** and applies **dynamic INT8 quantization**, cutting the size to ~1/4 with near-identical accuracy, running well on CPU:

```python
ort_model = ORTModelForSequenceClassification.from_pretrained("./xlmr-vihsd-v0", export=True)
quantizer = ORTQuantizer.from_pretrained("./xlmr-vihsd-onnx")
quantizer.quantize(save_dir="./xlmr-vihsd-onnx-int8",
                   quantization_config=AutoQuantizationConfig.avx2(is_static=False))
```

![ONNX export](/images/5-Workshop/5.3/onnx-export.png)

Two artifacts are needed for deployment: `model.onnx` + `tokenizer.json` (uploaded to S3 as the distribution source for the whole team).

#### 5. Packaging the Lambda container image

The inference handler depends only on `onnxruntime + tokenizers` (NO torch/transformers) → a much smaller image and faster cold starts:

```dockerfile
FROM public.ecr.aws/lambda/python:3.11
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY src/ ${LAMBDA_TASK_ROOT}/
COPY artifacts/ ${LAMBDA_TASK_ROOT}/artifacts/
ENV MODEL_DIR=/var/task/artifacts
CMD ["handler.lambda_handler"]
```

Processing flow in `handler.py`: validate input (empty → 400, over 2,000 characters → 400) → tokenize + run ONNX → softmax to label + confidence → if confidence < `CONFIDENCE_THRESHOLD` (0.7) call Bedrock for arbitration → collect flagged terms for UI highlighting → write to DynamoDB → return JSON with CORS headers. The model is **lazy-loaded once per container** to benefit from warm starts.

#### 6. Local testing with the Lambda RIE

The AWS base image ships with the **Runtime Interface Emulator** — the whole handler can be tested locally without spending a cent on AWS:

```bash
docker build -t fcaj-moderation:local .
docker run -d -p 9000:8080 -e ENABLE_BEDROCK=false -e TABLE_NAME= fcaj-moderation:local
curl -s -X POST "http://localhost:9000/2015-03-31/functions/function/invocations" \
  -d @events/offensive.json
```

![Local RIE test](/images/5-Workshop/5.3/rie-test.png)

The local suite (`scripts/test_local.sh`) drives four events — a clean sentence, teencode profanity, an empty string and a 2,001-character string — and all four return the expected status code. Measured on the RIE: the tokenizer loads in **2.86 s**, the ONNX session in **3.59 s**, and steady-state inference settles at **~11 ms per sentence** on CPU. This step is only a developer smoke test that shortens the feedback loop; the evidence that actually counts is the same handler answering correctly **inside AWS**, documented in section 5.4 with the Lambda console test results and CloudWatch logs.

#### 7. Pushing the image to Amazon ECR

```bash
aws ecr create-repository --repository-name fcaj-moderation --region ap-southeast-1
aws ecr get-login-password --region ap-southeast-1 \
  | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com
docker build --platform linux/amd64 -t <ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com/fcaj-moderation:v1 .
docker push <ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com/fcaj-moderation:v1
```

{{% notice tip %}}
The `--platform linux/amd64` flag is mandatory when building on ARM machines (Apple Silicon) — without it Lambda fails with `exec format error`.
{{% /notice %}}

![ECR repository](/images/5-Workshop/5.3/ecr-repo.png)

The image is now on ECR — on to the backend deployment.
