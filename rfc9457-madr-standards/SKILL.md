---
name: rfc9457-madr-standards
description: >-
  Mandatory skill for designing, documenting, and implementing HTTP API error handling (RFC 9457 Problem Details - obsoletes RFC 7807),
  Architectural Decision Records (MADR 3.0), and technical architecture or API specifications.
  Auto-triggers whenever working with: RFC 9457, RFC 7807, Problem Details, API error format, error response,
  application/problem+json, MADR, ADR, Architectural Decision Records, architecture documentation, API tech spec,
  API specification, C4 model, system structure, Mermaid diagrams, API design, idempotency key.
  Tự động kích hoạt khi prompt tiếng Việt liên quan đến: chuẩn RFC 9457, RFC 7807, định dạng lỗi API, Problem Details,
  viết ADR, tài liệu quyết định kiến trúc MADR, Markdown Architectural Decision Records, viết tài liệu kỹ thuật API,
  tech spec, tài liệu cấu trúc hệ thống, kiến trúc phần mềm, mô hình C4 model, sơ đồ Mermaid, thiết kế API RESTful,
  idempotency key, tài liệu kiến trúc, đặc tả API, schema lỗi API.
  Agent MUST follow these standards when designing APIs or authoring architecture/tech docs.
---

# RFC 9457 & MADR Architecture Standards

Quy chuẩn bắt buộc cho mọi **đặc tả lỗi HTTP API (RFC 9457 Problem Details - thay thế và kế thừa RFC 7807)**, **ghi nhận quyết định kiến trúc (MADR 3.0 - Markdown Architectural Decision Records)** và **tài liệu kỹ thuật hệ thống / API** được sinh ra hoặc chỉnh sửa bởi AI Agent trên mọi nền tảng kỹ thuật.

Áp dụng mô hình **Progressive Disclosure**: file `SKILL.md` này tóm tắt nguyên tắc cốt lõi, mẫu nhanh và checklist nghiệm thu. Hướng dẫn chi tiết kèm ví dụ đa ngôn ngữ nằm trong thư mục [references/](./references/).

---

## 📚 Danh Mục Quy Chuẩn & Tài Liệu Chi Tiết

| Phần | Trụ Cột Quy Chuẩn | Phạm Vi Áp Dụng | Tài Liệu Chi Tiết |
| :--- | :--- | :--- | :--- |
| **I** | **Định Dạng Lỗi HTTP API (RFC 9457)** | 5 trường chuẩn (`type`, `title`, `status`, `detail`, `instance`), `traceId`, `invalidParams` | [01-rfc9457-problem-details.md](./references/01-rfc9457-problem-details.md) |
| **II** | **Quyết Định Kiến Trúc MADR (ADR 3.0)** | Mẫu MADR 3.0, Context, Drivers, Options, Outcome, Pros & Cons, Đánh số & Vòng đời | [02-madr-architectural-decision-records.md](./references/02-madr-architectural-decision-records.md) |
| **III** | **Đặc Tả Kỹ Thuật API (API Tech Spec)** | RESTful modeling, Headers, Request/Response Schema, Idempotency-Key, Phân trang | [03-api-technical-documentation.md](./references/03-api-technical-documentation.md) |
| **IV** | **Tài Liệu Cấu Trúc Hệ Thống (Structure)** | Mô hình C4 (Context, Container, Component), Sơ đồ Mermaid (Sequence, ERD) | [04-system-structure-documentation.md](./references/04-system-structure-documentation.md) |

---

## ⚡ Cheatsheet: Nguyên Tắc Cốt Lõi Cần Nhớ

### 1. Chuẩn Định Dạng Lỗi RFC 9457 (Problem Details)
- ✅ **Header bắt buộc**: `Content-Type: application/problem+json`.
- ✅ **5 trường tiêu chuẩn**:
  - `type`: URI phân loại lỗi (mặc định `"about:blank"` nếu chưa có tài liệu riêng).
  - `title`: Tóm tắt ngắn gọn loại lỗi (cố định cho mọi lỗi cùng `type`, viết cho người đọc).
  - `status`: Mã HTTP Status Code số nguyên (`400`, `401`, `403`, `404`, `409`, `422`, `500`), trùng khớp 100% với HTTP Status của gói tin.
  - `detail`: Mô tả cụ thể nguyên nhân của lần phát sinh lỗi hiện tại.
  - `instance`: URI ngữ cảnh / tài nguyên phát sinh lỗi (ví dụ: `"/api/v1/orders/123"`).
