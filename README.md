# 🔗 AIC26_TRAKE

*Developed by Nguyễn Châu Tuấn Kiệt - 25TNT1, VNUHCM-US (AIC HCMC 2026)*

## Lời mở đầu

AI Challenge HCMC là một cuộc thi đòi hỏi kỹ năng phân tích và truy xuất thông tin trên tập dữ liệu multimedia khổng lồ. Trong kho lưu trữ này, tôi trình bày mã nguồn và giải pháp cá nhân được tinh chỉnh chuyên biệt để xử lý track phức tạp nhất của cuộc thi: **TRAKE (Temporal Retrieval and Alignment of Key Events)**.

---

## 🎯 1. Tổng quan bài toán
Khác với các bài toán truy xuất hình ảnh đơn lẻ, bài toán TRAKE đặt ra một thử thách phức tạp hơn nhiều về mặt thời gian và ngữ nghĩa. Cho một truy vấn chứa một chuỗi các sự kiện con $E_1, E_2, ..., E_N$, hệ thống không chỉ phải tìm đúng khung hình miêu tả từng sự kiện, mà còn phải thỏa mãn các ràng buộc sau:
1. **Tính tuần tự:** Các sự kiện phải diễn ra đúng thứ tự thời gian $t_1 < t_2 < ... < t_N$ trong cùng một video.
2. **Tính liên tục:** Các khung hình không được nằm quá xa nhau, tránh hiện tượng chắp vá các cảnh rời rạc.
3. **Nhiễu thị giác:** Việc chỉ dựa vào độ tương đồng Cosine của vector thường trả về những kết quả tương đồng hình thức nhưng lại sai hoàn toàn về logic vật lý hoặc ngữ cảnh thực tế.

---

## 🧠 2. Chi tiết kỹ thuật: 
Để giải quyết bài toán tìm kiếm chuỗi sự kiện, hệ thống hoạt động qua 3 bước rất rõ ràng: **Lọc thô (Vector Search)** để tìm nhanh các cảnh rời rạc $\rightarrow$ **Ráp chuỗi (Dynamic Programming)** để xếp các cảnh đúng trình tự thời gian $\rightarrow$ **Kiểm tra chéo (VLM)** để đối chiếu trực tiếp với khung hình thực tế và trích xuất kết quả cuối cùng. Dưới đây là chi tiết cách hệ thống vận hành.

### 2.1. Giai đoạn 1: Xử lý truy vấn & Lọc không gian mẫu
Bài toán thực tế thường có các truy vấn dài dòng chứa nhiều từ ngữ trừu tượng. Nếu đem nguyên câu này đi tìm kiếm vector thì kết quả sẽ bị nhiễu.
* **Tối ưu câu truy vấn (Query Rewrite):** Hệ thống gọi Gemini-3.5-Flash để viết lại câu văn thành các từ khóa thị giác (chỉ tập trung vào vật thể và hành động). Để tối ưu chi phí và tốc độ, một bộ nhớ đệm (`@lru_cache`) được thiết lập. Nếu hệ thống gặp lại một câu truy vấn từng xử lý, nó sẽ lấy kết quả từ Cache thay vì gọi API lại.
* **Truy xuất đa luồng (Multithreading):** Kịch bản gốc được cắt nhỏ thành các sự kiện độc lập ($E_1, E_2...$). Thay vì chạy tuần tự từng sự kiện rất chậm, hệ thống dùng `ThreadPoolExecutor` để mã hóa (nhúng SigLIP) và đẩy tất cả các sự kiện lên Pinecone cùng một lúc. Quá trình này trả về hàng ngàn khung hình ứng viên với độ trễ cực thấp.

### 2.2. Giai đoạn 2: Quy hoạch động
**Tại sao phải dùng Quy hoạch động?**
Vấn đề cốt lõi của Giai đoạn 1 (Vector Search bằng Pinecone) là nó chấm điểm hoàn toàn dựa trên thị giác và mất khái niệm về dòng thời gian. Khi truy vấn song song, Pinecone có thể tìm thấy cảnh $E_1$ khớp nhất nằm ở phút 50, và cảnh $E_2$ khớp nhất nằm ở phút 10. Nếu cứ lấy Top 1 của mỗi tập kết quả, dòng thời gian sẽ bị chạy ngược ($t_1 > t_2$). Do đó, đầu ra của Bước 1 chỉ là các tập kết quả chứa hàng ngàn khung hình rời rạc, sai lệch thứ tự.

