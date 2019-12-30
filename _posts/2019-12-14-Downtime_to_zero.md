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




# Replication