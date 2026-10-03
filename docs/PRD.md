# TEAM CHARTER (BẢN ĐIỀU LỆ NHÓM)

**Dự án: UniSupport - Student Support Management System**

---

# PHẦN I: TỔNG QUAN VÀ BỐI CẢNH (CONTEXT & STRATEGY)

## 1. Bối cảnh & Tuyên bố vấn đề (Context & Problem Statement)

**Bối cảnh:** Aurora University hiện phục vụ khoảng **3.000 sinh viên**. Quy trình tiếp nhận yêu cầu hỗ trợ (hành chính, học tập, cơ sở vật chất...) đang phân tán qua **Email, Microsoft Forms, tin nhắn cá nhân**.

**Vấn đề cốt lõi:**

- Thất lạc thông tin, phản hồi chậm (trung bình **3–5 ngày**).
- Sinh viên không thể tra cứu ai đang xử lý đơn của mình.
- Nhân viên trùng lặp hoặc bỏ sót công việc do không có công cụ điều phối tập trung.
- Ban Quản lý không có số liệu để đánh giá hiệu suất phục vụ của các phòng ban.

---

## 2. Giải pháp tổng thể (Proposed Solution)

Xây dựng hệ thống **Web Application UniSupport** tập trung hóa luồng giao tiếp giữa Sinh viên và Nhà trường.

**Kiến trúc luồng:**

```text
Sinh viên gửi Ticket
        ↓
Hệ thống điều phối về Queue Phòng ban
        ↓
Nhân viên xử lý & Phản hồi
        ↓
Đóng Ticket & Đánh giá
```

---

## 3. Nguyên tắc phát triển mở rộng (Extensibility Principles)

- **Không làm vỡ tính năng cũ:** Các Module sau chỉ tiêu thụ (consume) hoặc mở rộng (extend) dữ liệu từ Module trước.
- **Single Source of Truth:** Bảng `Tickets` là trung tâm. Trạng thái Ticket chỉ chuyển dịch qua lại theo Sơ đồ trạng thái chuẩn (State Machine), không ghi đè dữ liệu lịch sử.

---

# PHẦN II: TIÊU CHUẨN KỸ THUẬT VÀ QUY TẮC DỮ LIỆU (TECHNICAL RULES)

## 1. Quy tắc định danh và định dạng Dữ liệu (Data Conventions)

### Mã Ticket ID

Định dạng:

```text
TK-YYYYMMDD-XXXX
```

Ví dụ:

```text
TK-20261001-0042
```

Tự động tăng trong ngày.

### Quy định File đính kèm

- Định dạng cho phép: `.png`, `.jpg`, `.jpeg`, `.pdf`.
- Dung lượng tối đa: **5MB / tệp**.
- Tối đa **3 tệp / lần gửi**.

### Phân quyền dữ liệu (Data Isolation)

**Sinh viên** chỉ có quyền `READ` các Ticket do chính tài khoản đó tạo:

```text
created_by = current_user
```

**Nhân viên** chỉ có quyền `READ/WRITE` các Ticket thuộc Phòng ban của mình:

```text
department_id = user.department_id
```

---

## 2. Sơ đồ chuyển đổi Trạng thái Ticket (Ticket State Machine)

Mọi chức năng xử lý Ticket phải tuân thủ nghiêm ngặt các chuyển dịch trạng thái sau:

```text
[Chờ tiếp nhận] ──(Nhân viên nhận)──> [Đang xử lý] ──(Nội dung OK)──> [Hoàn thành]
       │                                      │
       │                                      ├──(Thiếu hs)──> [Chờ bổ sung]
       │                                      │                     │
       │                                      │             (SV gửi bổ sung)
       │                                      │                     │
       └──(Chuyển phòng)──> [Chờ tiếp nhận] <───────────────────────┘
```

**Auto-close Rule:** Ticket ở trạng thái **Chờ bổ sung** quá **7 ngày (168 giờ)** không có tương tác từ Sinh viên sẽ tự động chuyển sang **Đã đóng (Closed)** bởi **System Job**.

---

# PHẦN III: ĐẶC TẢ CHỨC NĂNG CHI TIẾT (MODULE BY MODULE)

# MODULE 1: NỀN TẢNG XÁC THỰC (AUTHENTICATION BASE)

Chức năng này làm nền tảng cho toàn bộ hệ thống.

## 1.1. Đăng nhập hệ thống (Login)

**User Stories:** Là người dùng (Sinh viên, Nhân viên, Quản lý), tôi muốn đăng nhập bằng Email và Mật khẩu do trường cấp để truy cập đúng không gian làm việc của mình.

### Input Validation

| Trường | Điều kiện |
|---|---|
| `email` | Required, đúng định dạng Email (`@aurora.edu.vn`) |
| `password` | Required, độ dài từ 6–32 ký tự |

