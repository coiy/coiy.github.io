---
published: true
layout: single
title: "raw counts of tables"
category: PostgreSQL
comments: false
---

통계 정보가 가지고 있는 테이블 카운트 정보가 아니라 테이블의 true count를 알아내는 방법이다. 

먼저 count_rows라는 함수를 만들어둔다. 

```
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
그 뒤에 아래의 select 쿼리를 실행하면 pg_catalog, information_schema, sys 이외의 스키마에 생성돼 있는 테이블들의 true count를 구할 수 있다. 

```
select 
  table_schema,
  table_name, 
  count_rows(table_schema, table_name)
from information_schema.tables
where 
  table_schema not in ('pg_catalog', 'information_schema', 'sys') 
  and table_type='BASE TABLE'
order by 3 desc ;
```