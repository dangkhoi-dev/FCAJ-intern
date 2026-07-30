# Ghi chú thiết kế sơ đồ kiến trúc

> Sơ đồ do Trần Phan Đăng Khôi thiết kế và vẽ; Lê Đức và Trần Quân review.
> File này ghi lại các quy ước đã áp dụng, để lần sau sửa vẫn giữ được tính nhất quán.
>
> Mở `architect.drawio` bằng draw.io (app.diagrams.net), sửa xong thì
> **File → Export as → PNG**, Zoom **300%**, Border width **10**, tick **Transparent background = OFF**.
> Lưu đè `images/architect_final.png`.

---

## 0. Tải bộ icon AWS mới

Trong draw.io: **More Shapes** (nút dưới cùng thanh trái) → mục **Networking** → tick **AWS 19** hoặc **AWS 2025** nếu có → **Apply**.

Bộ icon cũ (AWS 17 trở về trước) có viền và tỉ lệ khác, trộn lẫn hai bộ trong một hình là lỗi bị soi đầu tiên. **Chỉ dùng một bộ duy nhất.**

### Màu theo nhóm dịch vụ — bắt buộc đúng

| Nhóm | Màu | Dịch vụ trong sơ đồ |
|---|---|---|
| Compute | Cam `#ED7100` | Lambda |
| Containers | Cam `#ED7100` | ECR |
| Database | Tím-hồng "Nebula" `#C925D1` | DynamoDB *(icon hiện tại đã đúng)* |
| Storage | Xanh lá `#7AA116` | S3 |
| App Integration | Hồng `#E7157B` | API Gateway |
| Front-End Web & Mobile | Đỏ `#DD344C` | Amplify |
| Machine Learning | Xanh lá `#01A88D` | Bedrock |
| Management & Governance | Hồng đỏ `#E7157B` | CloudWatch, Organizations |
| Security, Identity & Compliance | Đỏ `#DD344C` | IAM, WAF, Cognito |
| Networking & Content Delivery | Tím `#8C4FFF` | CloudFront |

---

## 1. Quy ước đường nối — phải nhất quán

| Loại | Kiểu đường | Dùng cho |
|---|---|---|
| **Luồng request / dữ liệu** | **Nét liền**, mũi tên đặc, dày 2pt | User → Amplify → API GW → Lambda → Bedrock → DynamoDB → response |
| **Control plane / cấu hình** | **Nét đứt**, mũi tên rỗng, dày 1pt | IAM role gắn vào Lambda |
| **Telemetry (log, metric)** | **Nét chấm**, dày 1pt, màu xám | mọi service → CloudWatch |
| **Build/Deploy time** | **Nét đứt màu xám**, ghi rõ `(deploy time)` | Colab → S3 → Docker → ECR → Lambda |

**Ba lỗi đang có trong bản hiện tại phải sửa:**

- `Save Chat History & Results` đang là **nét đứt** nhưng đó là ghi dữ liệu thật → đổi thành **nét liền**
- `Pull Container Image` đang vẽ chung mặt phẳng với luồng request → chuyển sang **làn deploy time**, ghi lại nhãn thành `Image pulled at function create/update (cached)`
- Vòng lặp `Confidence >= 0.7` trên Lambda → **xoá mũi tên**, thay bằng **hình thoi quyết định** (rhombus) hoặc chú thích cạnh Lambda

---

## 2. Cấu trúc khung bao — từ ngoài vào trong

```
┌─ AWS Organization ─────────────────────────────────────────────┐
│  (khung nét liền mảnh, nhãn góc trên trái, icon Organizations)  │
│                                                                 │
│  ┌─ AWS Account: fcaj-prod ─────────────────────────────────┐  │
│  │  (khung nét liền, màu xám nhạt)                           │  │
│  │                                                            │  │
│  │  ┌─ Region: ap-southeast-1 (Singapore) ─────────────────┐ │  │
│  │  │  (khung NÉT ĐỨT, đây là chuẩn AWS cho Region)         │ │  │
│  │  │                                                        │ │  │
│  │  │   ... toàn bộ service nằm ở đây ...                    │ │  │
│  │  └────────────────────────────────────────────────────────┘ │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘

User  ← nằm NGOÀI mọi khung
```

