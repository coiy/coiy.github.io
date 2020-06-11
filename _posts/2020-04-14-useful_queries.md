---
published: true
layout: single
title: "워크로드 조사를 위한 쿼리와 커맨드"
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

## pg_stat_user_tables
Mastering PostgreSQL 11에서 Hans-Jürgen Schönig가 인덱스가 필요한 테이블을 알아보려 할 때 pg_stat_user_tables 카탈로그 테이블을 대상으로 실행한다는 쿼리다. 핵심 아이디어는 시퀀셜 스캔이 빈번하게 발생하는 대용량 테이블을 조회해보는 것이다. seq_tup_read 값이 큰 테이블들이 쿼리 실행 결과의 상위를 차지한다.  

```sql 
SELECT schemaname, relname, seq_scan, seq_tup_read, 
seq_tup_read / seq_scan AS avg, idx_scan 
FROM pg_stat_user_tables 
WHERE seq_scan > 0 ORDER BY seq_tup_read DESC LIMIT 25;

```

## pg_statio_user_tables
인덱스가 필요한 상황인지 아닌지를 pg_stat_user_tables로 알아보고나서 캐시 사용 현황(caching behavior)을 파악해보려고 할 때는 pg_statio_user_tables 카탈로그가 가지고 있는 정보를 활용한다(heap_blks_, idx_blks_ 접두사가 붙은 컬럼들과 TOAST 테이블). Hans-Jürgen Schönig는 pg_statio_user_tables도 좋은 정보를 많이 가지고 있지만 문제 해결에 실마리가 되어주는 것은 pg_stat_user_tables의 정보일 때가 더 많다고 한다. 


## pg_stat_user_indexes
```sql
SELECT schemaname, relname, indexrelname, idx_scan, 
pg_size_pretty(pg_relation_size(indexrelid)) AS idx_size, 
pg_size_pretty(sum(pg_relation_size(indexrelid)) OVER (ORDER BY idx_scan, indexrelid)) AS total 
FROM pg_stat_user_indexes ORDER BY 6 ;
```

## wraparound 측정하기  
```sql
WITH max_age AS ( 
    SELECT 2000000000 as max_old_xid
        , setting AS autovacuum_freeze_max_age 
        FROM pg_catalog.pg_settings 
        WHERE name = 'autovacuum_freeze_max_age' )
, per_database_stats AS ( 
    SELECT datname
        , m.max_old_xid::int
        , m.autovacuum_freeze_max_age::int
        , age(d.datfrozenxid) AS oldest_current_xid 
    FROM pg_catalog.pg_database d 
    JOIN max_age m ON (true) 
    WHERE d.datallowconn ) 
SELECT max(oldest_current_xid) AS oldest_current_xid
    , max(ROUND(100*(oldest_current_xid/max_old_xid::float))) AS percent_towards_wraparound
    , max(ROUND(100*(oldest_current_xid/autovacuum_freeze_max_age::float))) AS percent_towards_emergency_autovac 
FROM per_database_stats
```

## 리스닝 포트 확인 
EDB의 기본 포트인 5444에서 리스닝 대기중 프로세스를 출력한다. 
```bash 
[enterprisedb@EFM_EDB_11_master ~]$ netstat -lnp | grep :5444
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 0.0.0.0:5444            0.0.0.0:*               LISTEN      5682/edb-postgres
tcp6       0      0 :::5444                 :::*                    LISTEN      5682/edb-postgres

```

