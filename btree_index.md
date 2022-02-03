Btree index là loại index mặc định khi Postgres khởi tạo index. Btree index có thể sử dụng cho query với điều kiện bằng (=) hoặc trong khoảng (> hoặc <).

### Cấu trúc của index
Các node lá của btree sẽ lưu trữ thông tin của tuple, bao gồm tất cả các column được index và ctid của tuple ấy. Nếu index không phải index 1 phần (partial index), mỗi tuple trên heap đều có một index entry tương ứng. Vì vậy nếu bạn tạo 1 index trên tất cả các cột của 1 bảng, dung lượng của index sẽ gần bằng dung lượng của chính bảng đó.
Giờ tạo thử 1 index nhé:
```sql
TRUNCATE TABLE sample_table;
SELECT generate_sample_data(10, 100);
CREATE INDEX sample_table_a ON sample_table (a);
```

Sử dụng helper function mình đã viết ở phần đầu để kiểm tra nội dung index. Cũng giống với bảng, index cũng được tổ chức thành các page 8kB. Page đầu tiên là metadata page, nên chúng ta sẽ kiểm tra page thứ hai.

```sql
-- Mỗi index entry được join với tuple để dễ tham chiếu
SELECT * FROM index_page('sample_table_a', 1);
  ctid  | itemoffset |          data           |                id                |  a  | b
--------+------------+-------------------------+----------------------------------+-----+----
 (0,6)  |          1 | 03 00 00 00 00 00 00 00 | 30f8ac0882f0d0c225e3486329674b1f |   3 | 99
 (0,2)  |          2 | 1a 00 00 00 00 00 00 00 | 224729c7197babf912320c17031a546b |  26 | 39
 (0,9)  |          3 | 20 00 00 00 00 00 00 00 | c397a7a7f095f648e2bc49e4525861cd |  32 | 15
 (0,8)  |          4 | 34 00 00 00 00 00 00 00 | 39b0c0be36668daee3b832e9efdc2a84 |  52 | 80
 (0,4)  |          5 | 3f 00 00 00 00 00 00 00 | c7ec43a1b05fd9b43e2ebdfcf74fb70e |  63 | 52
 (0,3)  |          6 | 42 00 00 00 00 00 00 00 | 01424e97414ab1c2e2427f8a154db693 |  66 | 43
 (0,10) |          7 | 4b 00 00 00 00 00 00 00 | e97c4e18392719537c1ad135fec7ff50 |  75 | 17
 (0,1)  |          8 | 4c 00 00 00 00 00 00 00 | 46f18acd5ff27f4cc480523718a78b98 |  76 | 23
 (0,7)  |          9 | 63 00 00 00 00 00 00 00 | 71b49ac7fbff890dd068a2a7390fbd8e |  99 | 21
 (0,5)  |         10 | 64 00 00 00 00 00 00 00 | a8e1a51a8f1c025bb073d6758a077e25 | 100 | 97
(10 rows)
```

Mỗi index entry bao gồm giá trị của cột được index được lưu ở cột data và ctid tương ứng của tuple ở trên heap. Các index entry được sắp xếp theo thứ tự tăng dần của giá trị a (đây là chiều sort mặc định, có option để sort theo chiều ngược lại).

### Index trên nhiều cột
Btree cho phép bạn tạo index trên nhiều cột. Lúc này các index entry sẽ được sort trên cả 2 trường, giống như lúc chúng ta ORDER BY ấy.