**Sửa so với bản hiện tại:**

- **Amplify phải nằm TRONG khung Region.** Hiện đang nằm ngoài — sai, vì Amplify Hosting là dịch vụ regional
- **Organizations không vẽ thành icon lơ lửng** mà là khung bao ngoài cùng
- **IAM đặt cạnh khung Account**, mũi tên nét đứt trỏ vào **Lambda** (không phải API Gateway)
- **Xoá khung nét đứt rỗng** ở dưới cùng

---

## 3. Ghi chú VPC — quan trọng nhất

Kiến trúc này **cố ý không dùng VPC của khách hàng**. Người review không đọc được ý đồ đó nếu không ghi ra. Thêm một text box màu vàng nhạt, viền nét đứt, đặt cạnh Lambda:

```
No customer VPC - by design.
Lambda runs in the AWS-managed VPC. No private resources are accessed,
so a customer VPC would add:
  • NAT Gateway  -> hourly charge with zero traffic
  • ENI attachment -> longer cold start
DynamoDB, Bedrock and API Gateway are reached over AWS public endpoints
with IAM authentication (SigV4), not over the public internet.
```

Chính dòng này biến "quên vẽ VPC" thành "đã cân nhắc và quyết định". Đừng bỏ.

**Không được vẽ:** public subnet, private subnet, NAT Gateway, Internet Gateway — kiến trúc này không có. Vẽ vào là sai sự thật.

---

## 4. Đánh số luồng request

Đặt số trong hình tròn nhỏ ngay cạnh mỗi mũi tên, theo đúng thứ tự:

| # | Từ → Đến | Nhãn trên mũi tên |
|---|---|---|
| ① | User → CloudFront / Amplify Hosting | `HTTPS` |
| ② | Amplify → AWS WAF | `Rate-based rule` |
| ③ | WAF → API Gateway (REST) | `POST /moderate` |
| ④ | API Gateway → Lambda | `Proxy integration` |
| ⑤ | Lambda → *hình thoi quyết định* | `confidence >= threshold?` |
| ⑥ | Hình thoi (nhánh No) → Bedrock | `confidence < 0.7` |
| ⑦ | Lambda → DynamoDB | `PutItem` (nét liền) |
| ⑧ | Lambda → API Gateway → User | `JSON response` |

Nhánh **Yes** của hình thoi đi thẳng tới ⑦, không cần mũi tên vòng lại Lambda.

---

## 5. Thành phần phải bổ sung

Bản hiện tại thiếu những thứ sau. Ba cái đầu là bắt buộc.

| Thành phần | Vị trí | Vì sao |
|---|---|---|
| **AWS WAF** | Giữa Amplify và API Gateway | API đang public không giới hạn. Bất kỳ ai có URL đều gọi được Bedrock bằng tiền của nhóm |
| **Amazon Cognito** *(hoặc API key + Usage Plan)* | Gắn vào API Gateway dạng authorizer | Cùng lý do trên. Nếu chưa làm thật thì vẽ nét đứt và ghi `(planned)` |
| **S3** | Làn deploy time | Báo cáo viết S3 là nơi phân phối `model.onnx` + `tokenizer.json`, nhưng sơ đồ không có → sơ đồ mâu thuẫn văn bản |
| **SQS Dead Letter Queue** | Cạnh Lambda, nét đứt | Request lỗi hiện mất luôn |
| **CloudTrail** | Cạnh khung Account | Audit log cấp tổ chức |

Thành phần chưa triển khai thì vẽ **nét đứt + nhãn `(planned)`** và chú thích ở legend. Vẽ như đã có là khai sai.

---

## 6. Làn "Build / Deploy time" — vẽ tách riêng

Đặt thành một dải ngang phía dưới, ngăn bằng một đường kẻ mảnh, nhãn **`Build / Deploy time (not in request path)`**:

```
Google Colab ──► S3 (model artefacts) ──► docker build ──► ECR ──► UpdateFunctionCode ──► Lambda
   (train)         model.onnx                  (local)      v1        (image cached)
                   tokenizer.json
```

Toàn bộ làn này dùng **nét đứt màu xám**. Điều này trả lời dứt điểm hiểu lầm "mỗi request Lambda phải pull image từ ECR".

---

## 7. Legend — bắt buộc có

