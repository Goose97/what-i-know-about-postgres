Việc tạo index có thể cải thiện rất nhiều read performance với cái giá phải trả là write performance của bạn. Khi bạn insert hay update một row, 1 tuple mới sẽ được thêm vào. Mỗi index trên bảng đều phải duy trì thêm 1 index entry ứng với tuple mới kia, ngay cả khi chúng ta chỉ thay đổi các cột không nằm trong index. Hiện tượng này là *write amplification*, nếu bạn 10 index trên 1 bảng, 1 lần insert vào bảng sẽ dẫn đến 11 write operation thay vì 1. Đây là những cách mình dùng để khắc phục vấn đề này:

### Cân nhắc bỏ index
Bạn có thể kiểm tra tần suất của sử dụng của các index để cân nhắc đến việc bỏ chúng. Nếu bạn có nhiều index đơn trên 1 cột thì có thể cân nhắc đến việc bỏ chúng đi và tạo 1 index gộp. Postgres còn có 1 loại [index đặc biệt](https://www.postgresql.org/docs/10/bloom.html) sử dụng Bloom filter, có khả năng thay thế nhiều index đơn mà vẫn đáp ứng tốt nhu cầu query phức tạp.

### Tạo partial index
Đây là một cách rất hữu hiệu nhưng đòi hỏi bạn nắm rõ các query pattern của ứng dụng mình.

### Tận dụng HOT update
HOT (Heap-Only Tuple) update là một kĩ thuật được Postgres sử dụng để khắc phục chính xác vấn đề trên: **khi một row được update liên tục trên cột không nằm trong index**. Cùng nhìn vào ví dụ cụ thể nhé:

```sql
-- Insert 1 bản ghi đầu tiên và kiểm tra nội dung trên index
SELECT generate_sample_data(1, 100);
CREATE INDEX sample_table_a ON sample_table (a);
SELECT * FROM index_page('sample_table_a', 1);
 ctid  | itemoffset |          data           |                id                | a  | b
-------+------------+-------------------------+----------------------------------+----+----
 (0,1) |          1 | 5e 00 00 00 00 00 00 00 | dd08f309f3187cc64bf9b5cdaf133b0f | 94 | 13
(1 row)

-- Giờ thử cập nhật trên cột có index nhé. Trong trường hợp này, Postgres sẽ không áp dụng được HOT-update, 
-- vì vậy chúng ta sẽ có 2 index entry
UPDATE sample_table SET a = a + 10;
SELECT * FROM index_page('sample_table_a', 1);
 ctid  | itemoffset |          data           |                id                |  a  | b
-------+------------+-------------------------+----------------------------------+-----+----
 (0,1) |          1 | 5e 00 00 00 00 00 00 00 |                                  |     |
 (0,2) |          2 | 68 00 00 00 00 00 00 00 | dd08f309f3187cc64bf9b5cdaf133b0f | 104 | 13
(2 rows)

-- Bây giờ thử update trên cột không có index xem. Chúng ta không có thêm index entry nào,
-- đồng nghĩa với việc ta loại bỏ được phần write operation với index
UPDATE sample_table SET b = b + 10;
SELECT * FROM index_page('sample_table_a', 1);
 ctid  | itemoffset |          data           | id | a | b
-------+------------+-------------------------+----+---+---
 (0,1) |          1 | 5e 00 00 00 00 00 00 00 |    |   |
 (0,2) |          2 | 68 00 00 00 00 00 00 00 |    |   |
(2 rows)

-- Kiểm tra xem nội dung trên heap có gì nhé
SELECT * FROM heap_page('sample_table', 0);
 ctid  | state  |       t_xmin        |       t_xmax        | t_ctid
-------+--------+---------------------+---------------------+--------
 (0,1) | normal | 1314022 (committed) | 1314024 (committed) | (0,2)
 (0,2) | normal | 1314024 (committed) | 1314025 (committed) | (0,3)
 (0,3) | normal | 1314025 (committed) | 0 (aborted/invalid) | (0,3)
(3 rows)
```

Vậy Postgres đang làm trò ảo thuật gì ở đây? Cụ thể, Postgres làm những việc sau:
1. Đánh dấu một flag đặc biệt cho tuple được update, thông báo rằng tuple này đã được HOT-update.
2. Insert một tuple mới vào *cùng page* và set các metadata như bình thường (t_xmin, t_xmax, t_ctid, ...).
3. Khi truy vấn từ index, Postgres sẽ được trỏ với tuple cũ (ctid = (0,2)). Postgres nhận ra flag đặc biệt và đi theo pointer của t_ctid để tìm ra tuple mới nhất. Các t_ctid nối với nhau tạo thành một chuỗi, gọi là HOT-update chain.

Như bạn thấy đấy, toàn bộ quá trình trên không hề đả động gì đến việc cập nhật index, khiến HOT-update trở nên cực kì hữu hiệu trong trường hợp này. Tuy nhiên, bạn cần đảm bảo 2 điều kiện sau để HOT-update diễn ra:
1. Update của bạn có thể HOT được: update của bạn không thay đổi trên các cột đã có index. Nếu bạn có 10 index, chỉ cần update của bạn liên quan tới 1 trong 10 index trên là update đã không HOT được nữa rồi. Vì vậy nếu bạn có index trên những cột được tạo tự động, ví dụ như updated_at (cập nhật sau mỗi lần được update) thì nhiều khả năng là HOT update sẽ không hiệu quả.
2. Page chứa tuple được update phải còn đủ chỗ cho tuple mới. Đây là một quyết định của team Postgres khi thiết kế phần này, giữ HOT-update chain chỉ tồn tại trong cùng một page. Để đảm bảo page luôn đủ chỗ, bạn có thể can thiệp bằng 2 cách:
  - Vacuum bảng thường xuyên hơn để dọn dẹp các dead tuple.
  - Cài đặt fillfactor của bảng xuống mức < 100 (đây là mức mặc định). Fillfactor quyết định xem bao nhiêu phần trăm dung lượng của page sẽ được dùng để lưu trữ. Ví dụ nếu bạn cài đặt fillfactor của bảng là 80, Postgres sẽ chỉ thêm tuple vào page cho đến khi đạt 80% dung lượng thôi, phần còn lại sẽ để trống để dành cho HOT-update. Tuy nhiên nhược điểm là dung lượng bảng của bạn sẽ tăng lên 25%. Dung lượng bảng lớn hơn cũng đồng nghĩa với việc performance của 1 số read operation cũng giảm, vì Postgres sẽ phải fetch nhiều page hơn. Vì vậy lựa chọn fillfactor phù hợp phụ thuộc vào rất nhiều yếu tố, bao gồm tần suất update của ứng dụng cũng với tài nguyên của hệ thống và cần được benchmark kĩ càng.

```sql
-- Kiểm tra số lần HOT-update
SELECT n_tup_hot_upd FROM pg_stat_user_tables WHERE relname = 'table_name';
```

Nếu các bạn muốn tìm hiểu cụ thể hơn về HOT-update, mình khuyên nên đọc file [README.HOT](https://github.com/postgres/postgres/blob/master/src/backend/access/heap/README.HOT) ở trong source code của Postgres, rất đáng đọc nhé.

### Tài liệu tham khảo
https://use-the-index-luke.com/blog/2016-07-29/on-ubers-choice-of-databases
https://www.cybertec-postgresql.com/en/hot-updates-in-postgresql-for-better-performance/
https://habr.com/en/company/postgrespro/blog/483768/
https://medium.com/nerd-for-tech/postgres-fillfactor-baf3117aca0a