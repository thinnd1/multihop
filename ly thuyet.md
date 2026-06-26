Đây là các metric (chỉ số đánh giá) phổ biến trong NLP và Information Retrieval:

EM (Exact Match) — Đo xem câu trả lời của model có khớp chính xác từng ký tự với đáp án chuẩn hay không. Đúng = 1, sai = 0. Ví dụ: đáp án là "Hà Nội", model trả lời "Hà Nội" → 1, trả lời "thành phố Hà Nội" → 0.

F1 (Token-level F1) — Tính precision và recall ở mức từ/token giữa câu trả lời của model và đáp án, rồi lấy trung bình điều hòa. Mềm dẻo hơn EM vì cho điểm một phần khi trả lời đúng nhưng thừa hoặc thiếu vài từ. Ví dụ: đáp án "Hà Nội", model trả lời "thành phố Hà Nội" → F1 vẫn > 0 vì có chứa "Hà Nội".

Recall@K — Trong bài toán retrieval (truy xuất tài liệu), đây là tỷ lệ tài liệu đúng nằm trong top K kết quả mà hệ thống trả về. Ví dụ: có 5 tài liệu liên quan, hệ thống trả về top 10 mà chứa 3 trong số đó → Recall@10 = 3/5 = 0.6.

Supporting Fact F1 — Metric đặc thù của các bài toán multi-hop QA (như HotpotQA). Thay vì đánh giá câu trả lời cuối, nó đánh giá xem model có chỉ đúng các câu/đoạn văn chứa bằng chứng (supporting facts) dùng để suy luận ra đáp án hay không. Cũng tính theo precision/recall/F1 ở mức câu (sentence-level).

Hallucination (Ảo giác) — Hiện tượng model sinh ra thông tin sai hoặc bịa đặt mà không có cơ sở trong dữ liệu đầu vào hay thực tế. Có hai loại chính:

- Intrinsic hallucination: mâu thuẫn trực tiếp với nội dung nguồn (ví dụ: tài liệu nói "năm 2020" nhưng model trả lời "năm 2018").
- Extrinsic hallucination: thêm thông tin không thể xác minh từ nguồn (ví dụ: tài liệu không nhắc đến dân số nhưng model tự bịa ra một con số).

Tóm lại, EM và F1 đo chất lượng câu trả lời, Recall@K đo chất lượng truy xuất, Supporting Fact F1 đo khả năng suy luận đúng bước, và Hallucination là hiện tượng cần giảm thiểu.

Đây là một pipeline RAG (Retrieval-Augmented Generation) cho bài toán hỏi đáp tiếng Việt. Giải thích từng thành phần:


PhoBERT / BGE-m3 — Đây là các embedding model (mô hình biểu diễn văn bản thành vector). PhoBERT là mô hình pre-trained đặc thù cho tiếng Việt (dựa trên RoBERTa), còn BGE-m3 là mô hình embedding đa ngôn ngữ của BAAI, hỗ trợ tốt tiếng Việt. Vai trò ở đây là encode câu hỏi và các đoạn văn bản thành vector để so sánh độ tương đồng.

Retriever — Bộ truy xuất tài liệu. Nhận vector từ bước trên, tìm kiếm trong kho tài liệu (knowledge base) những đoạn văn bản liên quan nhất với câu hỏi. Thường dùng các phương pháp như dense retrieval (tìm kiếm theo cosine similarity giữa các vector) hoặc kết hợp sparse retrieval (BM25).

Evidence — Các đoạn văn bản bằng chứng được Retriever trả về. Đây là những đoạn context chứa thông tin cần thiết để trả lời câu hỏi. Có thể là 1 hoặc nhiều đoạn (top-k passages).

Qwen2-7B — Đây là LLM (Large Language Model) của Alibaba với 7 tỷ tham số, đóng vai trò reader/generator. Nó nhận đầu vào gồm câu hỏi + evidence, rồi sinh ra câu trả lời. Qwen2-7B được chọn vì hỗ trợ tốt tiếng Việt và kích thước vừa phải để chạy được trên phần cứng hạn chế.

Answer — Câu trả lời cuối cùng được LLM sinh ra dựa trên evidence đã truy xuất.


Tóm lại luồng hoạt động: Câu hỏi được encode thành vector (PhoBERT/BGE-m3) → tìm tài liệu liên quan (Retriever) → lấy ra bằng chứng (Evidence) → đưa vào LLM (Qwen2-7B) để sinh câu trả lời (Answer). Đây là kiến trúc RAG điển hình, giúp giảm hallucination vì LLM trả lời dựa trên evidence thực tế thay vì chỉ dựa vào bộ nhớ nội tại.


