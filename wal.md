### WAL là gì?
WAL (Write-Ahead Log) là một cơ chế để đảm bảo tính toàn vẹn của dữ liệu (data-integrity) trong Postgres đồng thời tăng hiệu năng. Những thay đổi cần được persist xuống disk (thay đổi bảng hoặc index) phải được log lại trước. Đây là 1 concept rất hay, thay vì trực tiếp modify row X của table Y, chúng ta chỉ cần ghi lại những thay đổi cần thực hiện vào 1 file log rồi persist xuống disk. Bằng cách này, Postgres có thể tạm gác lại các heavy-operation (write vào table heap) rồi vào khoảng thời gian thích hợp thực hiện batch nhiều operation.

### WAL giải quyết bài toán gì?
#### Đảm bảo data-integrity
Khi chúng ta thực hiện thay đổi trên một tuple, những thay đổi này sẽ được thực hiện ngay trên shared buffer. Những page có thay đổi chưa sync xuống disk được gọi là dirtied page, trái ngược với clean page. Trong quá trình sử dụng, thỉnh thoảng những page này sẽ bị evicted khỏi buffer và write những thay đổi ấy xuống disk. Tuy nhiên, luôn luôn có một lượng dirtied page nhất định tồn tại trên buffer và khi Postgres crash, những thay đổi trên các page ấy sẽ biến mất.
WAL giải quyết bài toán này bằng cách lưu lại các thay đổi ấy vào permanent storage. Khi Postgres crash sau khi restart, Postgres sẽ chuyển sang recovery mode. Lúc này, Postgres sẽ thực thi tất cả các thay đổi trong WAL chưa được flush xuống disk.
*Lưu ý*: thính thoảng bạn sẽ thấy Postgres restart rất lâu chính là do có nhiều thay đổi cần flush xuống từ WAL.
#### Tăng performance
Thay vì **eagerly** thực hiện các update, Postgres đơn giản lưu lại những gì sẽ được làm và **lazily** xử lý chúng. Việc này làm giảm IO đồng thời tăng hiệu năng vì update được thực hiện theo batch.

