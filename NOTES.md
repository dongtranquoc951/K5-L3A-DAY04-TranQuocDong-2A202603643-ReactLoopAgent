# Ghi chú phân tích · Day 04

Họ và tên:
MSSV:

So sánh hai kết quả: `eval_before.json` (**87.64/100**) và `eval_after.json` (**92.00/100**). File after là kết quả người dùng tự chạy với phiên bản agent đã tối ưu.

## 1. Case được chọn

- **H6**: kiểm tra GitHub Actions, đọc incident ban đầu rồi theo liên kết tới incident gộp để tìm nguyên nhân, ảnh hưởng và giới hạn dữ liệu.
- Before đạt **40/100**. Agent nhắc liên kết nhưng chưa đọc nguồn gộp, nên chưa đủ bằng chứng để trả lời về nguyên nhân; cũng không nêu giới hạn dữ liệu.
- H1 có lỗi tương tự và đạt 60/100 trước cải tiến.
- Chọn H6 vì có bằng chứng cụ thể trong trace, có thể cải thiện quá trình thu thập bằng chứng mà không hard-code câu trả lời.

## 2. Bằng chứng và nguyên nhân

Trace H6 trong `eval_before.json`, trích lược:

```text
step 1: get_status(provider=github, component=GitHub Actions)
step 1: get_incident_details(provider=github, incident_id=h02n9572mx9y)
        update dẫn tới https://www.githubstatus.com/incidents/xr3myzj7ms51
        updates_total=4; trả về 3 updates
step 2: model stop -> validation_missing_source
step 3: model stop -> completed
```

Không có call đọc `xr3myzj7ms51`; hai tool calls đều ở step 1. Câu trả lời chỉ dẫn shortlink của incident ban đầu: `https://stspg.io/7pwv28hvkmz8`. Shortlink này có thật trong tool result, không phải nguồn bịa.

Nguyên nhân có thể xảy ra: model dừng sau thông tin ban đầu, bị phân tán bởi 40 mục nhiễu trong status; kiểm tra một URL chưa đủ phát hiện bước đọc nguồn liên quan còn thiếu.

## 3. Thay đổi trong agent.py

- Trích ID từ liên kết HTTPS đúng host status của cùng provider GitHub/Cloudflare. Nếu còn incident liên quan chưa thử đọc, ghi `validation_unread_incidents` và yêu cầu model đọc tiếp trong `max_steps`. Không tự tải URL ngoài; link tự trỏ, chu trình và tool đã lỗi không bị ép retry vô hạn.
- Giới hạn yêu cầu model sửa thiếu nguồn tối đa một lần mỗi run. Nếu vẫn thiếu, bổ sung phụ lục **Nguồn incident đã đọc (tham khảo)** bằng URL đã quan sát, ghi `validation_source_appendix`, tránh lặp chỉ để sửa URL. Nếu lần đầu thiếu nguồn đã ở bước cuối, vẫn trả `max_steps` theo contract.
- Chấp nhận shortlink từ tool và canonical URL dựng từ host đã cấu hình cùng ID được tool xác nhận. So khớp URL đầy đủ; thêm phụ lục cho incident đã đọc còn thiếu nguồn.
- Thêm metadata `_agent_evidence` vào tool message: số updates trả về/bị lược bỏ, số body bị cắt, URL nguồn và ID liên quan. Ví dụ 9 updates, trả 3 là thiếu 6 updates; khác với việc một body có `truncated=true`. Trace giữ nguyên kết quả tool gốc.
- Chỉnh prompt: trả lời trực tiếp câu hỏi thông thường/viết thơ; ID cụ thể cần đọc chi tiết; trạng thái hiện tại cần status tool; phân biệt snapshot/lịch sử/hiện tại và phạm vi ảnh hưởng; không coi 50 incident là dữ liệu đầy đủ cho cả năm; không suy diễn timeout thành operational/outage.
- Giữ contract loop, xử lý lỗi, trace, usage và budget. Không sửa fixtures, dataset, scenarios, scoring hay rubric; không hard-code ID hoặc câu trả lời vào agent.

Kiểm tra offline sau tối ưu: **45 tests trong tests/ + 3 tests trong bonus/ = 48 tests đạt**. Tests kiểm tra đọc nguồn liên quan, budget, chu trình/tool lỗi, link ngoài host, giới hạn sửa nguồn, canonical URL, bổ sung nguồn và tính metadata mà không thay trace gốc.

## 4. So sánh before/after

Hai report có cùng cấu hình: provider agent/judge `xkiro`, agent `mistralai/codestral-2508`, judge `mistralai/mistral-large-2512`, `max_steps=8`, dataset core và snapshot lưu sẵn. Đã đối chiếu đủ 22 ID, câu hỏi, expected và scenario: tất cả khớp. Mỗi report chấm đủ 22/22 case, không có case unscored.

| Chỉ số | eval_before.json | eval_after.json |
|---|---:|---:|
| Điểm tổng /100 | 87.64 | **92.00** |
| H1 /100 | 60 | **80** |
| H6 /100 | 40 | **100** |
| Model calls agent, toàn bộ | 51 | 50 |
| Model calls agent, H6 | 3 | 4 |
| Token agent, toàn bộ | 50,129 | 63,122 |
| Token agent, H1 | 5,350 | 7,610 |
| Token agent, H6 | 5,693 | 10,861 |
| Token judge | 37,431 | 32,406 |
| Tổng token agent + judge | 87,560 | 95,528 |
| Runs completed / max_steps | 22 / 0 | 22 / 0 |
| Chi phí bằng tiền | Không được cung cấp | Không được cung cấp |

