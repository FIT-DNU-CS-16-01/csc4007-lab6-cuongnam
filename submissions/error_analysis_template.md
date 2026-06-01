# Lab 6 – Báo cáo Phân tích Lỗi Sai

## 1. Định nghĩa bài toán

- Bài toán chọa: **Phân loại cảm xúc (Sentiment Analysis) cho đánh giá phim IMDB**
- Đầu vào: Một đoạn review phim bằng tiếng Anh.
- Schema đầu ra: JSON với sentiment (tích cực/tiêu cực/trung lập/hỗn hợp), các câu chứng minh, giải thích và độ tự tin
- LLM sử dụng: Llama 3 (thông qua API Ollama Local)
- Phương pháp thực thi prompt: Mô hình Local thông qua API Ollama

## 2. Tập kiểm tra

- Số lượng đánh giá: 30
- Số đánh giá tích cực: 13
- Số đánh giá tiêu cực: 17
- Số đánh giá dễ dàng: 15 (cảm xúc rõ ràng, ngôn ngữ đơn giản)
- Số đánh giá hỗn hợp: 4 (chứa cả khen văn và chỉ trích)
- Số đánh giá mơ hổ: 1 (đọc thế hoặc cách giải thích phức tạp)
- Số đánh giá keyword-trap: 2 (phủ định bập mĩ như không tốt nhưng tập thể tích cực)
- Số đánh giá dài: 2 (nhiều mệnh đề, cấu trúc câu phức tạp)

## 3. Prompt v1 – Baseline Prompt

**File:** `prompts/prompt_template_v1.txt`

```
You are CineSense, a movie review analysis assistant.
Given one IMDB movie review, analyze its sentiment.
Return the answer in JSON format with the following fields:
- sentiment: positive, negative, neutral, or mixed
- evidence: exact phrases from the review
- short_explanation: one or two sentences
Review:
{review_text}
```

**Giải thích:**
- **Chức năng**: Hướng dẫn đơn giản để phân loại cảm xúc của bài đánh giá và cung cấp các bằng chứng hỗ trợ.
- **Định dạng đầu ra**: JSON gồm các trường sentiment, evidence và short_explanation.
- **Điểm mạnh**: Ngắn gọn, dễ hiểu, ít ràng buộc và dễ triển khai.
- **Điểm yếu tiềm ẩn**: Không có hướng dẫn xử lý phủ định (negation), đánh giá hỗn hợp (mixed review), hoặc ngăn chặn hiện tượng ảo giác (hallucination); cũng không yêu cầu mô hình cung cấp mức độ tự tin (confidence).

## 4. Prompt v2 – Improved Prompt

**File:** `prompts/prompt_template_v2.txt`

```
You are CineSense, a careful movie review analysis assistant.
Task:
Analyze one IMDB movie review using only the information explicitly stated in the review.
Rules:
1. Do not infer facts that are not present in the review.
2. If the review contains both positive and negative opinions, use "mixed".
3. Evidence must be exact phrases copied from the review.
4. Return valid JSON only. Do not add any explanation outside JSON.
5. If the sentiment is unclear, use "neutral" or "mixed" and set confidence to "low".
Output schema:
{
"sentiment": "positive | negative | neutral | mixed",
"evidence": [ "exact phrase from the review"],
"short_explanation": "one sentence",
"confidence": "high | medium | low"
}
Review:
{review_text}
```

**giải thích:**
- **Những cải tiến so với v1:** Bổ sung các quy tắc rõ ràng để ngăn mô hình suy diễn hoặc tạo thông tin không có trong bài đánh giá; thêm trường confidence; làm rõ cách xử lý các đánh giá có cả ý kiến tích cực và tiêu cực. 
- **Các quy tắc mới:** Không được suy luận thêm thông tin ngoài nội dung review, bắt buộc trả về JSON hợp lệ, và yêu cầu đánh giá mức độ tự tin.
- **Thay đổi trong đầu ra:** Bổ sung trường ```confidence``` và yêu cầu mọi bằng chứng phải được sao chép nguyên văn từ bài đánh giá.

## 5. Prompt v3 – CoT-inspired Prompt

**File:** `prompts/prompt_template_v3_cot.txt`