Đây là pipeline Multi-hop RAG — nâng cấp so với RAG thông thường, dành cho các câu hỏi phức tạp cần suy luận nhiều bước. Giải thích từng khối:


Question Q — Câu hỏi gốc từ người dùng. Thường là câu hỏi phức tạp, không thể trả lời chỉ bằng một đoạn văn bản duy nhất. Ví dụ: *"Tổng thống Mỹ ký hiệp định Paris sinh năm bao nhiêu?"* — cần biết ai ký, rồi mới tra năm sinh.

Tách câu hỏi con (Question Decomposition) — Bước quan trọng nhất khác biệt so với RAG thường. Dùng LLM hoặc rule-based để chia câu hỏi phức tạp thành nhiều câu hỏi đơn giản hơn. Ví dụ trên sẽ được tách thành: (1) *"Ai là tổng thống Mỹ ký hiệp định Paris?"* → (2) *"Người đó sinh năm bao nhiêu?"*. Mỗi câu hỏi con có thể truy xuất độc lập hoặc tuần tự.

Retriever BM25/BGE — Bộ truy xuất kết hợp hai phương pháp:
- BM25: tìm kiếm theo từ khóa (sparse retrieval), khớp chính xác từ ngữ, nhanh và hiệu quả cho câu hỏi có thuật ngữ cụ thể.
- BGE: tìm kiếm theo ngữ nghĩa (dense retrieval), dùng embedding vector để tìm đoạn văn có ý nghĩa tương đồng dù dùng từ khác.

Kết hợp cả hai gọi là hybrid retrieval, giúp bù trừ điểm yếu của nhau. Retriever chạy cho từng câu hỏi con để lấy evidence tương ứng.

Knowledge Base — Kho tri thức chứa toàn bộ tài liệu đã được index sẵn (dạng chunks/đoạn văn). Retriever tìm trong đây và trả về các đoạn evidence liên quan cho từng câu hỏi con.

Qwen2 Reasoner — LLM Qwen2 nhưng ở đây đóng vai trò reasoner (bộ suy luận), không chỉ đơn thuần sinh câu trả lời. Nó phải:
- Tổng hợp evidence từ nhiều câu hỏi con
- Suy luận chuỗi (chain of reasoning): dùng đáp án câu hỏi con trước làm đầu vào cho câu hỏi con sau
- Kiểm tra tính nhất quán giữa các evidence
- Sinh ra câu trả lời cuối cùng dựa trên toàn bộ chuỗi suy luận

Answer — Câu trả lời cuối cùng, là kết quả của quá trình suy luận nhiều bước qua nhiều evidence.


So sánh với pipeline trước (RAG thường): Pipeline trước chỉ retrieve một lần rồi đưa thẳng vào LLM. Pipeline này thêm bước tách câu hỏi con và dùng hybrid retrieval, giúp trả lời được các câu hỏi phức tạp cần kết nối thông tin từ nhiều nguồn khác nhau.


## FiD (Fusion-in-Decoder)

FiD là kiến trúc do Izacard & Grave (2021, Facebook AI) đề xuất, chuyên dùng cho bài toán open-domain question answering. Có hai phiên bản theo kích thước:


### Ý tưởng cốt lõi

Trong RAG thông thường, tất cả evidence được nối (concatenate) lại thành một chuỗi dài rồi đưa vào model → dễ bị giới hạn bởi chiều dài context.

FiD làm khác: encode riêng từng passage, nhưng decode chung.

Bước Encoder: Mỗi passage được ghép với câu hỏi thành một cặp riêng biệt, rồi encode độc lập qua encoder của T5. Ví dụ có 100 passages thì chạy encoder 100 lần, mỗi lần cho một cặp (question + passage_i). Điều này giúp model xử lý được rất nhiều passages mà không bị giới hạn context length.

Bước Decoder: Tất cả output của encoder được nối lại và đưa vào decoder. Decoder dùng cross-attention để "nhìn" đồng thời tất cả passages, rồi sinh câu trả lời. Đây chính là lý do gọi là "Fusion-in-Decoder" — việc tổng hợp thông tin xảy ra ở bước decode.


### FiD-base vs FiD-large

Cả hai đều dựa trên kiến trúc T5 (Text-to-Text Transfer Transformer):

