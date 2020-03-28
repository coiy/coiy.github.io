---
published: true
layout: single
title: "Archived Wal의 용량 부족 현상"
category: PostgreSQL
comments: false
---

archive_mode를 on으로 하고 archive_command 파라미터에 자신의 상황에 맞는 커맨드와 함께 path를 지정하면 pg_wal(또는 pg_xlog)에 있는 WAL 파일이 해당 path로 아카이빙된다. 이렇게 아카이빙된 WAL 파일(pg_wal의 WAL파일과 구별하기 위해 이하 아카이브 WAL로 통일)은 Point In Time Recovery에 사용되기 때문에 basebackup을 취득한 시점을 기준으로 아카이브 WAL을 많이 보유할수록 복원시점을 더 먼 과거로 설정할 수 있다.

Continuous Archiving을 위한 세팅 조건 

|파라미터 |  설정값 |  설명
| --------------------------- | ------------------------- |------------------------- |
|wal_level |  archive   | archive 이상이어야 한다. |
|max_wal_size |  1GB  | 보통 pg_wal(또는 pg_xlog) 디렉토리에 저장되는 WAL 파일들의 총합을 뜻한다. soft limit이라서 약간 초과할 수 있다.  |
|wal_keep_segments |  10   |   최소한으로 존재해야 하는 WAL 파일 숫자를 정의한다.    |
|archive_mode |  on   | 아카이브 모드로 쓰려면 필수로 on이어야 한다.   |
|archive_command |  cp %p /archive/%f   | WAL 파일을 어디로 어떻게 아카이브 할지 정의한다. 자신의 환경에 따라 아카이브 path는 다를 수 있다.      |
|archive_timeout |  0  | 설정된 시간값에 도달하면 WAL이 아카이브된다. 디폴트는 0 값으로 이 기능은 disabled 돼 있다.  |

아카이브 WAL을 계속 쌓아나갈 수는 없기 때문에 crontab으로 
정해진 삭제 정책을 준수하는 선에서 아카이브 디렉토리의 아카이브 WAL 파일을 삭제하여 용량 관리를 하도록 안내한다.   

```bash
# 매일 새벽3시에 /archive 디렉토리에 저장된 아카이브 WAL 파일 중에서 최근 3주(21일)분만 남기고 삭제하는 crontab의 예
00 3 * * * find /archive/* -mtime +21 -exec rm -rf {} \;
```

보통 아카이브 WAL은 별도의 파티션을 마련하여 마운트시킨 곳에 저장되도록 하는데 용량관리를 하지 않은 탓에 해당 파일시스템이 다 차서 아카이브에 실패하는 경우가 드물지 않게 발생한다. 아카이빙 실패가 발생했을 때 어떤 로그가 기록되고 긴급하게 어떤 작업을 하면 되는지 테스트 환경에서
아카이빙 실패를 재현해보면서 확인해보자. 

도커에서 CentOS 7.6 이미지로 컨테이너를 만들고 EDB 11.7 버전을 설치한 호스트의 파일시스템 현황이다. 아카이브 WAL은 1.5GB 파일시스템의 마운트포인트인 /archive에 저장된다. pg_wal은 EDB의 디폴트 디렉토리인 /var/lib/edb/as11/data/pg_wal이고 이곳에 mal_wal_size만큼만 WAL이 쌓인다. 

```bash
[enterprisedb@EFM_EDB_11_master archive]$ df -h
Filesystem      Size  Used Avail Use% Mounted on
overlay         103G   28G   70G  29% /
tmpfs            64M     0   64M   0% /dev
shm              64M   12K   64M   1% /dev/shm
osxfs           932G  632G  300G  68% /pg_rman
tmpfs           1.5G  865M  636M  58% /archive
/dev/sda1       103G   28G   70G  29% /etc/hosts
cgroup_root      10M     0   10M   0% /sys/fs/cgroup
tmpfs           7.9G  8.4M  7.9G   1% /run
tmpfs           1.6G     0  1.6G   0% /run/user/999
[enterprisedb@EFM_EDB_11_master archive]$
```

## 아카이빙에 실패했을 때 일어나는 일 

/archive이 가득 차도록 하기 위해 쓰기 트랜잭션을 대량으로 발생시켰다. 먼저 EDB 로그를 확인해보면 아래와 같다. 해당 디바이스의 여유 공간이 없어서 00000001000000000000005F이라는 이름의 WAL 파일을 아카이빙 하는 데 실패했다는 로그가 찍혔다. 

