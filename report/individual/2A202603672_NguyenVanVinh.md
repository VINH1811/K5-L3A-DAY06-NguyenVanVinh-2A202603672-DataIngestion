# Báo cáo cá nhân — Day 6

Nguyễn Văn Vinh · 2A202603672 · L3A. Lần chạy được báo cáo: 2026-11-13 16:08 UTC (`results/run_info.json`).


## Dữ liệu và pipeline

- **Chủ đề, nguồn/quyền, số trang và quy tắc tính trang:** Cập nhật kinh tế Việt Nam, hai báo cáo "Taking Stock" của World Bank: *August 2023 — Making Public Investment Work for Growth* (58 trang vật lý) và *March 2025 — Electrifying Journeys* (62 trang vật lý), tải từ documents.worldbank.org ngày 2026-11-13. Trang bản quyền ghi © World Bank và không in giấy phép Creative Commons, nên **PDF không được đưa vào repo**; manifest có URL và SHA-256 để tải lại đúng file. Tính **88 trang nội dung** (40 + 48): phần tổng quan và thân chương 1, 2. Loại bìa, trang trống, trang bản quyền, mục lục, danh mục hình/bảng, từ viết tắt, lời cảm ơn, trang tiêu đề chương, tài liệu tham khảo và bìa sau. Số trang là **trang vật lý của PDF** (bắt đầu từ 1), không phải số in trên trang. Tổng 236.254 ký tự sau chuẩn hóa. Chi tiết: [data/manifest.json](../../data/manifest.json).
- **Chuẩn hóa, lỗi và giới hạn:** `pypdf` 6.21.4 trích từng trang; chuẩn hóa Unicode NFC, bỏ soft hyphen/zero-width, gộp khoảng trắng, giữ ngắt dòng và ngắt đoạn. Không trang nào trong 88 trang bị rỗng hay lỗi ([results/ingest_log.json](../../results/ingest_log.json)); không cần OCR. Giới hạn: text trích ra lẫn nhãn trục/số liệu của biểu đồ, số chú thích dính vào câu, header/footer mỗi trang, và ở vài trang hai cột thứ tự đọc bị trộn (ví dụ 2023 tr.21). Tôi không lọc các phần này.
- **Embedding, vector store, lệnh chạy, chi phí:** `google/embeddinggemma-2` (Google DeepMind, Apache 2.0) chạy local trên CPU qua `sentence-transformers` 6.3.0 / `transformers` 5.23.0 / `torch` 2.15.0, nạp chế độ chỉ văn bản, float32, **768 chiều**, vector chuẩn hóa L2. Theo model card: query thêm tiền tố `task: search result | query: `, document thêm `title: none | text: `. Vector store: **Chroma 1.5.9** `PersistentClient` lưu tại `starter/chroma_db/`, metric cosine, tìm kiếm **xấp xỉ** (HNSW mặc định). Lệnh: `python -m src.run --split all` từ `starter/`. Chi phí API: **0**. Không dùng pgvector của starter vì máy không có Docker; `starter/src/store.py` có bản pgvector nhưng **chưa từng được chạy**.
- **Metadata:** bắt buộc `document_id`, `source` (URL gốc), `page` (trang vật lý), cộng `char_start`/`char_end` trong trang. Bổ sung: `year` (số nguyên, lấy từ bìa: AUGUST 2023 / MARCH 2025) — **trường lọc**; `section` (chuỗi: `overview`, `economic_developments`, `special_focus`, ánh xạ từ mục lục sang dải trang); `document_type` (hằng số, chỉ mô tả). Chọn `year` vì hai báo cáo cùng nói về tăng trưởng, lạm phát, tín dụng, nợ xấu với số liệu khác nhau theo kỳ, nên câu hỏi không nêu kỳ sẽ mơ hồ nếu không lọc.

## Cấu hình và benchmark

- **A / B / C:**
  - **A** — cắt cố định 500 ký tự, overlap 50, theo từng trang; không lọc.
  - **B** — theo đoạn: gộp đoạn (ngăn bằng dòng trống) tới tối đa 800 ký tự, đoạn dài hơn thì tách ở cuối câu; không lọc.
  - **C** — dùng **đúng index của B** (cùng chunk, cùng vector); khác biệt duy nhất là áp dụng `filter` của câu hỏi khi câu đó có filter. Với 17 câu không có filter, C chạy không lọc và cho kết quả trùng B.
  - A–B chỉ đổi cách chunk (kéo theo kích thước chunk khác nhau: trung vị 500 so với 691,5 ký tự, nên đây không phải so sánh thuần "ranh giới" mà gồm cả kích thước). Tham số không được tinh chỉnh sau khi xem dev.