FiD-base — Dựa trên T5-base với khoảng 220 triệu tham số. Nhẹ hơn, chạy nhanh hơn, phù hợp khi tài nguyên hạn chế. Hiệu năng thấp hơn nhưng vẫn rất tốt so với các phương pháp trước đó.

FiD-large — Dựa trên T5-large với khoảng 770 triệu tham số. Nặng hơn nhưng cho kết quả state-of-the-art trên các benchmark như NaturalQuestions và TriviaQA. Khả năng tổng hợp thông tin từ nhiều passages tốt hơn.


### Ưu điểm chính

- Xử lý được nhiều passages (50–100) mà không bị tràn context, vì encode độc lập
- Hiệu quả cao: FiD-large từng đạt SOTA trên nhiều benchmark QA
- Phù hợp multi-hop: vì decoder cross-attend đồng thời nhiều passages nên có thể kết nối thông tin từ nhiều nguồn

### Hạn chế

- Chi phí tính toán encoder tỷ lệ tuyến tính với số passages
- Không có cơ chế tường minh để lọc bỏ passages nhiễu — decoder phải tự học cách bỏ qua thông tin không liên quan


Tóm lại, FiD là kiến trúc reader trong pipeline RAG, thay thế cho việc đơn giản nối tất cả context lại. Nó giải quyết bài toán "làm sao đưa nhiều evidence vào model cùng lúc" một cách hiệu quả bằng cách encode riêng, decode chung.


Đây là pipeline Graph-based Iterative RAG — RAG kết hợp đồ thị tri thức và truy xuất lặp nhiều vòng. Giải thích từng ô:


Question — Câu hỏi gốc từ người dùng. Thường là câu hỏi multi-hop phức tạp, ví dụ: *"Đạo diễn bộ phim đoạt giải Oscar 2024 sinh ra ở đâu?"*

Retrieve E₁ — Vòng truy xuất lần 1. Dùng câu hỏi gốc để tìm trong knowledge base, lấy ra tập evidence đầu tiên (E₁). Đây là những đoạn văn bản liên quan nhất với câu hỏi ban đầu. Ví dụ: tìm được đoạn nói *"Phim X đoạt giải Oscar 2024, đạo diễn là Y"*.

Graph G₁ — Từ evidence E₁, hệ thống xây dựng một đồ thị tri thức (knowledge graph). Các entity (thực thể) trở thành node, các quan hệ giữa chúng trở thành edge. Ví dụ: node "Phim X" → edge "đạo diễn bởi" → node "Y". Graph giúp hệ thống nhìn rõ cấu trúc thông tin đã thu thập và xác định còn thiếu gì để trả lời.

Sinh query mới — Dựa trên Graph G₁, hệ thống phân tích xem thông tin nào còn thiếu để trả lời câu hỏi gốc, rồi tự động sinh ra câu query mới. Ví dụ: graph đã có "đạo diễn là Y" nhưng chưa biết Y sinh ở đâu → sinh query mới: *"Y sinh ra ở đâu?"*. Đây là bước suy luận quyết định hướng tìm kiếm tiếp theo.

Retrieve E₂ — Vòng truy xuất lần 2. Dùng query mới vừa sinh để tìm thêm evidence bổ sung (E₂). Ví dụ: tìm được đoạn *"Y sinh năm 1970 tại thành phố Z"*. Quá trình này có thể lặp nhiều vòng (retrieve E₃, E₄...) tùy độ phức tạp câu hỏi.

Update Graph — Cập nhật đồ thị bằng cách thêm các entity và quan hệ mới từ E₂ vào graph cũ. Ví dụ: thêm node "thành phố Z" → edge "nơi sinh" → node "Y". Lúc này graph đã có đủ chuỗi suy luận: Phim X → đạo diễn Y → sinh tại Z.

Qwen2 — LLM Qwen2 nhận đầu vào gồm: câu hỏi gốc + toàn bộ graph đã cập nhật (hoặc evidence tương ứng). Dựa vào đó để suy luận và sinh câu trả lời cuối cùng.

Answer + Path — Đầu ra gồm hai phần:
- Answer: câu trả lời cuối cùng (*"Thành phố Z"*)
- Path: đường suy luận trên graph, thể hiện chuỗi logic từng bước (Phim X → đạo diễn Y → sinh tại Z). Path giúp câu trả lời giải thích được (explainable), người dùng thấy rõ model suy luận qua những bước nào.


