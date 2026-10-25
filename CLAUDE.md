# Smart Inventory & Asset Tracking

Ứng dụng quản lý đồ đạc / tài sản cá nhân: cây vị trí lưu trữ (Room → Cabinet → Shelf → Box → Drawer), vật phẩm có thông số động, ngưỡng tồn kho riêng, mượn/trả, quét QR/barcode, cảnh báo hết hạn/bảo hành, báo cáo.

## Cấu trúc repo

| Thư mục | Nội dung | Rule chi tiết |
| :--- | :--- | :--- |
| `BE_GoLang/` | REST API Go + Gin + GORM + PostgreSQL, chạy HTTP server hoặc AWS Lambda | [BE_GoLang/CLAUDE.md](BE_GoLang/CLAUDE.md) |
| `FE_Flutter/` | App mobile Flutter (iOS/Android), Provider state management | [FE_Flutter/CLAUDE.md](FE_Flutter/CLAUDE.md) |

- `BE_GoLang/` là **git repo riêng** (có `.git` lồng bên trong). Commit backend phải chạy git trong `BE_GoLang/`, không phải ở root.
- Nguồn đặc tả API chuẩn: [BE_GoLang/AGENTS.md](BE_GoLang/AGENTS.md) (schema, endpoints, business rules). `FE_Flutter/AGENtS.md` là bản cũ hơn, đã lỗi thời — không dùng làm nguồn chân lý. `FE_Flutter/openapi.yaml` là OpenAPI spec tham khảo.
- Khi spec và code lệch nhau, **code là chân lý**; cập nhật spec nếu thay đổi có chủ đích.

## Hợp đồng API giữa BE và FE (bắt buộc giữ đồng bộ)

- Base path `/api/v1`, auth bằng `Authorization: Bearer <JWT>`. Chỉ `/auth/register|login|google` và `/health` là public.
- Mọi response dùng envelope:
  - Thành công: `{ "success": true, "data": ..., "message": "...", "meta": {page, limit, total} }`
  - Lỗi: `{ "success": false, "error": { "code": "UPPER_SNAKE", "message": "...", "details": [] } }`
  - FE (`ApiService._handleResponse`) unwrap `data` và ném `ApiException(code, message)` từ `error` — đổi envelope là vỡ toàn bộ app.
- JSON field dùng **camelCase** (`boxId`, `minQuantity`, `customAttributes`); cột DB dùng snake_case. Ngoại lệ: `POST /auth/google` nhận cả `avatar_url`/`avatarUrl` và `id_token`.
- ID là UUID v4 dạng string.
- Thêm/đổi field hoặc endpoint: sửa **cả hai phía** trong cùng một thay đổi (Go model/DTO + `fromJson`/`toJson` trong `FE_Flutter/lib/models/inventory_models.dart` + method trong `FE_Flutter/lib/services/api_service.dart`), rồi cập nhật `BE_GoLang/AGENTS.md`.
- Không xóa các alias tương thích ngược trong `GET /alerts/summary` (`totalAlertCount`, `overdueLoansCount`, `warrantyEndingCount`) — FE đang đọc chúng.

## Business rules cốt lõi

- **Low stock:** `minQuantity != null && quantity + lentQuantity <= minQuantity` (`lentQuantity` = số đơn vị đang cho mượn một phần, đã trừ khỏi `quantity` nhưng vẫn thuộc về người dùng). `minQuantity = null` nghĩa là không theo dõi.
- **Item status:** `stored | inUse | lent | broken | disposed`. **Box type:** `room | cabinet | shelf | box | drawer | other`.
- **Chống vòng lặp cây box:** khi move, parent mới không được là chính nó hoặc con cháu của nó; `newParentId = null` → đưa về root.
- **Mượn/trả:** chỉ cho mượn item đang `stored`/`inUse`; lend → status `lent`; return → `actual_return_date = NOW()` và status về `stored`.
- **Scan lookup:** khớp `boxes.label` trước, sau đó `items.barcode` / `serial_number` / `id`; không thấy → 404 `NOT_FOUND`.
- **Đa người dùng:** mọi dữ liệu thuộc về một `user_id`; không bao giờ trả hoặc sửa dữ liệu của user khác.

## Chạy cả hệ thống ở local

```bash
cd BE_GoLang && make db-up && make run          # API tại http://localhost:8080
cd FE_Flutter && flutter run --dart-define=API_BASE_URL=http://localhost:8080
```
Android emulator tự dùng `http://10.0.2.2:8080`. iPhone thật: `./run_ios_release.sh` (tự dò IP LAN của Mac).

## Quy ước làm việc

- Tài liệu và trao đổi bằng tiếng Việt; tên định danh trong code, commit message bằng tiếng Anh theo Conventional Commits (`feat:`, `fix(box):`...), giống lịch sử commit hiện có.
- Push lên `main` của `BE_GoLang` sẽ **tự deploy lên AWS Lambda production** (GitHub Actions) — chỉ push khi người dùng yêu cầu rõ.
- Không commit `.env`, secret, file trong `BE_GoLang/uploads/`, `FE_Flutter/build/`, `.DS_Store`.