```bash
#EDB 로그
2020-03-27 10:52:59 KST LOG:  archive command failed with exit code 1
2020-03-27 10:52:59 KST DETAIL:  The failed archive command was: cp pg_wal/00000001000000000000005F /archive/00000001000000000000005F
cp: error writing '/archive/00000001000000000000005F': No space left on device
cp: failed to extend '/archive/00000001000000000000005F': No space left on device
2020-03-27 10:53:00 KST LOG:  archive command failed with exit code 1
2020-03-27 10:53:00 KST DETAIL:  The failed archive command was: cp pg_wal/00000001000000000000005F /archive/00000001000000000000005F
```

pg_wal 디렉토리 하위에 있는 archive_status 디렉토리의 현황을 확인해보면 다음과 같다. 00000001000000000000005F부터 아카이빙에 실패하여 .ready라는 접미어가 찍혀있는 걸 확인할 수 있다. 

```bash 
[enterprisedb@EFM_EDB_11_master archive_status]$ ls -al
total 8
drwx------ 2 enterprisedb enterprisedb 4096 Mar 27 01:52 .
drwx------ 3 enterprisedb enterprisedb 4096 Mar 27 01:52 ..
-rw------- 1 enterprisedb enterprisedb    0 Mar 27 01:47 00000001000000000000001E.00000028.backup.done
-rw------- 1 enterprisedb enterprisedb    0 Mar 27 01:51 00000001000000000000005D.done
-rw------- 1 enterprisedb enterprisedb    0 Mar 27 01:51 00000001000000000000005E.done
-rw------- 1 enterprisedb enterprisedb    0 Mar 27 01:51 00000001000000000000005F.ready
-rw------- 1 enterprisedb enterprisedb    0 Mar 27 01:51 000000010000000000000060.ready
-rw------- 1 enterprisedb enterprisedb    0 Mar 27 01:51 000000010000000000000061.ready
-rw------- 1 enterprisedb enterprisedb    0 Mar 27 01:51 000000010000000000000062.ready
-rw------- 1 enterprisedb enterprisedb    0 Mar 27 01:51 000000010000000000000063.ready
-rw------- 1 enterprisedb enterprisedb    0 Mar 27 01:51 000000010000000000000064.ready
-rw------- 1 enterprisedb enterprisedb    0 Mar 27 01:51 000000010000000000000065.ready
-rw------- 1 enterprisedb enterprisedb    0 Mar 27 01:51 000000010000000000000066.ready
```

아카이브 모드에서 아카이빙에 실패하면 pg_wal의 WAL 파일이 재사용되지 못하기 때문에 max_wal_size의 설정값을 초과하여 pg_wal 디렉토리가 커지게 된다. (이런 현상을 PG 커뮤니티에서는 bloated pg_wal이라고 부른다)

해야 할 일은 아카이브 path을 여유 공간을 확보하여 아카이빙 작업에 문제가 없도록 하는 것이다. 스탠바이에서는 [pg_archivecleanup 유틸리티](https://www.postgresql.org/docs/11/pgarchivecleanup.html)로 .backup 파일을 파라미터로 넘겨주면서 정리하는 방법이 있지만 이 글에서는 마스터에서 해당 아카이브 WAL보다 작은 값을 가진 파일들을 전부 다른 곳으로 이동하는 방법으로 용량을 확보하려고 한다. 

```bash 
# 파일명이 00000001000000000000005F 작은 값들을 추려내서 파일로 저장한다. 
ls /archive | awk '$0 < "00000001000000000000005F"' > /tmp/temp.sh

# vim editor 기준 
# 모든 라인의 맨 처음을 mv /archive/로 치환한다.
:%s/^/mv \/archive\//g

# 모든 라인의 맨 마지막을 /tmp(이동시키려는 디렉토리)로 치환한다.
:%s/$/ \/tmp/g

# /tmp/temp.sh에 실행권한을 준다. 
chmod +x /tmp/temp.sh

# 셸스크립트를 실행한다.  
source /tmp/temp.sh
```

여기까지 문제가 없었다면 아카이브 WAL path에 여유공간이 확보되었을 것이다. 로그를 모니터링 하면서 select pg_switch_wal();을 실행해 wal 로그를 한번 스위치 해준다. 

## 참고 레퍼런스 
* [Continuous Archiving and Point-in-Time Recovery](https://www.postgresql.org/docs/11/continuous-archiving.html)
* [PostgreSQL WAL Archiving](https://www.opsdash.com/blog/postgresql-wal-archiving-backup.html)
* [When will PostgreSQL execute archive_command](https://dba.stackexchange.com/questions/51578/when-will-postgresql-execute-archive-command-to-archive-wal-files)