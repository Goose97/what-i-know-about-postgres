Phần này sẽ bàn đến một số cách là Postgres áp dụng để thực thi câu query. Mình sẽ chỉ nói về câu SELECT vì đây là phần phức tạp nhất. Trước khi nói cụ thể, mình phải làm rõ một số khái niệm trước:
1. Index: index là một secondary table, nó lưu lại dữ liệu chính xác như bảng của bạn nhưng được lưu trữ trong những data structure đặc biệt để phục vụ cho thao tác truy vấn nhanh hơn. Một ví dụ cho dễ hiểu nhé:
  Bạn là giáo viên chủ nhiệm một lớp học gồm 30 học sinh với tên họ khác nhau. Hãy tưởng tượng 30 học sinh trong lớp ứng với 30 row trong bảng. Vị trí ngồi của các học sinh chính là vị trí sắp xếp trên disk của các row, hay như ta biết chính là ctid. Nhiệm vụ của bạn là tìm 1 em học sinh theo tên họ (SELECT * FROM students WHERE fullname = ?). Cách đơn giản nhất là bạn có thể kiểm tra từng dãy một, từng bàn một cho đến khi tìm ra em học sinh ấy. Hoặc hiệu quả hơn, bạn lập một danh sách lớp cho tất cả học sinh trong lớp, bao gồm tên họ và vị trí ngồi. Tên họ của các học sinh sẽ được sắp xếp theo thứ tự xuất hiện trong bảng chữ cái. Khi tìm kiếm theo tên, ta có thể vận dụng tính chất này để tăng tốc độ truy vấn, ví dụ dùng binary search chẳng hạn. Cái danh sách lớp mà chúng ta vừa tạo chính là một index. Danh sách các học sinh (các row) được sắp xếp theo một ý đồ đặc biệt nhằm phục vụ câu truy vấn (tìm kiếm theo tên).
2. Selectivity: selectivity của một điều kiện lọc là tỉ lệ giữa số row thoả mãn điều kiện trên tổng số row. Ví dụ bạn muốn lọc tất cả các công dân Việt Nam có độ tuổi > 5, ta nói điều kiện lọc này có selectivity cao. Còn nếu bạn muốn lọc tất cả công dân Việt Nam có độ tuổi > 95, ta nói điều kiện này có selectivity thấp.

Trước tiên hãy khởi tạo dữ liệu test nhé:
```sql
TRUNCATE TABLE sample_table;
SELECT generate_sample_data(1000, 100);
```

### Scan
Khi Postgres tìm kiếm một bản ghi, chúng ta gọi đó là scan. Với scan, Postgres có 3 plan chính là sequential scan, index scan và bitmap index scan.

#### Sequential scan
Plan này là dễ hiểu nhất. Khi scan một bảng, Postgres sẽ đọc từng tuple trên từng page một. Plan này có thể sử dụng tối ưu cho các điều kiện có selectivity cao.
**Điểm mạnh**: plan này Postgres có thể dùng được với mọi query. IO access pattern là sequential --> nhanh hơn random access.
**Điểm yếu**: nếu kết quả đầu ra chỉ có một số ít row thì cách này rất chậm, nhiều operation lãng phí.
```sql
EXPLAIN SELECT * FROM sample_table WHERE a > 20;
                            QUERY PLAN
------------------------------------------------------------------
 Seq Scan on sample_table  (cost=0.00..24.75 rows=932 width=41)
   Filter: (a > 20)
```

#### Index scan
Đây sẽ là plan chúng ta gặp nhiều nhất. Postgres sẽ sử dụng index để tìm kiếm trên điều kiện lọc. Kết quả tìm kiếm trên index sẽ cho ra ctid của các row thoả mãn, lúc đó Postgres chỉ cần fetch từng row lên là được. Plan này có thể sử dụng tối ưu cho các điều kiện có selectivity thấp.
**Điểm mạnh**: với trường hợp kết quả đầu ra chỉ có một số ít row thì cách này rất hiệu quả, giảm số lượng page cần fetch xuống mức tối thiểu.
**Điểm yếu**: phải duy trì index song song với bảng. Đồng thời câu query phải thực hiện 2 lần truy vấn, 1 lần vào index và 1 lần vào bảng.
```sql
EXPLAIN SELECT * FROM sample_table WHERE a > 99;
                                     QUERY PLAN
------------------------------------------------------------------------------------
 Index Scan using sample_table_a on sample_table  (cost=0.28..8.29 rows=1 width=41)
   Index Cond: (a > 99)
```

