---
title: "Huấn luyện và đóng gói model"
date: 2026-07-13
weight: 3
chapter: false
pre: " <b> 5.3. </b> "
---

Mục này trình bày toàn bộ pipeline từ dữ liệu thô đến một Docker image inference sẵn sàng deploy: chuẩn bị dataset → huấn luyện → đánh giá → xuất ONNX INT8 → đóng gói Lambda container → test local → push lên ECR.

#### 1. Dataset và tiền xử lý

Nhóm sử dụng **ViHSD** (Vietnamese Hate Speech Detection, ~33.000 bình luận) với 3 nhãn: `0 = CLEAN`, `1 = OFFENSIVE`, `2 = HATE`, chia train/val/test theo bản gốc.

Đặc điểm quan trọng nhất của dataset là **mất cân bằng nhãn nghiêm trọng** — khoảng 82% là CLEAN:

![Phân bố nhãn](/images/5-Workshop/5.3/label-distribution.png)

Tiền xử lý gồm: chuẩn hóa unicode **NFC**, thay URL bằng token `<URL>`, mention bằng `<USER>`, gộp khoảng trắng. Một quyết định ngược trực giác nhưng quan trọng: **giữ nguyên teencode và từ tục** ("vcl", "đm"...) — đó chính là tín hiệu phân loại, lọc đi là mất thông tin.

```python
def clean_text(t):
    t = unicodedata.normalize("NFC", str(t))
    t = URL_RE.sub(" <URL> ", t)        # URL -> token
    t = MENTION_RE.sub(" <USER> ", t)   # mention -> token
    return re.sub(r"\s+", " ", t).strip()
```

#### 2. Huấn luyện trên Google Colab (GPU T4)

Notebook `train_text_classifier.ipynb` chạy trọn vẹn trên Colab GPU T4 miễn phí (~45–90 phút), seed 42 để tái lập:

![Colab GPU](/images/5-Workshop/5.3/colab-gpu.png)

**Baseline** trước: TF-IDF (1–2 gram) + Logistic Regression với `class_weight="balanced"` — mốc so sánh rẻ tiền mà model chính buộc phải vượt qua.

**Model chính**: fine-tune **XLM-RoBERTa-base**. Ban đầu nhóm đề xuất PhoBERT (tiếng Việt) + DistilBERT (tiếng Anh), nhưng quyết định chuyển sang XLM-R vì **một model duy nhất xử lý được đa ngôn ngữ** (khả năng zero-shot cross-lingual) và đơn giản hóa việc triển khai — chỉ cần đóng gói và phục vụ một model.

Cấu hình huấn luyện: `max_length=128` (dựa trên thống kê p95 độ dài câu), `batch=32`, `lr=2e-5`, warmup 10%, 3 epoch, fp16, early stopping theo **macro-F1**. Để chống lệch nhãn, nhóm override `compute_loss` dùng **weighted CrossEntropy** với trọng số nghịch đảo tần suất nhãn:

```python
class WeightedTrainer(Trainer):
    def compute_loss(self, model, inputs, return_outputs=False, **kwargs):
        labels = inputs.pop("labels")
        outputs = model(**inputs)
        loss_fct = torch.nn.CrossEntropyLoss(weight=class_weights.to(model.device))
        loss = loss_fct(outputs.logits, labels)
        return (loss, outputs) if return_outputs else loss
```

![Log huấn luyện](/images/5-Workshop/5.3/training-log.png)

#### 3. Kết quả đánh giá

Đo trên tập test của ViHSD, lấy từ cell đánh giá trong `train_text_classifier.ipynb`:

| Mô hình | Accuracy | Macro-F1 | F1 CLEAN | F1 OFFENSIVE | F1 HATE |
|---|---|---|---|---|---|
| TF-IDF + Logistic Regression (baseline) | 0.8250 | 0.6230 | 0.9084 | 0.4181 | 0.5427 |
| XLM-RoBERTa-base fine-tune (bản deploy) | 0.7898 | 0.6065 | 0.8828 | 0.3866 | 0.5502 |

