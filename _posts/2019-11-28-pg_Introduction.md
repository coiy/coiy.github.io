---
published: true
layout: single
title: "PostgreSQL 소개"
category: PostgreSQL
comments: false
---

이 문서는 데이터베이스 관리자와 시스템 개발자를 위해 PostgreSQL의 내부 구조를 설명한 것이다. PostgreSQL은 전세계에서 다양한 목적으로 널리 쓰이고 있는 오픈 소스 관계형 데이터베이스 시스템이다. PostgreSQL은 독특하면서 복잡한 기능들이 서로 공조하며 이뤄진 하위시스템들이 하나로 통합되어 있는 방대한 시스템이다. 그 방대함과 복잡함 때문에 PostgreSQL을 관리하고 통합하는 데 필수적인 내부 동작원리를 이해하기가 어려웠다. 이 문서의 주된 목적은 각 하위시스템들이 어떻게 동작하는지 설명하고 PostgreSQL의 전체적인 그림을 제공하는 데 있다.   

이 문서는 내가 2012년 일본에서 출간한 [PostgreSQL 바이블](https://www.amazon.co.jp/gp/product/4774153923/ref=dbs_a_def_rwt_hsch_vapi_taft_p1_i0)이라는 책의 2장을 토대로 썼으며 PostgreSQL 11 버전 이하를 다룬다.  

## 주요 내용 

- [1장: 데이터베이스 클러스터, 데이터베이스, 테이블](https://coiy.github.io/postgresql/pg_cluster/#) 
- 2장: 프로세스와 메모리 아키텍쳐 
- 3장: 쿼리 프로세싱
- 4장: Foreign Data Wrappers (FDW)
- 5장: 동시성 제어 (Concurrency Control)
- 6장: Vacuum 프로세싱 
- 7장: Heap Only Tuple (HOT) and Index-Only 스캔 
- 8장: 버퍼 매니저 
- 9장: WAL (Write Ahead Logging)
- 10장: 베이스 백업과 시점 지정 리커버리 (Base Backup and Point-In-Time Recovery)
- 11장: 스트리밍 리플리케이션 (Streaming Replication)

![context-hierarchy](/assets/pg_architect.png)  
> Resource from : [Resource from](http://www.interdb.jp/pg/index.html)  