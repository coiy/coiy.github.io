---
published: true
layout: single
title: "테이블 row count를 정확하게 구하기"
category: PostgreSQL
comments: false
---

PG & EDB에서 통계 정보가 가지고 있는 테이블에서 row count를 가져오는 것이 아니라 테이블의 exact row count를 알아내는 방법이다. 
먼저 내부에 select count(1) 구문이 포함돼 있는 count_rows라는 함수를 만든다. 

```sql
create or replace function 
count_rows(schema text, tablename text) returns integer
as
$body$
declare
  result integer;
  query varchar;
begin
  query := 'SELECT count(1) FROM ' || schema || '.' || tablename;
  execute query into result;
  return result;
end;
$body$
language plpgsql;

```
함수를 만든 다음 아래와 같이 select 쿼리문 속에 count_rows 함수가 호출되도록 하고 where 구문의 table_schema 조건 속에 검색하려는 스키마를 포함시키면 그 스키마가 소유한 테이블들의 exact row count를 구할 수 있다. 

```sql
select 
  table_schema,
  table_name, 
  count_rows(table_schema, table_name)
from information_schema.tables
where 
  table_schema in ('public', 'schema_what_you_want') 
  and table_type='BASE TABLE'
order by 3 desc ;
```

문제가 없다면 아래와 같은 결과를 볼 수 있다. 

```sql
edb=# select
edb-#   table_schema,
edb-#   table_name,
edb-#   count_rows(table_schema, table_name)
edb-# from information_schema.tables
edb-# where
edb-#   table_schema in ('public', 'coiy')
edb-#   and table_type='BASE TABLE'
edb-# order by 3 desc;
 table_schema | table_name | count_rows
--------------+------------+------------
 public       | jobhist    |         17
 public       | emp        |         14
 public       | dept       |          4
 public       | t1         |          0
 public       | t2         |          0
 coiy         | t1         |          0
(6 rows)

edb=#
edb=#
```