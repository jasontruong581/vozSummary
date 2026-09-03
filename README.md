# voz Điểm báo → Excel (chấm điểm chủ đề hot + top comment)

Quét box [Điểm báo](https://voz.vn/f/diem-bao.33/) của voz.vn mỗi ngày **10:00 giờ VN**,
xếp hạng chủ đề đang được bàn luận nhiều, và trích **top 20 comment được nhiều "Ưng" nhất**
trong mỗi thread — làm nguồn ý tưởng viết bài trên Threads.

---

## 1. Vì sao là Python chứ không phải Tampermonkey?

| Yêu cầu của bạn | Tampermonkey | Python |
|---|---|---|
| Chạy hàng ngày 10:00 | ❌ chỉ chạy khi bạn **tự mở** trang voz | ✅ cron / GitHub Actions |
| Đi vào 20 thread × nhiều trang | ⚠️ phải `fetch` chéo trang, dễ bị treo tab | ✅ |
| Xuất **file Excel** (.xlsx nhiều sheet, công thức, hyperlink) | ❌ chỉ tải được CSV thô | ✅ openpyxl |
| Lưu lịch sử để so sánh ngày qua ngày | ❌ | ✅ JSON + git |

→ Userscript không đáp ứng được điều kiện 0 (hẹn giờ). Repo này dùng Python.
Tampermonkey chỉ nên dùng nếu voz chặn hoàn toàn request ngoài trình duyệt — xem §6.

## 2. Cài đặt

```bash
git clone <repo-cua-ban> && cd voz-diembao
python -m venv .venv && source .venv/bin/activate     # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python test_parser.py                                 # 18 test, nên chạy trước
```

## 3. Chạy

```bash
python scrape_voz.py                       # quét đầy đủ → output/voz-diem-bao_YYYY-MM-DD.xlsx
python scrape_voz.py --max-threads 3       # test nhanh (~30 giây)
python scrape_voz.py --dry-run             # chỉ in danh sách thread, không vào chi tiết
python scrape_voz.py --top 30 --min-reactions 3
python scrape_voz.py --since-hours 24      # chỉ xét comment đăng trong 24h qua
```

Tham số hay dùng:

| Cờ | Mặc định | Ý nghĩa |
|---|---|---|
| `--top` | 20 | số comment top mỗi thread |
| `--min-reactions` | 1 | bỏ comment 0 Ưng (không ai đồng tình) |
| `--delay` | 1.2 | giây nghỉ tối thiểu giữa 2 request (có jitter) |
| `--max-pages` | 0 | giới hạn số trang/thread (0 = quét hết) |
| `--since-hours` | 0 | chỉ xét comment mới trong N giờ |
| `--impersonate` | – | `chrome` — dùng khi bị 403 (§6) |
| `--ignore-robots` | – | bỏ kiểm tra robots.txt (không khuyến khích) |

## 4. File Excel gồm 4 sheet

**`ThamSo`** — 4 ô vàng để bạn tự tinh chỉnh trọng số. Sửa xong, cả sheet `Tong quan` tự tính lại
(điểm và hạng là **công thức**, không phải số cứng):

```
Điểm gốc  = replies × W_REPLY + tổng_Ưng × W_UNG
Hot score = Điểm gốc / (tuổi_thread_giờ + AGE_OFFSET_H) ^ GRAVITY
```

Mặc định `W_REPLY=1`, `W_UNG=2`, `GRAVITY=0.8`, `AGE_OFFSET_H=6`.
`GRAVITY < 1` để thread mới không tự động đè thread cũ; đặt `GRAVITY = 0` nếu muốn bỏ hẳn yếu tố thời gian.

**`Tong quan`** — 1 dòng = 1 thread, đã sắp theo hot score:
hạng · tiêu đề · link thread · **link bài báo** · nguồn · replies · tổng Ưng · tuổi thread ·
điểm gốc · hot score · người đăng · thời gian · số trang đã quét · **trích bài báo (post #1)** · lỗi

**`Top comments`** (dạng dài — nên dùng để đọc & lọc) — 1 dòng = 1 comment:
hạng thread · tiêu đề · link bài báo · hạng comment · **Ưng** · tác giả · post # ·
link tới đúng comment · thời gian · nội dung. Có AutoFilter: lọc `Ưng ≥ 10` là ra ngay
những ý kiến được đồng tình nhất trong ngày.

**`Bang ngang`** — đúng format bạn mô tả: 1 dòng = 1 thread, các cột `Top 1` → `Top 20`.

Kèm theo `output/voz-diem-bao_YYYY-MM-DD.json` để so sánh giữa các ngày.

## 5. Hẹn giờ 10:00 bằng GitHub Actions

1. Push repo này lên GitHub (repo **private** cũng chạy được).
2. `Settings → Actions → General → Workflow permissions` → **Read and write** (để commit vào `history/`).
3. Workflow đã có sẵn ở `.github/workflows/daily.yml`, cron `0 3 * * *` = **10:00 ICT**.
4. Vào tab **Actions → Run workflow** bấm chạy thử một lần trước khi tin vào cron.
5. Tải file: Actions → chọn run → mục **Artifacts** (giữ 90 ngày), hoặc xem thư mục `history/` trong repo.

Lưu ý: cron của GitHub Actions **có thể trễ 5–30 phút** khi hạ tầng đông — nó không phải
bộ hẹn giờ chính xác đến giây. Nếu cần đúng 10:00, đặt cron sớm hơn (`0 2 * * *`).

<details>
<summary>Cách khác: chạy trên máy của bạn</summary>

```bash
# macOS / Linux — crontab -e
0 10 * * * cd /duong/dan/voz-diembao && .venv/bin/python scrape_voz.py >> cron.log 2>&1

# Windows — Task Scheduler: Create Task → Trigger: Daily 10:00
#   Action: Program = C:\...\voz-diembao\.venv\Scripts\python.exe
#           Arguments = scrape_voz.py
#           Start in  = C:\...\voz-diembao
```
</details>

## 6. Xử lý sự cố

| Triệu chứng | Cách xử lý |
|---|---|
| `403 tu voz.vn` | Cloudflare chặn IP datacenter (rất hay gặp trên GitHub Actions). Bỏ dấu `#` ở `curl_cffi` trong `requirements.txt`, rồi thêm `--impersonate chrome`. Vẫn chặn → dùng self-hosted runner hoặc chạy trên máy nhà. |
| `Khong parse duoc thread nao` | voz đổi markup. Chạy `python test_parser.py` rồi mở `parse_thread_list()` sửa selector. Script **cố tình dừng và báo lỗi** thay vì xuất file rỗng. |
| Excel thiếu link bài báo | Post #1 không có link ngoài (chỉ ảnh chụp báo). Cột để trống, dùng cột `Trich bai bao` thay thế. |
| Tất cả `Tuoi thread (gio)` = 0 | voz đổi thuộc tính thời gian. Xem `parse_post_time()` — timestamp thật nằm ở `data-timestamp`, còn `data-time` chỉ là chuỗi giờ (`"9:02 AM"`). |
| Chạy chậm | `--delay 1.0` là ngưỡng lịch sự thấp nhất nên dùng; đừng đặt 0. Hoặc `--max-pages 3`. |

## 7. Best practice đã áp dụng

**Với server voz**
- Rate limit **1.2 s + jitter** giữa các request, tuần tự (không song song).
- Retry có `backoff` chỉ cho 429/5xx, tôn trọng header `Retry-After`.
- Kiểm tra `robots.txt` trước khi quét (`urllib.robotparser`); chỉ đọc `/f/` và `/t/`, không đăng nhập, không POST.
- User-Agent trình duyệt **thường**. robots.txt của voz chặn các bot AI có tên
  (`GPTBot`, `ClaudeBot`, `CCBot`, `Bytespider`, …) — **đừng bao giờ** đặt UA thành các tên đó.

**Với code**
- Parse bằng **CSS selector có ý nghĩa cấu trúc** (`.structItemContainer-group--sticky`),
  không dựa vào màu chữ cam hay thứ tự dòng.
- Có **fallback** khi voz đổi markup, và log cảnh báo khi phải dùng fallback.
- **Fail loud**: parse ra 0 thread thì raise, không xuất file rỗng để bạn tưởng là "hôm nay không có gì".
- Lỗi từng thread được bắt riêng, ghi vào cột `Loi`, không làm sập cả lần chạy.
- 18 test với fixture copy từ markup thật; 2 test là **regression test** cho 2 lỗi
  chỉ lộ ra khi đối chiếu HTML thô (xem §8).
- Điểm số trong Excel là **công thức** → bạn tinh chỉnh trọng số ngay trong file, không cần sửa code.
- Xuất kèm JSON để lần chạy sau so sánh được (thread nào tăng nhiệt).

**Với bản quyền / nội dung**
- `robots.txt` của voz khai `Content-Signal: search=yes, ai-train=no, use=reference`.
  Dữ liệu này là **tư liệu tham khảo**: dùng để nắm chủ đề đang nóng và góc nhìn của cộng đồng.
- Khi viết bài Threads: **đừng copy nguyên văn comment của người khác**. Dùng nó làm
  *insight* ("dân mạng đang tranh luận về X theo 2 hướng…"), hoặc xin phép / dẫn nguồn nếu muốn trích.
- Link bài báo trong cột `Link bai bao` mới là nguồn nên dẫn.

## 8. Hai lỗi đã tìm ra khi đối chiếu HTML thật

Ghi lại vì đây là loại lỗi rất dễ bỏ sót — cả hai đều **không xuất hiện** khi test trên DOM
đã chạy JS, chỉ lộ ra khi parse HTML thô mà Python nhận được:

1. **`data-time` không phải timestamp.** voz render
   `<time datetime="2026-09-03T09:02:03+0700" data-timestamp="1788400923" data-time="9:02 AM">`.
   Đọc `data-time` sẽ ra `None` → tuổi thread = 0 → hot score sai toàn bộ.
   Timestamp thật ở **`data-timestamp`**.
2. **Post #1 của box Điểm báo thường CHỈ gồm quote bài báo + card unfurl.**
   Logic "bỏ blockquote để lấy lời của chính người post" (đúng cho comment) khi áp vào post #1
   sẽ ra **chuỗi rỗng** — mất sạch nội dung bài báo. Nên `post_text()` có cờ `keep_quotes`:
   bật cho post #1, tắt cho comment.

## 9. Về cách đếm "Ưng"

voz chỉ có **một** loại reaction (`data-reaction-id="1"`, tên "Ưng"), nên số reaction = số Ưng.
XenForo render dạng `<bdi>A</bdi>, <bdi>B</bdi>, <bdi>C</bdi> and 1,208 others`
→ đếm `= số thẻ <bdi>` + `số trong "and N others"` = 1211. (`parse_reactions()`)

## 10. Cấu trúc repo

```
scrape_voz.py                  script chính (1 file, ~830 dòng, có comment tiếng Việt)
test_parser.py                 18 test, chạy được không cần mạng
requirements.txt
.github/workflows/daily.yml    cron 10:00 ICT
output/                        file Excel + JSON
history/                       lịch sử do Actions commit (tuỳ chọn)
```
