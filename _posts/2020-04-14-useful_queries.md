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

## 실시간 temp file 사용량 조회하기 
깃헙에 올라와 있는 [이 글](https://gist.github.com/ng-pe/d80beb532b13d3936b8994c0f71059a0)을 참고하여 불필요한 부분만 수정했다. 

```
SELECT
        pg_stat_activity.pid AS pid,
        CASE WHEN LENGTH(pg_stat_activity.datname) > 16
            THEN SUBSTRING(pg_stat_activity.datname FROM 0 FOR 6)||'...'||SUBSTRING(pg_stat_activity.datname FROM '........$')
            ELSE pg_stat_activity.datname
            END
        AS database,
        pg_stat_activity.client_addr AS client,
        EXTRACT(epoch FROM (NOW() - pg_stat_activity.query_start)) AS duration,
        pg_stat_activity.wait_event IS NOT NULL AS wait,
        pg_stat_activity.usename AS user,
        pg_stat_activity.state AS state,
        pg_size_pretty(pg_temp_files.sum) as temp_file_size, pg_temp_files.count as temp_file_num,
        pg_stat_activity.query AS query
    FROM
        pg_stat_activity AS pg_stat_activity
INNER JOIN
(
SELECT unnest(regexp_matches(agg.tmpfile, 'pgsql_tmp([0-9]*)')) AS pid,
       SUM((pg_stat_file(agg.dir||'/'||agg.tmpfile)).size),
       count(*)
FROM
  (SELECT ls.oid,
          ls.spcname,
          ls.dir||'/'||ls.sub AS dir,
          pg_ls_dir(dir||'/'||ls.sub) AS tmpfile
   FROM
     (SELECT sr.oid,
             sr.spcname,
             'pg_tblspc/'||sr.oid||'/'||sr.spc_root AS dir,
             pg_ls_dir('pg_tblspc/'||sr.oid||'/'||sr.spc_root) AS sub
      FROM
        (SELECT spc.oid,
                spc.spcname,
                pg_ls_dir('pg_tblspc/'||spc.oid) AS spc_root,
                trim(TRAILING E'\n '
                     FROM pg_read_file('PG_VERSION')) AS v
         FROM
           (SELECT oid,
                   spcname
            FROM pg_tablespace
            WHERE spcname !~ '^pg_') AS spc) sr
      WHERE sr.spc_root ~ ('^PG_'||sr.v)
        UNION ALL
        SELECT 0,
               'pg_default',
               'base' AS dir,
               'pgsql_tmp' AS sub
        FROM pg_ls_dir('base') AS l WHERE l='pgsql_tmp' ) AS ls,
     (SELECT generate_series(1,2) AS i) AS gs
   WHERE ls.sub = 'pgsql_tmp') agg
GROUP BY 1
) as pg_temp_files on (pg_stat_activity.pid = pg_temp_files.pid::int)
WHERE
        pg_stat_activity.pid <> pg_backend_pid()
ORDER BY
        EXTRACT(epoch FROM (NOW() - pg_stat_activity.query_start)) DESC;
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
## 지금까지 생성된 WAL 파일의 총 용량
select pg_size_pretty(pg_current_wal_insert_lsn() - '0/00000000'::pg_lsn);

## vacuum clean-up을 막고 있는 세션 찾기 
```bash 
SELECT pid, age(backend_xmin), datname, usename, state, backend_xmin, substring(query, 1, 50) as shorten_query, age(now(), query_start) AS age
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL and pid <> pg_backend_pid()
ORDER BY age(backend_xmin) DESC;
```
