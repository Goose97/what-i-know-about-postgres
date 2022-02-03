Phần này sẽ đi xuống low-level của Postgres một chút, tuy nhiên rất cần thiết nếu bạn không muốn bị Postgres ma giáo.

### Cấu trúc của bảng
Các bảng và index trong Postgres được tổ chức dưới dạng tổ hợp của các page (một số tài liệu gọi là block) với kích thước cố định (mặc định là 8kB). Trên mỗi page sẽ lưu trữ metadata của page và nhiều page item. Với table page thì item là row còn với index page thì item là index entry. Lưu ý là một row chỉ có thể tồn tại trong một page, nghĩa là đối với những row lớn (> 2kB), Postgres phải có cách xử lý đặc biệt, gọi là [TOAST](https://www.postgresql.org/docs/9.5/storage-toast.html).

Mỗi page được định dạnh bằng một page number. Mỗi item trong page được định danh bằng offset. Vì vậy một row trong bảng sẽ được định danh bằng một cặp số, gọi là ctid. Đây chính là "địa chỉ nhà" của một row.
*Lưu ý*: khi Postgres cần fetch một row lên thì Postgres không chỉ fetch một mình row ấy mà sẽ fetch cả page.

```sql
-- Kiểm tra ctid bằng pseudo field
SELECT ctid, * FROM sample_table;
  ctid  |                id                | a  | b
--------+----------------------------------+----+----
 (0,1)  | fa8778f150ff3ea6fd9a0480f88a3fcb | 26 | 16
 (0,2)  | e65cbc18e09acffa621a6de6bae84cfb | 19 | 48
 (0,3)  | 45930b9b7c4317eff5d936d6178af184 | 32 | 50
 (0,4)  | 0774200e02f98a329e0bff4a470aad7b | 68 | 68
 (0,5)  | 45c775aeb5145d4eccdade8d1d2ff652 | 79 | 21
 (0,6)  | a34c1d10382af98e4eca9e7d4d0da120 | 79 | 78
 (0,7)  | 6a9df3ebd53684ad92c7cd1e3e17a496 | 66 | 56
 (0,8)  | 23977933100407a4530df6582d8f9927 | 22 | 37
 (0,9)  | 870e3141c97dc8bd386b996119b89db9 | 47 | 40
 (0,10) | 811f07e079c7c9d05bcedcda90fde1c6 | 91 |  0
 (0,11) | 0e34e8a0e0ddc1952ce69f924c7106dc | 67 | 27
 (0,12) | fa7abf5b6a7f684e8b36f7c51186067b |  1 | 46
 (0,13) | 3b110199f9bd6857139b11e04ba5f448 | 36 | 78
 (0,14) | ad4204317f714428ad63bec972ee895b | 54 | 46
 (0,15) | 712aff20481fc43d91c7d1774a0bd7d4 | 61 | 26
 (0,16) | 2b3249002c6a6d85025cabd3df738fa6 | 66 |  5
 (0,17) | e89a695bc04d469d11ed695c971323c4 | 15 | 70
 (0,18) | 6f2ad737a258c139cc2bb04afbd1c742 | 72 | 92
 (0,19) | d2099278f5868e6ec208c9f5b8e8fecb | 95 | 39
 (0,20) | bb4be20f1ddcba794f0196928aaba026 | 24 | 29
(20 rows)

-- Ước lượng số page và số tuple của một bảng
SELECT count(*) FROM sample_table;
 count
-------
 10020
(1 row)

SELECT relpages, reltuples FROM pg_class WHERE relname = 'sample_table';
 relpages | reltuples
----------+-----------
       94 |     10020
(1 row)
```

### Page được cache thế nào
Postgres sử dụng **double buffering**, nghĩa là ngoài IO buffer của kernel thì Postgres tự duy trì một internal buffer nữa. Khi Postgres cần fetch một page từ disk lên, Postgres sẽ kiểm tra trong buffer trước rồi mới issue read request xuống system. Sau khi fetch được page lên thì Postgres có thể sẽ cache lại trên buffer. Phần buffer này được gọi là shared buffers (có thể sửa trong config của Postgres qua config shared_buffers). Khi một page cần chỉnh sửa, Postgres cũng sẽ thực hiện chỉnh sửa ngay trên shared buffer luôn. Những page có thay đổi nhưng chưa được flush xuống disk gọi là dirtied page. Khi Postgres cần cache thêm page lên buffer nhưng không còn đủ chỗ, Postgres sẽ chọn ra một page để nhả ra khỏi buffer (page này gọi là victim page). Nếu page là dirtied page thì Postgres sẽ phải flush những thay đổi xuống disk nữa.
Đừng ngây thơ giống mình nghĩ rằng cứ tăng shared_buffers lên càng cao càng tốt nhé. Sẽ có một điểm rơi tối ưu dành cho shared_buffers, quá mức này thì tăng lên chỉ làm giảm hiệu năng thôi. Càng nhiều page bị double buffer hơn (lãng phí RAM) và khi write vào một buffer size lớn hơn cũng kém hiệu quả hơn. Có một [discussion](https://stackoverflow.com/questions/67016945/what-is-the-downside-to-increase-shared-buffer-in-postgresql) này khá hay về vấn đề này. Bạn nào muốn đọc sâu hơn về buffer manager của Postgres thì có thể tham khảo [tài liệu này](https://www.interdb.jp/pg/pgsql08.html).

### Tài liệu tham khảo
https://www.postgresql.org/docs/9.5/storage-page-layout.html
https://www.interdb.jp/pg/pgsql01.html
https://www.interdb.jp/pg/pgsql08.html
https://postgrespro.com/blog/pgsql/5967951
https://habr.com/en/company/postgrespro/blog/469087/