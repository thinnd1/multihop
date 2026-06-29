# Giải thích bài báo ViHERMES

## Ý tưởng chính

Bài báo giải quyết một vấn đề thực tế: khi người dùng hỏi về quy định y tế ở Việt Nam, câu trả lời thường không nằm gọn trong một văn bản duy nhất. Ví dụ, để biết một cơ sở khám chữa bệnh cần tuân thủ những gì, bạn phải đọc quy định về giấy phép ở nghị định A, quy định về chất lượng ở thông tư B, và quy định về giá dịch vụ ở thông tư C — rồi tổng hợp lại. Đây gọi là **suy luận đa bước (multihop reasoning)**.

Vấn đề là: chưa có bộ dữ liệu tiếng Việt nào được thiết kế riêng để đánh giá khả năng suy luận đa bước trên văn bản pháp quy y tế, và các hệ thống RAG thông thường cũng không hiểu được cấu trúc pháp lý (sửa đổi, bổ sung, dẫn chiếu giữa các văn bản).

Nhóm tác giả đóng góp hai thứ: một **bộ dữ liệu benchmark** (ViHERMES) và một **hệ thống hỏi đáp** dựa trên đồ thị tri thức pháp quy.

---

## Kiến trúc tổng thể

Hệ thống gồm hai phần lớn:

**Phần 1 — Xây dựng bộ dữ liệu ViHERMES** (5 bước)

Quy trình đi từ thu thập văn bản thô → tiền xử lý thành các đơn vị pháp lý (điều, khoản) → gom cụm ngữ nghĩa bằng Spherical K-Means để tìm các nhóm văn bản liên quan → lấy mẫu các bộ context từ nhiều văn bản khác nhau rồi đưa cho LLM sinh câu hỏi-đáp đa bước → chuyên gia kiểm tra và chia train/test. Mỗi mẫu trong dataset gồm câu hỏi, câu trả lời, số hop, evidence trích dẫn, và chuỗi suy luận — tất cả đều được gán nhãn tường minh.

**Phần 2 — Hệ thống hỏi đáp dựa trên đồ thị (Graph-Aware QA System)**

Trung tâm là một **đồ thị tri thức pháp quy (SRKG)** với ba loại node (Document, Clause, Article) và hai nhóm quan hệ: cấu trúc (Has_Article, Has_Clause) và ngữ nghĩa pháp lý (Amends, Replaces, Supplements, Refers_To). Các quan hệ này được trích xuất bằng rule-based pattern matching trên các mẫu câu pháp lý chuẩn, không dùng LLM — nên độ chính xác cao.

Quá trình trả lời câu hỏi theo mô hình **Seeded Retrieval + Propagation**:

1. **Seeded Retrieval**: Tìm top-K node liên quan nhất bằng điểm kết hợp dense (embedding) + sparse (BM25).
2. **Propagation**: Từ các seed node, mở rộng theo ba hướng — truy vết chuỗi sửa đổi/thay thế để tìm phiên bản hiệu lực mới nhất; mở rộng sang node được dẫn chiếu (chỉ 1 bước để tránh lan tràn); và truy ngược lên document gốc để lấy metadata.

Toàn bộ được điều phối bởi **hệ đa tác tử (MAS)** gồm 4 agent:

- **Interpreter**: Phân tích ý định câu hỏi, quyết định có cần truy vấn đồ thị không.
- **Pathfinder**: Thực hiện seeded retrieval + propagation trên SRKG.
- **Auditor**: Kiểm tra tính nhất quán và phát hiện hallucination.
- **Conductor**: Điều phối tổng thể, gọi LLM sinh câu trả lời cuối cùng.

---

## Ưu điểm

- **Lấp khoảng trống quan trọng**: Đây là benchmark đầu tiên cho QA đa bước trên văn bản pháp quy y tế tiếng Việt — một ngôn ngữ ít tài nguyên. Trước đó không có dataset nào kết hợp cả hai yếu tố "y tế pháp quy" và "suy luận đa bước".
- **Đồ thị tri thức dựa trên cấu trúc pháp lý thật**, không phải đồ thị thực thể tự động trích xuất kiểu GraphRAG/LightRAG. Điều này giúp truy vết chuỗi sửa đổi, bổ sung một cách chính xác — rất phù hợp với đặc thù văn bản luật.
- **Cơ chế propagation có kiểm soát**: Validity tracing đi theo chuỗi sửa đổi đến node cuối cùng (phiên bản hiệu lực), trong khi Refers_To chỉ mở rộng 1 bước — tránh "context drift" mà nhiều hệ thống graph-RAG khác gặp phải.
- **Kết quả thực nghiệm mạnh**: F1 đạt 0.8334, vượt HippoRAG 2 (0.8023) và LightRAG (0.7855), đồng thời latency chỉ ~14.7s — thấp hơn cả HippoRAG 2 (22s) dù đồ thị lớn hơn.
- **Ablation study rõ ràng**: Cho thấy mỗi component đều có đóng góp cụ thể, đặc biệt Interpreter (bỏ đi thì F1 giảm từ 0.83 xuống 0.65).

