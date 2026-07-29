---
title: "Event 3"
date: 2026-07-13
weight: 3
chapter: false
pre: " <b> 4.3. </b> "
---

### FCAJ — Agentic AI Build Week: ngày demo & pitch

&emsp;**Thời gian:** 25/07/2026

&emsp;**Địa điểm:** Văn phòng AWS Việt Nam

&emsp;**Vai trò:** Người tham dự

#### Nội dung sự kiện

Đây là ngày khép lại FCAJ Agentic AI Build Week — một hackathon trong đó các đội xây dựng sản phẩm agentic AI trên AWS rồi pitch trước cả hội trường. Phòng kín người, khoảng một trăm bạn, và format thì rất thẳng: mỗi đội lên sân khấu, trình bày bài toán đã chọn, demo đúng thứ họ thực sự làm được trong tuần đó, rồi nhận câu hỏi.

![Khai mạc ngày demo Agentic AI Build Week](/images/4-EventParticipated/event3-1.jpg)

*Phần khai mạc ngày demo tại văn phòng AWS.*

Bốn đội đã trình bày:

| Đội | Sản phẩm | Ý tưởng trong một dòng |
|---|---|---|
| **3KA** | The Hackathon Journey | Nhìn lại quá trình làm việc dưới áp lực thời gian, kể theo mạch nghi ngờ → nhập cuộc → tự hào |
| **One Team** | Đặt hàng bằng hội thoại có AI | Cho khách đặt hàng ngay trong app chat họ đang dùng, thay vì một app riêng |
| **Plan V** | Solution Architect native app | Một agent biến yêu cầu thành bản kiến trúc nháp, sơ đồ và IaC |
| **Signal Scout** | Phát hiện sớm thay đổi chiến lược của doanh nghiệp | Theo dõi tín hiệu công khai để nhận ra một công ty đang chuyển hướng trước khi được công bố |

Sản phẩm của Plan V nhắm vào một nút thắt có thật trong công việc của solution architect: đọc BRD hay PRD, phác kiến trúc ban đầu và vẽ sơ đồ dưới áp lực deadline. Agent của họ nhận cả yêu cầu dạng ngôn ngữ tự nhiên lẫn dạng có cấu trúc, sinh ra danh mục yêu cầu trong vài phút, phác các phương án kiến trúc có tính đến hybrid cloud, tạo sơ đồ draw.io chỉnh sửa được bằng đúng bộ icon AWS chính thức, xuất IaC, và kèm theo một ước tính chi phí định hướng ngay cạnh kiến trúc. Signal Scout trình bày giải pháp qua một canvas về tạo và phân phối giá trị, nêu AWS, LangFuse, TinyFish và Apify là các đối tác chính, và đưa kết quả tới người dùng qua một dashboard tự phục vụ. Bài pitch của One Team, tựa đề "Ordering Without Leaving the Chat", mở đầu bằng bối cảnh và bài toán trước khi cho xem sản phẩm.

![Một đội trình bày canvas tạo và phân phối giá trị](/images/4-EventParticipated/event3-2.jpg)

*Signal Scout trình bày canvas tạo và phân phối giá trị.*

![One Team trình bày "Ordering Without Leaving the Chat"](/images/4-EventParticipated/event3-3.jpg)

*One Team trình bày "Ordering Without Leaving the Chat".*

#### Bài học và giá trị nhận được

Thứ hữu ích nhất tôi mang về là slide chi phí của Signal Scout. Họ bóc kiến trúc ra theo từng dịch vụ — token Bedrock, AgentCore short-term memory, AgentCore runtime, WAF, Amplify Hosting, CloudWatch, Secrets Manager, DynamoDB, Lambda — và đưa con số tối thiểu, trung bình, tối đa hằng tháng cho mỗi dịch vụ, để người xem thấy rõ thành phần nào sẽ chiếm phần lớn hoá đơn khi lượng dùng tăng lên. Đề xuất của nhóm tôi thì ước tính "dưới 1 USD" bằng đúng một con số. Nhìn cách họ làm, tôi nhận ra một con số duy nhất không cho biết dịch vụ nào sẽ gãy trước khi tải tăng, và đó là format tôi sẽ dùng nếu tính lại chi phí cho dự án này.

Plan V làm tôi thay đổi cách hiểu về hai chữ "agentic". Trước buổi đó, hình dung của tôi về việc dùng LLM trên AWS về cơ bản chính là những gì dự án nhóm đang làm — một lệnh gọi model để phân loại một đoạn văn bản rồi trả kết quả. Sản phẩm của họ nối cả một quy trình nghiệp vụ lại với nhau, từ đọc tài liệu yêu cầu cho tới xuất ra mã hạ tầng, và giới thiệu cho tôi Bedrock AgentCore runtime cùng memory như những khối dựng sẵn mà tôi chưa từng dùng. Bài của One Team đưa ra một điểm về sản phẩm áp dụng được ngay cho hệ thống của nhóm: họ cố ý không xây một giao diện mới, mà đi tới nơi người dùng vốn đã ở đó. Dịch vụ kiểm duyệt của nhóm tôi hiện là một trang demo độc lập, nhưng vì nó thiết kế theo hướng API-first, đúng theo lập luận đó thì chỗ thuộc về nó là bên trong nền tảng cần kiểm duyệt, chứ không phải một trang riêng.

Bài của 3KA là bài tôi không ngờ mình lại thấy giá trị. Nó hoàn toàn không phải một bài kỹ thuật — đó là lời kể thành thật về khúc giữa của một quá trình xây dựng, đoạn mà sản phẩm chưa chạy được và bạn cũng không chắc nó sẽ chạy. Nghe một đội nói thẳng điều đó trước cả trăm người là một đối trọng cần thiết với những bản demo đã đánh bóng, và cũng là mô tả khá đúng về tuần làm việc của chính nhóm tôi.
