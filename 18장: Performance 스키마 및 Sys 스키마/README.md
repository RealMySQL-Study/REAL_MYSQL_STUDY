## Performance 스키마

MySQL 서버 내부 동작 및 쿼리 처리와 관련된 세부 정보들이 저장되는 테이블이 존재하는 스키마(데이터베이스)

성능을 분석하고 내부 처리 과정등을 모니터링 할 수 있음

수집한 정보를 메모리에 저장하다보니 활성화 되어있으면 CPU와 메모리등 리소스를 더 사용함

## Sys 스키마

Performance 스키마에 저장되어 있는 데이터를 사용자들이 더 쉽게 이해할 수 있는 형태로 출력하는 뷰, 스토어드 프로시저, 함수 등을 제공

Performance 스키마에 저장된 데이터를 참조하기 때문에 Performance 스키마가 활성화 되어 있어야 사용 가능(``performance_schema=ON``)

## 사용 예제

### 호스트 접속이력 확인

```SQL
SELECT * FROM PERFORMANCE_SCHEMA.HOSTS;
```
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/32b099fb-bace-46c3-b28d-fa4ae19ee4d1)

호스트가 NULL인 것들은 내부 스레드 이거나 연결 시 인증 실패한 것들

```SQL
SELECT HOST, CURRENT_CONNECTIONS
FROM PERFORMANCE_SCHEMA.HOSTS
WHERE CURRENT_CONNECTIONS > 0 AND HOST NOT IN ('NULL', '127.0.0.1')
ORDER BY HOST;
```
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/eb2b9d01-0167-412d-b0fa-0b348053451e)


### 미사용 DB 계정 확인

서버 구동 이후로 사용되지 않은 DB 계정을 확인 할 때
```SQL
SELECT DISTINCT 
	M_U.USER, 
    M_U.HOST
FROM MYSQL.USER M_U
LEFT JOIN PERFORMANCE_SCHEMA.ACCOUNTS PS_A
	ON M_U.USER = PS_A.USER 
    AND PS_A.HOST = M_U.HOST
LEFT JOIN INFORMATION_SCHEMA.VIEWS IS_V
	ON IS_V.DEFINER = CONCAT(M_U.USER, '@', M_U.HOST)
    AND IS_V.SECURITY_TYPE = 'DEFINER'
LEFT JOIN INFORMATION_SCHEMA.ROUTINES IS_R 
	ON IS_R.DEFINER = CONCAT(M_U.USER, '@', M_U.HOST)
    AND IS_R.SECURITY_TYPE = 'DEFINER'
LEFT JOIN INFORMATION_SCHEMA.EVENTS IS_E 
	ON IS_E.DEFINER = CONCAT(M_U.USER, '@', M_U.HOST)
LEFT JOIN INFORMATION_SCHEMA.TRIGGERS IS_T
	ON IS_T.DEFINER = CONCAT(M_U.USER, '@', M_U.HOST)
WHERE PS_A.USER IS NULL
	AND IS_V.DEFINER IS NULL
    AND IS_R.DEFINER IS NULL
    AND IS_E.DEFINER IS NULL
    AND IS_T.DEFINER IS NULL
ORDER BY M_U.USER, M_U.HOST;
```
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/8e283c35-5efd-43ca-b63d-5d26be5d517c)


### MySQL 총 메모리 사용량 확인

```SQL
SELECT * FROM SYS.MEMORY_GLOBAL_TOTAL;
```
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/146c41a0-dbb5-4306-a7d3-1a1f0354ba56)

MySQL 서버에 할당된 메모리의 전체 크기를 확인

사용중인 메모리보다 클 수 있음


### 스레드별 메모리 사용량 확인
```SQL
SELECT THREAD_ID, USER, CURRENT_ALLOCATED
FROM SYS.MEMORY_BY_THREAD_BY_CURRENT_BYTES
LIMIT 10;
```
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/d9a42e6e-105d-4e22-a750-6d87198579e1)

``SYS.MEMORY_BY_THREAD_BY_CURRENT_BYTES`` 뷰는 기본적으로 ``CURRENT_ALLOCATED`` 값 기준으로 내림차순으로 정렬

```SQL
-- 특정 스레드에 대해 구체적인 메모리 할당 내역을 확인 할 때
SELECT THREAD_ID,
	EVENT_NAME,
    SYS.FORMAT_BYTES(CURRENT_NUMBER_OF_BYTES_USED) AS CURRENT_ALLOCATED
FROM PERFORMANCE_SCHEMA.MEMORY_SUMMARY_BY_THREAD_BY_EVENT_NAME
WHERE THREAD_ID = 36
ORDER BY CURRENT_NUMBER_OF_BYTES_USED DESC;
```
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/244c730e-e3c8-45f5-9084-64435163cb74)

여기서 ``THREAD_ID``는 ``SHOW PROCESSLIST``의 `ID`` 와는 다름

```SQL
-- 아래와 같은 방법으로 PROCESSLIST의 ID로 THREAD_ID를 찾을 수 있음
SELECT THREAD_ID, PROCESSLIST_ID
FROM PERFORMANCE_SCHEMA.THREADS
WHERE PROCESSLIST_ID = 5;