Điểm tăng **4.36 điểm**, model calls agent giảm 1; token agent tăng **12,993 (+25.92%)**, tổng token cả agent và judge tăng **9.10%**. Cải thiện chất lượng trong lần chạy này đi kèm lượng token cao hơn baseline, không thể kết luận tối ưu làm giảm chi phí.

| Case | Before | After | Thay đổi |
|---|---:|---:|---:|
| E1 | 90 | 100 | +10 |
| E2 | 100 | 87 | -13 |
| E3 | 90 | 100 | +10 |
| E4 | 100 | 90 | -10 |
| E5 | 85 | 100 | +15 |
| E6 | 100 | 100 | 0 |
| A1 | 100 | 100 | 0 |
| A2 | 100 | 100 | 0 |
| A3 | 100 | 100 | 0 |
| A4 | 100 | 100 | 0 |
| A5 | 100 | 100 | 0 |
| A6 | 93 | 100 | +7 |
| A7 | 75 | 72 | -3 |
| A8 | 100 | 100 | 0 |
| H1 | 60 | 80 | +20 |
| H2 | 50 | 50 | 0 |
| H3 | 100 | 100 | 0 |
| H4 | 100 | 100 | 0 |
| H5 | 100 | 100 | 0 |
| H6 | 40 | 100 | +60 |
| H7 | 50 | 45 | -5 |
| H8 | 95 | 100 | +5 |

### Bằng chứng cải thiện

- **H6: 40 → 100**. After: step 1 đọc status và incident ban đầu; step 2 bị chặn bởi `validation_unread_incidents`; step 3 đọc incident gộp; step 4 hoàn tất, thêm nguồn incident ban đầu vào phụ lục. Câu trả lời mô tả nguyên nhân về thay đổi cấu hình/feature flags, ảnh hưởng và URL nguồn gộp.
- **H1: 60 → 80**. Đọc nguồn gộp ở step 3, hoàn tất ở step 4; bổ sung hai nguồn vào phụ lục. Không còn bỏ qua incident gộp trước khi trả lời.
- **A6: 93 → 100**. Phân biệt incident ban đầu chỉ trỏ sang nguồn gộp; đọc nguồn gộp trước khi nêu nguyên nhân và có nguồn cho cả hai incident.
- **A5/H3/H4 giữ 100**: thừa nhận giới hạn dữ liệu cả năm, đọc chi tiết ID không tồn tại và trả lời viết thơ trực tiếp.

### Lỗi còn lại

- **H2: 50**: chưa gọi `get_status`; incident lịch sử không đủ xác nhận trạng thái snapshot hiện tại của Actions.
- **H7: 50 → 45**: tool Cloudflare timeout nhưng câu trả lời vẫn khẳng định Cloudflare hoạt động bình thường. Prompt chưa bảo đảm model luôn từ chối yêu cầu bịa kết luận.
- **A7: 75 → 72**: dùng tên địa điểm `Phoenix` làm `incident_id`, nhận not found rồi chưa tìm đúng incident hay đánh giá quan hệ với lỗi tại Việt Nam.
- **E2: 100 → 87**: chỉ gọi status, chưa đọc update chi tiết để xác minh phạm vi non-FedRAMP; câu trả lời không có URL và gọi snapshot là trạng thái hiện tại.
- **H1 dù đạt 80**: chỉ mô tả incident gộp GitHub; Cloudflare và incident ban đầu chỉ có URL trong phụ lục. Chưa tổng hợp đủ hai bên, chưa nêu rõ không có bằng chứng chung nguyên nhân và giới hạn dữ liệu bị cắt.
- **H6 dù đạt 100**: câu trả lời cuối chưa nói rõ giới hạn bị cắt/lược bỏ và trạng thái snapshot Actions, dù judge nhận xét có. Đọc đủ nguồn không đồng nghĩa trả lời đủ mọi yêu cầu.
- **E4: 100 → 90**: giữ nguyên điểm chấm nhưng nhận xét judge có sai lệch; Code Scanning/Copilot thực sự xuất hiện trong update trả về, câu trả lời cũng đã nói sự cố được giải quyết. Không tự sửa điểm report.

## 5. Giới hạn và cách chạy lại

- Snapshot không phản ánh hiện tại thực tế; components rỗng không chứng minh một component operational hay không bị ảnh hưởng.
- Judge có thể nhận xét sai. Cần đối chiếu answer và trace, không chỉ dựa vào điểm số.
- Phụ lục nguồn chỉ liệt kê tài liệu đã đọc, không bảo đảm mọi claim đúng hoặc đã gán nguồn đúng theo từng claim.
- Theo liên kết chỉ hỗ trợ URL incident cùng provider GitHub/Cloudflare; không bảo đảm dữ liệu ngoài 50 incident gần nhất hoặc chuỗi dài trong budget.
- Prompt/metadata chưa bảo đảm model luôn làm đúng về timeout, thời gian, số update và phạm vi ảnh hưởng. Một lần chạy mỗi phiên bản chưa đủ chứng minh độ ổn định.
- Không có chi phí bằng tiền trong usage; không quy đổi token thành số tiền hoặc khẳng định eval miễn phí. Dòng model calls/token agent không bao gồm judge.

Chạy ở thư mục gốc dự án bằng PowerShell, key nạp từ `.env`. Lệnh dưới đây gọi API khi thực thi và ghi kết quả vào `eval_after.json`; nếu muốn giữ lần chạy hiện tại, dùng tên output khác.

```powershell
$env:PROVIDER="xkiro"
$env:JUDGE_PROVIDER="xkiro"
python -X utf8 -c "import sys; sys.path.insert(0, 'evals'); from providers import load_env; load_env(); from run_evals import main; sys.exit(main())" --model "mistralai/codestral-2508" --judge-model "mistralai/mistral-large-2512" --max-steps 8 --output eval_after.json
```
