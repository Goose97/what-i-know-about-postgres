### EXPLAIN dùng để làm gì?
Lệnh EXPLAIN sẽ cho ta thấy execution plan của một query bất kì. Cụ thể execution plan bao gồm việc bảng đích sẽ được scan thế nào (dùng sequential scan, index scan hay bitmap index scan), nếu có join thì thuật toán join nào sẽ được sử dụng và quan trọng nhất cost ước tính cho từng bước. Ví dụ đây là kết quả khi ta chạy EXPLAIN:

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM sample_table WHERE a > 800 AND b > 700;
                                                          QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on sample_table  (cost=9.17..13.19 rows=1 width=41) (actual time=0.140..0.149 rows=0 loops=1)
   Recheck Cond: ((b > 700) AND (a > 800))
   Buffers: shared read=2
   ->  BitmapAnd  (cost=9.17..9.17 rows=1 width=0) (actual time=0.119..0.128 rows=0 loops=1)
         Buffers: shared read=2
         ->  Bitmap Index Scan on sample_table_b  (cost=0.00..4.45 rows=22 width=0) (actual time=0.098..0.108 rows=0 loops=1)
               Index Cond: (b > 700)
               Buffers: shared read=2
         ->  Bitmap Index Scan on sample_table_a  (cost=0.00..4.47 rows=25 width=0) (never executed)
               Index Cond: (a > 800)
 Planning Time: 0.307 ms
 Execution Time: 0.329 ms
(12 rows)
```

### Cách đọc hiểu execution plan
Hãy để ý vào dấu `->`, đó là đánh dấu bắt đầu một execution node mới. Khi biểu diễn plan trên dưới dạng cây, trông nó sẽ như thế này:

```
                              Bitmap Heap Scan
                                      |
                                      |
                                  BitmapAnd
                                    /   \
                                   /     \
Bitmap Index Scan on sample_table_b       Bitmap Index Scan on sample_table_a
```

Thứ tự thực thi các node là **post-order traversal**, thực thi node con trước rồi mới đến node cha. Với từng node chúng ta sẽ có thêm những thông tin cụ thể như:
- cost: cost ước tính. Cost được biểu diễn dưới dạng hai số start_up_cost..total_cost (0.00..4.45). Start up cost là cost để node hiện tại tạo ra một row còn total cost là cost để tạo ra toàn bộ row. Lấy ví dụ với hash join node, start up cost sẽ lớn hơn một chút so với nested loop join vì ta phải khởi tạo được hash table trước. Trong hầu hết trường hợp, chúng ta chỉ quan tâm đến total cost
- rows: số row ước tính mà node này tạo ra. Nếu bạn chạy với option `ANALYZE` thì bạn sẽ thu được cả số row thực tế của node này khi thực thi (phần actual). Ví dụ node `Bitmap Index Scan on sample_table_b` có rows ước tính là 22 còn rows thực tế là 0
- width: [dung lượng trung bình](https://stackoverflow.com/questions/10964346/what-does-the-width-field-mean-in-postgresqls-explain) (theo byte) của các row mà node tạo ra
- time: thời gian thực thi thực tế của node này (chỉ xuất hiện với `ANALYZE` option)
- loops: số lần node này được thực thi. Tổng số row mà một node tạo ra sẽ bằng *rows \* loops*
- buffers: số lượng page của bảng được đọc ghi trên shared buffer. Có 4 loại page:
  - hit: số lượng page đã fetch những đã tồn tại trên shared buffer (hit cache)
  - read: số lượng page đã fetch những không tồn tại trên shared buffer (miss cache)
  - dirtied: số lượng page bị modified bởi query này
  - written: số lượng dirtied page bị write xuống disk
- Ngoài ra, mỗi một node sẽ có những thông tin cụ thể về execution plan của node ấy, ví nếu dùng index scan thì điều kiện scan là gì?

OK, giờ thử áp dụng những kiến thức vừa rồi vào plan phía trên nhé:
1. Bắt đầu từ các node lá trước. Ta thấy Postgres thực hiện bitmap index scan trên điều kiện a = ? (sử dụng index `sample_table_a`) và bitmap index scan trên điều kiện b = ? (sử dụng index `sample_table_b`). Bitmap index scan trên index `sample_table_b` theo estimate sẽ cho ra 22 rows nhưng thực tế số row là 0 (trong bước này Postgres phải read 2 page từ heap lên buffers read=2). Làm tương tự đối với index `sample_table_a`.
2. Sau đó hai bitmap sẽ được AND với nhau ở node `BitmapAnd` để tạo ra bitmap chứa các row cần phải fetch.
3. Cuối cùng node `Bitmap Heap Scan` sẽ sử dụng bitmap được tạo ra ở bên trên để fetch các row thoả mãn.
4. Note: Postgres thực hiện một bước tối ưu nhỏ ở đây. Kết quả scan trên index đầu tiên không cho ra row nào vì vậy Postgres không cần phải thực hiện bước scan trên index thứ hai (never executed)

Nhìn vào cost của từng bước, chúng ta sẽ nắm được bước nào chiếm tỉ trọng lớn trong tổng thời gian thực thi của cả plan, từ đó có thể cân nhắc các phương án tối ưu cần thiết.
*Lưu ý*: total cost của node cha bao gồm cả total cost của các node con. Vì vậy để so sánh cost giữa các node, mình thường sử dụng total cost - start up cost. Ví dụ trong plan trên, node `BitmapAnd` có thời gian thực thi ngắn nhất (9.17 - 9.17 = 0).

### Tài liệu tham khảo
https://www.postgresql.org/docs/9.1/sql-explain.html
https://www.pgmustard.com/docs/explain
https://www.cybertec-postgresql.com/en/how-to-interpret-postgresql-explain-analyze-output/