- **Benchmark:** 20 câu trong [benchmark/questions.json](../../benchmark/questions.json): q01–q15 dev, q16–q20 test; 9 câu trực tiếp, 5 diễn đạt lại, 3 điều kiện/ngoại lệ, 3 câu metadata. Ba câu có filter `year` (q14, q15 dev; q19 test); giá trị filter là **input giả định người dùng cung cấp trước** (đã chọn kỳ báo cáo), câu hỏi không chứa năm hay đáp án. Evidence được kiểm bằng code: `run.py` so khớp từng `evidence_quote` với text trang gold sau chuẩn hóa và từ chối chạy nếu thiếu; cả 20 câu đạt ([results/benchmark_validation.json](../../results/benchmark_validation.json)). Sự kiện xuất hiện ở cả phần tổng quan và chương được ghi đủ mọi trang hợp lệ. Benchmark được commit (`df83d95`, 21:45 +07) trước lần truy xuất đầu tiên. Rủi ro rò rỉ: dev và test cùng lấy từ hai tài liệu, chỉ khác sự kiện.
- **Tiêu chí chọn chiến lược (chốt trước khi xem test, ghi trong file benchmark):** Evidence Hit@3 trên dev cao nhất; hòa thì Hit@1 dev; hòa nữa thì ít chunk hơn.

## Kết quả

| Strategy | Split | Hit@1 (n/N) | Evidence Hit@3 (n/N) | Chunk count | Thời gian ingest | Chi phí |
|---|---|---|---|---|---|---|
| A | dev | 8/15 | 14/15 | 558 | 168,4 s | 0 |
| A | test | 4/5 | 4/5 | 558 | 168,4 s | 0 |
| B | dev | 11/15 | 15/15 | 370 | 144,0 s | 0 |
| B | test | 5/5 | 5/5 | 370 | 144,0 s | 0 |
| C | dev | 11/15 | 15/15 | 370 | 0 s thêm (dùng index B) | 0 |
| C | test | 5/5 | 5/5 | 370 | 0 s thêm (dùng index B) | 0 |

Thời gian ingest gần như toàn bộ là embedding trên CPU (A 167,7 s; B 143,6 s); chunking và ghi Chroma dưới 1 s. Tổng ký tự được embed: A 259.595 (nhiều hơn do overlap), B 235.971. Nạp lại lần hai không nhân đôi: A 558 → 558, B 370 → 370 bản ghi.

| Strategy | Coherence trung bình /2 (n) | Metadata đủ và đúng (n/N) | Filter tuân thủ, test (n/N) | Grounding 0/1/2 (số câu) và trung bình /2 |
|---|---|---|---|---|
| A | 1,0 (10) | 15/15 | 0/1 | 0 / 1 / 4 — 1,8 |
| B | 1,6 (10) | 15/15 | 1/1 | 0 / 0 / 5 — 2,0 |
| C | 1,6 (10) | 15/15 | 1/1 | 0 / 0 / 5 — 2,0 |

Filter tuân thủ trên dev (2 câu có filter): A 2/2, B 0/2, C 2/2.

**Trace, nhãn và cách chọn mẫu.** Trace đầy đủ top-3: `results/trace_A.jsonl`, `trace_B.jsonl`, `trace_C.jsonl`; chỉ số: `results/metrics.json`, `metrics.md`; nhãn: `results/audit_coherence.csv`, `audit_metadata.csv`, `audit_grounding.csv`, tổng hợp ở `audit_summary.json`.
- *Coherence:* 10 vị trí = điểm giữa của 10 trang nội dung đầu tiên theo thứ tự (2023 tr.11–17, 19–21), chọn theo quy tắc trong code trước khi có điểm; chấm chunk chứa vị trí đó ở mỗi chiến lược. C dùng chung chunk với B nên nhãn giống B.
- *Metadata:* 5 câu test × top-3 = 15 bản ghi mỗi chiến lược; đối chiếu **bằng code**: text chunk có nằm đúng trang PDF đã ghi không, `source` có khớp URL manifest không, `year`/`section` có khớp manifest không.
- *Grounding:* 5 câu test, chấm cả top-3 của từng chiến lược; đây là độ đủ của bằng chứng truy xuất, không phải độ đúng câu trả lời của LLM.

## Phân tích và kết luận

- **A–B (đổi chunking).** B hơn A ở mọi chỉ số tự động: Hit@1 dev 11/15 so với 8/15, Hit@3 dev 15/15 so với 14/15, test 5/5 so với 4/5, với ít chunk hơn 34% (370 so với 558) và ingest nhanh hơn khoảng 24 s. Coherence 1,6 so với 1,0: cả 10 chunk A đều bắt đầu hoặc kết thúc giữa từ/giữa câu, trong khi 6/10 chunk B trọn ý; 4 chunk B còn lại bị điểm 1 chủ yếu do nhiễu từ PDF (nhãn hình, chú thích, hết trang), không phải do cắt giữa câu.
  - *Ví dụ sai 1 — q17 (test), A, 2025 tr.52:* top-2 `…p0052:fixed500-50:002` kết thúc ở "…additional generation supply by 9–", con số "11 percent" rơi sang chunk sau không được truy xuất. Trang gold có trong top-3 nhưng không chunk nào chứa đủ evidence → miss (trace đánh dấu `evidence_split_suspected`). B giữ trọn câu và hit ở top-1.
  - *Ví dụ sai 2 — q10 (dev), A, 2025 tr.58:* cả top-3 đều là chunk ở tr.44 nói chung về xe hai bánh; đoạn khuyến nghị "banning new ICE mopeds from 2027" ở tr.58 không vào top-3 → miss. Câu hỏi diễn đạt lại ("petrol-engine", "stopping sales") trong khi nguồn viết "ICE", "banning". B cũng chỉ tìm được ở hạng 3.