So với mục tiêu macro-F1 ≥ 0.85 đặt ra trong proposal, mô hình fine-tune **chưa đạt** — và nói thẳng, nó còn **thấp hơn cả baseline TF-IDF** về macro-F1 (0.6065 so với 0.6230). Nhóm giữ nguyên con số đo được thay vì trình bày một kết quả đẹp hơn, vì chính khoảng cách này mới là bài học đáng giá nhất của cả dự án.

![Confusion matrix](/images/5-Workshop/5.3/confusion-matrix.png)

![So sánh baseline vs XLM-R](/images/5-Workshop/5.3/compare-baseline.png)

**Vì sao model fine-tune lại kém hơn.** Nguyên nhân nằm ở tài nguyên tính toán, không phải ở phương pháp. Không thành viên nào có GPU để train local, nên toàn bộ quá trình fine-tune chạy trên Google Colab bản miễn phí — nơi session bị thu hồi sau vài giờ và GPU thì không được bảo đảm. Thực tế nhóm chỉ chạy được **3 epoch trên XLM-RoBERTa-base**, không có thời gian tìm siêu tham số. Nhìn log huấn luyện thấy rõ: validation loss đã quay đầu tăng ở epoch 3 (0.7407 → 0.7647) trong khi macro-F1 chững lại quanh 0.606 — nghĩa là quá trình train bị cắt đúng vào giai đoạn mà một lịch huấn luyện dài hơn, tinh chỉnh kỹ hơn mới bắt đầu phát huy tác dụng. Một transformer 270 triệu tham số, train 3 epoch trên tập 24k mẫu lệch nhãn nặng, không đủ "ngân sách" để vượt một mô hình tuyến tính được regularize tốt ở lớp chiếm đa số.

**Điều mà con số tổng hợp che giấu.** Chỉ nhìn macro-F1 là đánh giá sai bản chất với một hệ thống kiểm duyệt. Đọc confusion matrix theo từng lớp, model fine-tune **bắt được nhiều nội dung độc hại hơn hẳn**:

| Nhãn | Recall — baseline | Recall — XLM-R |
|---|---|---|
| OFFENSIVE | 0.4482 | **0.5315** |
| HATE | 0.6977 | **0.7645** |

XLM-R bỏ sót ít câu OFFENSIVE và HATE hơn; cái nó đánh đổi là precision (456 câu CLEAN bị gắn nhầm nhãn OFFENSIVE), và chính điều đó kéo macro-F1 xuống. Với bài toán kiểm duyệt nội dung, một false positive chỉ tốn thêm một lần xem lại, còn một false negative nghĩa là nội dung xúc phạm lọt tới người dùng — nên nhóm chủ động chọn deploy XLM-R và **thiết kế kiến trúc để bù đúng điểm yếu đó**: mọi dự đoán có confidence < 0.7 đều được đẩy sang Amazon Bedrock (Claude Haiku) thẩm định lại, mà đó chính là vùng model cho precision thấp. Cơ chế cascade ở mục 5.4 vì vậy không phải để trang trí, nó là phương án xử lý cho đúng con số trong bảng này.

**Nếu làm lại, nhóm sẽ làm gì.** Train trên runtime GPU trả phí 8–10 epoch kèm early stopping theo macro-F1, quét learning rate và warm-up, thử `xlm-roberta-large` hoặc một encoder pretrain riêng cho tiếng Việt (PhoBERT) để đối chứng, và dùng focal loss thay cho class weighting đơn thuần cho tỉ lệ lệch 82/7/11.

Cuối cùng, có một điểm baseline không thể làm được dù cho bao nhiêu tài nguyên: dù chỉ train trên tiếng Việt, XLM-R vẫn phân loại đúng câu toxic **tiếng Anh** nhờ khả năng zero-shot đa ngôn ngữ — trong khi mô hình TF-IDF gắn với từ vựng tiếng Việt chỉ đoán ngẫu nhiên ở những câu này:

