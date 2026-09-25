# L3A Architecture Record

Tài liệu thiết kế kiến trúc hệ thống Multi-Agent điều tra khiếu nại thương mại điện tử (K4 L3A — Multi-Agent MCP + A2A).

## 1. System overview

Quy trình xử lý từ file đầu vào `inputs/<case_id>.json` qua các Agent và MCP Gateway đến file output `outputs/<case_id>.json` và observable trace `traces/trace.jsonl`:

```text
[Input Case] 
     │
     ▼
[Coordinator Agent] 
     ├── emit: task_assigned ──► [Specialist Agents]
     │                                ├── Order/Item Agent  ──► MCP Gateway (get_order, get_order_items, get_sellers)
     │                                ├── Payment Agent     ──► MCP Gateway (get_order_payments, get_payment_timeline, get_refund_timeline)
     │                                ├── Shipment Agent    ──► MCP Gateway (get_shipment_summary)
     │                                └── Policy Agent      ──► MCP Gateway (get_policy)
     │                                         │
     │                           (emit: tool_result_consumed, handoff)
     │                                         │
     ▼                                         ▼
[Verifier Agent] ◄─────────────────────────────┘
     │
     ├── Reconcile entities, cross-check claims vs authoritative evidence
     ├── Verify policy resolution, financial lines & responsible parties
     ├── Check consistency & validate JSON Schema
     ├── emit: verification_completed
     ▼
[Validated Output] ──► outputs/<case_id>.json
[Observable Traces] ──► traces/trace.jsonl (case_finalized)
```

## 2. Agent ownership

| Actor | Input | Trách nhiệm | MCP Tools được cấp quyền | Output / Handoff |
| --- | --- | --- | --- | --- |
| `coordinator` | `case` object từ `inputs/<case_id>.json` | Tiếp nhận case, phân tích scope khiếu nại sơ bộ, định tuyến phân rã nhiệm vụ cho các Specialist | Không gọi trực tiếp MCP tool | Giao việc tới các specialist qua sự kiện `task_assigned` |
| `order_agent` | `case_id`, `order_id` | Điều tra thực thể đơn hàng: trạng thái đơn hàng (`canceled`, `unavailable`, `delivered`), danh sách sản phẩm, giá bán, người bán | `get_order`, `get_order_items`, `get_sellers` | `ORDER_EVIDENCE_READY` bàn giao cho `verifier` |
| `payment_agent` | `case_id`, `order_id` | Điều tra tài chính: các đợt thanh toán, hình thức thanh toán (thẻ, voucher), sự kiện trừ tiền, lịch sử hoàn tiền | `get_order_payments`, `get_payment_timeline`, `get_refund_timeline` | `PAYMENT_EVIDENCE_READY` bàn giao cho `verifier` |
| `shipment_agent` | `case_id`, `order_id` | Điều tra logistics: timeline giao vận, hạn chót giao hàng của người bán, xác định trách nhiệm giao trễ giữa Seller và Đơn vị vận chuyển | `get_shipment_summary` | `SHIPMENT_EVIDENCE_READY` bàn giao cho `verifier` |
| `policy_agent` | `case_id`, `policy_version` | Tra cứu điều khoản chính sách có thẩm quyền cho phiên bản áp dụng, xác định ma trận bồi hoàn và chế tài tương ứng | `get_policy` | Phát sự kiện `policy_decided`, bàn giao `POLICY_EVIDENCE_READY` cho `verifier` |
| `verifier` | Dữ liệu tổng hợp từ các Specialist Agents và Policy | Đối soát chéo tính logic (Consistency), xác định nguyên nhân gốc rễ, tính toán số tiền hoàn BRL, kiểm tra tính toàn vẹn Schema | Không gọi trực tiếp MCP tool | Phát `verification_completed` và tạo output dict hoàn chỉnh |

## 3. A2A protocol

- **Message Correlation & Scope:** Mỗi tương tác liên tác tử được gắn chặt chẽ với duy nhất một `case_id`. Không cho phép chia sẻ hay rò rỉ trạng thái giữa các case.
- **Workflow State Progression:**
  - `case_received` (bởi `coordinator` tại CLI).
  - `task_assigned` (từ `coordinator` đến các `order_agent`, `payment_agent`, `shipment_agent`, `policy_agent`).
  - `tool_result_consumed` (được ghi nhận ngay khi một Specialist Agent tiếp nhận evidence hợp lệ từ MCP Gateway).
  - `handoff` (từ từng Specialist Agent chuyển giao kết quả đã thu thập tới `verifier`).
  - `policy_decided` (từ `policy_agent` truyền tải quy tắc chính sách đã giải mã tới `verifier`).
  - `verification_completed` (bởi `verifier` khi hoàn tất toàn bộ kiểm tra ràng buộc).
  - `case_finalized` (bởi `coordinator` tại CLI).
- **Tránh vòng lặp (Loop Prevention):** Luồng di chuyển theo đồ thị có hướng không chu trình (DAG: Coordinator $\rightarrow$ Specialists $\rightarrow$ Verifier). Không có cơ chế phản hồi lặp ngược (feedback loop) đa tầng không kiểm soát.

## 4. Evidence lifecycle

