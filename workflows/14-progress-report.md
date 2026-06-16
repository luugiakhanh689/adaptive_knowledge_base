# Workflow 14 — Báo cáo tiến độ dự án (local, no-server)

> Trigger: "báo cáo tiến độ", "report tiến độ", "tiến độ dự án", "sinh báo cáo" (confirm ý định trước).
> Cũng được Pha 2 (lịch 8:00, `workflows/08-schedule-sync.md`) gọi TỰ ĐỘNG sau khi quét Jira thành công.
>
> Sinh report từ **dữ liệu vault đã quét** (không server, không đẩy đi đâu). Nhấn mạnh: **thời gian
> (ước tính/đã log/còn lại), sprint đang chạy (active), người phụ trách (assignee)**.

## Bước 0 — Kiểm tra dữ liệu

- Đọc `vault_path` từ `config/factory-config.yaml`. Vault chưa có note Jira (`source: jira`) →
  báo nhẹ + gợi ý **`quét jira`** trước (workflow 01). KHÔNG sinh report rỗng.
- Khuyến nghị: nếu vault quét bằng bản < v1.1.0 (thiếu `time_*_s` / `sprint_state`) → nhắc **quét lại**
  để có đủ số liệu thời gian/sprint.

## Bước 1 — Sinh số liệu + dashboard

Chạy (Claude tự chạy trong sandbox; user chạy tay thì OS-dynamic — Windows `py`):

```bash
python3 tools/progress-report/build_report.py
```

Tạo trong `reports/`:
- `progress-data-<ngày>.json` — số liệu thô (nguồn cho UI inline).
- `progress-report-<ngày>.html` + `progress-report-latest.html` — **dashboard standalone** (mở bằng
  trình duyệt, chia sẻ được; phong cách tối glass như landing).

## Bước 2 — Hiển thị UI trong Cowork (inline)

Đọc `reports/progress-data-<ngày>.json` → **render dashboard NGAY trong chat** bằng `visualize`:
1. Gọi `mcp__visualize__read_me` (modules: `chart`) — nạp guideline 1 lần.
2. `mcp__visualize__show_widget` với một dashboard **TUÂN guideline visualize** (KHÔNG dùng nền tối/
   màu cứng của file standalone): nền trong suốt + biến CSS `--color-*`, icon Tabler (không emoji),
   số làm tròn. Gồm:
   - **Thẻ metric:** Tổng issue · % hoàn thành · Đã log/Ước tính · Còn lại · Sprint active.
   - **Donut trạng thái** (Done / Đang làm / Chưa làm) + legend HTML.
   - **Bar ngang theo assignee:** giờ Đã log vs Ước tính.
   - (Bảng chi tiết issue sprint/assignee → in dạng **markdown trong câu trả lời**, KHÔNG nhồi vào widget.)
3. Kèm tóm tắt text: tiến độ sprint active, top assignee theo tải, % burn time, danh sách rủi ro
   (quá hạn / active-sprint thiếu assignee/ước tính).

## Bước 3 — Báo file + bước kế

- Báo đường dẫn `reports/progress-report-latest.html` (mở bằng trình duyệt / gửi cho sếp).
- **Đề xuất bước kế (AskUserQuestion):** `[A] Đặt lịch 8:00 tự động pull→report (workflows/08) ·
  [B] Quét Jira lấy dữ liệu mới (workflows/01) · [C] Phân loại issue thành tri thức (workflows/03) · [D] Dừng`.

## Guardrails
- KHÔNG đẩy dữ liệu ra ngoài (local-only). KHÔNG ghi vào `docs/` KB chính — report là artifact ở `reports/`.
- `reports/` là DATA (gitignore + giữ khi update) — không commit báo cáo của user.
- Thiếu số liệu (issue thiếu time/sprint) → report vẫn chạy, nêu rõ "X issue thiếu dữ liệu", không bịa.