```sql
CREATE INDEX sample_table_a_b ON sample_table (a, b);
SELECT * FROM index_page('sample_table_a_b', 1);
  ctid  | itemoffset |          data           |                id                 |  a  | b
--------+------------+-------------------------+-----------------------------------+-----+----
 (0,6)  |          1 | 03 00 00 00 63 00 00 00 | 30f8ac0882f0d0c225e3486329674b1f  |   3 | 99
 (0,2)  |          2 | 1a 00 00 00 27 00 00 00 | 224729c7197babf912320c17031a546b  |  26 | 39
 (0,13) |          3 | 1a 00 00 00 58 00 00 00 | 224729c7197babf912320c17031a546a  |  26 | 88
 (0,9)  |          4 | 20 00 00 00 0f 00 00 00 | c397a7a7f095f648e2bc49e4525861cd  |  32 | 15
 (0,15) |          5 | 34 00 00 00 37 00 00 00 | 39b0c0be36668daee3b832e9efdc2a85  |  52 | 55
 (0,8)  |          6 | 34 00 00 00 50 00 00 00 | 39b0c0be36668daee3b832e9efdc2a84  |  52 | 80
 (0,4)  |          7 | 3f 00 00 00 34 00 00 00 | c7ec43a1b05fd9b43e2ebdfcf74fb70e  |  63 | 52
 (0,3)  |          8 | 42 00 00 00 2b 00 00 00 | 01424e97414ab1c2e2427f8a154db693  |  66 | 43
 (0,10) |          9 | 4b 00 00 00 11 00 00 00 | e97c4e18392719537c1ad135fec7ff50  |  75 | 17
 (0,1)  |         10 | 4c 00 00 00 17 00 00 00 | 46f18acd5ff27f4cc480523718a78b98  |  76 | 23
 (0,12) |         11 | 4c 00 00 00 1c 00 00 00 | 46f18acd5ff27f4cc480523718a78b989 |  76 | 28
 (0,14) |         12 | 4c 00 00 00 31 00 00 00 | 46f18acd5ff27f4cc480523718a78b12  |  76 | 49
 (0,7)  |         13 | 63 00 00 00 15 00 00 00 | 71b49ac7fbff890dd068a2a7390fbd8e  |  99 | 21
 (0,5)  |         14 | 64 00 00 00 61 00 00 00 | a8e1a51a8f1c025bb073d6758a077e25  | 100 | 97
(14 rows)
```

### Btree index được dùng trong trường hợp nào?
Một câu hỏi mà bản thân mình gặp rất nhiều: Tại sao Postgres không sử dụng index trong câu query này? Dưới đây là một số trường hợp mà btree index hoạt động hiệu quả.
#### Trong câu query tìm kiếm trên cột index
Lưu ý, Btree chỉ sử dụng được 5 toán tử (=, >, <, >=, <=) và chỉ có thế tìm kiếm hiệu quả trên các cột ở tiền tố. Ví dụ khi sử dụng index *sample_table_a_b* bên trên, tìm kiếm trên cột a hoặc tìm kiếm trên cột a + b sẽ hiệu quả, còn tìm kiếm theo b thì không. Lí do là vì bản thân b không được sắp xếp theo thứ tự nào cả. Ví dụ, với index gộp trên 3 cột a + b + c:
  - Tìm kiếm theo a: có thể sử dụng index
  - Tìm kiếm theo a + b: có thể sử dụng index
  - Tìm kiếm theo a + b + c: có thể sử dụng index
  - Tìm kiếm theo a + c: có thể sử dụng index cho mình điều kiện a rồi tiếp tục lọc theo điều kiện c
  - Tìm kiếm theo b + c (hoặc b, hoặc c): không thể sử dụng index
Ở bên trên mình sử dụng từ *có thể* vì Postgres sẽ cân nhắc thêm nhiều yếu tố nữa để quyết định có dùng index scan hay không.

#### Trong câu query có ORDER BY
Postgres có thể lợi dụng tính chất sắp xếp của dữ liệu trong index để tối ưu cho câu query có sử dụng ORDER BY, đặc biệt là khi kết hợp với LIMIT. Ví dụ câu query sau đây:
```sql
SELECT * FROM sample_table ORDER BY a LIMIT 10;
```
  - Trường hợp không có index trên cột a: Postgres sẽ scan toàn bộ bảng rồi sort và lấy ra 10 bản ghi đầu tiên. Nếu số lượng row trong bảng lớn, rất nhiều cột sẽ được đọc ra nhưng không nằm trong kết quả trả về.
  - Trường hợp có index trên cột a: Postgres chỉ cần scan theo thứ tự trong index cho đến khi tìm thấy đủ 10 row.

Index có thể scan theo chiều ngược lại nên index trên có thể dùng trong cả trường hợp *ORDER BY a DESC*. Đối với index gộp trên nhiều cột, ta cũng có thể suy luận tương tự, ví dụ xét index trên a + b:
  - ORDER BY a, b: có thể sử dụng index
  - ORDER BY b: không thể sử dụng index
  - WHERE a = ? ORDER BY b: có thể sử dụng index
  - WHERE a > ? ORDER BY b: không thể sử dụng index

*Lưu ý*: khi Postgres sử dụng Bitmap Index Scan, thứ tự sắp xếp trên sẽ bị mất do khi tạo thành bitmap, các tuple sẽ được sắp xếp theo thứ tự xuất hiện trên physical disk, thay vì thứ tự xuất hiện trong index.

### Tài liệu tham khảo
https://www.postgresql.org/docs/11/indexes.html
https://www.postgresql.org/docs/13/btree-implementation.html