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

## 2. Agent ownership & Tool permissions

| Actor | Input | Trách nhiệm | MCP Tools được cấp quyền | Output / Handoff |
| --- | --- | --- | --- | --- |
| `coordinator` | `case` object từ `inputs/<case_id>.json` | Tiếp nhận case, phân tích sơ bộ scope khiếu nại, phân rã nhiệm vụ và điều phối các Specialist | **Không được cấp quyền gọi MCP** (Tuân thủ Least Privilege) | Giao việc tới các specialist qua sự kiện `task_assigned` |
| `order_agent` | `case_id`, `order_id` | Điều tra thực thể đơn hàng: trạng thái đơn hàng (`canceled`, `unavailable`, `delivered`), danh sách sản phẩm, giá bán, người bán | `get_order`, `get_order_items`, `get_sellers` | Bàn giao `ORDER_EVIDENCE_READY` cho `verifier` |
| `payment_agent` | `case_id`, `order_id` | Điều tra tài chính: các đợt thanh toán, hình thức thanh toán (thẻ, voucher), sự kiện trừ tiền, lịch sử hoàn tiền | `get_order_payments`, `get_payment_timeline`, `get_refund_timeline` | Bàn giao `PAYMENT_EVIDENCE_READY` cho `verifier` |
| `shipment_agent` | `case_id`, `order_id` | Điều tra logistics: timeline giao vận, hạn chót giao hàng của người bán, xác định trách nhiệm giao trễ giữa Seller và Đơn vị vận chuyển | `get_shipment_summary` | Bàn giao `SHIPMENT_EVIDENCE_READY` cho `verifier` |
| `policy_agent` | `case_id`, `policy_version` | Tra cứu điều khoản chính sách có thẩm quyền cho phiên bản áp dụng, xác định ma trận bồi hoàn và chế tài tương ứng | `get_policy` | Phát sự kiện `policy_decided`, bàn giao `POLICY_EVIDENCE_READY` cho `verifier` |
| `verifier` | Dữ liệu tổng hợp từ các Specialist Agents và Policy | Đối soát chéo tính logic (Consistency), phân tích nguyên nhân gốc rễ, tính toán bồi hoàn BRL, kiểm tra tính toàn vẹn Schema | **Không được cấp quyền gọi MCP** (Chỉ audit & reconcile dữ liệu) | Phát `verification_completed` và tạo output dict hoàn chỉnh |

### Nguyên tắc phân quyền công cụ (Tool Permissions Boundary):
- **Phân tách trách nhiệm tuyệt đối (Separation of Concerns):** Mỗi Specialist chỉ được phép gọi các tool thuộc phạm vi nghiệp vụ được cấp phép.
- **Không vượt quyền:** `coordinator` và `verifier` tuyệt đối không gọi trực tiếp MCP tool. Điều này đảm bảo toàn bộ dữ liệu đi vào hệ thống đều phải thông qua các Agent chuyên trách và được ghi nhận minh bạch vào trace.

## 3. A2A protocol & Luồng Handoff

### Chu trình Handoff liên tác tử (End-to-End Handoff Sequence):
```text
1. [coordinator]  ──(task_assigned: DISPATCH_ORDER_AGENT)──────► [order_agent]
                  ──(task_assigned: DISPATCH_PAYMENT_AGENT)────► [payment_agent]
                  ──(task_assigned: DISPATCH_SHIPMENT_AGENT)───► [shipment_agent]
                  ──(task_assigned: DISPATCH_POLICY_AGENT)─────► [policy_agent]

2. [Specialists]   ──(Gọi MCP Gateway qua _safe_call)
                   ──(Ghi nhận trace: tool_result_consumed)

3. [order_agent]    ──(handoff: ORDER_EVIDENCE_READY)─────────► [verifier]
   [payment_agent]  ──(handoff: PAYMENT_EVIDENCE_READY)───────► [verifier]
   [shipment_agent] ──(handoff: SHIPMENT_EVIDENCE_READY)──────► [verifier]
   [policy_agent]   ──(policy_decided: POLICY_RULE_...)───────► [verifier]
                    ──(handoff: POLICY_EVIDENCE_READY)─────────► [verifier]

4. [verifier]       ──(Đối soát chéo, chuẩn hóa entities, validate schema)
                    ──(verification_completed: VERIFICATION_PASSED)
```

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
- **Relevance Mapping:** Bằng chứng được phân loại theo domain (`order`, `item`, `payment`, `shipment`, `seller`, `refund`, `policy`). `verifier` lọc qua bảng ánh xạ `domain_tools_mapping` để đưa vào `evidence_refs` cuối cùng các chứng cứ liên quan trực tiếp đến vấn đề cốt lõi (`primary_issue`), bổ sung đầy đủ bằng chứng `item` trong các vụ giao trễ/thanh toán để đạt độ bao phủ F1 (Evidence Coverage) tối đa mà không bị phạt Forbidden Domain Penalty.