#### Bitmap index scan
Còn những điều kiện có selectivity trung bình thì sao, Postgres sẽ sử dụng bitmap index scan. Giống với index scan, Postgres trước tiên sẽ truy vấn vào index. Sau khi thu được ctid của các row, Postgres sẽ thiết lập một bitmap phản ánh vị trí các row cần fetch. Ví dụ chúng ta có 10 row và có 3 row 1, 3, 8 thoả mãn điều kiện lọc. Bitmap của chúng ta sẽ trông như thế này: 1010000100. Bitmap này được sắp xếp theo thứ tự tăng dần của page number và tuple trong page. Sau đó Postgres sẽ dựa theo bitmap này để fetch các page cần thiết lên. Điểm thú vị của bitmap index scan là Postgres có thể sử dụng cùng lúc nhiều index. Ví dụ câu query có hai điều kiện trên cột a và b, Postgres có thể sử dụng 2 index riêng biệt trên a và b, tạo ra 2 bitmap rồi AND (OR) với nhau.
```sql
EXPLAIN SELECT * FROM sample_table WHERE a > 80;
                                  QUERY PLAN
-------------------------------------------------------------------------------
 Bitmap Heap Scan on sample_table  (cost=9.69..21.96 rows=182 width=41)
   Recheck Cond: (a > 80)
   ->  Bitmap Index Scan on sample_table_a  (cost=0.00..9.64 rows=182 width=0)
         Index Cond: (a > 80)
```

Như mình đã nói, 3 câu query trên có cấu trúc y hệt nhau, chỉ khác nhau ở selectivity nên plan mà Postgres sử dụng cũng khác nhau hoàn toàn.

### Join
Ngoài scan ra thì một tác vụ chúng ta hay thực hiện đó là join. Với join, Postgres có 3 plan chính là nested loop join, hash join và merge join.

#### Nested loop join
Đây là phương thức join đơn sơ nhất. Ví dụ join bảng A và B, bạn loop qua từng row của bảng A, sau đó ứng với mỗi row lại loop qua từng row của bảng B rồi tìm tất cả row thoả mãn điều kiện join. Nếu hai bảng lần lượt cho ra m và n row thì time complexity sẽ là O(m * n). Nested loop join có thể sử dụng index rất hiệu quả giúp hiệu năng tăng đáng kể. Nếu tồn tại index với điều kiện join trên bảng B, ta có thể lookup trực tiếp trên index mà không phải loop qua toàn bộ row nữa.
**Điểm mạnh**: giống với sequence scan thì plan này cân tất cả thể loại join, dù bảng có lớn nhỏ thế nào, điều kiện join là bằng, không bằng hay trong khoảng.
**Điểm yếu**: nếu m và n đều lớn, cộng với không có index thì nested loop join sẽ biến thành nỗi ác mộng.
```sql
EXPLAIN SELECT * FROM
	sample_table outer_table
	JOIN sample_table inner_table ON outer_table.a = inner_table.b
WHERE
	outer_table.a > 99

                                              QUERY PLAN
------------------------------------------------------------------------------------------------------
 Nested Loop  (cost=4.63..23.30 rows=10 width=82)
   ->  Index Scan using sample_table_a on sample_table outer_table  (cost=0.28..8.29 rows=1 width=41)
         Index Cond: (a > 99)
   ->  Bitmap Heap Scan on sample_table inner_table  (cost=4.35..14.91 rows=10 width=41)
         Recheck Cond: (b = outer_table.a)
         ->  Bitmap Index Scan on sample_table_b  (cost=0.00..4.35 rows=10 width=0)
               Index Cond: (b = outer_table.a)
```

#### Hash join
Muốn giải được 2 vòng loop thì thực ra có cách khá là đơn giản, loop qua các row của bảng A rồi tạo một hash table. Sau đó chỉ cần loop qua bảng B 1 lần rồi lookup các row tương ứng thông qua hash table thôi. Time complexity giờ chỉ còn O(m + n).
**Điểm mạnh**: performance cực nhanh, ngay cả với những bảng lớn.
**Điểm yếu**:
  - Không tận dụng được index trên điều kiện join (index trên WHERE thì vẫn thoải mái nhé).
  - Chỉ dùng được khi join trên điều kiện bằng (dùng hàm hash mà).
  - Trước khi join, Postgres phải khởi tạo một hash table trên một trong hai table (Postgres sẽ chọn table nhỏ hơn). Trong trường hợp hash table quá lớn, không chứa nổi trên `work_mem`, hash table sẽ tràn ra disk. Lúc này Postgres sẽ không sử dụng hash join nữa vì hash table hoạt động trên disk rất kém hiệu quả do lượng random access nhiều. Vì lí do này, cách tốt nhất để tối ưu hash join là giảm kích thước của hash table xuống bằng cách thêm nhiều điệu kiện trong WHERE hoặc SELECT ít cột hơn.
