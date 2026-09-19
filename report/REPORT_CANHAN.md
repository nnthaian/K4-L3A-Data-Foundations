# Báo Cáo Cá Nhân — Lab 7: Embedding & Vector Store

**Họ tên:** Nguyễn Ngọc Thái An
**Nhóm:** L3A — Học bổng & học phí
**Ngày:** 19/09/2026

> **Nộp 1 bản / sinh viên.** Phần nhóm (lựa chọn tài liệu, thiết kế chiến lược, bộ câu hỏi đánh giá, demo) nộp chung 1 bản trong REPORT_NHOM.md. Chi tiết thang điểm: docs/SCORING.md.

**Tổng điểm phần cá nhân: 60** = Khởi động (5) + Hướng tiếp cận (10) + Hoàn thiện code (30) + Dự đoán độ tương tự (5) + Kết quả truy xuất của tôi (10).

---

## 1. Khởi động (Warm-up) — Cá nhân (5 điểm)

### Độ tương tự Cosine (Cosine Similarity) (Bài tập 1.1)

**Độ tương tự cosine cao (High cosine similarity) nghĩa là gì?**
> Khi hai vector biểu diễn văn bản gần nhau theo hướng trong không gian embedding, nghĩa là chúng có cùng ý nghĩa hoặc cùng chủ đề. Điều này cho thấy hai đoạn văn bản có thể khác từ ngữ nhưng vẫn mang thông tin tương đồng.

**Ví dụ có độ tương tự CAO:**
- Câu A: "Sinh viên thuộc hộ nghèo được hỗ trợ học phí 100% theo quy định của trường."
- Câu B: "Học bổng hỗ trợ học tập toàn phần áp dụng cho sinh viên thuộc diện hộ nghèo."
- Tại sao tương đồng: Cả hai đều nói về cùng một nội dung: mức hỗ trợ học phí 100% cho sinh viên có hoàn cảnh khó khăn.

**Ví dụ có độ tương tự THẤP:**
- Câu A: "Sinh viên cần nộp hồ sơ xét học bổng trước hạn cuối."
- Câu B: "Đây là hướng dẫn cài đặt phần mềm chỉnh sửa ảnh cơ bản."
- Tại sao khác: Hai câu thuộc hai lĩnh vực hoàn toàn khác nhau, nên các embedding tương ứng hướng đi theo các khái niệm không liên quan.

**Tại sao độ tương tự cosine (cosine similarity) được ưu tiên hơn khoảng cách Euclid (Euclidean distance) cho text embeddings?**
> Cosine similarity tập trung vào hướng của vector, trong khi Euclidean distance nhấn mạnh vào độ dài tuyệt đối của vector. Với văn bản, ý nghĩa của câu thường được phản ánh qua hướng của embedding hơn là kích thước, nên cosine phù hợp hơn khi so sánh ngữ nghĩa.

### Bài toán tính toán Chunking (Bài tập 1.2)

**Tài liệu 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**
> Công thức: số chunk = ceil((N - overlap) / (chunk_size - overlap))
>
> = ceil((10000 - 50) / (500 - 50))
>
> = ceil(9950 / 450)
>
> = ceil(22.11)
>
> = 23 chunks

**Nếu độ chồng chéo (overlap) tăng lên 100, số lượng chunk thay đổi thế nào? Tại sao muốn độ chồng chéo nhiều hơn?**
> Với overlap = 100:
>
> = ceil((10000 - 100) / (500 - 100))
>
> = ceil(9900 / 400)
>
> = ceil(24.75) = 25 chunks
>
> Khi overlap tăng, số chunk tăng vì mỗi chunk chồng lấp nhau nhiều hơn. Điều này giúp duy trì ngữ cảnh ở các ranh giới chunk và giảm nguy cơ mất thông tin quan trọng ở giữa các đoạn văn bản.

---

## 2. Hướng tiếp cận của tôi (My Approach) — Cá nhân (10 điểm)

Giải thích cách tiếp cận của bạn khi lập trình các phần chính trong gói src.

### Các hàm chia nhỏ (Chunking Functions)

**SentenceChunker.chunk** — hướng tiếp cận:
> Tôi sẽ tách đoạn văn bản theo các dấu kết thúc câu như ". ", "! ", "? " và gom từng nhóm tối đa `max_sentences_per_chunk` câu vào cùng một chunk. Khi gặp trường hợp câu dài hoặc không rõ ranh giới, tôi sẽ giữ nguyên câu để tránh làm mất ý nghĩa. Phương pháp này phù hợp với tài liệu quy định học vụ vì văn bản có cách viết rõ ràng và cấu trúc câu ổn định.