- **B–C (thêm filter).** Trên các chỉ số bắt buộc B và C **bằng nhau**: Hit@1 và Hit@3 giống hệt ở cả dev và test, coherence và grounding giống nhau vì dùng chung chunk. Khác biệt chỉ thấy ở 3 câu có filter: B để lọt chunk của báo cáo sai năm vào top-3 ở cả hai câu dev (q14 top-1 là 2025 tr.32; q15 top-3 là 2023 tr.12), C thì không. Với q14, hạng của chunk đúng tăng từ 3 lên 2. Ở câu test duy nhất có filter (q19) B tình cờ trả về toàn chunk 2025 nên B và C trùng nhau; tức **trên test không quan sát được lợi ích nào của filter**.
- **Metadata hữu dụng ở đâu và hạn chế.** `year` giúp loại số liệu sai kỳ khi câu hỏi mơ hồ (q14, q15); `source` + `page` cho phép mở đúng trang PDF để kiểm chứng (cả 45 bản ghi kiểm tra đều khớp). Hạn chế: `page` là trang vật lý, lệch với số in trên trang ở báo cáo 2025 (trang vật lý 52 in số 42), người đọc trích dẫn dễ nhầm; `section` chỉ có 3 giá trị thô và chưa được dùng trong câu hỏi nào; `document_type` là hằng số nên không có giá trị lọc.
- **Chiến lược chọn.** Theo tiêu chí đã chốt, B và C hòa nhau ở cả ba bước (cùng Hit@3, Hit@1, số chunk) và cùng hơn A. Tôi chọn **C**, vì nó là B cộng thêm bảo đảm không trả chunk sai kỳ khi người dùng đã chỉ định kỳ báo cáo, không tốn thêm tài nguyên. Lưu ý trung thực: lý do phá thế hòa này (tuân thủ filter) **không nằm trong tiêu chí đã chốt trước**; nếu chỉ xét tiêu chí gốc thì kết luận là "B hoặc C, đều hơn A". Đánh đổi: C cần giá trị filter đúng từ người dùng; filter sai năm sẽ loại hết bằng chứng đúng. Chunk theo đoạn dài hơn (trung vị 691,5 ký tự) nên mỗi kết quả tốn ngữ cảnh hơn A.
- **Hạn chế.**
  - *Nhãn:* coherence và grounding chỉ được chấm một lượt, chưa có người chấm thứ hai độc lập; việc chấm không ẩn tên chiến lược nên có nguy cơ thiên lệch. Coherence của A đồng loạt bằng 1 cho thấy thang 0–2 phân biệt kém ở mẫu này.
  - *Mẫu nhỏ:* test chỉ 5 câu, 1 câu có filter; chênh lệch A–B trên test là 1 câu. Không thể kết luận B/C tốt hơn nói chung, chỉ trên corpus và bộ câu hỏi này. Mẫu coherence chỉ lấy từ 10 trang đầu của báo cáo 2023.
  - *Benchmark:* câu hỏi được viết sau khi đã đọc nguồn, nên có thể dễ hơn câu hỏi thật; 20 câu đều bằng tiếng Anh như corpus, chưa thử truy vấn tiếng Việt. Hit yêu cầu khớp nguyên văn `evidence_quote`, nên bằng chứng tương đương nhưng khác chữ sẽ bị tính miss nếu chưa liệt kê trong `alternatives`.
  - *Đo đạc:* thời gian đo một lần trên CPU máy cá nhân, không lặp lại; không đo thời gian truy vấn. Chroma dùng HNSW xấp xỉ; hai lần chạy đầy đủ cho cùng chỉ số và cùng top-3 trên các câu test, nhưng không bảo đảm tuyệt đối.
  - *Tái lập:* PDF không nằm trong repo; trace và bảng audit chứa các trích đoạn ngắn của báo cáo World Bank để người chấm đối chiếu.
- **Nếu làm lại hoặc mở rộng:** lọc header/footer, nhãn biểu đồ và chú thích trước khi chunk; cho chunk nối qua ranh giới trang (q11 cho thấy câu trong phần tổng quan bị cắt giữa tr.13–14); thử A và B cùng ngân sách ký tự để tách ảnh hưởng của kích thước khỏi ranh giới; thêm câu hỏi tiếng Việt và nhiều câu test có filter hơn; chạy bản pgvector với tìm kiếm chính xác để đối chiếu với HNSW; thử hybrid BM25 cho các câu diễn đạt lại như q10; nhờ một người khác chấm lại coherence/grounding mà không biết tên chiến lược.
