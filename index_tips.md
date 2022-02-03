### Kiểm tra mức độ sử dụng của index
Postgres duy trì statistics về index ở trong view `pg_stat_user_indexes`, trong đó `idx_scan` là số lần index được sử dụng trong một câu query.

### Index một phần (partial index)
Trong nhiều trường hợp, bạn có thể giảm thiểu dung lượng của index bằng cách tạo index trên một phần các row của bảng.
```sql
-- Nếu chỉ quan tâm đến sản phẩm nào đã có giá, chúng ta có thể tạo index 1 phần như sau
CREATE INDEX products_price ON products (price) WHERE price IS NOT NULL;
```

### Tạm thời disable index
Mình đã gặp trường hợp muốn drop 1 index nhưng không dám chắc có query nào sử dụng index này không. Thay vì drop index luôn thì bạn có thể [set 1 flag đặc biệt](https://stackoverflow.com/questions/6146024/is-it-possible-to-temporarily-disable-an-index-in-postgres) để planner bỏ qua index này.
```sql
UPDATE pg_class SET indisvalid = FALSE WHERE indexrelid = 'index_name'::regclass;
```
Sau một thời gian không thấy ảnh hưởng gì thì có thể drop thôi. Flag này được Postgres sử dụng khi tạo  index concurrently.