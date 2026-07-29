# Hướng dẫn xem local và đưa báo cáo lên GitHub Pages

> Cập nhật 29/07/2026. Mở PowerShell, `cd D:\AWS\fcj-workshop-template` rồi làm theo thứ tự dưới.

---

## Tình trạng hiện tại của website

| Hạng mục | Kết quả |
|---|---|
| Trang tiếng Anh / tiếng Việt | 33 / 33 — khớp nhau, không trang nào lẻ |
| Tham chiếu ảnh | 106 tham chiếu, **0 ảnh thiếu** |
| Link nội bộ | 32 link, **0 link gãy** |
| Link viết hoa (gây 404 trên GitHub Pages) | **0** |
| Ô `[FILL]` / `[ĐIỀN]` còn trống | **0** |
| Front matter | 66/66 file hợp lệ, đã chuẩn hoá `title:` |
| Bảng markdown lệch cột | 0 |

Website sẵn sàng deploy.

---

## Bước 0 — Nếu PowerShell chặn script

Chạy một lần cho mỗi cửa sổ PowerShell mới:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

---

## Bước 1 — Xem thử ở máy

```powershell
cd D:\AWS\fcj-workshop-template
.\2-XEM-LOCAL.ps1
```

Mở trình duyệt vào **http://localhost:1313/**. Sửa file `.md` nào rồi lưu lại là trang tự nạp lại, không phải chạy lại script. Dừng server bằng **Ctrl + C**.

Nếu báo *"hugo không tìm thấy"*, cài Hugo **extended** (bắt buộc bản extended, bản thường sẽ lỗi SCSS):

```powershell
winget install Hugo.Hugo.Extended
```

Cài xong **đóng PowerShell, mở lại**, rồi chạy lại script.

### Cần nhìn kỹ những trang này khi xem local

- **Trang chủ** — 8 mục ở "Nội dung báo cáo" phải bấm vào được hết, không mục nào 404.
- **1. Worklog** — đủ 12 tuần, mỗi tuần có dòng ngày tháng (Tuần 1 bắt đầu 11/05/2026, Tuần 12 kết thúc 31/07/2026).
- **3. Blogs** — phải có **3 bài**, mỗi bài có ảnh chụp bài đăng.
- **4. Events** — 3 sự kiện, buổi 06/06 phải có ảnh (`event1-1.jpg`).
- **5.3** — bảng chỉ số model đủ 12 số, dưới bảng có phần phân tích vì sao chưa đạt.
- **2. Proposal** — sơ đồ kiến trúc phải ghi **XLM-RoBERTa** và **Pull Container Image** (không phải PhoBERT / Pull model weights).

---

## Bước 2 — Đẩy lên GitHub

```powershell
.\3-DAY-LEN-GITHUB.ps1
```

Script làm 7 bước, tự động hết:

1. Kiểm tra đã cài git, tự đặt `user.name` / `user.email` nếu chưa có.
2. Clone hoặc pull repo `github.com/dangkhoi-dev/FCAJ-intern` về `D:\AWS\FCAJ-intern`.
3. Copy nội dung site sang bản sao đó.
4. Dọn rác (xoá `.git` lồng trong `themes/` — đây là nguyên nhân số 1 làm site build ra trang trắng).
5. Kiểm tra 13 file bắt buộc phải có; thiếu file nào là **dừng lại**, không push bậy.
6. Commit + push.
7. In ra link báo cáo.

> **Quan trọng:** luôn sửa file ở `D:\AWS\fcj-workshop-template`. Thư mục `D:\AWS\FCAJ-intern` chỉ là bản sao để đẩy lên, sửa ở đó lần chạy sau sẽ bị ghi đè.

Lần đầu chạy, git sẽ hỏi đăng nhập GitHub — đăng nhập bằng trình duyệt là xong.

---

## Bước 3 — Bật GitHub Pages (chỉ làm 1 lần duy nhất)

Sau khi bước 2 chạy xong và tab **Actions** trên GitHub đã hiện dấu tick xanh:

1. Vào `https://github.com/dangkhoi-dev/FCAJ-intern/settings/pages`
2. **Source** → chọn **Deploy from a branch**
3. **Branch** → chọn **`gh-pages`** → thư mục **`/ (root)`** → **Save**
4. Đợi 1–2 phút