Góc dưới bên phải, khung viền mảnh:

```
──────►   Request / data flow
- - - ►   Control plane (IAM, configuration)
· · · ►   Logs & metrics
- - - ►   Build / deploy time (grey)
◇         Decision point
(planned) Not yet implemented
```

Sơ đồ doanh nghiệp không có legend bị coi là chưa hoàn chỉnh.

---

## 8. Checklist trước khi export

- [ ] Chỉ dùng **một** bộ icon AWS, không trộn cũ mới
- [ ] DynamoDB đã đổi sang **màu xanh dương** (Database)
- [ ] Amplify nằm **trong** khung Region
- [ ] Organizations là **khung bao**, không phải icon rời
- [ ] Mũi tên IAM trỏ vào **Lambda**
- [ ] `Save Chat History` đã đổi sang **nét liền**
- [ ] Vòng lặp `Confidence >= 0.7` đã thay bằng **hình thoi**
- [ ] `Pull Container Image` đã chuyển xuống **làn deploy time**
- [ ] Đã có **text box ghi chú No customer VPC**
- [ ] Đã đánh số ① đến ⑧ theo luồng request
- [ ] Đã thêm **WAF** và **authorizer** (nét đứt nếu chưa làm)
- [ ] Đã thêm **S3** vào làn deploy time
- [ ] Đã có **legend**
- [ ] Đã xoá khung nét đứt rỗng thừa
- [ ] Không còn chữ **PhoBERT** ở đâu — model là **XLM-RoBERTa**
- [ ] Không vẽ subnet / NAT Gateway / Internet Gateway
- [ ] Export PNG ở **Zoom 300%**, tối thiểu 3000px chiều ngang

---

## 9. Bố cục tham khảo

```
                    ┌──────────── AWS Organization ─────────────────────────────────┐
                    │  ┌────────── AWS Account: fcaj-prod ──────────────────────┐   │
  ┌──────┐          │  │                                          [CloudTrail]  │   │
  │ User │──①HTTPS─►│  │ ┌ ─ ─ ─ ─  Region: ap-southeast-1  ─ ─ ─ ─ ─ ─ ─ ─ ┐  │   │
  └──────┘          │  │                                                       │  │   │
                    │  │ │ [Amplify]─②─►[WAF]─③─►[API GW]─④─►[Lambda]        │  │   │
                    │  │                            ▲            │  ◇ ⑤       │  │   │
       [IAM]- - - - - -│- - - - - - - - - - - - - - ┘            │  │         │  │   │
                    │  │ │                                        │  └─⑥─►[Bedrock] │
                    │  │                                          ⑦            │  │   │
                    │  │ │                                        └────►[DynamoDB]│  │
                    │  │  · · · · · · · · · · · · · · · · · · · ·►[CloudWatch]  │  │   │
                    │  │ └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘  │   │
                    │  └────────────────────────────────────────────────────────┘   │
                    └───────────────────────────────────────────────────────────────┘
  ─────────────────────────────────────────────────────────────────────────────────
  Build / Deploy time:  [Colab]- ->[S3]- ->[docker build]- ->[ECR]- ->[Lambda]
```

---

## 10. Nếu còn thời gian — vẽ thêm sơ đồ thứ hai

Một **sequence diagram** cho riêng đường đi của một request sẽ ăn điểm rất mạnh, vì nó thể hiện được thứ mà sơ đồ kiến trúc không thể hiện được: **thứ tự thời gian** và **rẽ nhánh**.

```
User        Amplify    API GW     Lambda      Bedrock    DynamoDB
 │            │          │          │            │           │
 ├──POST─────►│          │          │            │           │
 │            ├─────────►│          │            │           │
 │            │          ├─────────►│            │           │
 │            │          │          ├─ ONNX inference (~10 ms, CPU)
 │            │          │          │            │           │
 │            │          │      [conf >= 0.7?]   │           │
 │            │          │          ├──No───────►│           │
 │            │          │          │◄──label────┤           │
 │            │          │          ├───────────────PutItem─►│
 │            │          │◄─────────┤            │           │
 │◄───────────┴──────────┤ JSON     │            │           │
```

Vẽ bằng draw.io (shape **UML → Sequence**) hoặc PlantUML đều được.
