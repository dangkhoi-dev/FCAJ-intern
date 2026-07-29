---
title: "Event 1"
date: 2026-07-13
weight: 1
chapter: false
pre: " <b> 4.1. </b> "
---

### Buổi chia sẻ cộng đồng FCAJ / AWS Study Group

&emsp;**Thời gian:** 06/06/2026

&emsp;**Địa điểm:** Buổi sinh hoạt cộng đồng của chương trình First Cloud AI Journey (AWS Study Group)

&emsp;**Vai trò:** Người tham dự

#### Nội dung sự kiện

Đây là buổi chia sẻ cộng đồng với sáu bài trình bày liên tiếp, cố ý pha trộn nội dung kỹ thuật thực hành với các chủ đề về nghề nghiệp và làm việc nhóm. Diễn giả gồm cả kỹ sư đang đi làm lẫn sinh viên trong chương trình, và mỗi người đều trình bày thứ mình đã thực sự làm hoặc đã trải qua, chứ không phải một bài tổng quan chung chung.

| # | Diễn giả | Chủ đề |
|---|---|---|
| 1 | Bảo Huỳnh — Junior Cloud Native Developer, Endava Vietnam | Docker — công nghệ container hoá |
| 2 | Lê Hoàng Gia Đại — HUTECH | Kết hợp AWS WAF với machine learning để phát hiện tấn công mạng trên AWS |
| 3 | Nguyễn Quốc Bảo | Multiplayer trên cloud: kết nối client Godot bằng AWS WebSockets |
| 4 | Trương Huy Phước | Nghệ thuật làm việc nhóm hiệu quả |
| 5 | Việt Phát — Swinburne | Xây dựng ứng dụng GraphRAG với Amazon Bedrock và Amazon Neptune |
| 6 | Trần Trung Vinh — System Administrator, Central Retail Group | Từ IT helpdesk đến senior sysadmin |

Bài về Docker đi từ ảo hoá đến container hoá, so sánh máy ảo với container, rồi đọc từng dòng một Dockerfile và kết thúc bằng demo trực tiếp. Bài về AWS WAF lập luận rằng chỉ dựa vào rule thì không thể bắt được tấn công zero-day hay tấn công lai, và trình diễn một lớp phát hiện xâm nhập mạng huấn luyện bằng LightGBM trên bộ dữ liệu CSE-CIC-IDS2018, ghép vào WAF, Kinesis Data Firehose, Lambda, Security Hub và GuardDuty. Bài về Godot dựng luồng ghép trận multiplayer thời gian thực trên API Gateway WebSocket, Lambda và DynamoDB, rồi so sánh cách làm đó với AWS GameLift. Bài GraphRAG chỉ ra vì sao RAG thuần thất bại với câu hỏi nhiều bước suy luận, và trình bày hai hướng — hướng fully managed dùng Bedrock Knowledge Bases với Neptune Analytics, và hướng custom dùng LlamaIndex với Amazon Neptune. Hai bài phi kỹ thuật nói về công cụ phối hợp nhóm và hành trình của một diễn giả từ vị trí helpdesk lên senior system administrator.

#### Bài học và giá trị nhận được

Hai trong số các bài đã thay đổi cách tôi làm việc trên chính dự án của nhóm ngay trong tuần đó. Phần giải thích về layer caching trong bài Docker — rằng một instruction thay đổi sẽ làm hỏng toàn bộ layer phía sau nó — giải thích đúng lý do image Lambda container của tôi cứ build lại từ đầu mỗi lần; chuyển bước cài dependency lên trên bước copy model trong Dockerfile đã rút ngắn vòng lặp build của tôi đi rất nhiều. Bài AWS WAF còn liên quan trực tiếp hơn: lập luận cốt lõi của nó, rằng rule dựa trên chữ ký sẽ gãy khi gặp input chưa từng thấy, chính là lý do vì sao hệ thống kiểm duyệt của nhóm không thể chỉ là một danh sách từ cấm; và phần xử lý lệch nhãn trong bài gần như trùng khớp với phân bố lệch CLEAN/OFFENSIVE/HATE của ViHSD mà tôi đang phải xoay xở lúc đó.

Bài Godot dạy tôi một điều về serverless mà tôi chưa thực sự thấm: vì Lambda không giữ trạng thái, mọi state cần quan tâm đều phải đẩy xuống một nơi lưu trữ như DynamoDB, và cái giá của việc làm sai (scan một bảng ngày càng lớn ở mỗi message) sẽ phình theo lượng dùng. Điều đó khiến tôi nhìn bảng `ModerationHistory` của nhóm như một quyết định thiết kế chứ không phải thứ làm cho có. Bài GraphRAG mở rộng góc nhìn của tôi về Bedrock, vượt xa một lệnh gọi model đơn lẻ như dự án của nhóm đang dùng. Còn bài từ helpdesk lên sysadmin để lại cho tôi hai thói quen tôi cố giữ từ đó tới nay: đừng bao giờ test trên production, và hãy ghi lại cấu hình ngay khi còn nhớ vì sao mình cấu hình như vậy.

![Hình ảnh buổi 06/06](/images/4-EventParticipated/event1-1.jpg)
