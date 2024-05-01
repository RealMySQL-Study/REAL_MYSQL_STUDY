# TEXT와 BLOB

MySQL에서 량의 데이터를 저장하려면 TEXT나 BLOB 타입을 사용하는데, 두 타입은 거의 똑같은 설정이나 방식으로 작동함

차이점은 TEXT는 문자열을 저장하는 타입이라 문자 집합이나 콜레이션을 가진다는 점임

| 데이터 타입 | 필요 저장공간(L = 저장하려는 바이트수) | 저장 가능한 최대 바이트 수 |
| :---: | :--- | :--- | 
| TINYTEXT, TINYBLOB | L + 1바이트 | 2^8 - 1(255) |
| TEXT, BLOB | L + 2바이트 | 2^16 - 1(65,535) |
| MEDIUMTEXT, MEDIUMBLOB | L + 3바이트 | 2^24 - 1(16,777,215) |
| LONGTEXT, LONGBLOB | L + 4바이트 | 2^32 - 1(4,294,967,295) |

LONG 이나 LONG VARCHAR 도 있는데 MEDIUMTEXT와 동의어임

**고정 길이와 가변 길이 타입**
|  | 고정길이 | 가변길이 | 대용량 |
| :---: | :--- | :--- | :--- |
| 문자 데이터 | CHAR | VARCHAR | TEXT |
| 이진 데이터 | BINARY | VARVINARY | BLOB |

#### 사용시 주의 사항

+ 컬람 하나에 저장되는 값의 크기가 예측할 수 없이 클 때 사용한다.<br>
  하지만 레코드의 전체크기가 64KB를 넘지 않는 한도 내에서 VARCHAR 나 VARBINARY의 길이 제한이 없음
+ MySQL 에서는 일반적으로 하나의 레코드 크기가 64KB를 넘을 수 없음.<br>
  레코드 전체 크기가 64KB를 넘어서 더 큰 컬럼을 추가 할 수 없으면 일바 컬럼을 대용량 컬럼으로 전화해야함

```SQL
CREATE TABLE TB_VARCHAR(
	ID INT NOT NULL,
    BODY VARCHAR(6000),
    PRIMARY KEY(ID)
);

SHOW CREATE TABLE TB_VARCHAR;
/*
CREATE TABLE `tb_varchar` (
   `ID` int NOT NULL,
   `BODY` varchar(6000) DEFAULT NULL,
   PRIMARY KEY (`ID`)
 ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
-- 책에서는 CLOLATE 가 utf8mb4_general_ci 였음
*/
```


#### 인덱스 사용과 정렬

MySQL에서 인덱스 레코드의 모든 컬럼의 최대 제한 크기를 가지고 있음
> MyISAM : 1000 바이트<br>
> REDUNDANT 나 COMPACT 로우 포맷을 사용 하는 InnoDB : 767바이트<br>
> DYNAMIC 또는 COMPRESSED 로우 포맷을 사용하는 InnoDB : 3072바이트

BLOB 이나 TEXT 타입 컬럼에 인덱스를 생성할 때는 컬럼값의 몇바이트까지 인덱스를 생성할 것인지를 명시해야함

최대 제한크기를 넘어서는 인덱스는 생성할 수 없음

BLOB 이나 TEXT 컬럼으로 정렬을 수행해도 ``max_sort_length`` 시스템 변수에 설정된 길이까지만 정렬을 수행

일반적으로 1024 바이트로 설정되어 있는데 이 값을 줄이면 더 빠르게 정렬 가능

#### internal_tmp_storage_engine

쿼리의 특성에 따라 임시테이블을 생성해야 할 때가 있음

임시테이블을 메모리에 저장할때 ``internal_tmp_storage_engine`` 시스템 변수의 설정값에 따라 MEMORY 나 TempTable 스토리지 엔진 중 하나를 사용

8.0 버전부터는 MEMORY 스토리지 엔진이 TEXT, BLOB 타입을 지원하지 않음