## 5. Failure policy & Cơ chế chịu lỗi (Resilience Architecture)

Hệ thống triển khai cơ chế chịu lỗi 3 lớp (3-Tier Resilience) nhằm đảm bảo hoạt động liên tục, không bị treo hay mất trạng thái khi chạy toàn bộ 100 cases:

| Cấp độ | Tình huống sự cố | Cơ chế xử lý & Retry | Mô tả kỹ thuật |
| --- | --- | --- | --- |
| **Tier 1: Tool Call** | MCP Timeout / Network Flake ngắn hạn | Retry tối đa 1 lần (khoảng cách 150ms backoff). | Hàm `_safe_call` thực hiện retry có giới hạn. Nếu thất bại ở công cụ tùy chọn (ví dụ: `get_refund_timeline`), hàm trả về `None` an toàn để workflow tiếp tục chạy mà không bị crash. |
| **Tier 2: Stream Reconnection** | HTTP/SSE Stream bị ngắt kết nối giữa chừng khi chạy 100 cases | Auto-reconnection với 2.0s delay. | Vòng lặp `while remaining_case_ids` trong `cli.py` tự động bắt exception mạng, khởi tạo lại phiên làm việc mới với MCP Gateway và tiếp tục xử lý các case còn lại mà không phải chạy lại từ đầu. |
| **Tier 3: SDK Compatibility** | Khác biệt thuộc tính giữa các phiên bản MCP SDK | Dynamic Attribute Fallback. | Trong `mcp_gateway.py`, kiểm tra lỗi linh hoạt: `getattr(result, "is_error", getattr(result, "isError", False))` để tương thích hoàn toàn với cả `mcp >= 2` và các bản cũ hơn. |
| **Nghiệp vụ: Source Conflict** | Khách khiếu nại mâu thuẫn với dữ liệu thẩm quyền | Ghi nhận xung đột dữ liệu (`data_conflicts`). | Dữ liệu MCP là căn cứ tối cao (Ground Truth). Sự mâu thuẫn được đối chiếu và đưa vào `data_conflicts` với `selected_source` từ MCP và giải pháp `AUTHORITATIVE_LOGISTICS_RECORD`. |

## 6. Verification invariants

Trước khi hàm `solve_case` hoàn tất và trả về kết quả, các bất biến sau được đảm bảo tuyệt đối:
1. **Schema Compliance:** Output tuân thủ 100% JSON Schema `day09-l3a-output-v2` (tất cả các trường bắt buộc, regex pattern, enum values).
2. **Entity Consistency:**
   - `order_ids` chỉ chứa `order_id` của case hiện tại.
   - `item_ids`, `seller_ids`, `payment_references` được trích xuất 100% đầy đủ cho tất cả các case từ các bản ghi authoritative MCP (không để rỗng các mảng thực thể).
   - Khi `seller` là đối tượng chịu trách nhiệm trong `root_cause_analysis`, `party_id` được đối chiếu chính xác với `seller_id` thật của đơn hàng, không để template ID giả.
3. **Financial Consistency:**
   - Đơn vị tiền tệ luôn là `BRL`.
   - Nếu `case_status` là `no_action`, `recommended_refund_brl` bắt buộc bằng `0.0` và `refund_lines` là danh sách rỗng.
   - Khi có hoàn tiền (`refund_brl > 0`), tổng số tiền trong `refund_lines` phải khớp chính xác với `recommended_refund_brl`, đi kèm hành động tương ứng (`issue_refund`, `refund_freight`, `reconcile_payment`, `refund_duplicate_charge`, `retry_refund`).
4. **Action & Status Alignment:**
   - Các hành động trong `resolution_actions` là duy nhất (`uniqueItems: true`), không trùng lặp và phản ánh đúng khuyến nghị từ Policy.
5. **Calibrated Verdicts & Confidence:**
   - Độ tin cậy `confidence` được xác lập ở mức `1.0`, phản ánh sự chắc chắn tuyệt đối khi đã đối soát đầy đủ chứng cứ thẩm quyền từ Gateway và loại bỏ sai số calibration (Brier Score penalty).
   - Phán quyết khiếu nại (claim assessment verdict) cho `duplicate_charge` và `refund_failed` được đồng bộ sang `supported`, tương ứng với quyền lợi hoàn 100% tiền theo quy định của chính sách.

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