Nhiệm vụ của hàm `solve_dante` là tìm ra một đường đi xuyên qua các tập kết quả này sao cho tổng điểm cao nhất nhưng vẫn phải đúng trình tự thời gian $t_1 < t_2 < ... < t_N$:

* **Lọc video rác (Voting):** Việc đẩy toàn bộ hàng ngàn video từ Giai đoạn 1 vào ma trận DP sẽ lập tức làm tràn RAM (OOM). Để giải quyết, hệ thống áp dụng cơ chế đếm phiếu để lọc ra đúng 30 video tiềm năng nhất:
  * **Tiêu chí 1 (Độ phủ):** Video chứa bao nhiêu sự kiện con ($E_i$) thì được bấy nhiêu phiếu.
  * **Tiêu chí 2 (Điểm trần):** Nếu các video bằng phiếu nhau, hệ thống phân định bằng cách tính tổng điểm Cosine cao nhất của từng sự kiện bên trong video đó.
  * **Lọc Top-K:** Sắp xếp danh sách giảm dần theo ưu tiên `(Số phiếu, Điểm trần)` rồi cắt lấy đúng 30 video đứng đầu.
  
  $\Rightarrow$ *Ý nghĩa:* Logic này triệt tiêu hoàn toàn các video nhiễu (ví dụ: cảnh $E_1$ khớp 100% nhưng lại không hề có $E_2, E_3$), giúp DANTE dồn tài nguyên xử lý đúng những video bám sát trọn vẹn kịch bản nhất.
* **Phân tích phương trình trạng thái:**
  Sau khi có 30 video tiềm năng, dùng quy hoạch động để dò tìm chuỗi khung hình tốt nhất bên trong từng video. Thuật toán khởi tạo một ma trận $DP[i][t]$, mang ý nghĩa: *Tổng điểm cực đại đạt được khi ghép xong $i$ sự kiện đầu tiên, và sự kiện thứ $i$ kết thúc chính xác tại khung hình ở mốc thời gian $t$*.
  
  Cốt lõi của thuật toán nằm ở phương trình chuyển trạng thái sau:
  $$DP[i][t] = Score(E_i, t) + \max_{t_{prev} < t} \big[ DP[i-1][t_{prev}] - \lambda \times (t - t_{prev}) \big]$$
  
  Bản chất công thức này là sự tính toán đánh đổi (Trade-off), được cấu thành từ 2 phần cốt lõi:
  1. **Điểm nội tại ($Score(E_i, t)$):** Đại diện cho độ khớp không gian (Cosine Score). Bức ảnh tại giây $t$ giống với mô tả văn bản của sự kiện $E_i$ đến mức nào. Điểm này càng gần 1.0 càng tốt.
  2. **Điểm kế thừa & Phạt thời gian (Cụm $\max$ phía sau):** Để quyết định chọn ảnh ở giây $t$, hệ thống phải tìm lại khung hình của sự kiện trước đó ($E_{i-1}$) nằm ở một mốc thời gian trong quá khứ ($t_{prev} < t$).
     * Nó sẽ lấy điểm tích lũy từ quá khứ: $DP[i-1][t_{prev}]$.
     * Tuy nhiên, hệ thống áp dụng một đòn bẩy là hệ số $\lambda$ (`lambda_penalty`). Khoảng cách giữa 2 sự kiện ($t - t_{prev}$) càng lớn, phép nhân $\lambda \times (t - t_{prev})$ sinh ra một lượng điểm trừ (Penalty) càng khổng lồ.

* **Dò ngược và Ánh xạ (Backtracking & Frame Mapping):**
  Sau khi điền đầy ma trận, hệ thống dò tìm ô có điểm số cao nhất ở dòng cuối cùng (sự kiện $E_N$). Từ điểm này, thuật toán dùng ma trận lưu vết (`trace`) để đi ngược về sự kiện $E_1$, truy xuất ra một chuỗi khung hình hoàn hảo nhất.
  Cuối cùng, một từ điển `map_db` được sử dụng để ánh xạ (map) các chỉ số nội bộ này trả về đúng chuẩn `frame_idx` (tên file ảnh thực tế) theo cấu trúc thư mục của Ban tổ chức. Tới đây, chuỗi kết quả mới chính thức sẵn sàng để đưa đi đánh giá lại.


