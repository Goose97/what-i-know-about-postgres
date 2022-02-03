Khi bạn start Postgres lên, bạn sẽ thấy một vài process được khởi tạo bởi postgres user. Một số process mà bạn nên chú ý:
1. Server process: hay còn gọi là [postmaster](https://www.postgresql.org/docs/8.1/app-postmaster.html), là cha của tất cả các process còn lại.
2. Backend process: với mỗi connection từ client thì postmaster sẽ tạo ra một backend process để xử lý. Để kiểm tra thông tin về các backend process thì bạn có thể sử dụng view `pg_stat_activity`.
3. Background process: ngoài việc xử lý query từ client thì Postgres có rất nhiều tác vụ chạy ngầm, ví dụ như vacuum, checkpoint, statistic collector, WAL write, ... Nội dung cụ thể về những process này sẽ được bàn cụ thể ở chương sau.

Về phân vùng memory, chúng ta có 2 vùng memory chính là:
1. Local memory area: vùng memory này được cấp phát riêng cho từng backend process. Ví dụ [work_mem](https://postgresqlco.nf/doc/en/param/work_mem/) là một thành phần của local memory area.
2. Shared memory area: vùng nhớ này được postmaster allocate ngay lúc khởi động và như chữ *shared* gợi ý, vùng nhớ này có thể được access bởi tất cả backend process và background process. Ví dụ [shared_buffers](https://postgresqlco.nf/doc/en/param/shared_buffers/) và [wal_buffers](https://postgresqlco.nf/doc/en/param/wal_buffers/) là hai thành phần của shared memory area.

### Tài liệu tham khảo
https://www.interdb.jp/pg/pgsql02.html