### Acceptance Criteria (AC)

**AC 1.1:** Đăng nhập thành công trả về `JWT Token`, lưu Role người dùng và điều hướng đúng Màn hình chính:

- Sinh viên → Trang gửi đơn.
- Nhân viên → Queue phòng ban.
- Quản lý → Dashboard.

**AC 1.2:** Nhập sai Mật khẩu hoặc Email không tồn tại → Báo lỗi chung:

```text
"Thông tin đăng nhập không chính xác"
```

**AC 1.3:** Nhập sai mật khẩu **5 lần liên tiếp** → Tự động khóa tài khoản tạm thời **15 phút**.

---

## 1.2. Đổi mật khẩu (Change Password)

**User Stories:** Mọi người dùng sau khi đăng nhập có thể chủ động đổi mật khẩu cá nhân.

### Acceptance Criteria (AC)

**AC 2.1:** Yêu cầu nhập:

- Mật khẩu cũ.
- Mật khẩu mới.
- Xác nhận mật khẩu mới.

**AC 2.2:** Mật khẩu mới không được trùng Mật khẩu cũ.

---

# MODULE 2: PHÂN HỆ SINH VIÊN (STUDENT WORKSPACE)

Mở rộng trên Module 1: Sinh viên dùng Token xác thực để thực hiện các nghiệp vụ gửi đơn.

## 2.1. Tạo Ticket hỗ trợ mới (Create Ticket)

**User Stories:** Là Sinh viên, tôi muốn chọn phòng ban/danh mục và viết nội dung hỗ trợ để gửi lên hệ thống.

### Input Fields & Validation

| Trường dữ liệu | Kiểu dữ liệu | Bắt buộc | Điều kiện Validation / Ghi chú |
|---|---|---|---|
| `department_id` | Dropdown | Có | Chọn từ danh sách Phòng ban khả dụng |
| `category_id` | Dropdown | Có | Thuộc `department_id` đã chọn |
| `title` | Text Input | Có | Tối thiểu 10 ký tự, tối đa 150 ký tự |
| `description` | Textarea | Có | Tối thiểu 20 ký tự, tối đa 2000 ký tự |
| `attachments` | File Upload | Không | Tối đa 3 file, dung lượng 5MB/file |

### Acceptance Criteria (AC)

**AC 1.1:** Bấm **"Gửi yêu cầu"** → Tạo record trong DB với:

```text
status = 'PENDING'
```

(Chờ tiếp nhận).

**AC 1.2:** Hệ thống trả về Ticket ID trên màn hình và gửi Email xác nhận ngắn gọn về email sinh viên.

---

## 2.2. Theo dõi danh sách & Chi tiết Ticket (Ticket Tracking)

### Acceptance Criteria (AC)

**AC 2.1:** Trang danh sách hiển thị:

- Mã Ticket.
- Tiêu đề.
- Ngày tạo.
- Phòng ban phụ trách.
- Trạng thái (được gắn Badge màu tương ứng).

**AC 2.2:** Màn hình Chi tiết Ticket hiển thị luồng Timeline thời gian thực từ lúc tạo đến trạng thái hiện tại.

---

## 2.3. Bổ sung thông tin (Update Ticket Information)

Chức năng này **KHÔNG sửa dữ liệu cũ** mà ghi thêm phản hồi mới (**Append-only**).

**Điều kiện mở:** Ticket đang ở trạng thái:

```text
WAITING_FOR_STUDENT
```

(Chờ bổ sung).

### Acceptance Criteria (AC)

**AC 3.1:** Sinh viên nhập Nội dung bổ sung và/hoặc File đính kèm mới → Bấm **"Gửi bổ sung"**.

**AC 3.2:** Hệ thống cập nhật:

```text
status = 'PROCESSING'
```

(Đang xử lý) và đẩy thông báo cho Nhân viên phụ trách.

---

## 2.4. Đánh giá chất lượng dịch vụ (CSAT Rating)

**Điều kiện mở:** Ticket ở trạng thái:

```text
RESOLVED
```

(Hoàn thành).

### Acceptance Criteria (AC)

**AC 4.1:** Hiển thị Form Đánh giá:

- Chọn số sao từ **1 đến 5 sao** — bắt buộc.
- Nhận xét — không bắt buộc.

**AC 4.2:** Sau khi gửi đánh giá thành công, khóa Form (chỉ hiển thị dạng Read-only).

---

# MODULE 3: PHÂN HỆ NHÂN VIÊN (STAFF WORKSPACE)

Mở rộng trên Module 2: Nhân viên tiếp nhận các Ticket đã được Sinh viên khởi tạo.

## 3.1. Danh sách & Bộ lọc Queue phòng ban (Queue Management)