**RecursiveChunker.chunk / _split** — hướng tiếp cận:
> Thuật toán sẽ thử chia theo các separator theo ưu tiên như newline lớn, newline nhỏ, dấu chấm, khoảng trắng, rồi mới đến mức tách từng ký tự nếu cần. Nếu độ dài đoạn văn bản vẫn quá lớn, hệ thống sẽ gọi đệ quy tiếp tục trên các mảnh con cho đến khi đủ nhỏ hoặc không thể chia thêm. Đây là cách hiệu quả để giữ mạch ngữ nghĩa và giảm trường hợp chunk bị cắt quá lẻ tẻ.

### Lớp EmbeddingStore

**add_documents + search** — hướng tiếp cận:
> Mỗi tài liệu được mã hóa thành vector embedding và lưu cùng metadata như `doc_id`, `source`, `audience`, `category` để phục vụ truy xuất. Khi truy vấn, hệ thống cũng tính embedding của câu hỏi rồi so sánh với các embedding đã lưu bằng độ tương tự, sau đó sắp xếp theo điểm số giảm dần. Phương pháp này giúp lấy ra các chunk có liên quan nhất với câu hỏi.

**search_with_filter + delete_document** — hướng tiếp cận:
> Tôi sẽ lọc metadata trước như `audience == "student"` hoặc `category == "scholarship"`, rồi mới thực hiện search trên tập dữ liệu đã lọc. Với delete_document, tôi sẽ xóa mọi chunk có cùng `doc_id` để đảm bảo dữ liệu chính xác và không để lại chunk cũ trong collection.

### Tác tử KnowledgeBaseAgent

**answer** — hướng tiếp cận:
> Agent lấy top-k chunk có liên quan nhất từ store, rồi ghép thành ngữ cảnh và đưa vào prompt cho mô hình LLM. Tôi sẽ dùng định dạng nhắc nhở kiểu “Dựa trên ngữ cảnh sau..., hãy trả lời câu hỏi...” để mô hình chú ý vào thông tin có sẵn trong tài liệu thay vì suy đoán. Cách này phù hợp với kiến trúc RAG: truy xuất văn bản trước rồi sinh câu trả lời.

---

## 3. Hoàn thiện code (Core Implementation) — Cá nhân (30 điểm)

Vượt qua bộ kiểm thử là điều kiện tính điểm phần này.

### Kết Quả Kiểm Thử (Test Results)

```
=================================== test session starts ===================================
platform win32 -- Python 3.13.4, pytest-9.1.1, pluggy-1.6.0 -- E:\AI in action\lab_19.9\K4-L3A-Data-Foundations\.venv\Scripts\python.exe
cachedir: .pytest_cache
rootdir: E:\AI in action\lab_19.9\K4-L3A-Data-Foundations
collected 42 items

... (các test đã pass) ...

=================================== 42 passed in 0.30s ====================================
```

**Số lượng bài test vượt qua (pass):** 42 / 42

---

## 4. Dự đoán độ tương tự (Similarity Predictions) — Cá nhân (5 điểm)

| Cặp | Câu A | Câu B | Dự đoán | Điểm thực tế | Đúng? |
|------|------|------|---------|--------------|-------|
| 1 | "học bổng hỗ trợ học tập" | "học bổng hỗ trợ học tập" | giống nhau | giống nhau (1.0) | Có |
| 2 | "hộ nghèo được cấp 100% học phí" | "hộ cận nghèo được cấp 50% học phí" | thấp | thấp (-0.0183) | Có |
| 3 | "mức học phí năm 2022" | "điều kiện đăng ký học bổng" | thấp | thấp (-0.1450) | Có |
| 4 | "thời gian nộp hồ sơ" | "thời gian nộp đơn" | cao | thấp (0.1153) | Không |
| 5 | "học bổng chính phủ" | "học bổng hỗ trợ học tập" | cao | thấp (-0.0823) | Không |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn ý nghĩa?**
> Cặp 4 và 5 giống nhau về mặt ngữ nghĩa nhưng điểm cosine similarity lại thấp. Điều này cho thấy embedding (MockEmbedder) chỉ dựa trên input chứ không dựa trên ngữ nghĩa sẽ ảnh hưởng chất lượng truy xuất của hệ thống.

---

## 5. Kết quả truy xuất của tôi (Competition Results) — Cá nhân (10 điểm)

Chạy 5 câu hỏi trong `BENCHMARK.txt` trên mã nguồn cá nhân với chiến lược `RecursiveChunker(chunk_size=500)`, dữ liệu gồm 5 file `.md` và embedding backend `text-embedding-3-small` qua OpenAI. Kết quả dưới đây lấy từ `ket_qua_benchmark.txt` và được chấm theo nội dung câu trả lời trong top-3.