- ✅ **Trường mở rộng bắt buộc**: `traceId` (W3C Trace Context) để đối soát log phân tán, `timestamp` (ISO 8601 UTC), `errorCode` (Machine code).
- ✅ **Validation Error**: Chứa mảng `invalidParams` gồm `{ name, reason, code }`.
- ❌ **CẤM**: Trả HTTP 200 kèm `{ "isError": true }`, cấm lộ StackTrace ra client, cấm dùng cấu trúc tự chế tùy tiện.

### 2. Chuẩn Ghi Nhận Quyết Định Kiến Trúc MADR 3.0
- ✅ **Vị trí lưu trữ**: Lưu trong repository tại `docs/decisions/` (hoặc `docs/adr/`).
- ✅ **Quy ước đặt tên**: `NNNN-short-title-in-kebab-case.md` (ví dụ: `0001-use-postgresql-for-database.md`).
- ✅ **Khung sườn chuẩn MADR 3.0**:
  1. Title & Metadata (Status: `proposed | accepted | rejected | deprecated | superseded by ADR-XXXX`, Deciders, Date).
  2. Context and Problem Statement (Bối cảnh và vấn đề cần giải quyết).
  3. Decision Drivers (Yếu tố thúc đẩy quyết định: hiệu năng, bảo mật, thời gian giao hàng...).
  4. Considered Options (Các phương án cân nhắc 1, 2, 3).
  5. Decision Outcome (Phương án chọn và lý do cốt lõi).
  6. Positive & Negative Consequences (Hệ quả tích cực & Đánh đổi / rủi ro).
  7. Pros and Cons of each Option (Ưu và nhược điểm chi tiết từng phương án).
  8. Validation / Confirmation (Kế hoạch kiểm chứng thực tế).
- ❌ **CẤM**: Xóa bỏ các quyết định cũ khi đổi kiến trúc. Phải đánh dấu `superseded by [ADR-XXXX]` để bảo toàn lịch sử.

### 3. Quy Chuẩn Tài Liệu Kỹ Thuật API
- ✅ URL theo chuẩn RESTful: Danh từ số nhiều, chữ thường, `kebab-case` (`/api/v1/user-profiles`).
- ✅ Thao tác nhạy cảm (Tạo đơn, Thanh toán): Bắt buộc yêu cầu header `Idempotency-Key: <UUID>`.
- ✅ Phản hồi thành công: `201 Created` kèm header `Location`, `204 No Content` cho DELETE.
- ✅ Phân trang: Chuẩn hóa theo Offset (`pageIndex`, `pageSize`) hoặc Cursor (`cursor`, `limit`).

### 4. Quy Chuẩn Cấu Trúc Hệ Thống (C4 & Mermaid)
- ✅ Trực quan hóa kiến trúc theo 4 cấp độ của **C4 Model**:
  - Level 1: System Context (Ai dùng hệ thống, tích hợp hệ thống bên thứ 3 nào).
  - Level 2: Container (Web SPA, Mobile App, API Gateway, Microservices, DB, Message Queue).
  - Level 3: Component (Clean Architecture: API, Application, Domain, Infrastructure).
  - Level 4: Code (Sơ đồ lớp cho luồng nghiệp vụ phức tạp).
- ✅ Dùng mã **Mermaid chuẩn** (`flowchart TD`, `sequenceDiagram`, `erDiagram`) để vẽ sơ đồ trực tiếp trong Markdown.
- ✅ Liên kết chéo mọi công nghệ/mô hình trên sơ đồ tới các file ADR tương ứng.

---

## 📋 Checklist Nghiệm Thu (Audit Checklist)

- [ ] Phản hồi lỗi API có header `Content-Type: application/problem+json` không?
- [ ] Lỗi có đầy đủ 5 trường chuẩn `type`, `title`, `status`, `detail`, `instance` không?
- [ ] Mã `status` trong body JSON có trùng khớp 100% với HTTP status code của gói tin không?
- [ ] Có đính kèm trường `traceId` và `timestamp` cho truy vết phân tán không?
- [ ] Lỗi nghiệp vụ 500 có bị lộ StackTrace kỹ thuật của server ra ngoài không?
- [ ] File ADR có lưu trong `docs/decisions/` và đặt tên đúng định dạng `NNNN-kebab-case.md` không?
- [ ] File ADR có đầy đủ các mục: Context, Drivers, Options, Outcome, Pros & Cons không?
- [ ] Khi thay thế quyết định cũ, có cập nhật status thành `superseded by [ADR-XXXX]` không?
- [ ] API Tech Spec có định nghĩa rõ Request/Response schema và ma trận lỗi RFC 9457 không?
- [ ] Thao tác thanh toán/tạo đơn có đặc tả header `Idempotency-Key` không?
- [ ] Tài liệu kiến trúc có vẽ sơ đồ C4 Model và luồng nghiệp vụ bằng Mermaid không?
