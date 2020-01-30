---
published: true
layout: single
title: "pg_rman을 통해 이해하는 PG의 백업과 복구"
category: PostgreSQL
comments: false
---


# Overview 

2009년 12월부터 개발이 시작되어 이 글을 쓰고 있는 시점(2019년 12월)에도 유지•보수가 이어지고 있는 백업 솔루션이다. DB업계에서 사실상의 표준(de facto standard) 지위를 누리고 있는 오라클DBMS의 RMAN과 유사한 인터페이스를 제공함으로써 오라클DBMS를 중심으로 경력을 이어왔고 이제 막 PG로 진입한 DBA가 백업과 복구에 관한 문제에 직면하였을 때 그 솔루션으로 대안이 될 수 있다.  

C언어로 작성되었기에 github 저장소에서 소스(raw source file)를 내려받아서 설치하려면 컴파일이 필요하다. 레드햇 패밀리 계열 리눅스 OS 플랫폼을 대상으로는 PG 버전별로 [패키지 파일을 제공](https://github.com/ossc-db/pg_rman/releases/)하기에 직접 컴파일하여 설치하는 것보다는 자신의 환경에 맞는 패키지 파일을 받아서 설치하는 편을 추천한다. 디폴트 설치 경로는 PG의 각종 유틸리티가 존재하는 /usr/pgsql-1*/bin이다. 따라서 pg_rman을 edb에 도입한다면 /usr/edb/as1*/bin로 pg_rman을 옮겨둘 필요가 있다.             

유사한 기능을 제공하는 백업&복구 솔루션으로 EDB에서 제공하는 것으로는 BART, 오픈소스 방식으로 2ndQuadrant에서 유지•보수하는 BARMAN이 있다. 근간이 되는 백업&복구의 기본개념은 닮아 있다. 

백업 솔루션의 구체적인 사용법을 익히기 전에 DML 트랜잭션이 발생하면 그 내용(커밋 내용, 인덱스 포인트 정보, 체크섬 등을 포함한 해당 오퍼레이션의 다양한 정보)이 처리되는 과정을 복습하고 checkpoint, timeline 같은 개념도 되새겨보자. 


# 새로운 데이터가 insert 되는 상황에서 일어나는 일들 

![storage_and_buffer](/assets/buffers.png)

* PG는 장애에 대비해 모든 변경내용의 이력 정보를 영속적인 스토리지 영역에 저장하는데 이런 변경내용의 이력 정보를 저장하고 있는 데이터를 WAL(a.k.a XLOG 레코드)이라고 한다. 
* 테이블에 DML 작업을 하는 워크로드일 경우, 변경이 일어날 테이블의 페이지(page)가 메모리 상에 존재하지 않으면 해당 테이블 파일을 찾아서 Shared Buffer 메모리 상에 적재(Load)한다. 
* DML 트랜잭션 로그는 WAL 버퍼라는 메모리 상에 존재하다가 commit 또는 abort되는 즉시 스토리지 상의 WAL 세그먼트 파일에 저장된다(Write and Flush). 
* commit 또는 abort로 트랜잭션이 종결되면 LSN(Log Sequence Number)이 증가한다.  
* DML 트랜잭션이 종료되면 변경내용은 일단 메모리 상의 page에 저장된다. 즉 변경내용이 해당 테이블 데이터가 속한 스토리지 내의 파일에 바로 반영되는 것은 아니다. 

# CHECKPOINT

![memory_and_disk](/assets/memory_disk.png)

## CHECKPOINT의 기본 동작 
* shared buffer에서 변경이 발생한 모든 페이지(dirty pages)을 식별한다(identify).
* 버퍼(메모리)에 저장돼 있는 변경 블록을 디스크에 쓴다(flush). 
* 버퍼에 저장된 데이터(메타데이터 + 데이터)가 물리적인 디스크에 쓰였음을 보장하는 fsync 시스템 콜 호출을 통해 버퍼 캐시의 내용과 디스크의 파일 시스템을 동기화한다. 

## checkpoint와 관련된 주요 파라미터 

* checkpoint_timeout(디폴트 5min, 설정 가능한 값의 범위는 30s - 1d) 
* max_wal_size(max 사이즈에 도달하면 wal 새그먼트 파일(16진수 24자릿수로 된 디폴트 16MB의 파일, v12부터 initdb 단계에서 --wal-segsize 값을 MB단위로 지정하여 변경이 가능해짐)을 rename 후 재사용해야 하기 때문에 checkpoint를 발생시켜서 메모리 상의 변경내용을 영속적인 스토리지에 쓴다) 
* checkpoint_completion_target  
* log_checkpoints(로그에 checkpoint 발생 관련 로그를 기록하도록 한다)

## CHECKPOINT가 발생하는 상황 
* 사용자가 직접 checkpoint 명령어를 수행하거나 설정된 주기 및 특정 조건을 만족했을 때 checkpoint가 발생한다. 

* pg_start_backup, pg_basebackup, CREATE DATABASE 구문 또는 pg_ctl stop|restart 수행할 시에도 checkpoint가 발생한다. 

* checkpointer와 background writer라는 이름의 두 가지 프로세스가 checkpoint와 관계된 백그라운드 프로세스다.  

# checkpoint_timeout의 텀은 어떻게 설정하는 것이 좋을까? 
Mastering PostgreSQL 11의 저자이자 Cybertec이라는 PG 벤더사의 CEO인 Hans-Jurgen Schonig 씨에 따르면 9.6 버전 이후부터는 기본적으로 checkpoint 발생 텀은 길게 가져가는 것이 [바람직하다고 주장](https://www.cybertec-postgresql.com/en/checkpoint-distance-and-amount-of-wal/)한다. I/O 퍼포먼스 관점에서 좋고 checkpoint 텀이 길수록 full-page writes가 덜 발생하고 그렇다는 것은 WAL 발생이 준다는 것을 의미하기 때문이다. 

shared_buffers용으로 메모리 100GB를 할당했고 쓰기 작업이 빈번한 시스템이 있다고 하자. 이런 시스템에서 checkpoint 발생 조건이 충족되었을 때 쓰기 작업으로 메모리 상에 쌓인 100GB분의 변경내역을 한번에 디스크로 쓰려는 상황도 있을 수 있다. 메모리 상의 dirty pages를 디스크로 flush하는 작업은 대량의 I/O 작업을 동반하기 때문에 커다란 부하를 발생시키고 일상적인 쿼리(normal query)에 응답하는 데에 나쁜 영향을 주게 된다. 이런 상황에서 조정을 검토해봄직한 파라미터가 checkpoint_completion_target 이다. 

이 파라미터는 checkpoint_completion_target와 checkpoint_timeout의 값을 곱한 시간 동안 좀더 천천히 데이터를 쓰도록 시도한다. checkpoint_timeout(5min)와 checkpoint_completion_target(0.5) 값이 디폴트 값 그대로 설정돼 있다면 그 둘을 곱한 값은 2.5분(2분 30초)이고 이 값이 의미하는 바는 PostgreSQL 엔진에게 2.5분에 걸쳐 checkpoint 작업을 진행하라는 의미이다.      

예로 든 시스템으로 돌아와서 디스크에 써야 할 메모리 상의 변경내역이 100GB이고 디스크의 최대 쓰기 속도가 초당 1GB라고 해보자. 최대 쓰기 속도로 메모리 상의 변경내역 100GB을 디스크에 쓴다면 100초(1분 40초)가 걸릴 것이다. checkpoint_timeout가 5min이고 checkpoint_completion_target 값이 0.5라고 한다면 2분 30초 동안 쓰기 작업을 완료하면 되기 때문에 1GB/1s 최대 속도가 아니어도 700MB/1s 정도의 쓰기 속도이면 2분 30초 안에 쓰기 작업을 마칠 수 있고 그렇게 아낀 30% 정도의 쓰기 자원은 다른 목적을 위해 쓸 수 있다.       

checkpoint_completion_target 설정에 따른 디스크 I/O 변화를 [테스트한 글](https://www.depesz.com/2010/11/03/checkpoint_completion_target/). 

pgbench 스케일 팩터 값으로 15를 주어서 진행했고 생성된 데이터의 크기는 약 300MB. 

* shared_buffers = 1024MB
* log_checkpoints = on 
* checkpoint_segments = 30 (deprecated by max_wal_value since 9.5)
* checkpoint_timeout = 5min     
* checkpoint_warning = 5min (체크포인트가 설정된 시간 이상 진행되면 경고 로그를 남긴다)
* log_line_prefix = '%m %r %u %d %p'
* log_min_duration_statement = 5 


## checkpoint_completion_target = 0으로 설정했을 때의 %util 변화 

![memory_and_disk1](/assets/checkpoint_completion_target_0.png)
- Pgbench 가동 시간대 14:57:34 ~ 15:09:34
- %util은 디스크 read/write시의 CPU사용률

## checkpoint_completion_target = 0으로 설정했을 때의 await 변화 

![memory_and_disk2](/assets/checkpoint_completion_target_0_1.png)
- await는 해당 Device에 발생된 I/O request의 평균 서비스 타임(단위 ms). queue 대기시간과 서비스 시간을 포함한다. 

## checkpoint_completion_target = 0.3으로 설정했을 때의 %util 변화  
![memory_and_disk3](/assets/checkpoint_completion_target_3_1.png)
- Pgbench 가동 시간대 14:57:34 ~ 15:09:34

## checkpoint_completion_target = 0.3으로 설정했을 때의 await 변화 
![memory_and_disk4](/assets/checkpoint_completion_target_3_2.png)
- Pgbench 가동 시간대 14:57:34 ~ 15:09:34

## checkpoint_completion_target = 0.6으로 설정했을 때의 %util 변화 
![memory_and_disk5](/assets/checkpoint_completion_target_6_1.png)

## checkpoint_completion_target = 0.6으로 설정했을 때의 await 변화 
![memory_and_disk6](/assets/checkpoint_completion_target_6_2.png)

## checkpoint_completion_target = 0.9으로 설정했을 때의 %util 변화 
![memory_and_disk7](/assets/checkpoint_completion_target_9_1.png)

## checkpoint_completion_target = 0.9으로 설정했을 때의 await 변화 
![memory_and_disk8](/assets/checkpoint_completion_target_9_2.png)

## checkpoint_completion_target = 1로 설정했을 때의 %util 변화 
![memory_and_disk9](/assets/checkpoint_completion_target_10_1.png)

## checkpoint_completion_target = 1로 설정했을 때의 await 변화 
![memory_and_disk10](/assets/checkpoint_completion_target_10_2.png)

## checkpoint_completion_target 설정 값에 따른 slow 쿼리 수의 변화 
![memory_and_disk11](/assets/cct_ms.png)

100ms - 200ms 범위에서는 꼭 그렇다고 할 수 없지만 checkpoint_completion_target(이하 cct) 값이 1에 가까워질수록 200ms 이상 걸리는 쿼리의 수가 줄어드는 것을 확인할 수 있다. 

cct 값을 크게 할수록 좋아 보이는데 단점(drawback)은 무엇일까? 
* wal 새그먼트를 저장하는 pg_wal(v10 이전에는 pg_xlog)가 비대해질 수 있다. 
* 항상 기대한대로 동작하는 것은 아니다. share_buffers의 크기가 매우 크고 쓰기 작업이 많은 시스템에서는 아래 그림처럼 체크포인트가 거의 항시 발생한다.
* 체크포인트 텀이 길어질수록 강제로 종료된 상황이나 재기동 시에 수행되는 리커버리 과정에 시간이 더 오래 걸릴 수 있음을 인지하고 있어야 한다.   

![memory_and_disk12](/assets/cct_ms_1.png)
> pgbench 스케일 팩터 값으로 35를 주었을 때의 결과 그래프. (순간적으로 550MB 가량의 데이터 발생)


# Base Backup 

## logical backup
* pg_dump & pg_restore
* 데이터베이스가 클수록 시간이 오래 걸린다. 
## physical backup
* logical backup과 비교해서 데이터베이스를 백업하고 복원하는 데 걸리는 시간이 상대적으로 짧다. 

| <img width=200px/>구분 | SQL dump to an archive file: pg_dump -F c | SQL dump to a script file: pg_dump -F p or pg_dumpall | Filesystem backup using: pg_start_backup |
| --------------------------- | ------------------------------------------------------------ |------- |------- |
| 백업 타입  | Logical | Logical | Physical |
| PITR  | 불가 | 불가 | 가능 |
| Zero data loss  | No | No | Yes |
| 모든 DB 백업  | 한번에 하나씩 | pg_dumpall 사용시 가능 | 가능 |
| DB를 선택해 백업  | 가능 | 가능 | 불가 |
| 증분(Incremental) 백업  | 불가 | 불가 | 가능 |
| 백업 파일 압축  | 가능 | 가능 | 가능 |
| 백업 파일 여러 개로 분할  | 불가 | 불가 | 가능 |
| 병렬 백업  | 불가 | 불가 | 가능 |
| 병렬 복구 | 가능 | 불가 | 가능 |
| 백업 중에 DDL 허용  | 불가 | 불가 | 가능 |

> quoted from [PostgreSQL 10 Administration CookBook](https://www.amazon.com/PostgreSQL-Administration-Cookbook-management-maintenance/dp/1788474929)

| <img width=200px/>구분 |                                                                                                                        |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| 스킴                   | 어떤 프로토콜을 사용하는가? `http`, `https`,`ftp`, `rtsp`, `file` … 등등이 올 수 있다. 이어서 나오는 자원 정보가 어떤 프로토콜로 접근하는지 정의한다.                |
| 사용자 이름, 비밀번호         | 공용 자원이 아닌 경우, 몇몇 프로토콜은 계정과 비밀번호를 요구하기도 한다. 예)FTP                                                                       |
| 호스트                  | **서버** 를 뜻한다. naver.com, daum.net, google.com ...                                                                      |
| 포트                   | 서버에 접근하기 위한 포트를 정의한다. 잘 알려진 포트의 경우 생략할 수 있다. (브라우저가 스킴을 보고 자동으로 붙여준다) HTTP는 80, HTTPS는 443, FTP는 21, SFTP는 22 ...      |
| 경로                   | 호스트 내부에서 자원의 위치를 기술한다. 위에서 설명한 파일시스템과 유사한 방식이다. /baseball/kiwoom/1, /java/spring/starter ...                           |
| 파라미터                 | 자원을 정의하기 위한 키 / 밸류 조합. FTP에서는 바이너리, 텍스트 포맷이 있다. `type`파라미터가 i인 경우 바이너리를, a 인 경우 ASCII 포맷이다. `ftp://foo.org/bar;type=i` |
| 질의 (쿼리스트링)           | 자원의 형식을 구체화 하기 위해 질의를 포함할 수 있다. `naver.com/book/search?title=HTTP` 는 도서 리스트 중에서 `title`이 `HTTP` 인 것을 요청할 수 있다.         |
| 프래그먼트                | HTML 문서의 위치를 설명한다. #[TITLE]은 TITLE 섹션을 의미한다.                                                                           |






## pg_start_backup
1. 강제로 full-page write 모드로 전환시킨다. 
2. 현재의 WAL 새그먼터 파일을 스위치한다. 
3. checkpoint를 실행한다. 
4. base directory 최상단 path에 backup_label를 생성한다. 

## backup_label 파일에 담기는 정보 
* CHECKPOINT LOCATION : pg_start_backup를 실행했을 때 발생한 체크포인트가 기록된 LSN 위치. 
* START WAL LOCATION : PITR시에 사용되는 값은 아니고 SR시에 사용된다. 
* BACKUP METHOD : 백업 방식에 대한 설명. pg_start_backup 또는 pg_basebackup 값을 갖는다. 
* BACKUP FROM : 백업이 primary와 standby 중에 어디에서 발생했는지 기록한다. 
* START TIME : pg_start_backup가 실행된 시간. 
* LABEL : pg_start_backup 실행시 입력한 라벨 값. 
* START TIMELINE : v11부터 도입되었고 백업 시작 시점의 타임라인 값을 기록한다. 

## pg_stop_backup
1. pg_start_backup 시점에 강제로 full-page write 모드로 변경되었다면 이를 non-full-page write로 재설정한다.  
2. backup이 끝남을 알리는 XLOG를 쓴다. 
3. WAL 새그먼트 파일을 스위치한다. 
4. 백업 히스토리 파일 생성. 
5. backup_label 파일 삭제. 