### Cụ thể WAL hoạt động thế nào?
Một số khái niệm cần làm rõ trong phần này:
1. Wal segment: WAL sẽ được lưu vào các file có độ lớn cố định (mặc định là 16MB). Khi file này đầy, Postgres sẽ đổi sang WAL file mới (đoạn này sẽ block các process đang cập nhật vào WAL 1 chút). Bạn có thể chạy `pg_switch_wal` để kích hoạt switch bằng tay.
2. Wal buffers: mỗi thay đổi liên quan đến heap (insert/update/delete row hoặc index) đều phải ghi xuống WAL trước. Để tăng hiệu năng thì Postgres sẽ write xuống một buffer trên shared memory area trước, gọi là wal buffers. Buffer này sau mỗi transaction commit sẽ được flush xuống wal segment. Bạn có thể cấu hình dung lượng của buffer với parameter [wal_buffers](https://postgresqlco.nf/doc/en/param/wal_buffers/). Tuy nhiên mình thấy chỉnh lên cao cũng không có tác dụng mầy vì wal buffers được flush rất thường xuyên.

Như mình đề cập ở bên trên, trong tình huống Postgres được restart, Postgres sẽ *replay* tất cả các thay đổi ở trong WAL. Vậy nếu Postgres không bao giờ crash, số lượng file WAL sẽ tích tụ ngày càng nhiều và khiến cho lần tiếp theo restart có thể kéo dài hàng chục phút. Để giải quyết vấn đề này, Postgres có 1 background process gọi là checkpointer. Nhiệm vụ của checkpointer là flush những thay đổi trên WAL xuống heap sau đó đánh dấu thời điểm checkpoint. Bạn có thể hiểu WAL là một chuỗi tuần tự các thay đổi. Checkpoint là một điểm trong chuỗi đó sao cho tất cả các thay đổi trước thời điểm checkpoint được đảm bảo chắc chắn đã tồn tại trên heap (nghĩa là Postgres không cần quan tâm đến thời điểm trước checkpoint). Định kì, Postgres sẽ trigger checkpointer, flush từ WAL xuống heap rồi đánh dấu lại điểm checkpoint. Lần tiếp theo restart, Postgres chỉ cần replay lại từ điểm checkpoint thôi. Postgres sẽ trigger checkpointer khi một trong 2 điều kiện thoả mãn:
  - Số lượng wal segment vượt quá checkpoint_segments (mặc định là 3)
  - Thời gian từ lần cuối checkpoint vượt quá checkpoint_timeout (mặc định là 5 phút)
  
Việc cài đặt cho checkpointer hoạt động hiệu quả phụ thuộc nhiều vào tần suất insert/update/delete của ứng dụng của bạn. Để cho checkpointer chạy quá dày hay quá thưa đều có những ưu nhược điểm nhất định.

### Một số lock liên quan đến WAL
Thỉnh thoảng bạn sẽ thấy một vài backend process trong pg_stat_activity bị lock khi thao tác với WAL. Tuy nhiên các lock này đều là lightweight lock nên bạn không cần quá lo lắng
  - WALInsertLock: vì có nhiều backend process cùng thao tác với wal buffers nên các process sẽ block nhau khi insert vào WAL.
  - WALWriteLock: các backend process khi write vào WAL đều phải đợi wal buffers flush thành công xuống wal segment. Lúc này bạn sẽ thấy wait_event là WALWriteLock. WALWriteLock còn có thể xuất hiện trong trường hợp Postgres đổi wal segment.

### Một số tuning liên quan đến WAL
Một số config mình đã đề cập ở phần trên như `wal_buffers`, `checkpoint_segments` và `checkpoint_timeout` mình sẽ không nhắc lại nữa.
  - wal_level: quyết định xem bao nhiêu thông tin sẽ được ghi xuống WAL (càng verbose thì càng nặng những sẽ hỗ trợ nhiều feature hơn)
  - commit_delay: khi một process chuẩn bị flush wal buffers, nó sẽ đợi thêm một khoảng thời gian bằng với commit_delay. Trong khoảng thời gian đợi nếu có nhiều process nữa cùng commit thì những commit này sẽ được gộp lại với nhau và cùng flush xuống wal segment.
  - commit_siblings: nhược điểm của commit_delay đó là trong trường hợp process đợi nhưng không có process nào khác commit tiếp, làm lãng phí thời gian đợi. Để giảm khả năng trường hợp xảy ra, Postgres chỉ thực hiện delay nếu có ít nhất commit_siblings transaction đang active, mặc định là 5.
  - synchronous_commit: nếu bạn set synchronous_commit=off, Postgres sẽ không đợi wal buffers persist thành công xuống permanent storage. Vì vậy, có trường hợp các update của transaction sẽ bị mất:
    1. Transaction thực hiện update
    2. Transaction commit
    3. Process thực hiện flush wal buffers nhưng không đợi (async đó) mà thông báo transaction commit thành công luôn
    4. Application nhận được tín hiệu thành công và tiếp tục chạy
    5. Postgres crash, những thay đổi của transaction trên không kịp ghi xuống WAL
    6. Khi Postgres khởi động lại, những thay đổi này sẽ coi như mất vì không hề tồn tại trên WAL

  synchronous_commit đưa ra cho bạn một trade-off, nó tăng đáng kể write performance với cái giả phải trả là tỉ lệ mất dữ liệu nhỏ. Đối với những bảng dữ liệu không quá quan trọng (ví dụ như thống kê hay bảng log) hoặc có thể khởi tạo lại được, synchronous_commit là một cấu hình bạn nên cân nhắc.
  
Ngoài ra, khi tạo bảng trong Postgres, bạn có thể chỉ định bỏ qua WAL khi thao tác với bảng này:
```sql
CREATE UNLOGGED TABLE name (

);
```

### Tài liệu tham khảo
https://www.postgresql.org/docs/11/wal-intro.html
https://www.postgresql.org/docs/11/wal-configuration.html
https://www.postgresql.org/docs/11/runtime-config-wal.html
https://www.postgresql.fastware.com/blog/back-to-basics-with-postgresql-memory-components