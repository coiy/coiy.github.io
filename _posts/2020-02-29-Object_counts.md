---
published: true
layout: single
title: "오브젝트 카운트 구하기"
category: PostgreSQL
comments: false
---

as-is 오라클에서 to-be EDB로 마이그레이션을 진행하면서 작업 전후의 오브젝트 카운트가 일치하는지 확인하는 과정이 필요했다. EDB에서는 오라클과의 호환성 dba_objects라는 뷰가 제공되기 때문에 아래와 같은 쿼리로 조사하면 된다. 

```sql
select schema_name, object_type, count(*)
from dba_objects
where schema_name in ('PUBLIC')
group by schema_name, object_type
order by schema_name, object_type;
```

PG에서는 아래와 같은 쿼리로 확인할 수 있다. n.nspname 조건문에 원하는 스키마만 포함시키면 특정 스키마가 소유한 오브젝트 카운트를 뽑아낼 수 있다.  

```sql
SELECT
	n.nspname as schema_name
	,CASE c.relkind
	   WHEN 'r' THEN 'table'
	   WHEN 'v' THEN 'view'
	   WHEN 'i' THEN 'index'
	   WHEN 'S' THEN 'sequence'
	   WHEN 's' THEN 'special'
	   WHEN 'm' THEN 'materialized view'
	   WHEN 'p' THEN 'partitioned table'
	   WHEN 'f' THEN 'foreign table'
	   WHEN 'c' THEN 'composite type'
	END as object_type
	,count(1) as object_count
FROM pg_catalog.pg_class c
LEFT JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind IN ('r','v','i','S','s','m','p','f','c') and n.nspname  IN ('public')
GROUP BY  n.nspname,
	CASE c.relkind
	   WHEN 'r' THEN 'table'
	   WHEN 'v' THEN 'view'
	   WHEN 'i' THEN 'index'
	   WHEN 'S' THEN 'sequence'
	   WHEN 's' THEN 'special'
	   WHEN 'm' THEN 'materialized view'
	   WHEN 'p' THEN 'partitioned table'
	   WHEN 'f' THEN 'foreign table'
	   WHEN 'c' THEN 'composite type'
	END
ORDER BY n.nspname,
	CASE c.relkind
	   WHEN 'r' THEN 'table'
	   WHEN 'v' THEN 'view'
	   WHEN 'i' THEN 'index'
	   WHEN 'S' THEN 'sequence'
	   WHEN 's' THEN 'special'
	   WHEN 'm' THEN 'materialized view'
	   WHEN 'p' THEN 'partitioned table'
	   WHEN 'f' THEN 'foreign table'
	   WHEN 'c' THEN 'composite type'
	END;
```



