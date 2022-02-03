Khi một backend process nhận được một query từ client, các bước xử lý sau sẽ được thực hiện:
1. Parser
2. Rewriter
3. Planner
4. Executor

Mình sẽ chỉ tập trung vào planner và executor của Postgres vì đối với một backend developer thì đây là phần chúng ta phải làm việc với nhiều nhất.

### Nhiệm vụ của Planner
Khi nhận được một câu query đã được parse và rewrite, planner sẽ tìm kế hoạch query hiệu quả nhất. Planner làm việc này bằng cách giả định cost cho các tác vụ rồi lựa chọn plan có tổng cost nhỏ nhất. Cái này gọi là cost-based optimiation. Lưu ý rằng cost không phải là đơn vị đo thời gian và cũng không phải đơn vị tuyệt đối (đây là điều nhiều người hiểu nhầm). cost là đơn vị tương đối, giúp bạn ước tính thời gian thực thi tương đối của các phần trong plan. Ví dụ mặc định, Postgres gán cho seq_page_cost = 1.0. Đây là cost để Postgres lấy một page từ disk lên khi đang thực thi sequential scan (khái niệm page và sequential scan sẽ được làm rõ ở các chương sau).

### Debug planner
Bạn có thể dùng [EXPLAIN](https://www.postgresql.org/docs/9.1/sql-explain.html) để kiểm tra plan mà planner đề ra cho một câu query bất kì. Đây là một tool cực kì hữu dụng. Ngoài ra thì EXPLAIN còn có một vài option rất bá nữa:
1. ANALYZE: với option này thì Postgres sẽ chạy thật query của bạn và thu thập runtime statistic luôn và trả ra cũng với plan: thời gian + số row output
2. BUFFERS: với option này thì Postgres sẽ thu thập cả thông tin về IO activity lúc chạy query
Mình khuyên là bạn nên chạy `EXPLAIN (ANALYZE, BUFFERS) QUERY` nếu cần phải debug query. Mình sẽ có một chương riêng về cách đọc hiểu output của EXPLAIN.

### Làm thế nào mà Planner làm được như vậy?
Sở dĩ planner có thể ước lượng được cost của một query là vì Postgres duy trì statistics của các bảng. Statistics này được lưu trong bảng pg_stats. Ví dụ `most_common_vals` và `most_common_freqs` lưu lại các giá trị hay xuất hiện của một column cùng với tần suất của nó. Ví dụ khi bạn query tất cả các nhân viên có giới tính là nam trong bảng **employees**, Postgres sẽ ước lượng được có *khoảng* bao nhiêu nhân viên nam trong bảng ấy. Từ ước lượng này Postgres sẽ tính toán được cost ứng với từng plan.
Planner chỉ hoạt động hiệu quả khi sai số giữa thống kê và dữ liệu thực không quá lớn. Tuy nhiên rất ít trường hợp mình thấy planner của Postgres đưa ra một plan kém hiệu quả. Mà trong trường hợp có xảy ra thì dev cũng bất lực vì không thể bắt planner dùng một plan hay đưa gợi ý cho planner để nó chọn plan mà mình muốn, chỉ có cách điều chỉnh lại config của Postgres thôi. Theo mình thì đây cũng là một nhược điểm của Postgres planner.

### Tài liệu tham khảo
https://www.interdb.jp/pg/pgsql03.html
https://www.postgresql.org/docs/current/query-path.html
https://malisper.me/how-postgres-plans-queries/
https://www.postgresql.org/docs/9.5/planner-stats-details.html