# Kế hoạch 6 tuần triển khai hệ thống quản lý công việc nhóm realtime (Quarkus + WebSocket + JWT)

## 1) Mục tiêu dự án trong 6 tuần

Xây dựng MVP có thể demo nội bộ với các năng lực chính:

- Đăng nhập/đăng ký bằng JWT
- Quản lý project và thành viên theo vai trò
- Quản lý Task theo Kanban (`TODO/DOING/DONE`)
- Kéo thả task giữa cột + sắp xếp trong cột
- Đồng bộ realtime qua WebSocket theo từng project
- Đảm bảo an toàn cập nhật đồng thời (optimistic locking)

---

## 2) Phạm vi kỹ thuật

### Backend (Quarkus)
- REST API cho nghiệp vụ chính
- WebSocket để broadcast thay đổi realtime
- JPA/Panache + PostgreSQL/MySQL
- JWT security (`quarkus-smallrye-jwt`)
- Validation + Global exception handler
- OpenAPI + Health check

### Frontend (HTML/CSS/JS)
- Trang `login`, `dashboard`, `board`
- Module gọi API + module WebSocket
- Render Kanban + drag/drop
- Đồng bộ UI theo event realtime

---

## 3) Kế hoạch theo tuần

## Tuần 1 — Khởi tạo nền tảng + Authentication JWT

### Mục tiêu
Dự án chạy được end-to-end cho luồng auth cơ bản.

### Công việc chính
- Khởi tạo/cấu hình project Quarkus, dependencies cần thiết
- Tạo cấu trúc thư mục theo tầng kỹ thuật:
  - `config`, `controller`, `service`, `repository`, `entity`, `dto`, `mapper`, `exception`, `websocket`
- Hoàn thiện entity nền tảng:
  - `User`, `Project`, `ProjectMember`, `Task`, `Comment`, `BaseEntity`, enums
- Làm API auth:
  - `POST /auth/register`
  - `POST /auth/login`
- Hash mật khẩu bằng BCrypt
- Sinh JWT access token (và refresh token nếu triển khai ngay tuần 1)
- Cấu hình endpoint public/protected

### Đầu ra tuần 1
- User có thể đăng ký/đăng nhập và nhận JWT
- API protected xác thực được bằng Bearer token
- Có tài liệu API cơ bản trên Swagger

## Tuần 2 — Project + Membership + Authorization

### Mục tiêu
Hoàn chỉnh quản lý project và kiểm soát quyền theo thành viên.

### Công việc chính
- API project:
  - Tạo project
  - Lấy danh sách project của user
  - Lấy chi tiết project
  - Cập nhật/xóa project (theo quyền)
- API thành viên:
  - Thêm thành viên vào project
  - Cập nhật vai trò `ADMIN/MEMBER/VIEWER`
  - Xóa thành viên
- Rule phân quyền:
  - `VIEWER`: chỉ xem
  - `MEMBER`: thao tác task
  - `ADMIN`: quản trị project/member
- Chuẩn hóa xử lý lỗi:
  - `401`, `403`, `404`, `409`

### Đầu ra tuần 2
- Luồng “tạo project -> thêm member -> phân quyền” hoạt động
- Rule quyền được kiểm tra đầy đủ ở service layer

## Tuần 3 — Task Kanban + Optimistic Locking

### Mục tiêu
Task CRUD ổn định, hỗ trợ Kanban và chống ghi đè dữ liệu.

### Công việc chính
- API Task CRUD theo project
- Trường nghiệp vụ:
  - `status`, `priority`, `dueDate`, `assigneeId`, `position`, `version`
- API chuyên biệt cho board:
  - Move task giữa cột (`TODO/DOING/DONE`)
  - Reorder task trong cột
- Thiết kế `position/order` (chiến lược số thưa: `1000, 2000, 3000...`)
- Bật optimistic locking qua `version`
- Trả lỗi conflict khi update đồng thời

### Đầu ra tuần 3
- Board state trả về đúng cho frontend
- Có thể move/reorder task bằng API
- Concurrent update được xử lý an toàn (trả `409`)

## Tuần 4 — Realtime WebSocket theo room project

### Mục tiêu
Đồng bộ realtime đa người dùng trong cùng project.

