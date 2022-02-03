MVCC (Multi-version concurrency control) là kĩ thuật Postgres sử dụng để thiết kế transaction. Thiết kế này đảm bảo nhiều transaction có thể chạy đồng thời (concurrency) mà vẫn đảm bảo tính độc lập với nhau.

### Transaction ID
Mỗi transaction được Postgres định danh bằng 1 transaction ID, 1 số nguyên 32 byte tăng dần. Không gian của transaction ID là một circular buffer, nghĩa là khi transaction ID vượt quá giá trị max, nó sẽ được reset về 3 (0, 1, 2 là 3 giá trị đặc biệt không được dùng). Transaction ID được dùng để xác định transaction A xảy ra xong "quá khứ" hay "tương lai" đối với transaction hiện tại. Với mỗi transaction, 2 tỉ transaction phía trước (2**31 - 1) được coi là xảy ra trong tương lai và 2 tỉ transaction phía sau được coi là xảy ra trong quá khứ.

### Transaction snapshot
Transaction snapshot được dùng để lưu lại trạng thái của tất cả các transaction transaction của một database tại một thời điểm nhất định. Mỗi transaction có thể nằm 1 trong 3 trạng thái: IN_PROGRESS, COMMITTED và ABORTED.
```sql
SELECT txid_current_snapshot();
txid_current_snapshot
-----------------------
 1313934:1313940:1313934,1313937
(1 row)
```

Postgres biểu diễn snapshot với format `xmin:xmax:xip_list`. Trong đó:
- xmin: là transaction ID nhỏ nhất vẫn còn active. Nghĩa là tất cả các transaction trước thời điểm này đều đã hoàn thành, hoặc commit hoặc rollback.
- xmax: là transaction tiếp theo chưa được assign. Nghĩa là mọi transaction từ thời điểm này trở về sau đều chưa được khởi tạo.
- xip_list: danh sách các transaction đang active (ở status là IN_PROGRESS).

### Insert, delete và update
Mỗi row trong Postgres tồn tại cùng lúc nhiều version (multi-version). Từ bây giờ mình sẽ gọi các version này là tuple (giống trong doc). Row là khái niệm abstract mà application chúng ta biết còn tuple chính là internal representation của row. Mỗi khi ta update một row, Postgres sẽ không xoá tuple cũ mà thêm vào 1 tuple mới với dữ liệu đã được chỉnh sửa. Các tuple ngoài lưu dữ liệu còn lưu các metadata phục vụ cho MVCC model. Ta sẽ tập trung vào 3 field quan trọng là t_xmin, t_xmax và t_ctid.
- t_xmin: lưu transaction ID của transaction insert tuple này.
- t_xmax: lưu transaction ID của transaction delete tuple này. Nếu tuple chưa bị delete thì t_xmax = 0 (transaction có ID 0 được gọi là invalid transaction).
- t_ctid: là pointer trỏ đến version mới hơn của tuple này. Các version của một tuple sẽ tạo thành một linked list với pointer là t_ctid. Version cuối cùng có t_ctid là ctid của chính nó.

Vậy các tác vụ insert, delete và update có thể hiểu đơn giản là:
#### Insert: thêm tuple mới với t_xmin = transaction ID hiện tại và t_xmax = 0
```sql
BEGIN;
SELECT txid_current();
 txid_current
--------------
      1313962
(1 row)

SELECT generate_sample_data(10, 100);

-- Tất cả 10 bản ghi đều có xmin = 1313962 và t_xmax = 0
SELECT * FROM heap_page('sample_table', 0);
  ctid  | state  | t_xmin  |       t_xmax        | t_ctid
--------+--------+---------+---------------------+--------
 (0,1)  | normal | 1313962 | 0 (aborted/invalid) | (0,1)
 (0,2)  | normal | 1313962 | 0 (aborted/invalid) | (0,2)
 (0,3)  | normal | 1313962 | 0 (aborted/invalid) | (0,3)
 (0,4)  | normal | 1313962 | 0 (aborted/invalid) | (0,4)
 (0,5)  | normal | 1313962 | 0 (aborted/invalid) | (0,5)
 (0,6)  | normal | 1313962 | 0 (aborted/invalid) | (0,6)
 (0,7)  | normal | 1313962 | 0 (aborted/invalid) | (0,7)
 (0,8)  | normal | 1313962 | 0 (aborted/invalid) | (0,8)
 (0,9)  | normal | 1313962 | 0 (aborted/invalid) | (0,9)
 (0,10) | normal | 1313962 | 0 (aborted/invalid) | (0,10)
(10 rows)
```

#### Delete: set t_xmax của tuple = transaction ID hiện tại
```sql
BEGIN;
SELECT txid_current();
 txid_current
--------------
      1313967
(1 row)

DELETE * FROM sample_table;

-- Tất cả bản ghi đều có t_xmax = 1313968
SELECT * FROM heap_page('sample_table', 0);
  ctid  | state  |       t_xmin        | t_xmax  | t_ctid
--------+--------+---------------------+---------+--------
 (0,1)  | normal | 1313962 (committed) | 1313968 | (0,1)
 (0,2)  | normal | 1313962 (committed) | 1313968 | (0,2)
 (0,3)  | normal | 1313962 (committed) | 1313968 | (0,3)
 (0,4)  | normal | 1313962 (committed) | 1313968 | (0,4)
 (0,5)  | normal | 1313962 (committed) | 1313968 | (0,5)
 (0,6)  | normal | 1313962 (committed) | 1313968 | (0,6)
 (0,7)  | normal | 1313962 (committed) | 1313968 | (0,7)
 (0,8)  | normal | 1313962 (committed) | 1313968 | (0,8)
 (0,9)  | normal | 1313962 (committed) | 1313968 | (0,9)
 (0,10) | normal | 1313962 (committed) | 1313968 | (0,10)
(10 rows)
```

