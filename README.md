# TEAM CHARTER

## UniSupport - Student Support Management System

---

# I. Tổng quan và Bối cảnh

## 1. Bối cảnh & Tuyên bố vấn đề

### Bối cảnh

Aurora University hiện phục vụ khoảng **3.000 sinh viên**.

Quy trình tiếp nhận các yêu cầu hỗ trợ như:

- Hành chính
- Học tập
- Cơ sở vật chất
- Các vấn đề hỗ trợ sinh viên khác

hiện đang bị phân tán qua nhiều kênh:

- Email
- Microsoft Forms
- Tin nhắn cá nhân

### Vấn đề cốt lõi

- Thông tin có nguy cơ bị **thất lạc**.
- Thời gian phản hồi chậm, trung bình khoảng **3–5 ngày**.
- Sinh viên không thể biết **ai đang xử lý yêu cầu của mình**.
- Nhân viên có thể **xử lý trùng lặp hoặc bỏ sót công việc** do không có hệ thống điều phối tập trung.
- Ban Quản lý không có dữ liệu tổng hợp để **đánh giá hiệu suất phục vụ của các phòng ban**.

---

## 2. Giải pháp tổng thể

Xây dựng **Web Application UniSupport** nhằm tập trung hóa luồng giao tiếp và xử lý yêu cầu giữa **Sinh viên** và **Nhà trường**.

### Luồng xử lý tổng quát

```text
Sinh viên gửi Ticket
        ↓
Hệ thống đưa Ticket vào Queue của Phòng ban
        ↓
Nhân viên tiếp nhận và xử lý
        ↓
Phản hồi / yêu cầu bổ sung nếu cần
        ↓
Hoàn thành Ticket
        ↓
Sinh viên đánh giá chất lượng dịch vụ
```

---

## 3. Nguyên tắc phát triển mở rộng

### Không làm vỡ tính năng cũ

Các module sau chỉ:

- `consume` dữ liệu từ module trước
- hoặc `extend` chức năng dựa trên dữ liệu đã tồn tại

Không thay đổi logic cốt lõi theo cách gây ảnh hưởng đến các module đã hoàn thành.

### Single Source of Truth

Bảng `Tickets` là nguồn dữ liệu trung tâm của hệ thống.

Trạng thái Ticket chỉ được thay đổi theo **Ticket State Machine** đã định nghĩa.

Không được ghi đè hoặc xóa dữ liệu lịch sử xử lý Ticket.

---

# II. Tiêu chuẩn kỹ thuật và Quy tắc dữ liệu

## 1. Data Conventions

### Ticket ID

Định dạng:

```text
TK-YYYYMMDD-XXXX
```

Ví dụ:

```text
TK-20261001-0042
```

Trong đó:

- `YYYYMMDD`: ngày tạo Ticket
- `XXXX`: số thứ tự tự động tăng trong ngày

---

### File đính kèm

Các định dạng được phép:

```text
.png
.jpg
.jpeg
.pdf
```

Giới hạn:

- Tối đa **5 MB / file**
- Tối đa **3 file / lần gửi**

---

### Data Isolation

#### Sinh viên

Sinh viên chỉ được phép `READ` các Ticket do chính tài khoản đó tạo.

```text
created_by = current_user.id
```

#### Nhân viên

Nhân viên chỉ được phép `READ / WRITE` các Ticket thuộc phòng ban của mình.

```text
department_id = current_user.department_id
```

---

## 2. Ticket State Machine

Mọi chức năng xử lý Ticket phải tuân thủ các trạng thái và