### Công việc chính
- Xây dựng WebSocket endpoint
- Quản lý room theo `project:{projectId}`
- Chuẩn hóa event:
  - `TASK_CREATED`
  - `TASK_UPDATED`
  - `TASK_DELETED`
  - `TASK_MOVED`
  - `TASK_REORDERED`
- Chuẩn payload:
  - `eventId`, `type`, `projectId`, `taskId`, `actorUserId`, `timestamp`, `data`
- Luồng backend:
  - REST ghi DB thành công -> phát event -> WS broadcast room
- Bảo mật WS:
  - Xác thực JWT khi kết nối/join room
  - Verify user thuộc project

### Đầu ra tuần 4
- 2+ user mở cùng board thấy thay đổi realtime ngay khi có thao tác

## Tuần 5 — Frontend hoàn chỉnh luồng nghiệp vụ

### Mục tiêu
Hoàn chỉnh giao diện và tương tác chính cho demo.

### Công việc chính
- `login.html`: đăng nhập, lưu token
- `dashboard.html`: danh sách project, điều hướng board
- `board.html`: Kanban 3 cột, modal task
- Tách module JS:
  - `js/api/*` gọi REST
  - `js/websocket/board-socket.js` realtime
  - `js/board/*` render + drag/drop + modal
- Luồng board:
  - Load state bằng REST khi mở trang
  - Kết nối WS và join room project
  - Khi kéo thả: gọi API move/reorder
  - Nhận event WS và cập nhật UI

### Đầu ra tuần 5
- Demo được luồng đầy đủ: login -> vào board -> CRUD + kéo thả + realtime

## Tuần 6 — Hardening + Test + Release nội bộ

### Mục tiêu
Ổn định chất lượng, bảo mật và tài liệu hóa để bàn giao/demo.

### Công việc chính
- Unit test cho service quan trọng
- Integration test cho auth/project/task
- Test case phân quyền và concurrent update
- Bổ sung:
  - Health check
  - Logging chuẩn
  - OpenAPI hoàn thiện
- Rà soát bảo mật JWT:
  - Expiry policy
  - Secret/key management
  - (Nếu có refresh token) rotation/revocation strategy
- Hoàn thiện tài liệu:
  - README chạy local
  - API usage
  - Kịch bản demo

### Đầu ra tuần 6
- Bản v1 nội bộ sẵn sàng review/demo
- Có test cơ bản và tài liệu vận hành

---

## 4) Milestone kiểm tra cuối mỗi tuần

- **Cuối tuần 1:** Auth JWT chạy ổn định
- **Cuối tuần 2:** Project + membership + quyền hoàn chỉnh
- **Cuối tuần 3:** Task board API hoàn chỉnh + chống conflict
- **Cuối tuần 4:** Realtime hoạt động theo room project
- **Cuối tuần 5:** Frontend MVP hoàn thiện end-to-end
- **Cuối tuần 6:** Test + hardening + release nội bộ

---

## 5) Rủi ro và cách giảm rủi ro

- Rủi ro chậm tiến độ realtime -> ưu tiên REST đúng trước, WS broadcast sau
- Rủi ro bug phân quyền -> tập trung test service-layer theo ma trận role
- Rủi ro conflict dữ liệu kéo thả -> bắt buộc dùng `version` + API trả `409`
- Rủi ro bảo mật JWT -> token ngắn hạn + refresh/rotation phù hợp
- Rủi ro frontend rối logic -> tách rõ `api`, `websocket`, `board`

---

## 6) Checklist hoàn thành dự án (Definition of Done)

- [ ] Đăng nhập JWT + gọi API protected thành công
- [ ] Quản lý project và thành viên theo role
- [ ] CRUD task đầy đủ
- [ ] Kanban 3 cột + drag/drop + reorder
- [ ] Realtime đồng bộ đa người dùng theo project
- [ ] Xử lý concurrent update an toàn
- [ ] Có test cốt lõi + tài liệu README/API/demo

---

## 7) Backlog ưu tiên khi thiếu thời gian

1. Auth JWT + Project + Membership + Task CRUD
2. Kanban + move/reorder
3. WebSocket realtime
4. Comment, tối ưu UI, tính năng phụ

Nếu cần cắt scope để kịp 6 tuần, giữ đúng thứ tự trên.
