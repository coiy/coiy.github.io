---
published: true
layout: single
title: "EDB version 11 새로운 기능"
category: PostgreSQL
comments: false
---

- EDB사의 제공하는 Version 11 공식 [Release Notes](https://get.enterprisedb.com/docs/EPAS_Release_Notes_v11.pdf?_ga=2.140986673.2062198911.1576200190-181288000.1576200190) 
- 일본 HP Enterprise에서 공개한 [PostgreSQL 11 New Features with Example](https://h50146.www5.hpe.com/products/software/oe/linux/mainstream/support/lcc/pdf/PostgreSQL_11_New_Features_beta1_en_20180525-1.pdf)
- 2ndQuadrant사의 PostgreSQL 11의 [새로운 기능 소개](https://www.2ndquadrant.com/en/blog/tag/postgresql-11-new-features/) 
- AWS RDS의 [PostgreSQL 매뉴얼](https://docs.aws.amazon.com/ko_kr/AmazonRDS/latest/UserGuide/CHAP_PostgreSQL.html) 
- MS Azure For PostgreSQL [문서](https://docs.microsoft.com/en-us/azure/postgresql/) 


# Overview 

PostgreSQL 11 버전에서는 160가지 이상의 새로운 기능 추가와 버그 픽스가 이루어졌다.  

## 분석 쿼리의 성능 향상 

## 아래 조건을 충족시 parallel query가 적용되는 범위 확장
* Hash Join 연산 
* Append 연산 
* on SELECT INTO 구문
* CTAS, CREATE INDEX, CREATE MV 구문 등에도 적용된다 

## 파티션 테이블의 기능 강화 
* 해시(HASH partition) 파티션 지원
* 파티션 테이블에 PRIMARY KEY, UNIQUE KEY 생성 가능 
* 각 파티션에 자동으로 인덱스 생성 가능 
* 파티션 테이블에 대한 FOREIGN KEY 생성 가능 
* 어느 파티션에도 해당하지 않는 값에 대한 디폴트 파티션 지정 가능

## Ver 11에서 폐지된 기능들 (✔️ 이관시 챙겨야 할 사항)  
* CREATE FUNCTION 구문에서의 WITH 절 비호환 
* 시스템 카탈로그 뷰에서 추가되고 삭제된 컬럼들이 존재 


# 새로운 기능 자세히 보기 

## 1. 좀더 다양한 상황에서 병렬 쿼리(parallel query) 사용이 가능해졌다. 

* 해시 조인이 발생하는 상황
* UNION ALL 구문 같은 append 처리
* CTAS 구문
* MV(Materialized View) 생성 구문

| Parameter                | Description                                                  |
| --------------------------- | ------------------------------------------------------------ |
| max_parallel_workers_per_gather  | 하나의 쿼리당 수행 가능한 최대 worker 프로세스 개수를 설정하는 값. 기본 설정값은 0이 므로 Parallel 처리를 원한다면 0보다 큰 값으로 설정한다. 이 값은 세션 레벨에서도 변경할 수 있다. |
| dynamic_shared_memory_type              | 서버가 사용해야 하는 동적 공유 메모리 구현을 설정하는 값. enum 타입의 값으로 플랫폼에 따라 유닉스 계열 OS에는 posix, System V 공유 메모리 형식의 OS에는 sysv, 윈도우의 경우는 windows, 이 기능 비활성을 하려면 none으로 설정한다.  |
| 

```sql
explain select count(*) from t_random inner join s_random on t_random.s = s_random.s;
```

Version 11 
![Version 11](/assets/parallel_q_v11.png)

Version 10 
![Version 11](/assets/parallel_q_v10.png)


병렬 처리가 실행될 수 있는 상황과 조건에 대한 더 자세한 내용은 [PostgreSQL 공식 웹사이트의 관련 글](https://www.postgresql.org/docs/11/when-can-parallel-query-be-used.html) 참조. 


```sql
explain select count(*) from t_random UNION ALL select count(*) from s_random ; 
```
Version 11 
![Version 11](/assets/parallel_u_q_v11.png)

Version 10 
![Version 11](/assets/parallel_u_q_v10.png)



```sql
explain create table para_1 as select count(*) from t_random ; 
```
## 2. B-tree 타입의 인덱스에 한해서 병렬 쿼리를 통한 CREATE INDEX가 가능해졌다.  


numberic 타입의 랜덤 더미 데이터 100만 건을 작성한다. 

```sql 
CREATE TABLE t_demo (data numeric);
 
CREATE OR REPLACE PROCEDURE insert_data(buckets integer)
LANGUAGE plpgsql
AS $$
   DECLARE
      i int;
   BEGIN
      i := 0;
      WHILE i < buckets
      LOOP
         INSERT INTO t_demo SELECT random()
            FROM generate_series(1, 1000000);
         i := i + 1;
         RAISE NOTICE 'inserted % buckets', i;
         COMMIT;
      END LOOP;
      RETURN;
   END;
$$;
 
CALL insert_data(100);
```

| Parameter                | Description                                                  |
| --------------------------- | ------------------------------------------------------------ |
| max_parallel_maintenance_workers  | 11 버전부터 도입된 파라미터로 정수값을 사용하며 B-tree 타입의 인덱스를 빌드할 때 영향을 주고 max_worker_processes에서 정의된 풀 숫자에 제한을 받기 때문에 그 이상은 부여할 수 없다. 디폴트 값은 2.    |
|

병렬로 인덱스를 빌드할 때 영향을 주는 두 파라미터 값을 확인한다. 
```sql 
SHOW max_worker_processes;
SHOW max_parallel_maintenance_workers;
```
인덱스 작성. 
```sql 
CREATE INDEX idx1 ON t_demo (data);
```

max_parallel_maintenance_workers를 0으로 변경한 후 인덱스 작성. 
```sql
SET max_parallel_maintenance_workers TO 0;
CREATE INDEX idx2 ON t_demo (data);

```

병렬 쿼리를 통한 B-tree 타입의 [인덱스 생성에 관한 글](https://www.cybertec-postgresql.com/en/postgresql-parallel-create-index-for-better-performance/). 

## 3. 해시 파티션 지원 
PostgreSQL 11 버전부터 해시 파티션도 지원 시작. EDB에서는 10버전부터 오라클과의 호환성 차원에서 해시 파티션이 이미 가능함. 

