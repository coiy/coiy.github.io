---
published: true
layout: single
title: "테이블 row count를 정확하게 구하기"
category: PostgreSQL
comments: false
---


# PL/SQL이란 무엇인가 

* 절차적인 처리가 가능하다.  
SQL 구문만으로는 실행하기 어려운 복잡한 처리가 실행 가능하다. 예를 들면 어떤 데이터를 검색하고 그 데이터를 조건으로 하여 레코드를 갱신할지 말지 판단하는 일련의 처리를 실행할 수 있다.    

* 오라클과의 친화성이 높다.  
오라클에서 개발한 절차적(Procedural Language for SQL) 프로그래밍 언어이기 때문에 오라클에서 사용가능한 모든 데이터 타입을 지원한다. 따라서 SQL과 PL/SQL 사이의 데이터 타입 변환 작업이 필요없다. 

* 이식성이 높다.  
OS가 무엇이든 오라클이 동작하는 환경에서라면 PL/SQL 구문을 수정하지 않고도 실행할 수 있다. 

* 퍼포먼스 측면에서 유리할 수 있다.   
프로그래밍 언어에서 여러 SQL 구문으로 처리하는 경우에는 다양한 미들웨어를 거치면서 오라클 서버와 통신하기 때문에 실행하는 SQL 구문의 수가 많을수록 오버헤드가 걸릴 수 있는 반면  PL/SQL은 블록에 포함돼 있는 다양한 SQL를 하나의 프로그램 단위로 실행하여 전송하기 때문에 오버헤드를 줄일 수 있다. PL/SQL 블록에 네임을 붙어주면 오라클 내부에 구문 분석이 완료된 상태로 저장해둘 수도 있는데(Stored) 이렇게 해두면 구문 분석이 생략되어 부하를 더 줄일 수도 있다. 

 ![wal_segment_file](/assets/plsql1.png)
> 👆SQL 단위로 처리하는 경우 Image from [プロとしてのOracle PL/PSQL入門](https://www.amazon.co.jp/dp/B00QJINRLG/ref=dp-kindle-redirect?_encoding=UTF8&btkr=1)


 ![wal_segment_file](/assets/plsql2.png)
> 👆PL/SQL 블록 단위로 처리하는 경우 Image from [プロとしてのOracle PL/PSQL入門](https://www.amazon.co.jp/dp/B00QJINRLG/ref=dp-kindle-redirect?_encoding=UTF8&btkr=1)


# PL/SQL 구문의 기본 구조 
DECLARE, BEGIN, END 사이에 중첩 구조(Nested)가 있다고 해도 논리적으로는 1개의 블록으로 취급된다. 

```sql 
<<label>>   -- this is optional
DECLARE
-- this section is optional
  number1 NUMBER(2);
  number2 number1%TYPE := 17;             -- value default
  text1   VARCHAR2(12) := 'Hello world';
  text2   DATE         := SYSDATE;        -- current date and time
BEGIN
-- this section is mandatory, must contain at least one executable statement
  SELECT street_number
    INTO number1
    FROM address
    WHERE name = 'INU';
EXCEPTION
-- this section is optional
   WHEN OTHERS THEN
     DBMS_OUTPUT.PUT_LINE('Error Code is ' || TO_CHAR(sqlcode));
     DBMS_OUTPUT.PUT_LINE('Error Message is ' || sqlerrm);
  -- nested block 
  DECLARE
  
  BEGIN 
  
  END;
  -- nested block end 
END;
```

## 사용 가능한 데이터 타입
* NUMBER 
* CHAR
* VARCHAR2
* DATE
* BOOLEAN
* Composite 타입 (RECORD, TABLE, VARRAY 등)

## %TYPE, %ROWTYPE 속성 
PL/SQL을 사용하다보면 테이블에서 검색한 데이터를 직접 변수에 대입하는 경우가 자주 있는데 오라클의 데이터를 직접 취급하는 경우에 대응하는 변수를 %TYPE, %ROWTYPE으로 정의하면 매우 편리하다. 



# PL/SQL의 종류 

* 익명 블록 (PL/SQL anonymous block)
* 함수 (Function)
* 프로시저 (Procedure)
* 패키지 (Package)
* 트리거 (Trigger)