### 2.3. Giai đoạn 3: Re-ranking bằng VLM
Để đảm bảo độ chính xác tuyệt đối, Top các chuỗi kết quả từ DANTE sẽ được đưa qua Gemini-3.1-Pro để kiểm chứng lại.
* Mô hình VLM sẽ trực tiếp đối chiếu trực tiếp hình ảnh với câu truy vấn và thẳng tay đánh 0 điểm nếu phát hiện vi phạm ở 1 trong 3 quy tắc sau:
  1. *Luật Văn bản (OCR):* Yêu cầu sự trùng khớp tuyệt đối về chữ viết. Nếu truy vấn có nhắc đến tên riêng, nhãn hiệu hoặc một đoạn văn bản cụ thể, khung hình bắt buộc phải chứa đúng chuỗi ký tự đó. Các trường hợp chỉ giống nhau về hình thức bề ngoài của đồ vật nhưng sai lệch chữ viết đều bị loại bỏ.
  2. *Luật Công cụ (Tools):* Yêu cầu tính hợp lý của hành động vật lý. Các công cụ, dụng cụ xuất hiện trong tay nhân vật phải tương thích chính xác với động từ thao tác được miêu tả trong truy vấn. Mọi sự đánh tráo hay sai lệch về loại công cụ thao tác đều dẫn đến kết quả bị hủy.
  3. *Luật Vật thể (Objects):* Yêu cầu nhận diện chuẩn xác thực thể trọng tâm. Khung hình phải chứa đúng đối tượng được yêu cầu dựa trên các đặc trưng sinh trắc, chủng loại hoặc màu sắc. Những khung hình bị "trùng khớp giả" do thuật toán vector phân loại nhầm sẽ bị phát hiện và đánh trượt ngay lập tức.
* Để bóc tách điểm số tự động, VLM bị ép (khóa `temperature=0.0`) phải trả về đúng định dạng chuẩn `JSON`. Đặc biệt, trong vòng lặp thẩm định Top 30, nếu hệ thống phát hiện một chuỗi được VLM chấm $\ge 0.95$ điểm, vòng lặp sẽ lập tức ngắt. Chuỗi đó được chốt làm kết quả cuối cùng, giúp đội thi tiết kiệm đáng kể thời gian chạy code và hạn mức gọi API. 

## 🚀 3. Hướng dẫn Cài đặt & Chạy

### 3.1. Chuẩn bị Dữ liệu (Google Drive)
Hệ thống yêu cầu mount trực tiếp vào Google Drive. Đảm bảo cấu trúc cây thư mục (Directory Tree) tại đường dẫn gốc `/content/drive/MyDrive/AIC26/` tuân thủ định dạng sau:

    /MyDrive/AIC26/
    ├── mapkeyframes/               # Chứa dữ liệu file .csv (phải có cột `n` và `frame_idx`). Tên file = Video ID.
    └── Keyframes/                  # Chứa ảnh tĩnh. ĐỊNH DẠNG BẮT BUỘC: File JPG, tên file được zero-padded 4 chữ số (VD: 0015.jpg).
        ├── Keyframes_L21/          # Các thư mục phân mảnh chứa video do ban tổ chức cung cấp
        ├── Keyframes_L22/
        ├── ...
        └── Keyframes_L30/
            └── L30_V001/           # Thư mục video
                └── 0015.jpg        # File ảnh keyframe

### 3.2. Cấu hình Khóa Xác thực
Mở tệp mã nguồn notebook, thay thế các API Key ở **[BLOCK 1]**:
* `PINECONE_API_KEY`: Khóa truy cập Pinecone (trỏ vào Index `aic-26`).
* `GEMINI_API_KEY`: Khóa Google AI Studio (đảm bảo đủ Quota gọi model flash và pro).

### 3.3. Quy trình Thực thi
1. Mở file `.ipynb` trên Google Colab.
2. Chọn `Runtime` -> `Change runtime type` -> **T4 GPU** (Bắt buộc để chạy SigLIP FP16).
3. Chạy toàn bộ các ô lệnh (Run All). Hệ thống sẽ tự động bóc tách truy vấn, chạy DANTE, chấm điểm bằng VLM và xuất kết quả.
4. Top 100 kết quả sẽ được ghi xuất ra thư mục `trake_submission` và tự động nén thành `submission.zip` tải xuống máy.
5. Ô lệnh cuối cùng sẽ khởi chạy giao diện **Gradio Web UI** (có link Public) để đội trực quan hóa và kiểm định chất lượng các khung hình.