---
published: true
layout: single
title: "EDB version 12 새로운 기능"
category: PostgreSQL
comments: false
---
# Overview

## PostgreSQL v12 
- 파티션 테이블 개선 
    - 파티션 테이블에 대한 참조 정합성 제약 
    - Fast Run-time Pruning
    - Attach Partition with Share Update Exclusive Lock
    - 파티션 테이블에 대한 고속 COPY

- 인덱스 
    - B-tree 인덱스의 사이즈가 더욱 작아짐 
    - REINDEX CONCURRENTLY 

- SQL 
    - 생성열 지원 
    - CTE 퍼포먼스 향상 

- 운용 관리 
    - pg_checksum 
    - pg_stat_progress_cluster / pg_stat_progress_create_index 
    - recovery.conf 폐지 (postgresql.conf로 통합) 
    - 플러거블(Pluggable) 스토리지 엔진

## EDB v12 
- 오라클 호환성 
    - interval partitioning 지원 
    - 복합 트리거(Compound Trigger) 지원
    - 집계함수 추가(LISTAGG/MEDIAN)
    - CAST함수의 컬렉션 대응 
    - SYS_GUID 지원 
    - 데이터 딕셔너리 뷰 추가 
    - SELECT UNIQUE 구문 지원 
    - PG v12에서 OID가 폐지됨에 따라 ROWID 

- 스탠바이 DB의 논리적인 리플리케이션 슬롯(Logical Replication Slot) 지원 

## EDB 12에서 지원하는 파티션 
- 레인지(Range) 파티션 : 하나 이상의 파티션 키-컬럼. 값의 범위로 (Less than, MAXVALUE)로 정의하는 통상의 예
- 리스트(List) 파티션 : 정확한 컬럼 값에 근거하여 파티션을 분류
- 해시(Hash) 파티션 : 지정된 컬럼의 해시값에 근거하여 데이터를 균등하게 분할
- 🆕 인터벌(Interval) 파티션 

![Partitions](/assets/partition.png)

