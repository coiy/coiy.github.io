---
published: true
layout: single
title: "설정값을 확인하고 적용하는 방법"
category: PostgreSQL
comments: false
---

PostgreSQL를 전문적으로 다루는 것을 본업으로 하기 시작하면서 데이터 디렉토리 내에 존재하는 conf 파일의 역할과 그 파일에 기술된 여러 설정 값들의 의미를 깊이 이해하는 것이 얼마나 중요한지 깨닫고 있다. 어떤 설정값들은 그 값을 파악하는 것만으로도 현재 시스템의 status을 가늠하는 데 핵심적인 역할을 하기도 하고 문제 상황의 실마리를 찾는 데 도움을 주기도 한다.     

정기점검 계약을 맺은 곳 또는 정기점검 계약을 맺은 곳은 아니지만 뭔가 트러블이 발생한 업체로부터 컨설팅 의뢰를 받으면 최우선으로 확인하는 것 또한 해당 시기의 로그와 함께 현재 PostgreSQL DB의 설정값이다.   

설정값을 확인하려고 할 때 psql 클라이언트로 접속하여 show parameter 커맨드로 조회하거나 postgresql.conf 또는 postgresql.auto.conf 설정 파일을 텍스트에디터로 열어서 찾으려는 설정값을 검색하는 방법을 많이들 쓰는 것 같다. 

다른 방법으로는 pg_settings라는 카탈로그 뷰를 활용하는 것도 있는데 나는 이 방법으로 설정값을 확인하는 편이다. 현재의 설정값은 어떤 설정 파일에 명시된 값이 적용된 것인지 확인할 수 있다. 

```sql 
postgres=# select name, setting, source, sourcefile, sourceline from pg_settings where name = 'shared_buffers';

      name      | setting |       source       |            sourcefile             | sourceline
----------------+---------+--------------------+-----------------------------------+------------
 shared_buffers | 65536   | configuration file | /u01/pg11/data/conf.d/custom.conf |         19
(1 row)

```

설정값 변경의 효과를 다양한 레벨에서 테스트 해볼 수 있다. 나는 세션 레벨에서만 적용하려고 할 때 사용하는 SET parameter=value 구문으로 설정값 변경의 효과를 테스트 해본다. 각 레벨별 설정값 변경 구문은 아래 그림을 참고하자. 


![timeline](/assets/pg_setting_value.png)
> Image from [Where do my Postgres settings come from?](https://mydbanotebook.org/post/postgres-settings/) 





