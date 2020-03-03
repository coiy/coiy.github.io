---
published: true
layout: single
title: "MTK 세팅하기"
category: PostgreSQL
comments: false
---


1
오라클 인스턴트 클라이언트가 있는 디렉토리에서 ojdbc 파일을 jre/lib/ext 디렉토리에 복사. 

```bash 
$ readlink -f $(which java)
```

2
conf파일의 oracle_home 값으로 /pg_rman/instantclient_19_6 추가 