- **Validation:** Mọi phản hồi từ MCP Gateway được bọc trong phong bì `day09-mcp-evidence-v1` và được validate lập tức thông qua `Contracts.validate_evidence` trước khi giải nén payload.
- **Audit & Provenance:** Thu thập chính xác mã định danh bằng chứng `evidence_ref` (định dạng `ev_...`) do Gateway cấp phát trong chính phiên làm việc của case đó.
- **Audit Logging:** Mỗi khi một Agent sử dụng kết quả bằng chứng, sự kiện `tool_result_consumed` được phát ra trace với danh sách `evidence_refs` tương ứng.
- **Relevance Mapping:** Bằng chứng được phân loại theo domain (`order`, `item`, `payment`, `shipment`, `seller`, `refund`, `policy`). `verifier` chỉ đưa vào `evidence_refs` cuối cùng các chứng cứ liên quan trực tiếp đến vấn đề cốt lõi (`primary_issue`), loại bỏ các domain không liên quan để tránh bị phạt Forbidden Domain Penalty.

## 5. Failure policy

| Tình huống lỗi | Cơ chế Retry? | Phương án Fallback | Trace Event / Decision Code |
| --- | --- | --- | --- |
| **MCP Timeout** | Retry tối đa 1 lần (khoảng cách 150ms). | Bỏ qua tool bị timeout nếu không bắt buộc, hoặc sử dụng dữ liệu từ nguồn tin cậy thay thế đã thu thập. | Ghi nhận fallback, không để sập tiến trình toàn bộ case. |
| **Tool Error / Not Found** (ví dụ: `get_refund_timeline` khi đơn hàng chưa từng có yêu cầu hoàn tiền) | Không retry nếu là lỗi nghiệp vụ (404/Empty). | Xác định không tồn tại bản ghi hoàn tiền (refund = None/Empty), chuyển sang đánh giá theo các bằng chứng thanh toán khác. | Không phát `tool_result_consumed` với bằng chứng lỗi. |
| **Source Conflict** (ví dụ: Khách khiếu nại giao trễ nhưng vận chuyển xác nhận đúng hạn) | Không cần retry. | Dữ liệu MCP là thẩm quyền tối cao (Ground Truth). Đưa sự khác biệt vào trường `data_conflicts` của output với resolution code cụ thể. | `verification_completed` ghi nhận xử lý xung đột dữ liệu. |
| **Invalid Specialist Result** | Không retry. | `verifier` tiến hành kiểm tra độc lập và chuẩn hóa về mã lỗi an toàn (`unsupported_claim` hoặc quy tắc mặc định theo policy). | `VERIFICATION_PASSED` với cấu trúc chuẩn. |

## 6. Verification invariants

Trước khi hàm `solve_case` hoàn tất và trả về kết quả, các bất biến sau được đảm bảo tuyệt đối:
1. **Schema Compliance:** Output tuân thủ 100% JSON Schema `day09-l3a-output-v2` (tất cả các trường bắt buộc, regex pattern, enum values).
2. **Entity Consistency:**
   - `order_ids` chỉ chứa `order_id` của case hiện tại.
   - `item_ids`, `seller_ids`, `payment_references` được trích xuất trực tiếp từ các bản ghi authoritative MCP.
   - Khi `seller` là đối tượng chịu trách nhiệm trong `root_cause_analysis`, `party_id` được đối chiếu chính xác với `seller_id` thật của đơn hàng, không để template ID giả.
3. **Financial Consistency:**
   - Đơn vị tiền tệ luôn là `BRL`.
   - Nếu `case_status` là `no_action`, `recommended_refund_brl` bắt buộc bằng `0.0` và `refund_lines` là danh sách rỗng.
   - Khi có hoàn tiền (`refund_brl > 0`), tổng số tiền trong `refund_lines` phải khớp chính xác với `recommended_refund_brl`, đi kèm hành động tương ứng (`issue_refund`, `refund_freight`, `reconcile_payment`, `refund_duplicate_charge`, `retry_refund`).
4. **Action & Status Alignment:**
   - Các hành động trong `resolution_actions` là duy nhất (`uniqueItems: true`), không trùng lặp và phản ánh đúng khuyến nghị từ Policy.
5. **Confidence Bounds:** Độ tin cậy `confidence` được giữ ở mức `1.0`, phản ánh sự chắc chắn tuyệt đối khi đã đối soát đầy đủ chứng cứ thẩm quyền từ Gateway và loại bỏ sai số calibration.

## 7. Reproducibility

- **Ngôn ngữ & Môi trường:** Python 3.13 (64-bit), môi trường ảo `.venv`.
- **Thư viện phụ thuộc:** `mcp>=2,<3`, `httpx2>=2,<3`, `jsonschema[format]>=4.25,<5`, `python-dotenv>=1.1,<2`.
- **Concurrency Limit:** Chạy tuần tự và kiểm soát timeout (30.0s connect, 300.0s read) nhằm bảo vệ MCP server tránh rate-limit.
- **Lệnh thực thi chuẩn:**
  ```powershell
  # 1. Kích hoạt môi trường
  .\.venv\Scripts\Activate.ps1

  # 2. Kiểm tra input hợp lệ
  day09 validate-inputs

  # 3. Chạy toàn bộ 100 case
  day09 run

  # 4. Xác minh tính hợp lệ của output và trace
  day09 validate

  # 5. Đóng gói nộp bài
  day09 package --output dist/submission.zip
  ```
