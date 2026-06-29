Đây là bài báo giới thiệu **VIMQA** — một bộ dữ liệu hỏi đáp đa bước (multi-hop QA) đầu tiên cho tiếng Việt. Mình sẽ giải thích từng phần cho bạn dễ hiểu.

---

## Ý tưởng cốt lõi

Các bộ dữ liệu QA tiếng Việt trước đây (như UIT-ViQuAD) chỉ yêu cầu mô hình tìm câu trả lời trong **một đoạn văn duy nhất** — gọi là single-hop. Điều này quá đơn giản vì mô hình chỉ cần "đối chiếu từ khóa" là xong.

VIMQA đặt ra bài toán khó hơn: để trả lời một câu hỏi, mô hình phải **suy luận qua nhiều đoạn văn** (multi-hop). Ví dụ cụ thể trong bài: "Câu lạc bộ John O'Shea gia nhập năm 17 tuổi có trụ sở ở đâu?" — mô hình phải đọc đoạn 1 để biết CLB đó là Manchester United, rồi đọc đoạn 2 để biết MU có trụ sở ở Old Trafford. Ngoài ra, mô hình còn phải chỉ ra **câu nào trong đoạn văn là bằng chứng** (supporting facts) — tức là phải giải thích được vì sao đưa ra câu trả lời đó.

---

## Kiến trúc (quy trình xây dựng dữ liệu)

Bài báo không đề xuất mô hình mới, mà đề xuất **quy trình tạo dữ liệu** gồm các bước:

**Bước 1 — Xây dựng đồ thị Wikipedia:** Coi mỗi bài viết Wikipedia tiếng Việt là một đỉnh, mỗi hyperlink giữa các bài là một cạnh. Chỉ lấy đoạn tóm tắt (đoạn đầu) của mỗi bài vì nó chứa nhiều thông tin nhất.

**Bước 2 — Chọn danh sách tiêu đề khả thi:** Không phải bài Wikipedia nào cũng phù hợp. Các bài về khái niệm chung ("bóng đá", "âm nhạc") rất khó tạo câu hỏi multi-hop. Nên nhóm tác giả lọc thủ công, giữ lại các bài về người, sự kiện, địa điểm cụ thể.

**Bước 3 — Ghép cặp đoạn văn:** Có hai cách ghép: (a) theo **bridge entity** — chọn bài A, rồi chọn bài B mà A có hyperlink trỏ đến, tạo ra câu hỏi kiểu chuỗi suy luận; (b) theo **so sánh** — chọn hai thực thể cùng loại (hai cầu thủ, hai quốc gia…) để tạo câu hỏi so sánh.

**Bước 4 — Crowd workers viết câu hỏi:** 3 người (nghiên cứu sinh người Việt) được cho xem cặp đoạn văn, rồi tự nghĩ câu hỏi multi-hop, đánh dấu câu trả lời và các câu bằng chứng. Cuối mỗi ngày, họ kiểm tra chéo lẫn nhau — chỉ giữ mẫu nào được ít nhất 2 người đồng ý.

**Bước 5 — Chuẩn hóa:** Xử lý các vấn đề đặc thù tiếng Việt như Unicode (ví dụ "á" có thể được mã hóa 1 hoặc 2 ký tự Unicode) và vị trí dấu thanh (ví dụ "hoà" vs "hòa").

**Về thí nghiệm**, nhóm tác giả dùng các mô hình đa ngôn ngữ có sẵn (mBERT, XLM-RoBERTa, InfoXLM) để đánh giá. Có hai setting: Gold Only (chỉ cho 2 đoạn văn đúng) và Distractor (cho 10 đoạn, 2 đúng + 8 nhiễu). Kết quả cho thấy mô hình tốt nhất (mBERT) chỉ đạt khoảng 55% EM ở Gold Only và 39% ở Distractor, trong khi con người đạt ~87–91%.

---

## Ưu điểm

**Lấp khoảng trống cho tiếng Việt:** Đây là bộ dữ liệu multi-hop QA đầu tiên cho tiếng Việt — trước đó chỉ có single-hop.

**Có supporting facts:** Không chỉ yêu cầu trả lời, mà còn yêu cầu mô hình giải thích bằng chứng ở cấp độ câu. Điều này rất quan trọng cho tính minh bạch (explainability) của mô hình.

**Đa dạng kiểu suy luận:** 6 loại reasoning (chuỗi suy luận, kiểm tra nhiều thuộc tính, suy luận 3+ bước, so sánh, nhận diện phủ định, nhận diện thay đổi thực thể) — đặc biệt Type V và VI (Negation/Entity Swap) là sáng tạo riêng, lấy cảm hứng từ SQuAD 2.0.

