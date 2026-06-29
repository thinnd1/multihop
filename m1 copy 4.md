# Tổng hợp ý chính bài báo: Hierarchical Graph Network for Multi-hop Question Answering (HGN)

## 1. Bối cảnh và vấn đề

Bài báo giải quyết bài toán **multi-hop question answering** (trả lời câu hỏi đa bước) trên dataset **HotpotQA**, nơi mô hình phải tổng hợp thông tin từ nhiều đoạn văn để trả lời câu hỏi và đồng thời chỉ ra các "supporting facts" (câu chứa bằng chứng).

**Hạn chế của các phương pháp trước:**
- Các mô hình dùng entity graph chỉ tập trung vào dự đoán câu trả lời, không hỗ trợ tốt việc tìm supporting facts.
- Graph đơn thuần (chỉ entity, hoặc paragraph-entity) không đủ sức xử lý câu hỏi phức tạp nhiều bước.

## 2. Ý tưởng cốt lõi

Đề xuất **Hierarchical Graph Network (HGN)** — một đồ thị **phân cấp dị thể** (heterogeneous hierarchical graph) tích hợp 4 loại node với mức độ chi tiết khác nhau:

- **Question node** (câu hỏi)
- **Paragraph nodes** (đoạn văn)
- **Sentence nodes** (câu)
- **Entity nodes** (thực thể)

Các node ở các cấp khác nhau **bổ trợ lẫn nhau** để giải đồng thời nhiều sub-task: chọn đoạn văn, tìm supporting facts, dự đoán câu trả lời.

## 3. Kiến trúc 4 module

**(1) Graph Construction** — Xây dựng đồ thị qua 2 bước:
- Chọn paragraph hop 1 bằng title matching + paragraph ranker (RoBERTa).
- Tìm paragraph hop 2 thông qua **hyperlink của Wikipedia** trong các câu của paragraph hop 1.
- Định nghĩa **7 loại cạnh** kết nối các node (Q–P, Q–E, P–S, S–P qua hyperlink, S–E, P–P, S–S liền kề).

**(2) Context Encoding** — Dùng **RoBERTa-large** + bi-attention + BiLSTM để tạo biểu diễn ban đầu cho từng node (lấy hidden state ở vị trí start/end của span, max-pooling cho câu hỏi).

**(3) Graph Reasoning** — Dùng **Graph Attention Network (GAT)** với attention coefficient phụ thuộc vào **loại cạnh** để lan truyền thông tin giữa các node theo cơ chế message passing.

**(4) Multi-task Prediction** — Output của graph được dùng cho:
- Paragraph selection (từ P nodes)
- Supporting facts prediction (từ S nodes)
- Entity prediction (từ E nodes — chỉ làm regularization)
- Answer span extraction (qua **gated attention** kết hợp context M với graph H′)
- Answer type classification (span / entity / yes / no)

Hàm loss tổng hợp: `L = L_start + L_end + λ₁L_para + λ₂L_sent + λ₃L_entity + λ₄L_type`.

## 4. Kết quả thực nghiệm

Đạt **state-of-the-art** tại thời điểm submit trên HotpotQA:

| Setting | Joint EM | Joint F1 | Cải thiện |
|---------|----------|----------|-----------|
| Distractor | 47.11 | 74.21 | +2.44 / +1.48 |
| Fullwiki (ALBERT-xxlarge) | 37.92 | 62.26 | +2.57 / +1.08 |

**Ablation study** xác nhận từng thành phần đều có ích:
- Plain RoBERTa: Joint F1 = 71.02
- + PS graph (Q–P–S): 73.83 (+2.81)
- + Entity nodes: 74.13
- + Edges P–P, S–S: **74.37** (graph đầy đủ)

Hiệu suất vượt DFGN, EPS, SAE khi dùng **cùng pre-trained model**, chứng tỏ gain đến từ **thiết kế kiến trúc**, không chỉ từ RoBERTa.

## 5. Phân tích lỗi

Phân loại 100 lỗi mẫu thành 6 nhóm:
- **Annotation sai** (9%)
- **Nhiều đáp án đúng** (24%) — phổ biến nhất, ví dụ "EPA" vs "Environmental Protection Agency"
- **Discrete reasoning** (15%) — mô hình yếu khi cần so sánh số học
- **Commonsense / external knowledge** (16%)
- **Multi-hop sai** (16%) — đi nhầm nhánh
- **MRC span sai** (20%) — tìm đúng đoạn nhưng chọn sai span

Theo loại câu: comparison yes/no dễ nhất (88.5 F1), bridge và comparison-span khó hơn (~74 F1).

## 6. Đóng góp chính

1. **Đề xuất HGN** — đồ thị phân cấp dị thể hợp nhất 4 loại node.
2. Các node ở các mức **bổ trợ lẫn nhau** trong multi-task learning, cung cấp tín hiệu giám sát cho cả supporting facts lẫn answer.
3. **SOTA trên cả 2 setting** của HotpotQA tại thời điểm công bố (EMNLP 2020).

## 7. Hạn chế và hướng phát triển

- Hiện phụ thuộc **Wikipedia hyperlinks** để nối đa hop — có thể thay bằng entity linking để tổng quát hóa.
- Giới hạn 2-hop, 4 paragraph cho HotpotQA; muốn mở rộng cần sliding window hoặc transformer cho long sequence (Longformer, BigBird).
- Fullwiki dùng retriever rời — hướng tương lai: huấn luyện chung HGN và retriever.