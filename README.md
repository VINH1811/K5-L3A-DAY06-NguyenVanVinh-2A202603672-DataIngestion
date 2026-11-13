# Day 06 · Data Ingestion Engineering

**Lab cá nhân** · **4 giờ** · **Dữ liệu tự chọn**

Biến tài liệu thành kho tri thức có thể truy xuất cho AI agent, rồi đo xem cách chuẩn bị dữ liệu nào hiệu quả nhất.

| 📄 Dữ liệu | 🧩 Chiến lược | 🎯 Benchmark |
|---|---|---|
| Tối thiểu **30 trang nội dung** | Tối thiểu **3 cấu hình** | Tối thiểu **20 câu hỏi** |

## Bài nộp — cách setup và chạy lại

Bài làm của Nguyễn Văn Vinh (2A202603672). Báo cáo: [report/individual/2A202603672_NguyenVanVinh.md](report/individual/2A202603672_NguyenVanVinh.md).

| Thành phần | Vị trí |
|---|---|
| Code pipeline | [starter/src/](starter/src/) (điểm vào: `run.py`) |
| Manifest corpus | [data/manifest.json](data/manifest.json) |
| Benchmark 20 câu + gold | [benchmark/questions.json](benchmark/questions.json) |
| Cấu hình A/B/C | `STRATEGIES` trong [starter/src/run.py](starter/src/run.py), [artifacts/experiment_log.csv](artifacts/experiment_log.csv) |
| Kết quả, trace, nhãn audit | [results/](results/) |

Corpus là hai báo cáo "Taking Stock" của World Bank (8/2023 và 3/2025). File PDF **không nằm trong repo**; tải lại theo URL trong manifest. Embedding: `google/embeddinggemma-2` chạy local (768 chiều, chi phí 0). Vector store: Chroma lưu trên đĩa tại `starter/chroma_db/` (máy không có Docker nên không dùng pgvector).

Cần Python 3.11–3.13, khoảng 5 GB đĩa trống và mạng cho lần đầu (thư viện + model ~1,5 GB). Chạy trong PowerShell:

```powershell
cd starter
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
Copy-Item .env.example .env        # rồi đặt EMBEDDING_MODEL=google/embeddinggemma-2 và VECTOR_STORE=chroma
curl.exe -L -o data\wb-taking-stock-2023-08.pdf "https://documents.worldbank.org/curated/en/099408408092319382/pdf/IDU0301a3e760c732046e509d3106e81810de4b9.pdf"
curl.exe -L -o data\wb-taking-stock-2025-03.pdf "https://documents.worldbank.org/curated/en/099919303102515323/pdf/IDU-409e0901-e76f-4c35-b13a-248c3fb8c4c0.pdf"
python -m src.run --split dev      # chỉ chấm 15 câu dev
python -m src.run --split all      # dev + test, sinh bảng audit nếu chưa có
```

Mỗi lần chạy mất khoảng 6 phút trên CPU. `run.py` dừng với thông báo lỗi nếu corpus dưới 30 trang nội dung, benchmark sai cấu trúc hoặc có `evidence_quote` không nằm trên trang gold. Mã SHA-256 của hai PDF đã dùng được ghi trong `results/run_info.json`.

Ba bảng `results/audit_*.csv` chứa nhãn chấm tay và không bị ghi đè khi chạy lại. Nhãn hiện có là **bản nháp do AI (Claude) chấm**, ghi bằng [starter/scripts/fill_audit_labels.py](starter/scripts/fill_audit_labels.py); xem mục hạn chế trong báo cáo.

---

Phần dưới đây là đề bài gốc.

## Bắt đầu nhanh

Dùng [starter PostgreSQL + pgvector](starter/README.md) để dựng database và hoàn thiện các hàm TODO, hoặc tự chọn vector store tương đương. Starter chỉ là khung: **bạn tự hiện thực pipeline và benchmark**.

## 1. Xây pipeline

![Đọc tài liệu, chuẩn hóa, chia chunk, tạo vector, lưu và kiểm tra](assets/ingestion-pipeline.svg)

Chọn tài liệu được phép sử dụng; giữ **ID tài liệu, nguồn, trang/vị trí** và metadata phù hợp. Dùng embedding model thật qua API hoặc chạy local; lưu và truy xuất bằng pgvector, Qdrant, Chroma hoặc một store tương đương.

## 2. So sánh và đánh giá

![Ba chiến lược được đánh giá trên cùng corpus và benchmark dev/test](assets/benchmark-workflow.svg)

Dùng cùng dữ liệu, model và câu hỏi cho cả ba cấu hình. Tách **15 câu dev / 5 câu test** trước khi chạy; chọn cấu hình trên dev, rồi đánh giá test sau khi cố định. Mỗi câu hỏi cần có đoạn bằng chứng và trang nguồn; thiết kế bộ câu hỏi và đánh giá theo [BENCHMARK.md](BENCHMARK.md).

Đánh giá theo năm khía cạnh: **độ chính xác truy xuất**, **tính mạch lạc của chunk**, **tính hữu dụng của metadata**, **chất lượng grounding** và **tác động của chiến lược dữ liệu**. Giải thích lựa chọn bằng kết quả, ví dụ lỗi và các đánh đổi.

## 3. Chuẩn bị bài nộp

1. Chạy pipeline và lưu cấu hình, kết quả top-3, trace cùng nhãn đánh giá thủ công.
2. Điền [thông tin học viên](STUDENT.md), [nhật ký thí nghiệm](artifacts/experiment_log.csv) và bản sao [mẫu báo cáo cá nhân](report/individual/MSSV_HoTen.md).
3. Kiểm tra mã nguồn, cách chạy, manifest dữ liệu và checklist trong [SUBMISSION.md](SUBMISSION.md).

## Tài liệu hướng dẫn

| Bạn cần tìm | Tài liệu |
|---|---|
| Setup baseline PostgreSQL + pgvector | [starter/README.md](starter/README.md) |
| Các bước làm và cách tính chỉ số | [GUIDE.md](GUIDE.md) |
| Tiến độ trong 4 giờ | [CHECKPOINTS.md](CHECKPOINTS.md) |
| Năm tiêu chí benchmark và bộ câu hỏi | [BENCHMARK.md](BENCHMARK.md) |
| Tiêu chí chấm điểm | [RUBRIC.md](RUBRIC.md) |
| Quy tắc sử dụng dữ liệu, AI và API | [RULES.md](RULES.md) |
| Cách nộp bài | [SUBMISSION.md](SUBMISSION.md) |

[data/format-example.json](data/format-example.json) chỉ minh họa định dạng, không phải corpus đủ điều kiện. Phần xử lý và benchmark do bạn tự hiện thực; không bắt buộc xây chatbot.