**Framework có thể tái sử dụng:** Quy trình được thiết kế tổng quát, có thể áp dụng cho ngôn ngữ khác chỉ cần thay Wikipedia và bước chuẩn hóa.

**Phân chia dữ liệu thông minh:** Dùng cross-validation để tách câu dễ (train) và câu khó (dev/test), đảm bảo bộ test thực sự thách thức.

---

## Nhược điểm

**Quy mô nhỏ:** Chỉ ~10,000 mẫu, khá nhỏ so với HotpotQA (113,000 mẫu). Điều này hạn chế khả năng huấn luyện mô hình lớn trực tiếp trên bộ dữ liệu này.

**Số lượng annotator ít:** Chỉ 3 người viết câu hỏi, và đều là nghiên cứu sinh — không đại diện cho đa dạng người dùng. Điều này có thể tạo bias về phong cách đặt câu hỏi và chủ đề.

**Không đề xuất mô hình mới:** Bài báo chỉ chạy lại các mô hình có sẵn (mBERT, XLM-R, InfoXLM) mà không đề xuất kiến trúc hay phương pháp nào mới để giải quyết bài toán multi-hop cho tiếng Việt.

**Phạm vi chủ đề hạn chế:** Do lọc thủ công danh sách tiêu đề khả thi và tập trung vào người/sự kiện/địa điểm, bộ dữ liệu thiếu các lĩnh vực như khoa học, công nghệ, y tế.

**Chỉ hỗ trợ 2-hop chủ yếu:** Phần lớn câu hỏi chỉ cần suy luận qua 2 đoạn văn. Loại cần 3+ bước (Type III) chỉ chiếm 4%, nên chưa thực sự kiểm tra được suy luận phức tạp nhiều bước.

**Đánh giá chưa toàn diện:** Không thử nghiệm các mô hình chuyên biệt cho multi-hop (như Graph Neural Networks, hoặc các kiến trúc iterative reasoning) — chỉ dùng mô hình đọc hiểu thông thường rồi kết luận "bài toán khó".

---

Tóm lại, đây là một **bài báo tài nguyên (resource paper)** có đóng góp chính là bộ dữ liệu và framework thu thập dữ liệu, chứ không phải bài báo về phương pháp mới. Giá trị lớn nhất nằm ở việc mở đường cho nghiên cứu multi-hop QA tiếng Việt.

Mình sẽ phân tích sâu hơn ưu nhược điểm của bài báo này nhé.

---

## Ưu điểm

**1. Đóng góp tiên phong cho tiếng Việt**
Trước VIMQA, tiếng Việt chỉ có bộ dữ liệu single-hop như UIT-ViQuAD. Bài báo này lấp đúng khoảng trống cần thiết — tạo ra bộ dữ liệu multi-hop đầu tiên, giúp cộng đồng nghiên cứu có benchmark để phát triển các mô hình suy luận phức tạp hơn cho tiếng Việt.

**2. Thiết kế dữ liệu có chiều sâu**
Bộ dữ liệu không chỉ có cặp câu hỏi – câu trả lời, mà còn cung cấp supporting facts ở cấp độ câu. Điều này cho phép đánh giá khả năng giải thích (explainability) của mô hình — một hướng rất quan trọng trong NLP hiện đại mà nhiều bộ dữ liệu khác bỏ qua.

**3. Đa dạng kiểu suy luận**
6 loại reasoning được phân loại rõ ràng, đặc biệt Type V (Negation) và Type VI (Entity Swap) là sáng tạo riêng lấy cảm hứng từ SQuAD 2.0 nhưng kết hợp vào bối cảnh multi-hop. Điều này tăng tính thách thức và thực tế của bộ dữ liệu.

**4. Framework tổng quát hóa được**
Quy trình thu thập dữ liệu được thiết kế module hóa: đổi Wikipedia sang ngôn ngữ khác, thay bước chuẩn hóa là có thể áp dụng cho ngôn ngữ mới. Đây là đóng góp có giá trị thực tiễn cao cho các ngôn ngữ ít tài nguyên khác.

**5. Xử lý tốt các vấn đề đặc thù tiếng Việt**
Bài báo nhận diện và giải quyết hai vấn đề mà nhiều nghiên cứu tiếng Việt bỏ sót: chuẩn hóa Unicode (một ký tự vs tổ hợp ký tự) và vị trí dấu thanh ("hoà" vs "hòa"). Điều này nâng cao chất lượng dữ liệu đáng kể.