---

## Nhược điểm / Hạn chế

- **Phụ thuộc vào rule-based extraction**: Quan hệ pháp lý được trích bằng pattern matching trên các mẫu câu chuẩn. Nếu văn bản viết không theo đúng mẫu chuẩn hoặc dùng cách diễn đạt khác, hệ thống sẽ bỏ sót quan hệ.
- **Chưa xử lý hiệu lực theo thời gian (temporal validity)**: Chính tác giả thừa nhận ở phần kết luận rằng hệ thống chưa mô hình hóa rõ ràng thời gian có hiệu lực của từng quy định — một yếu tố rất quan trọng trong thực tế pháp lý.
- **Dataset sinh bởi LLM rồi kiểm duyệt**: Dù có expert review, bản chất pipeline vẫn dùng LLM sinh câu hỏi-đáp, nên chất lượng phụ thuộc vào khả năng của LLM với tiếng Việt pháp lý. Không rõ tỷ lệ bị loại sau kiểm duyệt là bao nhiêu.
- **Phạm vi hẹp**: Chỉ tập trung vào pháp quy y tế Việt Nam. Tác giả nói có thể mở rộng sang domain khác, nhưng chưa kiểm chứng.
- **LLM backbone là GPT-4o-mini**: Kết quả có thể khác đáng kể nếu dùng model khác. Chưa có thí nghiệm so sánh trên nhiều LLM backbone.
- **Chưa đánh giá trên user thật**: Toàn bộ đánh giá là tự động (F1, LLM-as-Judge), chưa có human evaluation end-to-end từ phía người dùng thực tế (bác sĩ, nhân viên y tế).

## Ưu điểm

**Về mặt đóng góp khoa học**, bài báo lấp đúng một khoảng trống rõ ràng: trước ViHERMES, không có benchmark nào cho QA đa bước trên văn bản pháp quy y tế tiếng Việt. Nhóm tác giả không chỉ tạo dataset mà còn đề xuất cả hệ thống kèm theo, tạo thành một "package" hoàn chỉnh cho cộng đồng nghiên cứu.

**Thiết kế đồ thị tri thức SRKG rất phù hợp với bài toán.** Thay vì dùng LLM trích xuất entity-relation tự động như GraphRAG hay LightRAG (dễ sinh ra quan hệ sai hoặc thiếu), nhóm tác giả dựa vào cấu trúc pháp lý có sẵn trong văn bản luật — vốn đã được viết theo quy chuẩn rõ ràng (điều, khoản, sửa đổi, bổ sung). Cách tiếp cận rule-based này cho độ chính xác cao hơn hẳn trong domain pháp quy so với các phương pháp trích xuất tự động.

**Cơ chế propagation ba hướng được thiết kế có chủ đích.** Validity tracing đi theo chuỗi sửa đổi đến phiên bản cuối cùng (hiệu lực), contextual supplementation chỉ mở rộng 1 bước qua Refers_To để tránh lan tràn, và provenance retrieval lấy metadata gốc. Mỗi hướng giải quyết một nhu cầu thực tế cụ thể trong suy luận pháp lý, không phải mở rộng đồ thị một cách tùy tiện.

**Pipeline sinh dataset cũng đáng chú ý.** Việc dùng semantic clustering để gom nhóm các đơn vị pháp lý liên quan trước khi sinh câu hỏi giúp đảm bảo các context trong một bộ thực sự có mối liên hệ ngữ nghĩa, chứ không phải ghép ngẫu nhiên. Ràng buộc các context phải đến từ văn bản khác nhau cũng buộc câu hỏi phải thực sự đa bước liên văn bản.

**Kết quả thực nghiệm thuyết phục.** Hệ thống đạt F1 cao nhất (0.8334) và Recall@5 cao nhất (0.8461), đồng thời latency chỉ 14.7s — thấp hơn đáng kể so với HippoRAG 2 (22s) dù đồ thị có nhiều node và edge hơn. Ablation study cũng cho thấy rõ vai trò của từng component: bỏ Interpreter thì F1 rơi gần 18 điểm, bỏ Auditor thì LLM Judge giảm mạnh — mỗi agent đều có lý do tồn tại.

