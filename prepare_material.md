Những ghi chép trong bài viết này được áp dụng với Postgres 11.8. Đây là định nghĩa relation và một số helper function được mình sử dụng trong các ví dụ:
```sql
CREATE TABLE sample_table (
	id varchar(255) PRIMARY KEY,
	a integer,
	b integer
);

CREATE EXTENSION pageinspect;

CREATE FUNCTION generate_sample_data (sample_size int, range int)
	RETURNS void
	LANGUAGE plpgsql
	AS $$
BEGIN
	INSERT INTO sample_table
	SELECT
		md5(random()::text),
		random() * range,
		random() * range
	FROM
		generate_series(1, sample_size) ON CONFLICT ON CONSTRAINT sample_table_pkey DO nothing;
END;
$$;

CREATE FUNCTION heap_page(relname text, pageno integer)
	RETURNS TABLE (ctid tid, state text, t_xmin text, t_xmax text, t_ctid tid)
	LANGUAGE SQL
	AS $$
	SELECT 
		(pageno, lp)::text::tid AS ctid,
		CASE lp_flags
			WHEN 0 THEN 'unused'
			WHEN 1 THEN 'normal'
			WHEN 2 THEN 'redirect to ' || lp_off
			WHEN 3 THEN 'dead'
		END AS state,
		t_xmin || 
		CASE 
			WHEN (t_infomask & 256) > 0 THEN ' (committed)'
			WHEN (t_infomask & 512) > 0 THEN ' (aborted/invalid)'
		ELSE
			''
		END AS t_xmin,
		t_xmax ||
		CASE 
			WHEN (t_infomask & 1024) > 0 THEN ' (committed)'
			WHEN (t_infomask & 2048) > 0 THEN ' (aborted/invalid)'
		ELSE
			''
		END AS t_xmax,
		t_ctid
	FROM
		heap_page_items(get_raw_page(relname, pageno))
	ORDER BY
		lp;
$$;

CREATE FUNCTION index_page(relname text, pageno integer)
	RETURNS TABLE (ctid tid, itemoffset smallint, data text, id text, a integer, b integer)
	LANGUAGE SQL
	AS $$
		SELECT ctid, itemoffset, data, id, a, b
		FROM bt_page_items(relname, pageno) index_page
		LEFT JOIN sample_table heap_page
		ON index_page.ctid = heap_page.ctid
		ORDER BY index_page.itemoffset
	$$;
```