SELECT SYS.PS_THREAD_ID(5);
```
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/64cc51ba-0ff0-4d44-ac89-d53d40939745)

![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/b79f9083-d78c-4077-a601-f4e7b0784b86)


### 미사용 인덱스 확인
```SQL
SELECT *
FROM SYS.SCHEMA_UNUSED_INDEXES;
```
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/73b5610e-e142-423b-b236-ee97dfeb60f0)

사용하지 않는 인덱스라면 제거하는 것이 좋음

제거할 때는 안전하게 인덱스를 ``INVISIBLE``상태로 만들어서 인정 기간동안 실제로 사용하지 않는지 확인 후 지우는 것이 좋음

```SQL
ALTER TABLE USERS ALTER INDEX IX_NAME INVISIBLE;

SELECT TABLE_NAME, 
	INDEX_NAME,
    IS_VISIBLE
FROM INFORMATION_SCHEMA.STATISTICS
WHERE TABLE_SCHEMA = ''
	AND TABLE_NAME = ''
    AND INDEX_NAME = '';
```

### 중복된 인덱스 확인
```SQL
-- 인덱스 구성이 같거나, 한 인덱스 구성이 다른 인덱스 구성에 포함 될때
SELECT * FROM SYS.SCHEMA_REDUNDANT_INDEXES
```
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/8069141c-b01a-46ec-8319-4bcff2d8151c)


### 변경이 없는 테이블 목록 확인
```SQL
SELECT T.TABLE_SCHEMA,
	T.TABLE_NAME,
    T.TABLE_ROWS,
    TIO.COUNT_READ,
    TIO.COUNT_WRITE
FROM INFORMATION_SCHEMA.TABLES AS T
JOIN PERFORMANCE_SCHEMA.TABLE_IO_WAITS_SUMMARY_BY_TABLE AS TIO
	ON TIO.OBJECT_SCHEMA = T.TABLE_SCHEMA
    AND TIO.OBJECT_NAME = T.TABLE_NAME
WHERE T.TABLE_SCHEMA NOT IN ('mysql', 'performance_schema', 'sys')
	AND TIO.COUNT_WRITE = 0
ORDER BY T.TABLE_SCHEMA, T.TABLE_NAME;
```
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/39dc258d-fac4-4881-b9bf-21f277ffadfa)


### I/O 요청이 많은 테이블 목록 확인
```SQL
SELECT * FROM SYS.IO_GLOBAL_BY_FILE_BY_BYTES
WHERE FILE LIKE '%ibd';
```
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/094b9084-fef4-4b5e-be43-83c349f975dd)


### 테이블별 작업량 통계 확인
```SQL
SELECT TABLE_SCHEMA,
	TABLE_NAME,
	ROWS_FETCHED,
    ROWS_INSERTED,
    ROWS_UPDATED,
    ROWS_DELETED,
    IO_READ,
    IO_WRITE
FROM SYS.SCHEMA_TABLE_STATISTICS
WHERE TABLE_SCHEMA NOT IN ('mysql', 'performance_schema','sys');
```

### 테이블의 Auto-Increment 컬럼 사용량 확인
```SQL
SELECT TABLE_SCHEMA,
	TABLE_NAME,
    COLUMN_NAME,
    AUTO_INCREMENT AS CURRENT_VALUE,
    MAX_VALUE,
    ROUND(AUTO_INCREMENT_RATIO * 100, 2) AS USAGE_RATIO
FROM SYS.SCHEMA_AUTO_INCREMENT_COLUMNS;
```
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/de0dbc55-9dbf-4277-b62c-8ca86c87991e)


### 풀 테이블 스캔 쿼리 확인
```SQL
SELECT DB,
	QUERY,
	EXEC_COUNT,
    SYS.FORMAT_TIME(TOTAL_LATENCY) AS FORMATTED_TOTAL_LATENCY,
    ROWS_SENT_AVG,
    ROWS_EXAMINED_AVG,
    LAST_SEEN
FROM SYS.X$STATEMENTS_WITH_FULL_TABLE_SCANS
ORDER BY TOTAL_LATENCY DESC;
```
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/811a573d-da74-403d-92af-96099ec4098e)


### 자주 실행되는 쿼리 목록 확인
```SQL
SELECT DB,
	EXEC_COUNT,
    QUERY
FROM SYS.STATEMENT_ANALYSIS
ORDER BY EXEC_COUNT DESC;
```
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/5b8be813-77f2-42b3-a9be-de4930ab2fc7)


### 실행 시간이 긴 쿼리 목록 확인
```SQL
SELECT QUERY,
	EXEC_COUNT,
    SYS.FORMAT_TIME(AVG_LATENCY) AS FORMATTED_AVG_LATENCY,
    ROWS_SENT_AVG,
    ROWS_EXAMINED_AVG,
    LAST_SEEN
FROM SYS.X$STATEMENT_ANALYSIS
ORDER BY AVG_LATENCY DESC;
```
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/b52d5132-319c-4c37-b9e5-24c0bcbd5ff5)


### 정렬 작업을 수행한 쿼리 목록 확인
```SQL
SELECT * FROM SYS.STATEMENTS_WITH_SORTING ORDER BY LAST_SEEN DESC LIMIT 1;
```
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/b5665c50-109b-4ed3-bf47-f1e189c62e24)

### 임시 테이블을 생성하는 쿼리 목록 확인
```SQL
SELECT * FROM SYS.STATEMENTS_WITH_TEMP_TABLES LIMIT 10;
```
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/ce743ab2-9e9a-4df8-9136-803c3aa0988c)


### 트랜잭션이 활성 상태인 커넥션에서 실행한 쿼리 내역확인
```SQL
```

### 쿼리 프로파일링
```SQL
```

### ALTER 작업 진행률 확인
```SQL
```

### 메타데이터락 대기 확인
```SQL
```

### 데이터 락 대기 확인
```SQL
```