#### Update: delete tuple hiện tại, insert tuple với giá trị mới và cập nhật t_ctid của tuple cũ trỏ tới tuple mới.
```sql
BEGIN;
TRUNCATE TABLE sample_table;
SELECT txid_current();
 txid_current
--------------
      1313975
(1 row)

SELECT generate_sample_data(1, 100);

-- Sau khi insert chúng ta có 1 tuple
SELECT * FROM heap_page('sample_table', 0);
 ctid  | state  | t_xmin  |       t_xmax        | t_ctid
-------+--------+---------+---------------------+--------
 (0,1) | normal | 1313975 | 0 (aborted/invalid) | (0,1)

UPDATE sample_table SET a = a + 10, b = b + 20;

-- Sau khi insert chúng ta có 2 tuple, 1 tuple được insert mới và 1 tuple bị delete. t_ctid của tuple
-- bị xoá trỏ đến version mới hơn của nó
SELECT * FROM heap_page('sample_table', 0);
 ctid  | state  | t_xmin  |       t_xmax        | t_ctid
-------+--------+---------+---------------------+--------
 (0,1) | normal | 1313975 | 1313975             | (0,2)
 (0,2) | normal | 1313975 | 0 (aborted/invalid) | (0,2)
(2 rows)

COMMIT;
```

Có thể nói mỗi tuple trong Postgres là một *immutable object* (giống với cách tiếp cận của Elixir). Điểm mạnh của mô hình này là xử lý concurrency rất dễ dàng mà không cần nhiều logic lock phức tạp. Điểm yếu là performance bị ảnh hưởng (nhất là write performance) đồng thời tạo ra nhiều dead resource, cần phải có logic để garbage collect.

### Visibility check
Mỗi row trong Postgres tồn tại cùng lúc nhiều version. Nhưng rõ ràng bạn không muốn query của bạn trả ra 2 user Alice đúng không? Vì vậy một transaction cần đảm bảo chỉ có thể nhìn thấy nhiều nhất là 1 tuple của cùng một row thôi. Postgres đạt được điều này bằng cách thực hiện visibility check trên mỗi tuple. Visibility check hiểu đơn giản là một tác vụ: cho transaction ID hiện tại, transaction snapshot và 1 tuple bất kì, câu hỏi là transaction hiện tại có thể **"nhìn thấy"** tuple này không? Visibility check có rất nhiều rules (vì phải handle nhiều edge cases) nên mình sẽ không liệt kê hết ra ở đây, nhưng để minh hoạ cho dễ hiểu thì nó sẽ như thế này:
1. Nếu status của t_xmin là `ABORTED` (transaction insert tuple này bị rollback) --> không nhìn thấy
2. Nếu status của t_xmin là `IN_PROGRESS` (transaction insert tuple này vẫn đang active) --> chỉ nhìn thấy nếu t_xmin = transaction ID hiện tại (chính transaction hiện tại insert tuple này)
3. Nếu status của t_xmin là `COMMITTED` (transaction insert tuple đã commit)
- Nếu t_xmax = 0 hoặc status của t_xmax là `ABORTED` --> có nhìn thấy
- Nếu status của t_xmax là `COMMITTED` --> không nhìn thấy
- Nếu status của t_xmax là `IN_PROGRESS` --> chỉ không nhìn thấy nếu t_xmax = transaction ID hiện tại (chính transaction hiện tại delete tuple này)

### Isolation level
Như đã đề cập ở đầu, MVCC sinh ra để đảm bảo các transaction có thể chạy đồng thời mà vẫn đảm bảo tính độc lập (isolation). Tuy nhiên, isolation cũng có nhiều cấp độ. Sau đây là 4 cấp độ được định nghĩa trong SQL standard, sắp xếp theo thứ tự isolation tăng dần:
1. Uncomitted read: một transaction có thể đọc được các thay đổi chưa commit của một transaction khác. Hiện tượng này gọi là **dirty read**. Mình không rõ trong trường hợp nào thì cần sử dụng isolation level này. Postgres không support level này.
2. Committed read: để tránh **dirty read** thì ta có thể dùng level này (đây là level mặc định của Postgres). Một transaction chỉ có thể đọc được các thay đổi đã được commit của một transaction khác. Trước khi thực thì một statement, transaction hiện tại sẽ tạo ra 1 transaction snapshot và sử dụng trong visibility check. Điều này đồng nghĩa là hai statement trong cùng một transaction, thực thi trong hai thời điểm khác nhau có thể có hai transaction snapshot khác nhau dẫn đến kết quả đọc ra khác nhau. Ví dụ:
```
t=0: Transaction A đếm số sản phẩm bút chì đã bán được trong tháng này, kết quả là 43.
t=1: Transaction B ghi nhận thêm đơn hàng gồm 3 bút chì.
t=2: Transaction A đếm lại số sảm phẩm bút chì đã bán trong tháng, kết quả là 46.
```
Hiện tượng này gọi là **non-repeatable read**.
3. Repeatable-read: để khắc phục **non-repeatable read** thì ta có thể dùng level này. Khác với committed read, transaction chỉ tạo transaction snapshot một lần khi thực thi statement đầu tiên. Snapshot này sẽ tồn tại đến khi transaction hoàn thành.
4. Serializable: đây là isolation level cao nhất mà Postgres support. Chưa đọc @@

### Tài liệu tham khảo
https://habr.com/en/company/postgrespro/blog/479512/
https://www.interdb.jp/pg/pgsql05.html
https://www.postgresql.org/docs/9.5/transaction-iso.html
https://malisper.me/postgres-mvcc