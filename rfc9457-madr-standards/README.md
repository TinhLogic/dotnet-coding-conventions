# RFC 9457 & MADR Architecture Standards

[![RFC 9457](https://img.shields.io/badge/IETF-RFC%209457%20(Obsoletes%207807)-blue.svg)](https://datatracker.ietf.org/doc/html/rfc9457)
[![MADR 3.0](https://img.shields.io/badge/Architecture-MADR%203.0-green.svg)](https://adr.github.io/madr/)
[![OpenAPI 3.1](https://img.shields.io/badge/API-OpenAPI%203.1-orange.svg)](https://www.openapis.org/)
[![C4 Model](https://img.shields.io/badge/Architecture-C4%20Model-purple.svg)](https://c4model.com/)

> **Ngôn ngữ / Languages**: [Tiếng Việt](#ti%E1%BA%BFng-vi%E1%BB%87t) | [English](#english)

---

# Tiếng Việt

Bộ quy chuẩn kỹ thuật toàn diện về **định dạng lỗi API chuẩn quốc tế RFC 9457 (Problem Details for HTTP APIs - thay thế RFC 7807)**, **ghi nhận quyết định kiến trúc theo chuẩn MADR 3.0 (Markdown Architectural Decision Records)** và **phương pháp viết tài liệu kỹ thuật API & cấu trúc hệ thống (C4 Model & Mermaid)**.

## 🎯 4 Trụ Cột Kỹ Thuật

| Phần | Trụ Cột Quy Chuẩn | Phạm Vi Áp Dụng | Tài Liệu Chi Tiết |
| :--- | :--- | :--- | :--- |
| **I** | **Định Dạng Lỗi HTTP API (RFC 9457)** | 5 trường chuẩn (`type`, `title`, `status`, `detail`, `instance`), `traceId`, `invalidParams` | [`01-rfc9457-problem-details.md`](./references/01-rfc9457-problem-details.md) |
| **II** | **Quyết Định Kiến Trúc MADR (ADR 3.0)** | Mẫu MADR 3.0, Context, Drivers, Options, Outcome, Pros & Cons, Đánh số & Vòng đời | [`02-madr-architectural-decision-records.md`](./references/02-madr-architectural-decision-records.md) |
| **III** | **Đặc Tả Kỹ Thuật API (API Tech Spec)** | RESTful modeling, Headers, Request/Response Schema, Idempotency-Key, Phân trang | [`03-api-technical-documentation.md`](./references/03-api-technical-documentation.md) |
| **IV** | **Tài Liệu Cấu Trúc Hệ Thống (Structure)** | Mô hình C4 (Context, Container, Component), Sơ đồ Mermaid (Sequence, ERD) | [`04-system-structure-documentation.md`](./references/04-system-structure-documentation.md) |

Bản tóm tắt, cheatsheet và checklist nghiệm thu nhanh nằm tại [`SKILL.md`](./SKILL.md).

---

# English

Comprehensive engineering standards for **RFC 9457 Problem Details for HTTP APIs (obsoletes RFC 7807)**, **MADR 3.0 (Markdown Architectural Decision Records)**, and **API & System Structure Technical Documentation (C4 Model & Mermaid)**.

## 🎯 4 Core Architectural Pillars

1. **RFC 9457 Problem Details**: Standardized HTTP API error payload with mandatory 5 members (`type`, `title`, `status`, `detail`, `instance`) and enterprise extensions (`traceId`, `timestamp`, `errorCode`, `invalidParams`).
2. **MADR 3.0 Architecture Decision Records**: Transparent decision logging with structured Context, Decision Drivers, Considered Options, Outcome, Positive/Negative Consequences, and Pros/Cons analysis.
3. **API Technical Specifications**: RESTful resource modeling, request/response contracts, authorization matrices, distributed `Idempotency-Key` implementation, and cursor/offset pagination.
4. **System Structure Documentation**: C4 Model visual hierarchy (Context, Container, Component, Code), Mermaid sequence flows, entity-relationship diagrams, and cross-linking with ADRs.
