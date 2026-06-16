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

## Bước 0.5 — LÀM MỚI dữ liệu trước khi report (Pha 2)

Kiểm tra độ mới: `python3 tools/jira-to-obsidian/import_jira.py --check-fresh` (Windows `py`) → JSON
`{last_import, is_stale, age_days, done_today}`. **`done_today:true` & `is_stale:false` → BỎ QUA làm mới**
(dữ liệu đủ mới), sang Bước 1. Ngược lại, làm mới theo LOẠI Jira (đọc `JIRA_BASE_URL`/config):

**A) Jira Cloud (`*.atlassian.net`, có MCP Atlassian):** tự kéo qua MCP → nạp vào vault → report:
1. `since` = `last_import` (chưa có → kéo full).
2. MCP `searchJiraIssuesUsingJql`: `project = <KEY> AND updated >= "<since>"` (hoặc `project=<KEY>` nếu
   full), `fields:["*all"]`. Kết quả lớn MCP **tự lưu ra file** → dùng path đó; nhỏ (inline) → ghi ra
   `reports/_mcp-pull.json`. (KHÔNG nạp cả khối vào ngữ cảnh — xử lý qua file.)
3. Lấy map tên field 1 lần: `getJiraIssue` 1 issue `expand=names` → ghi `{id:name}` ra `reports/_mcp-names.json`.
4. Nạp vào vault (tái dùng toàn bộ logic ghi note): `python3 tools/jira-to-obsidian/import_jira.py
   --from-mcp <file> --names reports/_mcp-names.json --since` (cờ `--since` để bật idempotent-per-day).
5. Reindex: `python3 tools/kb-indexer/build_index.py --root .`.
> Phiên scheduled nền **thiếu MCP** → coi như không kéo được → xử như nhánh "cũ" của B (báo cũ + nhắc mở Cowork gõ "báo cáo tiến độ").

**B) Jira self-host (token, MCP/nền KHÔNG tới host nội bộ):** KHÔNG tự kéo.
- `is_stale:false` → report bình thường (Bước 1).
- `is_stale:true` → **vẫn sinh report (dữ liệu CŨ, có banner)** ở Bước 1, RỒI in **lệnh terminal copy-paste**
  điền sẵn đường dẫn thật (OS-dynamic) để user tự kéo:
  `python3 "<TOOL_DIR>/import_jira.py" --since` (Windows `py "<TOOL_DIR>\import_jira.py" --since`).
  Nhắc: "Chạy lệnh trên để cập nhật, rồi gõ **'báo cáo tiến độ'** lại → báo cáo mới." User kéo xong → chạy lại workflow.

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