**Tính mở và tái tạo được.** Dataset và code được công khai trên GitHub, giúp cộng đồng có thể kiểm chứng và xây dựng tiếp.

---

## Nhược điểm

**Rule-based extraction là con dao hai lưỡi.** Đúng là nó chính xác với văn bản viết đúng chuẩn, nhưng trong thực tế, không phải mọi văn bản pháp quy đều tuân thủ nghiêm ngặt mẫu câu chuẩn. Một thông tư viết "điều chỉnh nội dung tại Điều X" thay vì "sửa đổi Điều X" có thể bị bỏ sót. Bài báo không phân tích tỷ lệ coverage của rule-based extraction trên corpus thực tế, nên không biết hệ thống bỏ lọt bao nhiêu quan hệ.

**Thiếu mô hình hóa thời gian hiệu lực.** Chính tác giả thừa nhận điểm này ở phần kết luận. Trong thực tế pháp lý, một điều khoản có thể chỉ có hiệu lực từ ngày X đến ngày Y, hoặc bị đình chỉ tạm thời. Validity tracing hiện tại chỉ đi theo chuỗi sửa đổi đến node cuối cùng, nhưng không kiểm tra liệu node đó có đang còn hiệu lực tại thời điểm hỏi hay không. Đây là một thiếu sót khá nghiêm trọng cho một hệ thống pháp quy.

**Dataset sinh bằng LLM có rủi ro về chất lượng.** Câu hỏi và câu trả lời được LLM tạo ra rồi con người kiểm duyệt, nhưng bài báo không báo cáo tỷ lệ reject rate, inter-annotator agreement, hay bất kỳ chỉ số đo chất lượng annotation nào. Không rõ bao nhiêu mẫu bị loại, tiêu chí loại cụ thể ra sao, và liệu expert review có đủ sâu để phát hiện lỗi pháp lý tinh vi hay không.

**Đánh giá phụ thuộc nhiều vào LLM-as-Judge.** F1 chỉ đo overlap bề mặt ở mức token, còn LLM Judge dùng GPT-4o đánh giá correctness/completeness. Nhưng GPT-4o đánh giá văn bản pháp lý tiếng Việt tốt đến đâu thì chưa được kiểm chứng. Không có human evaluation từ chuyên gia pháp lý để đối chiếu với điểm LLM Judge, nên không biết metric này đáng tin cậy đến mức nào trong domain này.

**Chỉ dùng một LLM backbone duy nhất (GPT-4o-mini).** Tất cả các hệ thống đều chạy trên cùng một model, nên kết quả phản ánh hiệu quả của phương pháp retrieval hơn là khả năng tổng quát. Nếu đổi sang model khác (Gemini, Claude, hoặc model mã nguồn mở), thứ hạng giữa các phương pháp có thể thay đổi. Đặc biệt, GPT-4o-mini không phải model mạnh nhất cho tiếng Việt.

**Phạm vi domain hẹp và chưa kiểm chứng khả năng tổng quát hóa.** Tác giả nói SRKG có thể áp dụng cho domain pháp quy khác, nhưng toàn bộ thiết kế — từ rule-based extraction đến cấu trúc node — đều gắn chặt với đặc thù văn bản pháp quy y tế Việt Nam. Chưa có thí nghiệm nào trên domain khác (thuế, lao động, giáo dục) để chứng minh tính tổng quát.

**Quy mô corpus và test set chưa lớn.** Test set có 1560 mẫu chia đều cho 5 mức hop (312 mẫu/mức). Với domain pháp quy phức tạp, số lượng này có thể chưa đủ để đánh giá toàn diện, đặc biệt ở các mức hop cao (4-5) nơi diversity của reasoning pattern bị hạn chế.

**Thiếu phân tích lỗi định tính.** Bài báo báo cáo số liệu tổng hợp nhưng không đưa ra case study về những trường hợp hệ thống trả lời sai, propagation đi lệch hướng, hay Auditor bỏ sót hallucination. Phân tích lỗi sẽ giúp hiểu rõ hơn giới hạn thực sự của hệ thống.

---

Tóm lại, bài báo có đóng góp rõ ràng và thiết kế hệ thống hợp lý cho domain cụ thể, nhưng còn nhiều điểm cần bổ sung — đặc biệt về temporal validity, chất lượng annotation, và khả năng tổng quát hóa — trước khi có thể triển khai trong thực tế.