### Acceptance Criteria (AC)

**AC 1.1:** Nhân viên đăng nhập chỉ xem được các Ticket có:

```text
department_id = user.department_id
```

trùng với phòng ban của mình.

**AC 1.2:** Bộ lọc hỗ trợ:

- Lọc theo Trạng thái:
  - Chờ tiếp nhận.
  - Đang xử lý.
  - Chờ bổ sung.
- Lọc theo Khoảng thời gian.
- Tìm kiếm theo Mã Ticket/Tiêu đề.

---

## 3.2. Tiếp nhận Ticket (Claim Ticket)

**Condition:** Ticket có:

```text
status = 'PENDING'
```

### Acceptance Criteria (AC)

**AC 2.1:** Bấm **"Tiếp nhận"** → Cập nhật:

```text
assignee_id = current_user.id
status = 'PROCESSING'
```

**AC 2.2: Race Condition Handling:** Nếu 2 nhân viên bấm **"Tiếp nhận"** cùng millisecond, hệ thống dùng **DB Lock**.

Người bấm sau nhận thông báo:

```text
"Ticket đã được tiếp nhận bởi [Tên Nhân Viên 1]"
```

và tự động Refresh giao diện.

---

## 3.3. Yêu cầu Sinh viên bổ sung thông tin (Request Info)

**Condition:** Ticket có:

```text
status = 'PROCESSING'
```

và:

```text
assignee_id = current_user.id
```

### Acceptance Criteria (AC)

**AC 3.1:** Nhân viên nhập Lý do yêu cầu bổ sung → Chuyển:

```text
status = 'WAITING_FOR_STUDENT'
```

---

## 3.4. Chuyển tiếp Ticket sai thẩm quyền (Re-route Ticket)

### Acceptance Criteria (AC)

**AC 4.1:** Chọn Phòng ban mới + Nhập Lý do chuyển → Hệ thống cập nhật:

```text
department_id = new_dept_id
assignee_id = NULL
status = 'PENDING'
```

Ticket chuyển sang Queue của Phòng ban mới.

---

## 3.5. Hoàn thành Ticket (Resolve Ticket)

### Acceptance Criteria (AC)

**AC 5.1:** Nhân viên nhập:

- Nội dung giải quyết — Bắt buộc.
- File kết quả — Không bắt buộc.

Sau đó chuyển:

```text
status = 'RESOLVED'
```

---

# MODULE 4: PHÂN HỆ QUẢN LÝ (MANAGER WORKSPACE)

Mở rộng trên Module 1, 2, 3: Tổng hợp toàn bộ dữ liệu giao dịch để báo cáo và cấu hình.

## 4.1. Dashboard Báo cáo & Thống kê (Analytics Dashboard)

**Dữ liệu đầu vào:** Toàn bộ bảng `Tickets` và `Ratings` đã sinh ra ở Module 2 & 3.

### Acceptance Criteria (AC)

**AC 1.1:** Hiển thị **4 thẻ chỉ số tổng quan (KPI Cards):**

- Tổng số Ticket tiếp nhận (theo khoảng thời gian chọn).
- Tỷ lệ xử lý đúng hạn (%) - Xác định Ticket quá hạn khi xử lý **>48 giờ làm việc** mà chưa chuyển `RESOLVED`.
- Số lượng Ticket quá hạn hiện tại.
- Điểm đánh giá hài lòng trung bình (**CSAT Score từ 1.0 đến 5.0**).

**AC 1.2:** Biểu đồ xu hướng: Biểu đồ cột thể hiện số lượng Ticket theo từng Nhóm vấn đề.

---

## 4.2. Quản lý Tài khoản & Phân quyền (User Management)

### Acceptance Criteria (AC)

**AC 2.1:** Tạo mới / Chỉnh sửa tài khoản người dùng:

- Tên.
- Email.
- Chức vụ.
- Vai trò:
  - Student.
  - Staff.
  - Manager.
- Phòng ban.

**AC 2.2:** Vô hiệu hóa tài khoản (**Soft Delete / Deactivate**) → Tài khoản không thể đăng nhập, nhưng dữ liệu lịch sử Ticket cũ vẫn giữ nguyên.

---

## 4.3. Quản lý Danh mục Hỗ trợ (Category Management)

### Acceptance Criteria (AC)

**AC 3.1:** Thêm mới / Sửa tên / Ẩn (**Deactivate**) các Nhóm vấn đề thuộc từng Phòng ban.

**AC 3.2:** Khi một Nhóm vấn đề bị Ẩn:

- Nó không xuất hiện ở Dropdown chọn của Sinh viên khi tạo đơn mới.
- Toàn bộ Ticket cũ thuộc Nhóm này vẫn hiển thị bình thường ở màn hình tra cứu/báo cáo.