Image From [Oracle Docs](https://docs.oracle.com/cd/B12037_01/server.101/b10743/partconc.htm)

## interval partitioning 지원 
- Oracle에서는 11g부터 지원하기 시작한 기능 
- 새로운 튜플이 기존의 파티션 어디에도 해당하지 않을 때 자동으로 해당하는 파티션을 생성 
- 오라클에서 제공되는 인터벌 함수와 호환성이 있기 때문에 NUMTOYMINTERVAL, NUMTODSINTERVAL, TO_YMINTERVAL 등의 함수로 사용자가 인터벌을 정의하여 사용 
- 아직 커뮤니티 PG 버전에서는 사용 불가능한 기능  
- 사용자가 정의한 인터벌에 따라 필요한 파티션을 생성해주므로 MAXVALUE 구문은 같이 사용할 수 없음


```sql 
create table my_part_tab 
( id int, 
dummy text, 
created date)
partition by range (created)
( partition my_part_tab_1 values less than (to_date('2019.02.01','YYYY.MM.DD')));

⬇️ 파티션 range에 포함되는 값을 insert하므로 문제 없이 성공하는 쿼리 
insert into my_part_tab (id,dummy,created) values (1,'aaa',to_date('2019.01.05','YYYY.MM.DD'));

⬇️ 파티션 range에 포함되는 않는 값을 insert하므로 ERROR 발생  
insert into my_part_tab (id,dummy,created) values (1,'aaa',to_date('2019.02.05','YYYY.MM.DD'));

⬇️ 해당 range의 파티션을 미리 만들어두고 insert하면 성공하나 edb v12부터는 좀더 우아한 interval partitioning으로 가능!   
alter table my_part_tab add partition my_part_tab_2 values less than (to_date('2019.03.01','YYYY.MM.DD'));
insert into my_part_tab (id,dummy,created) values (1,'aaa',to_date('2019.02.05','YYYY.MM.DD'));
```                        

```sql 
create table my_part_tab 
(id int, 
dummy text, 
created date)
partition by range (created)
interval (numtoyminterval(1,'month'))
( partition my_part_tab_1 values less than (to_date('01.02.2019','YYYY.MM.DD')));

⬇️ 해당하는 파티션이 있기에 문제 없이 insert가 가능한 쿼리 
insert into my_part_tab (id,dummy,created) values (1,'aaa',to_date('05.01.2019','YYYY.MM.DD'));

⬇️ 해당하는 파티션이 없음에도 insert 가능!  
insert into my_part_tab (id,dummy,created) values (1,'aaa',to_date('05.02.2019','YYYY.MM.DD'));
```

## 파티션 테이블에 대한 외래키 작성 가능 

⬇️ 참조되는 테이블 
```sql 
CREATE TABLE employees( 
empno numeric(8,0) not null ,
ename varchar(32) ,
gender char(1) ,
birthday date ,
deptno number(2) ,
job varchar(9) ,
hiredate timestamp ,
postno char(7) ,
adress varchar2(1000) ,
telno varchar2(20) ,
mobileno varchar2(20))
PARTITION BY RANGE (empno)
(PARTITION employees_0100 VALUES LESS THAN (100) ,
PARTITION employees_0200 VALUES LESS THAN (200) ,
PARTITION employees_MAX VALUES LESS THAN (MAXVALUE) );

CREATE UNIQUE INDEX employees_idx0 on employees(empno);
```
⬇️ 참조하는 테이블 
```sql 
CREATE TABLE emp_incident ( 
inci_no number(8) ,
inc_date date ,
inc_type char(3) ,
empno number(8) ,
description clob ,
remark clob );

ALTER TABLE emp_incident ADD CONSTRAINT ref_empincident_employees FOREIGN KEY (empno) REFERENCES employees(empno);
```

EDB v11에서는 `ERROR:  cannot reference partitioned table "employees"` 파티션 테이블인 employees를 참조 테이블로 설정할 수 없다는 에러 메시지가 뜨지만 v12에서는 가능하다. 다만 파티션의 키컬럼이자 참조되는 컬럼인 empno에 대해 `UNIQUE INDEX`가 생성돼 있어야 한다. 현재는 empno를 포함한 복합 UNIQUE INDEX를 만들어도 에러가 발생하므로 empno 단일 컬럼을 대상으로 한 UNIQUE INDEX를 작성할 필요가 있다.  


## _TAB_PRIVS, _COL_PRIVS, _TAB_DEPENDENCIES 딕셔너리 뷰 지원 

- 이전까지는 information_schema.table_privileges 이용
- 오라클과의 호환성을 강화하는 측면에서 3가지 딕셔너리 뷰를 지원 
- 특정 유저에게 설정된 테이블 권한을 조회하기가 수월해짐 

## 복합 트리거(Compound Trigger) 지원 
- 오라클에서는 11g부터 지원한 기능 
- 전역 변수 사용이 가능하면서 Before & After, Statement & Row 레벨 등 다양한 타이밍에 실행이 가능 
- 복합 트리거는 DML 트리거여야 한다 
- 파티션 테이블에서는 사용 불가 
- 내부에서 PRAGMA AUTONOMOUS_TRANSACTION 선언을 포함시킬 수 없다 
- 오라클에서 ORA-04091 table mutating 에러를 피하는 방법의 하나로 Compound Trigger 사용할 수 있다
- 오라클과의 구문 호환성에 관해서는 [가이드 문서](https://www.enterprisedb.com/edb-docs/d/edb-postgres-advanced-server/user-guides/database-compatibility-for-oracle-developers-guide/12/Database_Compatibility_for_Oracle_Developers_Guide.1.086.html) 참조 

⬇️ 기본 문법 형태 
```sql
CREATE OR REPLACE TRIGGER <복합트리거 명>
FOR INSERT OR UPDATE OR DELETE ON <테이블 명>
COMPOUND TRIGGER 
-- 필요한 Global Variables 정의하는 부분  
-- 여기서 정의된 전역 변수들은 각각의 타이밍 포인트에서 사용될 수 있다

-- DML 구문이 실행되기 전에 실행시키고 싶은 것 
BEFORE STATEMENT IS 
BEGIN
    <처리 부분 1>
END BEFORE STATEMENT; 

-- each row change가 실행되기 전에 실행시키고 싶은 것 :NEW :OLD 변수 사용가능 
BEFORE EACH ROW IS 
BEGIN 
    <처리 부분 2>
END BEFORE EACH ROW;

-- each row change가 발생한 뒤에 실행시키고 싶은 것 :NEW :OLD 변수 사용가능
AFTER EACH ROW IS 
BEGIN 
    <처리 부분 3>
END AFTER EACH ROW;

-- DML 구문 처리를 마친 뒤에 실행시키고 싶은 것 
AFTER STATEMENT IS
BEGIN 
    <처리 부분 4>
END AFTER STATEMENT; 
END <복합트리거 명>;
/
```

마스터 테이블을 오디팅(auditing)하는 테이블을 예로 들어보면 
```sql
--Master Table
CREATE TABLE employees(
    emp_id  varchar2(50) NOT NULL PRIMARY KEY,
    name    varchar2(50) NOT NULL, 
    salary  number NOT NULL
);

--Audit Table
CREATE TABLE aud_emp(
    upd_by    varchar2(50) NOT NULL, 
    upd_dt    date NOT NULL,
    emp_id     varchar2(50) NOT NULL,
    action     varchar2(50) NOT NULL, 
    field     varchar2(50) NOT NULL, 
    from_value varchar2(50) NOT NULL,
    to_value varchar2(50) NOT NULL);
```
마스터 테이블에서 DML 이벤트가 발생하면 오딧 테이블에 기록하는 복합 트리거 작성 
```sql 
CREATE OR REPLACE TRIGGER aud_emp
FOR INSERT OR UPDATE
ON employees
COMPOUND TRIGGER
  
  TYPE t_emp_changes       IS TABLE OF aud_emp%ROWTYPE INDEX BY INTEGER;
  v_emp_changes            t_emp_changes;
  
  v_index                  INTEGER       := 0;
  v_threshhold    CONSTANT INTEGER       := 5; --maximum number of rows to write in one go.
  v_user          VARCHAR2(50); --logged in user
  
  PROCEDURE flush_logs
  IS
    v_updates       CONSTANT INTEGER := v_emp_changes.count();
  BEGIN

    FORALL v_count IN 1..v_updates
        INSERT INTO aud_emp
             VALUES v_emp_changes(v_count);

    v_emp_changes.delete();
    v_index := 0; --resetting threshold for next bulk-insert.

  END flush_logs;

  AFTER EACH ROW
  IS
  BEGIN
        
    IF INSERTING THEN
        v_index := v_index + 1;
        v_emp_changes(v_index).upd_dt       := SYSDATE;
        v_emp_changes(v_index).upd_by       := SYS_CONTEXT ('USERENV', 'SESSION_USER');
        v_emp_changes(v_index).emp_id       := :NEW.emp_id;
        v_emp_changes(v_index).action       := 'Create';
        v_emp_changes(v_index).field        := '*';
        v_emp_changes(v_index).from_value   := 'NULL';
        v_emp_changes(v_index).to_value     := '*';

    ELSIF UPDATING THEN
        IF (   (:OLD.EMP_ID <> :NEW.EMP_ID)
                OR (:OLD.EMP_ID IS     NULL AND :NEW.EMP_ID IS NOT NULL)
                OR (:OLD.EMP_ID IS NOT NULL AND :NEW.EMP_ID IS     NULL)
                  )
             THEN
                v_index := v_index + 1;
                v_emp_changes(v_index).upd_dt       := SYSDATE;
                v_emp_changes(v_index).upd_by       := SYS_CONTEXT ('USERENV', 'SESSION_USER');
                v_emp_changes(v_index).emp_id       := :NEW.emp_id;
                v_emp_changes(v_index).field        := 'EMP_ID';
                v_emp_changes(v_index).from_value   := to_char(:OLD.EMP_ID);
                v_emp_changes(v_index).to_value     := to_char(:NEW.EMP_ID);
                v_emp_changes(v_index).action       := 'Update';
          END IF;
        
        IF (   (:OLD.NAME <> :NEW.NAME)
                OR (:OLD.NAME IS     NULL AND :NEW.NAME IS NOT NULL)
                OR (:OLD.NAME IS NOT NULL AND :NEW.NAME IS     NULL)
                  )
             THEN
                v_index := v_index + 1;
                v_emp_changes(v_index).upd_dt       := SYSDATE;
                v_emp_changes(v_index).upd_by       := SYS_CONTEXT ('USERENV', 'SESSION_USER');
                v_emp_changes(v_index).emp_id       := :NEW.emp_id;
                v_emp_changes(v_index).field        := 'NAME';
                v_emp_changes(v_index).from_value   := to_char(:OLD.NAME);
                v_emp_changes(v_index).to_value     := to_char(:NEW.NAME);
                v_emp_changes(v_index).action       := 'Update';
          END IF;
                       
        IF (   (:OLD.SALARY <> :NEW.SALARY)
                OR (:OLD.SALARY IS     NULL AND :NEW.SALARY IS NOT NULL)
                OR (:OLD.SALARY IS NOT NULL AND :NEW.SALARY IS     NULL)
                  )
             THEN
                v_index := v_index + 1;
                v_emp_changes(v_index).upd_dt      := SYSDATE;
                v_emp_changes(v_index).upd_by      := SYS_CONTEXT ('USERENV', 'SESSION_USER');
                v_emp_changes(v_index).emp_id      := :NEW.emp_id;
                v_emp_changes(v_index).field       := 'SALARY';
                v_emp_changes(v_index).from_value  := to_char(:OLD.SALARY);
                v_emp_changes(v_index).to_value    := to_char(:NEW.SALARY);
                v_emp_changes(v_index).action      := 'Update';
          END IF;
                       
    END IF;

    IF v_index >= v_threshhold THEN
      flush_logs();
    END IF;

   END AFTER EACH ROW;

  -- AFTER STATEMENT Section:
  AFTER STATEMENT IS
  BEGIN
     flush_logs();
  END AFTER STATEMENT;

END aud_emp;
```
-- DML문 실행 
```sql
INSERT INTO employees VALUES (1, 'emp1', 16000);
INSERT INTO employees VALUES (2, 'emp2', 20000);
INSERT INTO employees VALUES (3, 'emp3', 16000);
INSERT INTO employees VALUES (4, 'emp1', 18000);
INSERT INTO employees VALUES (5, 'emp2', 20000);

UPDATE employees 
   SET salary = 2000
 WHERE salary > 10000;
```


ORA-04091 table mutating 에러가 발생하는 상황도 복합트리거로 해결하는 것을 보며 알아보자. 

```sql 
CREATE TABLE EMP(EMPNO INT, ENAME varchar(100), SAL INT, DEPTNO INT);

⬇️ ORA-04091 table mutating 에러가 발생하는 상황 
CREATE OR REPLACE TRIGGER emp_alert_trig
    AFTER INSERT ON emp
    FOR EACH ROW 
    DECLARE
        -- PRAGMA AUTONOMOUS_TRANSACTION(EDB v11부터 지원); 
        cnt NUMBER; 

BEGIN 
    SELECT count(*) into cnt from emp; 
    DBMS_OUTPUT.PUT_LINE('All employees cnt: ' || cnt );
END;     

⬇️ INSERT 쿼리 
INSERT INTO emp (empno, ename) values (1111, 'LEE');

⬇️ Compound Trigger 구문을 사용한 경우  
CREATE OR REPLACE TRIGGER emp_alert_trig
    FOR INSERT ON emp 
    COMPOUND TRIGGER 
        var_sal NUMBER := 10000;
        cnt NUMBER;

    BEFORE STATEMENT IS
     BEGIN
       var_sal := var_sal + 1000;
            DBMS_OUTPUT.PUT_LINE('Before Statement: ' || var_sal);
     END BEFORE STATEMENT;    

    AFTER EACH ROW IS 
        BEGIN 
            DBMS_OUTPUT.PUT_LINE('Insert Completed.');    
        END AFTER EACH ROW;

    AFTER STATEMENT IS 
        BEGIN 
            var_sal := var_sal + 1000;
            DBMS_OUTPUT.PUT_LINE('Before Statement: ' || var_sal);
            SELECT count(*) INTO cnt from emp;      
            DBMS_OUTPUT.PUT_LINE('All employees cnt: ' || cnt );
            
        END AFTER STATEMENT;          
END emp_alert_trig;
```

## SELECT UNIQUE 구문 지원 
- 기능적으로는 DISTINCT 구문과 동일

## 윈도우 함수 LISTAGG, MEDIAN 지원 
- 오라클에서는 11g부터 지원하기 시작한 함수들로 EDB v12에서도 지원을 시작 
- LISTAGG : 복수의 행을 모아 하나의 행으로 집약해서 표시 
- MEDIAN : 해당 컬럼 속의 중위값을 취득
```sql 
SELECT job, 
LISTAGG(ename, ',') WITHIN GROUP (order by ename) 
FROM employees 
GROUP BY job 
ORDER BY job ;
```

## CAST함수에서 Collection 데이터 타입 지원 
- 오라클과의 호환성을 위해 EDB는 version 9.4부터 Collection 데이터 타입(Nested Table, Associative Array, Varray)을 지원해옴
- 빌트인 데이터타입(TIMESTAMP, VARCHAR 등)을 형변환하는 것은 기본적으로 지원
- 12 버전부터는 CAST 구문에서 MULTISET 표현식 안의 서브쿼리가 반환하는 컬렉션 타입의 값도 취할 수 있도록 지원   


## %TYPE과 %ROWTYPE를 Package DDL에서도 지원 
- 9.5버전부터 Function, Procedure에서는 사용 가능했고 적용 범위를 Package에도 확대 
- 테이블이나 뷰의 컬럼 데이터형, 크기, 속성 등을 그대로 사용할 수 있다 

## SYS_GUID() 함수 지원 
- 16바이트(이론적으로는 2^128까지 넘버링 가능)로 구성된 고유한 전역 식별자를 반환하는 함수로 오라클과의 호환성을 강화하는 차원으로 12 버전부터 지원하기 시작 
- 11 버전까지는 uuid-ossp라는 익스텐션을 설치하여 비슷한 함수를 구현해야 했다 

## 생성열 지원 
- Oracle 12c Release 1의 [Identity Columns](https://oracle-base.com/articles/12c/identity-columns-in-oracle-12cr1)와 유사한 기능  
 
```sql 
create table test_table ( a number(5), b varchar(20), c varchar(20), d varchar(20) GENERATED ALWAYS AS (b || c) STORED) ;
```
- 생성열에 직접 INSERT하는 것은 불가 
```sql 
INSERT INTO test_table(a, b, c, d)
VALUES (1, 'AA', 'BB', 'CC');
```

## pg_checksums 
- PG & EDB 11 버전의 pg_verify_checksums의 이름이 변경되면서 기능이 추가되었다  
- ⚠️ 클러스터를 내리고 (offline) 진행해야 하는 점을 운영 환경에서는 고려해야 함 