**Link nộp:** `https://dangkhoi-dev.github.io/FCAJ-intern/`

Từ lần sau chỉ cần chạy lại `.\3-DAY-LEN-GITHUB.ps1`, link tự cập nhật, không phải vào Settings nữa.

---

## GitHub Action hoạt động thế nào

File `.github/workflows/hugo.yml` chạy tự động mỗi lần push lên nhánh `main` (hoặc bấm tay ở tab **Actions** → **Run workflow**).

Nó làm 6 việc:

1. Checkout mã nguồn kèm lịch sử đầy đủ (theme cần `.GitInfo` để hiện ngày cập nhật).
2. Xoá `public/` cũ để không dính file thừa từ lần build trước.
3. **Kiểm tra theme có tồn tại không** — nếu thiếu `themes/hugo-theme-learn/theme.toml` thì báo lỗi rõ ràng thay vì build ra trang trắng.
4. **Tự tính `baseURL`** từ tên repo. Nghĩa là đổi tên repo cũng không vỡ CSS, không phải nhớ sửa `config.toml`.
5. Build bằng Hugo extended 0.134.3 với `--minify --gc`.
6. Deploy `public/` sang nhánh `gh-pages` (dùng `force_orphan` nên nhánh này luôn sạch, không phình lịch sử).

Có `concurrency` nên push liên tiếp thì build cũ bị huỷ, tránh hai lần deploy đè lên nhau.

### Nếu Action đỏ

Vào tab **Actions** → bấm vào lần chạy đỏ → mở bước bị lỗi. Ba lỗi hay gặp:

| Triệu chứng | Nguyên nhân | Cách sửa |
|---|---|---|
| `Thieu themes/hugo-theme-learn` | Thư mục theme chưa được push lên | `git add -f themes/ && git commit -m "add theme" && git push` |
| Site lên nhưng **trắng trơn, không CSS** | Có `.git` lồng trong `themes/` nên GitHub coi nó là submodule | Script bước [4/7] đã tự xử lý; nếu vẫn còn thì xoá hẳn `D:\AWS\FCAJ-intern` rồi chạy lại script |
| Trang chủ vào được nhưng 8 mục đều **404** | Link viết hoa chữ đầu, mà GitHub Pages phân biệt hoa–thường | Đã sửa hết; nếu thêm link mới thì nhớ viết thường (`1-worklog/` chứ không phải `1-Worklog/`) |

---

## Hai thứ tôi vừa sửa trong script push

**1. Script đang xoá mất Blog 2 và Blog 3.** Bước [4/7] có danh sách "dọn rác" chứa `content\3-BlogsPosted\3.2-Blog2` và `3.3-Blog3` — sót lại từ lúc báo cáo chỉ có 1 bài blog. Nếu chạy nguyên bản, hai bài blog vừa dựng lại sẽ bị xoá ngay trước khi push. Đã gỡ hai dòng đó và thêm chúng vào danh sách file bắt buộc kiểm tra ở bước [5/7], cùng với `blog2.png`, `blog3.png` và `event1-1.jpg`.

**2. Không đẩy file cá nhân lên repo public.** Đã thêm `*.docx` và `CHECKLIST-SCREENSHOT.md` vào danh sách loại trừ, để `THONG-TIN-CAN-DIEN.docx` (có tên cán bộ hướng dẫn, số điện thoại) không lên GitHub.

---

## Về quyền riêng tư — đã kiểm tra

Script chỉ đồng bộ đúng 6 thư mục: `.github`, `archetypes`, `content`, `layouts`, `static`, `themes`, cộng vài file lẻ ở gốc.

Nghĩa là **các thư mục sau KHÔNG được đẩy lên GitHub**, và điều đó là đúng:

- `HCMUT_RULE\` — hơn 1.000 file hồ sơ thực tập của hàng chục doanh nghiệp khác.
- `D2_D3\` — form D3 chứa **họ tên và MSSV của 117 sinh viên**. Đây là dữ liệu cá nhân của người khác, tuyệt đối không được để lên repo public.
- `Template\` — báo cáo LaTeX nộp Khoa.
- `THONG-TIN-CAN-DIEN.docx`

Nếu sau này bạn sửa script, giữ nguyên nguyên tắc này.