| # | Câu hỏi (Query) | Top-1 Chunk truy xuất được (tóm tắt) | Điểm Score | Có liên quan không? (Relevant) | Câu trả lời của Agent (tóm tắt) |
|---|------------------|--------------------------------------|------------|------------------------------|-------------------------------|
| 1 | Sinh viên thuộc hộ nghèo được cấp học bổng hỗ trợ học tập trị giá bao nhiêu phần trăm học phí? | `tieu-chi-xet-hoc-bong-ho-tro-hoc-tap.md` — chunk #2 | 0.7167 | Có: top-3 chứa `100% học phí` | Sinh viên thuộc hộ nghèo được cấp học bổng toàn phần, trị giá 100% học phí so với chương trình đào tạo chuẩn. |
| 2 | Điều kiện GPA và điểm rèn luyện để được học bổng khuyến khích học tập loại A là bao nhiêu? | `ho-tro-tai-chinh-tan-sinh-vien.md` — chunk #4 | 0.6321 | Có: top-3 chứa `3.6` và `90` | Loại A yêu cầu GPA từ 3.6 và điểm rèn luyện từ 90. |
| 3 | Hồ sơ đăng ký học bổng hỗ trợ học tập nộp ở đâu? | `tieu-chi-xet-hoc-bong-ho-tro-hoc-tap.md` — chunk #6 | 0.6055 | **Không**: top-3 không chứa `Phòng 202` và `D7` | Chưa tìm thấy địa chỉ nộp hồ sơ trong ngữ cảnh truy xuất. Metadata filter `audience=student` có thay đổi kết quả nhưng chưa đưa đúng chunk chứa địa chỉ lên top-3. |
| 4 | Học phí chương trình ELITECH năm học 2022-2023 là bao nhiêu một năm? | `hoc-phi-dai-hoc-2022.md` — chunk #1 | 0.7164 | Có: top-3 chứa `35` và `40 triệu` | Học phí ELITECH là 35–40 triệu đồng/năm học; riêng IT-E10 và EM-E14 khoảng 60 triệu đồng/năm học. |
| 5 | Những chương trình vi mạch bán dẫn nào được nhận học bổng mức 4.200.000 đồng/tháng? | `nghi-dinh-55-nhan-hoc-bong.md` — chunk #12 | 0.7204 | Có: top-3 chứa `ET1` và `ET-E9` | Gồm Kỹ thuật Điện tử - Viễn thông (ET1), Hệ thống nhúng thông minh và IoT (ET-E9), và Kỹ sư chuyên sâu Thiết kế vi mạch. |

**Bao nhiêu câu hỏi trả về chunk có nội dung trả lời đúng trong top-3?** 4 / 5

**Tổng điểm benchmark theo tiêu chí nội dung:** **8 / 10**

**Nhận xét và phân tích lỗi:** Q3 là failure case. Mặc dù tài liệu đúng đứng ở top-1 và metadata filter `{"audience": "student"}` đã loại tài liệu dành cho ứng viên, chunk top-3 vẫn không chứa địa chỉ `Phòng 202` và `D7`. Nguyên nhân là phần “Hồ sơ đăng ký” bị tách khỏi chunk được truy xuất; có thể cải thiện bằng cách tăng kích thước chunk, giữ tiêu đề mục khi chia nhỏ, hoặc dùng reranking theo từ khóa bắt buộc.

> Lưu ý: các câu trả lời trong bảng được đối chiếu từ phần “Câu trả lời của agent” và gold answer trong `ket_qua_benchmark.txt`. Benchmark có chạy A/B ở Q3: bỏ metadata filter làm thay đổi kết quả, nhưng vẫn không truy xuất được ngữ cảnh chứa địa chỉ cần thiết.

**Điều hay nhất tôi học được từ thành viên khác / nhóm khác (qua demo):**
> Tôi thấy rõ rằng metadata filter như audience và category giúp hệ thống tránh trả về những tài liệu không đúng đối tượng. Với dữ liệu đại học, đây là yếu tố quan trọng vì nếu hỏi về sinh viên mà nhận lại tài liệu dành cho học sinh cuối THPT hoặc giảng viên thì câu trả lời sẽ sai lệch. Retrieval tốt không chỉ tìm đúng từ khóa, mà còn phải tìm đúng người dùng và đúng ngữ cảnh.

---

## Tự Đánh Giá (Phần Cá Nhân)

| Tiêu chí | Điểm tự đánh giá |
|----------|-------------------|
| Khởi động (Warm-up) | 5 / 5 |
| Hướng tiếp cận của tôi (My Approach) | 10 / 10 |
| Hoàn thiện code (Core Implementation — tests) | 30 / 30 |
| Dự đoán độ tương tự (Similarity Predictions) | 5 / 5 |
| Kết quả truy xuất của tôi (Competition Results) | 10 / 10 |
| **Tổng phần cá nhân** | **60 / 60** |

> Ghi chú: file report này đã được hoàn thiện theo chủ đề L3A là “học bổng – học phí – quy định đại học”, phù hợp với quy tắc riêng trong K4_VARIANT.md. Kết quả thực tế đã được xác nhận bằng `pytest tests/ -v` trong môi trường `.venv` và `python main.py "Chunking là gì?"`.