```
You are CineSense, a careful NLP/LLM assistant for IMDB movie review analysis.

Goal:
Classify the overall sentiment of the review as positive or negative.

Think internally before answering:
1. Identify positive clues in the review.
2. Identify negative clues in the review.
3. Decide which attitude dominates overall.
4. Check whether the review is mixed, ambiguous, or a keyword trap.
5. Check that every evidence phrase is copied exactly from the review.

Rules:
1. Use only information explicitly stated in the review.
2. Do not invent facts about the movie, actors, director, awards, box office, or audience reactions.
3. Do not reveal your full chain-of-thought.
4. Return valid JSON only. Do not include markdown fences.
5. If the review is mixed or ambiguous, use confidence = "medium" or "low".

Output schema:
{
  "sentiment": "positive | negative",
  "dominant_reason": "short final reason without full chain-of-thought",
  "positive_clues": ["exact phrase from the review", "..."],
  "negative_clues": ["exact phrase from the review", "..."],
  "evidence_phrases": ["exact phrase from the review", "..."],
  "confidence": "high | medium | low"
}

Review:
{review_text}
```

**Giải thích:**
- **Suy luận theo từng bước:** Khuyến khích mô hình thực hiện suy luận nội bộ theo trình tự như xác định các dấu hiệu tích cực, xác định các dấu hiệu tiêu cực, đánh giá thái độ chiếm ưu thế và kiểm tra các trường hợp đặc biệt.
- **Xử lý Chain-of-Thought:** Quá trình suy luận được thực hiện nội bộ nhưng không được hiển thị trong kết quả JSON trả về.
- **Định dạng đầu ra:** Cấu trúc chi tiết hơn với các trường riêng cho ```positive_clues```, ```negative_clues``` và ```dominant_reason```.
- **Cải tiến:** Yêu cầu mô hình kiểm tra các trường hợp keyword-trap và nội dung mơ hồ; đồng thời buộc phải liệt kê cả các dấu hiệu tích cực và tiêu cực trước khi đưa ra quyết định cuối cùng.

## 6. Quantitative comparison

| Metric | Prompt v1 | Prompt v2 | Prompt v3 CoT | Comment |
|---|---:|---:|---:|---|
| Accuracy | 36.7% | 60.0% | 56.7% | Correct sentiment prediction rate |
| Valid JSON rate | 0% | 93.3% | 83.3% | Percentage of valid JSON outputs |
| Evidence exactness rate | N/A | 85.0% | 80.0% | Percentage of outputs with exact evidence from the review |
| Hallucination count | 30 | 4 | 6 | Number of outputs that invent unsupported content |
| Outside knowledge count | 30 | 0 | 1 | Number of outputs that use facts not stated in the review |
| Overconfidence count | N/A | 2 | 3 | Number of uncertain cases with too high confidence |
| Error count | 30 | 12 | 13 | Total errors (sentiment + JSON + evidence) |

## 7. Error buckets

| Error bucket | Count v1 | Count v2 | Count v3 CoT | Example review_id | Comment |
|---|---:|---:|---:|---|---|
| wrong_sentiment | 0 | 4 | 4 | R008, R002, R004, R025 | Predicted sentiment differs from gold label |
| invalid_json | 30 | 2 | 5 | R026, R033 | Output is not valid JSON or has extra prose |
| hallucinated_evidence | 0 | 4 | 6 | R026, R033, R046 | Evidence phrase is not present in review |
| outside_knowledge | 0 | 0 | 1 | R??  | Uses facts not in review (awards, actors, box office) |
| missed_positive_aspect | 0 | 2 | 3 | R045, R041 | LLM ignores important positive phrase |
| missed_negative_aspect | 0 | 0 | 0 | N/A | LLM ignores important negative phrase |
| keyword_trap | 0 | 6 | 5 | R008, R002, R004, R025, R035 | Model follows surface words and misses overall attitude |
| mixed_review_failure | 0 | 2 | 3 | R026, R033, R046 | Fails to weigh both praise and criticism |
| overconfident | 0 | 2 | 3 | R008, R035 | High confidence on ambiguous or mixed review |
| cot_not_helpful | 0 | 0 | 2 | R021, R008 | CoT still fails or gives no improvement |

## 8. Ba lỗi đáng chú ý

### Lỗi 1: Mẫu Keyword-Trap – R008

* Review ID: R008

* Loại review: keyword_trap

* Nhãn cảm xúc đúng (Gold sentiment): positive

* Phiên bản prompt: v1 (thất bại), v2 và v3 cũng thất bại

