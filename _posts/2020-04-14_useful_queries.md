---
published: true
layout: single
title: "워크로드 조사를 위한 쿼리"
category: PostgreSQL
comments: false
---

CYBERTEC CEO인 Hans-Jürgen Schönig의 [PostgresConf South Africa 2019 강연](https://www.youtube.com/watch?v=5M2FFbVeLSs)에서 pg_stat_statements에서 long running 쿼리를 조회하는 방법을 설명하며 예시로 든 쿼리다.   

```sql
select 
substring(query, 1, 50) as shorten_query, 
round(total_time::numeric, 2) as total_time, 
calls, round(mean_time::numeric, 2) as mean, 
round((100 * total_time / sum(total_time::numeric) over ())::numeric, 2) as percentage_overall 
from pg_stat_statements 
order by total_time desc 
limit 20; 

```


```sql
select schemaname, relname, seq_scan, seq_tup_read, 
idx_scan, seq_tup_read / seq_scan as avg 
from pg_stat_user_tables 
where seq_scan > 0
order by seq_tup_read desc ;

```