가능하면 ``internal_tmp_storage_engine``시스템 변수를 "TempTable"로 설정해서 대용량 타입을 포함한 결과도 메모리를 사용 할 수 있게 하는 것이 좋음

#### max_allow_packet

BLOB 이나 TEXT 컬럼을 포함한 테이블에 INSERT, UPDATE 작업을 할 때 SQL 문이 매우 길어질 수 있음

``max_allow_packet`` 시스템 변수 값보다 큰 SQL문장은 MySQL 서버로 전송되지 못하고 오류가 발생하는데

BLOB이나 TEXT를 사용한다면 ``max_allow_packet`` 시스템 변수를 필요한만큼 늘려서 설정하는 것이 좋음

#### ROW_FORMAT

BLOB 이나 TEXT 타입 컬럼의 데이터가 어떻게 저장될지 결정하는 요소는 테이블의 ROW_FORMAT 옵션임

별도로 지정하지 않으면 ``innodb_default_row_format`` 시스템 변수를 적용(보통 dynamic으로 되어있음)

```sql
show variables like 'innodb_default_row_format';
```
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/5bb1d49d-c947-4c8e-afb6-a809fec01737)


8.0버전에서는 모든 ROW_FORMAT(REDUNDANT, COMPACT, DYNAMIC, COMPRESSED) 에서는 가능하면 TEXT와 BLOB 모두 다른 레코드와 같이 저장하려고 함

그런데 레코드의 최대 길이 제한때문에 불가능함

```sql
CREATE TABLE TB_LARGE_OBJECT(
	ID INT NOT NULL PRIMARY KEY,
    FD_BLOB BLOB,
    FD_TEXT TEXT
);
```

5.6 버전에서의 기본 ROW_FORMAT은 COMPACT 였고 5.7부터는 DYNAMIC임

COMPACT 는 모든 ROW_FORMAT의 바탕이 되는 포맷

+ DYNAMIC = COMPACT + 몇 가지 규칙
+ COMPRESSED = DYNAMIC + 압축 관련 규칙

COMPACT 포맥에 저장 가능한 레코드의 최대길이는 데이터 페이지(데이터 블록) 크기(16KB)의 절반인 8126바이트(8K - 데이터 페이지 관리용 공간)임

이경우 BLOB 이나 TEXT 타입 컬럼을 가능한 레코드에 같이 포함해서 저장하려고 함

이 제한을 넘어버리면 용량이 큰 컬럼 순서대로 외부 페이지(Off-page 또는 External-page)로 옮기면서 레코드의 크리를 8126 바이트이하로 맞춤

| FD_BLOB의 길이 | FD_TEXT의 길이 | FD_BLOB의 저장 위치 | FD_TEXT의 저장 위치 |
| ---: | ---: | :--- | :--- |
| 3000 | 3000 | PK 페이지 | PK 페이지 |
| 3000 | 10000 | PK 페이지 | 외부 페이지 |
| 10000 | 10000 | 외부 페이지 | 외부 페이지 |

마지막 경우처럼 저장시 길이가 16KB를 넘으면 컬럼의 값을 나눠서 여러개의 외부페이지에 저장하고 각 페이지를 체인으로 연결함

![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/ee89231e-6650-4123-bdc5-61ddb9837cdb)


외부 페이지로 저장하는 경우 COMPACT 와 REDUNDANT 레코드 포맷을 사용하는 테이블에서는<br>
외부 페이지로 저장된 TEXT나 BLOB 컬럼의 앞쪽 768바이트(BLOB 프리픽스)만 잘라서 PK 페이지에 같이 저장

DYNAMIC이나 COMPRESSED 레코드 포맷에서는 BLOB 프리픽스를 PK 페이지에 저장하지 않음

BLOB 프리픽스는 인덱스를 생성할때 도움이 되기도 하지만 저장 효율을 낮추게 될 수 있음<br>
또 BLOB 프리픽스는 PK 페이지에 저장할 수 있는 레코드 수를 줄이는데, BLOB이나 TEXT 컬럼을 거의 참조하지 않는 쿼리는 성능이 더 떨어짐