Điểm khác biệt so với hai pipeline trước:
- So với RAG thường: thêm graph + truy xuất lặp, không chỉ retrieve một lần
- So với Multi-hop RAG (tách câu hỏi con): thay vì tách câu hỏi trước, pipeline này xây graph rồi tự phát hiện thiếu gì để sinh query mới — linh hoạt hơn vì không cần biết trước cần bao nhiêu bước
- Đầu ra có path suy luận, giúp kiểm chứng và giảm hallucination


## Tổng quan 4 model

Đây là 4 biến thể của kiến trúc **reader** trong open-domain QA, kết hợp giữa hai chiều: kích thước model (base/large) và có/không dùng knowledge graph (FiD/KG-FiD).



### FiD (Fusion-in-Decoder) — đã giải thích ở trên

Nhắc lại ngắn gọn: **encode riêng từng passage, decode chung**. Mỗi cặp (question + passage) được encode độc lập, rồi tất cả output encoder được nối lại cho decoder cross-attend để sinh câu trả lời.

- **FiD-base**: dựa trên T5-base (~220M tham số)
- **FiD-large**: dựa trên T5-large (~770M tham số)



### KG-FiD (Knowledge Graph enhanced Fusion-in-Decoder)

Đây là phiên bản **nâng cấp** của FiD, được đề xuất bởi **Yu et al. (2022)**. Ý tưởng cốt lõi: FiD gốc đối xử với tất cả passages **như nhau**, không biết passage nào quan trọng hơn hay chúng liên quan đến nhau thế nào. KG-FiD giải quyết vấn đề này bằng cách **thêm knowledge graph vào giữa encoder và decoder**.

**Cách hoạt động của KG-FiD:**

**Bước 1 — Encode (giống FiD):** Mỗi cặp (question + passage_i) được encode độc lập qua T5 encoder, tạo ra các representation riêng biệt.

**Bước 2 — Xây dựng graph giữa các passages (KHÁC FiD):** Đây là bước mới. Hệ thống xây một **đồ thị liên kết giữa các passages** dựa trên:
- Các **entity chung** xuất hiện trong nhiều passages (ví dụ: passage 1 và passage 5 cùng nhắc đến "Nguyễn Du" → nối cạnh)
- Quan hệ từ knowledge graph bên ngoài (như Wikidata, Freebase)

Mỗi passage trở thành một **node** trong graph, các cạnh thể hiện **mối liên hệ ngữ nghĩa** giữa chúng.

**Bước 3 — Graph Neural Network (GNN):** Dùng GNN để **truyền thông tin giữa các passage nodes**. Nhờ đó, mỗi passage representation được **bổ sung thông tin** từ các passages liên quan. Passage quan trọng (kết nối nhiều) sẽ được tăng trọng số, passage nhiễu (cô lập) sẽ bị giảm.

**Bước 4 — Rerank/Filter:** Dựa trên graph, hệ thống có thể **xếp hạng lại** hoặc **loại bỏ passages nhiễu** trước khi đưa vào decoder.

**Bước 5 — Decode (giống FiD):** Decoder cross-attend vào các passage representations đã được **làm giàu bởi graph**, sinh câu trả lời.



### So sánh 4 model

| Model | Backbone | Tham số | Knowledge Graph | Đặc điểm |
||||||
| **FiD-base** | T5-base | ~220M | Không | Nhanh, nhẹ, baseline |
| **FiD-large** | T5-large | ~770M | Không | Mạnh hơn, tốn tài nguyên hơn |
| **KG-FiD-base** | T5-base + GNN | ~220M + GNN | **Có** | Thêm graph giúp lọc nhiễu, hiệu quả hơn FiD-base |
| **KG-FiD-large** | T5-large + GNN | ~770M + GNN | **Có** | Kết hợp model lớn + graph, kết quả tốt nhất |



### Tại sao thêm Knowledge Graph lại tốt hơn?

**FiD gốc** có điểm yếu: nó encode passages **độc lập**, không biết passage A và passage B có liên quan nhau. Decoder phải tự học cách kết nối, dễ bị nhiễu bởi passages không liên quan.

**KG-FiD** giải quyết bằng cách cho các passages **"nói chuyện" với nhau** qua graph trước khi vào decoder. Kết quả là decoder nhận được representations đã được **cấu trúc hóa** và **lọc nhiễu**, nên sinh câu trả lời chính xác hơn, đặc biệt với câu hỏi multi-hop cần kết nối thông tin từ nhiều passages.



Tóm lại: FiD = encode riêng decode chung. KG-FiD = FiD + **thêm graph nối các passages** ở giữa để passages hiểu ngữ cảnh của nhau trước khi decoder tổng hợp. Base/large chỉ khác kích thước backbone T5.