* Điều gì đã xảy ra: Review có nội dung: "The film is **not perfect**, but it has **heart, energy**, and a **cast that clearly understands the material**." Cả v2 và v3 đều dự đoán **negative** vì tập trung vào cụm từ "not perfect" ở bề mặt câu chữ mà không đánh giá đúng các tín hiệu tích cực mạnh hơn xuất hiện sau từ "but".

* Tại sao lỗi này quan trọng: Đây là một dạng lỗi nghiêm trọng và thường gặp trong các bài đánh giá thực tế. Cấu trúc phủ định kết hợp đối lập ("not X, but Y") thường cho thấy ý kiến chính nằm ở phần sau từ "but". Lỗi này chứng minh rằng ngay cả khi prompt đã có các quy tắc rõ ràng, mô hình vẫn có thể thất bại nếu không hiểu đúng ý nghĩa ngữ nghĩa của cấu trúc đối lập.

* Prompt v2 hoặc Prompt v3 đã cố gắng khắc phục như thế nào: v2 bổ sung quy tắc "Nếu review chứa cả ý kiến tích cực và tiêu cực thì sử dụng mixed". Tuy nhiên, v2 vẫn thất bại vì không nhận diện được rằng từ "but" làm thay đổi trọng tâm cảm xúc của câu. v3 yêu cầu mô hình xác định riêng các dấu hiệu tích cực và tiêu cực trước khi đưa ra kết luận cuối cùng, điều này về lý thuyết có thể giúp xử lý tốt hơn nhưng vẫn không ngăn được lỗi xảy ra.

### Lỗi 2: Thất bại khi xử lý Review Hỗn Hợp – R026

* Review ID: R026

* Loại review: mixed

* Nhãn cảm xúc đúng (Gold sentiment): negative

* Phiên bản prompt: v2 và v3

* Điều gì đã xảy ra: Review có nội dung: "The film looks **expensive**, but the story is **thin and the characters are hard to care about**." Mặc dù bộ phim được khen về mặt hình ảnh, các nhận xét tiêu cực về cốt truyện và nhân vật mới là nội dung chiếm ưu thế. Tuy nhiên, v2 đôi khi sinh JSON không hợp lệ hoặc dự đoán là positive/neutral. v3 đôi khi tạo JSON đúng định dạng nhưng vẫn không đánh giá được rằng các yếu tố tiêu cực mới là nội dung chính.

* Tại sao lỗi này quan trọng: Các review hỗn hợp vốn khó xử lý hơn vì mô hình phải cân nhắc mức độ ảnh hưởng của từng ý kiến thay vì chỉ nhận diện từ khóa cảm xúc. Kết quả này cho thấy chỉ cải thiện prompt là chưa đủ; khả năng suy luận về mức độ quan trọng của các nhận xét trong review vẫn còn hạn chế.

* Prompt v2 hoặc Prompt v3 đã cố gắng khắc phục như thế nào: v2 quy định rõ cách xử lý review hỗn hợp và yêu cầu sử dụng confidence = "low" trong các trường hợp khó xác định. v3 bổ sung bước suy luận "Decide which attitude dominates overall", tức là xác định cảm xúc nào chiếm ưu thế. Tuy nhiên, cả hai prompt đều không cung cấp ví dụ hoặc cơ chế đánh trọng số cụ thể cho từng loại nhận xét, dẫn đến kết quả không ổn định.

### Lỗi 3: Đầu ra JSON Không Hợp Lệ – R045 (v1) / R033 (v2)

* Review ID: R045 (v1: JSON không hợp lệ), R033 (v2: bằng chứng bị hallucination)

* Loại review: long_review / mixed

* Nhãn cảm xúc đúng (Gold sentiment): negative

* Phiên bản prompt: v1 (thất bại hoàn toàn), v2 và v3 (cải thiện một phần)