```sql
EXPLAIN SELECT
	*
FROM
	sample_table outer_table
	JOIN sample_table inner_table ON outer_table.a = inner_table.b
WHERE
	outer_table.a > 90;
                                       QUERY PLAN
-----------------------------------------------------------------------------------------
 Hash Join  (cost=37.40..58.49 rows=795 width=82)
   Hash Cond: (outer_table.a = inner_table.b)
   ->  Bitmap Heap Scan on sample_table outer_table  (cost=4.90..15.92 rows=81 width=41)
         Recheck Cond: (a > 90)
         ->  Bitmap Index Scan on sample_table_a  (cost=0.00..4.88 rows=81 width=0)
               Index Cond: (a > 90)
   ->  Hash  (cost=20.00..20.00 rows=1000 width=41)
         ->  Seq Scan on sample_table inner_table  (cost=0.00..20.00 rows=1000 width=41)
```

#### Merge join
Với plan này thì Postgres trước hết sẽ sort hai bảng A và B theo điều kiện sort. Khi cả hai đã được sort rồi thì việc join rất đơn giản, chỉ cần dùng 2 pointer scan đồng thời cả 2 bảng. Time complexity sẽ là O(m\*log(m) + n\*log(n)).
**Điểm mạnh**:
  - Không bị phụ thuộc vào độ lớn `work_mem` như hash join, có thể hoạt động tốt kể cả khi dùng disk.
  - Bước sort có thể tận dụng được index và bỏ qua hoàn toàn bước sort vì với một số index (ví dụ BTREE), dữ liệu đã được sort trước rồi.

**Điểm yếu**:
  - Bước sort có thế ngốn nhiều thời gian.
  - Giống như hash join, chỉ có thể sử dụng khi join trên điều kiện bằng.
```sql
-- Phần này mình phải giảm work_mem xuống mức thấp nhất là 64kB để planner không thể dùng hash join nữa
EXPLAIN SELECT
	*
FROM
	sample_table outer_table
	JOIN sample_table inner_table ON outer_table.a = inner_table.b
                                               QUERY PLAN
----------------------------------------------------------------------------------------------------
 Merge Join  (cost=321885.37..64855038.71 rows=4301949209 width=82)
   Merge Cond: (inner_table.b = outer_table.a)
   ->  Sort  (cost=160435.39..162078.81 rows=657369 width=41)
         Sort Key: inner_table.b
         ->  Seq Scan on sample_table inner_table  (cost=0.00..16020.69 rows=657369 width=41)
   ->  Materialize  (cost=160435.39..163722.23 rows=657369 width=41)
         ->  Sort  (cost=160435.39..162078.81 rows=657369 width=41)
               Sort Key: outer_table.a
               ->  Seq Scan on sample_table outer_table  (cost=0.00..16020.69 rows=657369 width=41)

-- Giờ hãy cải thiện plan trên bằng cách tạo index trên hai cột a và b
CREATE INDEX sample_table_a ON sample_table (a)
CREATE INDEX sample_table_b ON sample_table (b)

-- Voila, bước sort mất tiêu luôn
                                                      QUERY PLAN
----------------------------------------------------------------------------------------------------------------------
 Merge Join  (cost=0.85..152687128.74 rows=10170434212 width=82)
   Merge Cond: (outer_table.a = inner_table.b)
   ->  Index Scan using sample_table_a on sample_table outer_table  (cost=0.42..64045.70 rows=1010756 width=41)
   ->  Materialize  (cost=0.42..66569.86 rows=1010756 width=41)
         ->  Index Scan using sample_table_b on sample_table inner_table  (cost=0.42..64042.97 rows=1010756 width=41)
```

### Tài liệu tham khảo
https://www.postgresql.org/docs/8.1/planner-stats-details.html
https://use-the-index-luke.com/sql/join/hash-join-partial-objects
https://use-the-index-luke.com/sql/explain-plan/postgresql/operations
https://malisper.me/category/query-execution/