**6. Phân chia train/test thông minh**
Dùng cross-validation với mô hình baseline để phân loại câu dễ (train-normal) và câu khó (train-hard, dev, test). Dev và test chỉ chứa câu khó, đảm bảo benchmark thực sự đo được năng lực suy luận chứ không bị "lạm phát điểm" bởi câu dễ.

**7. Hai setting đánh giá bổ trợ nhau**
Gold Only kiểm tra khả năng suy luận thuần túy, Distractor kiểm tra thêm khả năng lọc nhiễu. Kết hợp cả hai cho bức tranh toàn diện hơn về năng lực mô hình.

---

## Nhược điểm

**1. Quy mô dữ liệu nhỏ**
Chỉ ~10,000 mẫu — bằng chưa đến 1/10 HotpotQA (113,000). Với lượng dữ liệu này, việc fine-tune các mô hình lớn dễ bị overfit, và kết quả thí nghiệm có thể chưa phản ánh đúng năng lực thực sự của mô hình.

**2. Đội ngũ annotator quá nhỏ và thiếu đa dạng**
Chỉ 3 người viết câu hỏi, đều là nghiên cứu sinh — cùng nền tảng học thuật, cùng lĩnh vực. Điều này dẫn đến bias nghiêm trọng: phong cách đặt câu hỏi đơn điệu, chủ đề tập trung vào những gì nhóm quen thuộc (chủ yếu thể thao, nhân vật nổi tiếng). So với HotpotQA dùng Amazon Mechanical Turk với hàng trăm worker, mức độ đa dạng của VIMQA kém hơn nhiều.

**3. Không đề xuất mô hình hay phương pháp mới**
Bài báo chỉ chạy lại các mô hình có sẵn (mBERT, XLM-R, InfoXLM) theo cách đơn giản nhất rồi kết luận "dataset khó". Không có bất kỳ đề xuất nào về kiến trúc mới, kỹ thuật mới, hay cách tiếp cận multi-hop chuyên biệt. Phần thí nghiệm mang tính "chấm điểm benchmark" hơn là đóng góp khoa học.

**4. Thiếu vắng các mô hình multi-hop chuyên dụng**
Các mô hình nổi tiếng cho multi-hop QA như DFGN (Graph Neural Network), SAE, HGN, hay Longformer đều không được thử nghiệm. Việc chỉ dùng mô hình single-hop rồi nói "bài toán khó" là chưa thuyết phục — có thể kết quả sẽ khác nhiều nếu dùng đúng mô hình thiết kế cho multi-hop.

**5. Chủ yếu chỉ 2-hop, thiếu suy luận phức tạp thực sự**
Type III (cần 3+ bước suy luận) chỉ chiếm 4%. Phần lớn câu hỏi là 2-hop đơn giản. Trong thực tế, nhiều câu hỏi cần chuỗi suy luận dài hơn, và bộ dữ liệu chưa phản ánh được điều này.

**6. Phương pháp retrieval trong Distractor setting quá đơn giản**
Dùng BM25 (thuật toán dựa trên từ khóa) để trích xuất 2 đoạn văn từ 10 đoạn, rồi mới cho mô hình đọc. Nếu BM25 chọn sai đoạn, mô hình QA không có cơ hội trả lời đúng. Bài báo không phân tích riêng lỗi do retrieval vs lỗi do reasoning, nên khó biết bottleneck thực sự nằm ở đâu.

**7. Thiếu phân tích lỗi (error analysis)**
Bài báo chỉ báo cáo con số EM/F1 mà không phân tích mô hình sai ở đâu, sai kiểu nào, kiểu suy luận nào khó nhất. Một error analysis chi tiết sẽ giúp định hướng nghiên cứu tương lai rõ ràng hơn nhiều.

**8. Kiểm tra chất lượng inter-annotator agreement chưa rõ**
Bài nói "chỉ giữ mẫu được hơn 1 worker đồng ý" nhưng không báo cáo chỉ số đồng thuận (Cohen's Kappa hay Fleiss' Kappa). Không có con số cụ thể thì khó đánh giá chất lượng annotation thực sự.

---

## Đánh giá tổng thể

Đây là một **resource paper** (bài báo tài nguyên) có giá trị — đóng góp chính nằm ở bộ dữ liệu và framework, không phải ở phương pháp. Với bối cảnh tiếng Việt thiếu tài nguyên multi-hop QA, đóng góp này có ý nghĩa thực tiễn. Tuy nhiên, bài báo sẽ mạnh hơn nếu có quy mô dữ liệu lớn hơn, đội annotator đa dạng hơn, thí nghiệm với mô hình multi-hop chuyên dụng, và phân tích lỗi chi tiết.