* Điều gì đã xảy ra:

  * **v1 (R045):** Mô hình trả về kết quả dưới dạng Markdown thay vì JSON thuần túy (`Here is the analysis: \n\```json {sentiment: ...} ````). Hệ thống không thể parse kết quả và giá trị `pred_sentiment` bị gán là NaN.
  * **v2 (R033):** JSON được parse thành công nhưng các bằng chứng như "character decision feels forced" lại không xuất hiện nguyên văn trong review gốc. Đây là hiện tượng hallucinated evidence (mô hình tự tạo ra bằng chứng).

* Tại sao lỗi này quan trọng: Việc v1 không trả về JSON hợp lệ khiến toàn bộ hệ thống không thể sử dụng kết quả. Trong khi đó, mặc dù v2 đã tạo được JSON hợp lệ, việc sinh ra bằng chứng không chính xác vẫn làm giảm độ tin cậy và khả năng giải thích của hệ thống.

* Prompt v2 hoặc Prompt v3 đã cố gắng khắc phục như thế nào: v2 bổ sung quy tắc "Return valid JSON only. Do not add any explanation outside JSON." Nhờ đó tỷ lệ JSON hợp lệ tăng từ 0% lên 93,3%. v3 kế thừa quy tắc này và vẫn duy trì tỷ lệ JSON hợp lệ ở mức 83,3%, mặc dù việc bổ sung các trường suy luận mới làm tăng số lượng lỗi định dạng so với v2.

---

## 9. Nhận xét và Đánh giá

### LLM làm tốt điều gì?

Prompt v2 đã cải thiện đáng kể tính hợp lệ của đầu ra JSON (từ 0% lên 93,3%) nhờ bổ sung quy tắc rõ ràng *"chỉ trả về JSON hợp lệ"*. Cả v2 và v3 đều xử lý khá tốt các đánh giá mang cảm xúc tích cực hoặc tiêu cực rõ ràng, với độ chính xác lần lượt là 60% và 56,7%. Llama 3 tuân thủ hướng dẫn tương đối tốt khi các yêu cầu được mô tả cụ thể và rõ ràng.

### Loại đánh giá nào gây khó khăn nhất?

Các bài đánh giá dạng **keyword-trap** (ví dụ: *"not perfect, but..."*) là khó xử lý nhất. Cả v2 và v3 đều thất bại với những trường hợp này mặc dù đã có quy tắc hướng dẫn rõ ràng. Ngoài ra, các đánh giá hỗn hợp, trong đó có một số lời khen bề ngoài nhưng nội dung chính lại mang tính chỉ trích (R026, R033, R046), cũng gây nhiều khó khăn. Những bài đánh giá dài với nhiều khía cạnh khác nhau đòi hỏi mô hình phải cân nhắc và đánh giá mức độ quan trọng của từng ý, điều mà LLM vẫn còn hạn chế.

### Prompt v2 có cải thiện so với Prompt v1 không? Hãy đưa ra bằng chứng.

**Có, cải thiện rất đáng kể.**

Độ chính xác tăng từ 0% lên 60% (18 dự đoán đúng so với 0). Tỷ lệ JSON hợp lệ tăng từ 0% lên 93,3%.

Nguyên nhân thất bại hoàn toàn của v1 là mô hình thường bọc kết quả trong Markdown thay vì trả về JSON thuần túy. Các quy tắc bổ sung trong v2 như:

* "return valid JSON only"
* "do not infer facts"
* "evidence must be exact"

đã khắc phục các lỗi nền tảng này. Việc bổ sung cấu trúc rõ ràng cùng các ràng buộc cụ thể giúp tăng thêm 18 dự đoán chính xác.

### Prompt v3 (CoT) có cải thiện so với Prompt v2 không? Hãy đưa ra bằng chứng.

**Không. Hiệu suất thực tế giảm nhẹ.**

Prompt v3 đạt độ chính xác 56,7%, thấp hơn v2 (60%). Tỷ lệ JSON hợp lệ cũng giảm từ 93,3% xuống 83,3%.

v3 bổ sung các bước suy luận theo hướng Chain-of-Thought như:

1. Xác định các dấu hiệu tích cực.
2. Xác định các dấu hiệu tiêu cực.
3. Đánh giá thái độ chiếm ưu thế.
4. Kiểm tra các trường hợp keyword-trap và mơ hồ.

Tuy nhiên trên thực tế:

* Mô hình vẫn thất bại với trường hợp keyword-trap R008 dù đã có hướng dẫn rõ ràng.
* Việc đánh giá các review hỗn hợp vẫn không nhất quán (R026, R033).
* Các trường thông tin bổ sung trong JSON làm tăng số lượng lỗi định dạng (5 JSON không hợp lệ ở v3 so với 2 ở v2).

### Chain-of-Thought có làm kết quả tệ hơn, dài hơn hoặc khó phân tích hơn không?

**Có, ở cả ba khía cạnh.**

Đầu ra của v3 dài hơn do phải bổ sung các trường:

* positive_clues
* negative_clues
* dominant_reason

Sự phức tạp tăng thêm này dẫn tới:

* 5 đầu ra JSON không hợp lệ ở v3 so với 2 ở v2.
* Thời gian phản hồi dài hơn do mô hình phải thực hiện thêm các bước suy luận.

Quan trọng hơn, CoT không giúp khắc phục các lỗi liên quan đến keyword-trap hoặc review hỗn hợp. Điều này cho thấy vấn đề nằm ở khả năng hiểu ngữ nghĩa của mô hình hơn là ở quy trình suy luận được mô tả trong prompt.

### Nếu tiếp tục cải thiện, cần làm gì?

1. Bổ sung 2–3 ví dụ cụ thể trong prompt hệ thống để minh họa cách xử lý keyword-trap (ví dụ: *"not perfect, but..."* nên được phân loại là tích cực).
2. Xây dựng quy tắc hoặc trọng số rõ ràng hơn cho từng loại cảm xúc (ví dụ: nếu nội dung chê chất lượng thực thi nhưng khen ý tưởng, nên nghiêng về tiêu cực).
3. Tách riêng bước trích xuất bằng chứng và bước phân loại cảm xúc thành hai giai đoạn độc lập.
4. Sử dụng các ví dụ minh họa có giải thích (in-context examples) thay vì chỉ đưa ra quy tắc chung.
5. Cân nhắc sử dụng mô hình kết hợp (ensemble), trong đó v2 đảm nhiệm việc tạo JSON ổn định còn v3 hỗ trợ đánh giá độ tin cậy cho các trường hợp mơ hồ.

## 10. Kết luận

**Prompt v2 là lựa chọn đáng tin cậy nhất để triển khai trong hệ thống CineSense.**

Prompt v1 đạt độ chính xác 0% và tỷ lệ JSON hợp lệ 0%, cho thấy gần như không thể sử dụng trong thực tế. Llama 3 xem đây như một tác vụ NLP tổng quát và thường sinh thêm phần giải thích dưới dạng Markdown, khiến kết quả không thể phân tích tự động.

Prompt v2 đã khắc phục triệt để vấn đề này nhờ các ràng buộc rõ ràng như:

* "return valid JSON only"
* "do not infer facts"
* "evidence must be exact"

Kết quả đạt được:

* Độ chính xác: 60%
* Tỷ lệ JSON hợp lệ: 93,3%

Mặc dù Prompt v3 áp dụng phương pháp Chain-of-Thought nhằm cải thiện khả năng suy luận (xác định dấu hiệu, cân nhắc mức độ ảnh hưởng, kiểm tra keyword-trap), kết quả thực nghiệm lại cho thấy sự suy giảm:

* Độ chính xác giảm còn 56,7%.
* Tỷ lệ JSON hợp lệ giảm còn 83,3%.
* Vẫn mắc lỗi ở các mẫu keyword-trap như R008 (*"not perfect, but..."* bị dự đoán là tiêu cực).
* Tăng độ phức tạp của đầu ra và làm phát sinh thêm lỗi phân tích JSON.

Từ góc độ triển khai thực tế, Prompt v2 mang lại sự cân bằng tốt nhất giữa:

1. Đầu ra JSON ổn định và dễ xử lý (93,3% hợp lệ).
2. Độ chính xác chấp nhận được (18/30 dự đoán đúng).
3. Cấu trúc đơn giản, không yêu cầu xử lý thêm các trường suy luận phức tạp.

Phần lớn các lỗi còn lại xuất phát từ những thách thức ngữ nghĩa thực sự như keyword-trap, cảm xúc hỗn hợp và các bài đánh giá dài có nhiều khía cạnh. Những vấn đề này có thể được giải quyết hiệu quả hơn bằng cách bổ sung ví dụ minh họa (few-shot / in-context learning) thay vì chỉ thêm các quy tắc hướng dẫn.

Do đó, khuyến nghị hiện tại là **triển khai Prompt v2**, đồng thời thu thập các trường hợp thất bại tiêu biểu (R008, R002, R004, R026, ...) để sử dụng làm ví dụ huấn luyện trong một phiên bản Prompt v4 trong tương lai.

