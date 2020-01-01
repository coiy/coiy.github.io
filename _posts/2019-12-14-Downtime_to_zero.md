---
published: true
layout: single
title: "다운타임을 최소화하는 메이저 버전 업그레이드"
category: PostgreSQL
comments: false
---

메이저 버전을 업그레이드 하는 방법은 크게 3가지 방법이 있고 PG 개발그룹(PGDG)에서도 이 [3가지 방법](https://www.postgresql.org/docs/10/upgrading.html)을 권고하고 있다. 

- pg_dumpall
- pg_upgrade 👈 오늘 중점적으로 다룰 방법 
- via Replication


# pg_dumpall (또는 pg_dump) 
특징 및 주의점 
- Logical 백업 방식 
- Plain Text SQL문으로 백업
- 클러스터 레벨로 백업
- 데이터베이스 레벨로 백업을 하고자 할 때에는 pg_dump 사용  
- 데이터베이스 레벨로 스키마 변경 없이 단순히 펌프만 받을 때는 pg_dump -Fc 옵션을 사용하면 덤프 파일 용량이 최소 4배~7배 줄어듬 
- 100GB 이하 급의 작은 클러스터에 적합   
- 상위 버전의 pg_dumpall/pg_dump를 이용하고 pg_dump의 full path 명시하여 실행


단점 
- 데이터베이스가 크면 클수록 다운타임도 더 필요하고 그 시간을 예측하기가 어려움    
- 따라서 수백 GB급 또는 TB 이상의 사이즈인 클러스터를 대상으로는 최선의 옵션이라고 보기 어려움
- as-is, to-be 이중으로 스토리지 공간이 필요함   




# pg_upgrade

1. 업그레이드 하고자 하는 버전의 EDB 설치 
2. as-is에서 사용 중인 extension 미리 설치 
3. initdb로 새 버전의 EDB 클러스터 생성 
4. -c 옵션으로 as-is와 to-be 간의 호환성 체크  

## ✔️ 3번 단계까지 진행한 후 본격적인 업그레이드 작업을 시작하기 전에 -c 옵션으로 호환성 체크를 해보는 것이 중요
```sql 
[enterprisedb@EDB_11_6 data]$ /usr/edb/as12/bin/pg_upgrade -c
Performing Consistency Checks on Old Live Server
------------------------------------------------
Checking cluster versions                                   ok
Checking database user is the install user                  ok
Checking database connection settings                       ok
Checking for prepared transactions                          ok
Checking for reg* data types in user tables                 ok
Checking for contrib/isn with bigint-passing mismatch       ok
Checking for tables WITH OIDS                               ok
Checking for invalid "sql_identifier" user columns          ok
Checking for presence of required libraries                 ok
Checking database user is the install user                  ok
Checking for prepared transactions                          ok
```

장점 
- 링크 모드를 이용하면 다운 타임을 최소화하면서 엔진 업그레이드가 가능 

단점 
- 소스(source)와 타겟(target) 시스템이 같은 호스트여야 한다 