![Demo dự đoán](/images/5-Workshop/5.3/demo-predict.png)

#### 4. Xuất ONNX + quantize INT8

Model PyTorch gốc nặng >1GB và cần torch để chạy — không phù hợp Lambda. Nhóm dùng `optimum` xuất sang **ONNX** rồi **quantize dynamic INT8**, giảm kích thước còn ~1/4 với độ chính xác gần như không đổi, chạy tốt trên CPU:

```python
ort_model = ORTModelForSequenceClassification.from_pretrained("./xlmr-vihsd-v0", export=True)
quantizer = ORTQuantizer.from_pretrained("./xlmr-vihsd-onnx")
quantizer.quantize(save_dir="./xlmr-vihsd-onnx-int8",
                   quantization_config=AutoQuantizationConfig.avx2(is_static=False))
```

![Export ONNX](/images/5-Workshop/5.3/onnx-export.png)

Hai artifact cần cho bước deploy: `model.onnx` + `tokenizer.json` (upload lên S3 làm nguồn phân phối cho cả nhóm).

#### 5. Đóng gói Lambda container image

Handler inference chỉ phụ thuộc `onnxruntime + tokenizers` (KHÔNG cần torch/transformers) → image nhỏ hơn nhiều, cold start nhanh hơn:

```dockerfile
FROM public.ecr.aws/lambda/python:3.11
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY src/ ${LAMBDA_TASK_ROOT}/
COPY artifacts/ ${LAMBDA_TASK_ROOT}/artifacts/
ENV MODEL_DIR=/var/task/artifacts
CMD ["handler.lambda_handler"]
```

Luồng xử lý trong `handler.py`: validate input (rỗng → 400, quá 2000 ký tự → 400) → tokenize + chạy ONNX → softmax ra nhãn + confidence → nếu confidence < `CONFIDENCE_THRESHOLD` (0.7) thì gọi Bedrock phân xử → tìm từ vi phạm cho UI highlight → ghi DynamoDB → trả JSON kèm CORS header. Model được **lazy-load một lần cho mỗi container** để tận dụng warm start.

#### 6. Test local bằng Lambda RIE

Base image của AWS có sẵn **Runtime Interface Emulator** — test toàn bộ handler ngay trên máy, không tốn một đồng AWS nào:

```bash
docker build -t fcaj-moderation:local .
docker run -d -p 9000:8080 -e ENABLE_BEDROCK=false -e TABLE_NAME= fcaj-moderation:local
curl -s -X POST "http://localhost:9000/2015-03-31/functions/function/invocations" \
  -d @events/offensive.json
```

![Test RIE local](/images/5-Workshop/5.3/rie-test.png)

Bộ test local (`scripts/test_local.sh`) chạy bốn event — câu sạch, câu tục teencode, chuỗi rỗng và chuỗi 2001 ký tự — cả bốn đều trả về đúng status code mong đợi. Đo thực tế trên RIE: tokenizer nạp trong **2,86 s**, ONNX session **3,59 s**, inference ổn định ở mức **~11 ms/câu** trên CPU. Bước này chỉ là smoke test giúp rút ngắn vòng lặp phát triển; minh chứng thực sự có giá trị là chính handler đó chạy đúng **trên AWS**, được trình bày ở mục 5.4 với kết quả test trên Lambda console và CloudWatch logs.

#### 7. Push image lên Amazon ECR

```bash
aws ecr create-repository --repository-name fcaj-moderation --region ap-southeast-1
aws ecr get-login-password --region ap-southeast-1 \
  | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com
docker build --platform linux/amd64 -t <ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com/fcaj-moderation:v1 .
docker push <ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com/fcaj-moderation:v1
```

{{% notice tip %}}
Flag `--platform linux/amd64` là bắt buộc nếu build trên máy ARM (Mac M-series) — thiếu nó Lambda sẽ lỗi `exec format error`.
{{% /notice %}}

![ECR repository](/images/5-Workshop/5.3/ecr-repo.png)

Image đã nằm trên ECR — sang bước triển khai backend.
