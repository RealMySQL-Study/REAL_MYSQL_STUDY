# 15.7 JSON 타입

5.7 버전부터 JSON 타입을 지원하고 8.0 부터는 기능과 개선사항이 추가됨

## 저장 방식

내부적으로는 바이너리 포맷인 BSON 타입으로 변환해서 저장함 => BLOB 이나 TEXT 타입 컬럼에 저장하는 것보다 공간 효율이 높음

```sql
CREATE TABLE TB_JSON(
	ID INT ,
    FD JSON
);
-- JSON 컬럼의 값이 이진 포맷으로 변환됐을때 길이가 몇 바이트인지 확인하는 예제
INSERT INTO TB_JSON VALUES
(1, '{"USER_ID":1234567890}'),
(2, '{"USER_ID":"1234567890"}');

SELECT ID, FD,
	JSON_TYPE(FD -> "$.USER_ID") AS FIELD_TYPE,
    JSON_STORAGE_SIZE(FD) AS BYTE_SIZE
FROM TB_JSON;
```
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/defa4706-666d-4fd7-8a3a-e7f4611469e5)

정수 타입과 문자열 타입으로 저장한 차이가 있어서 7바이트의 차이가 생김

```SQL
-- JSON 도큐먼트 {"a":"x", "b":"y", "c":"z"}
SELECT JSON_STORAGE_SIZE('{"a":"x", "b":"y", "c":"z"}') AS BINARY_LENGTH;
```
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/f4590e3c-b0f6-4ea6-9c24-3f5ffa9f80e0)

실제 이진 데이터는 JSON의 타입, JSON 어트리뷰트 개수, JSON 도큐먼트 길이, 키의 주소, 키 길이, 값의 타입, 값 주소, 키, 값의 길이, 값 등으로 이루어져 있음

JSON 도큐먼트를 구성하는 키의 위치와 키의 이름이 JSON 필드 값보다 먼저 나열되어 있어서<br>
JSON 컬럼의 특정 필드만 참조하거나 특정 필드의 값만 (길이가 변하지 않는)업데이트 하는 경우<br>
JSON 컬럼의 값을 모두 읽어보지 않고도 즉시 변경 가능

매우 큰 JSON 도큐먼트가 저장되면 16KB 단위로 여러 페이지에 저장함

5.7 버전까지는 BLOB 페이지들은 단순 링크드 리스트처럼 관리됐었음

이런 형태는 JSON 필드의 부분 업데이트를 효율적으로 처리할 수 없어서 8.0 버전부터는<br>
BLOB 페이지 들의 인덱스를 관리하고 각 인덱스는 실제 BLOB  데이터를 가진 페이지들의 링크를 갖도록 개선

<br>

## 15.7.2 부분 업데이트 성능

8.0 버전부터 JOSN 타입에 대해 부분 업데이트(Partial Update) 기능을 제공

JSON_SET(), JSON_REPLACE(), JSON_REMOVE() 를 이용해 특정 필드값을 변경, 삭제 하는 경우에만 작동

```SQL
UPDATE TB_JSON
SET FD = JSON_SET(FD, '$.USER_ID', "12345")
WHERE ID = 2;
-- KEY 컬럼으로 WHERE 절 안걸었다고 오류 떠서 유니크키 만들어주고 돌림
SELECT ID, FD,
	JSON_STORAGE_SIZE(FD),
    JSON_STORAGE_FREE(FD)
FROM TB_JSON;
```

![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/385c1205-072d-46ce-9dfb-f3e7eff20edc)

부분 업데이트로 처리 되었는지 확인하는 명확한 방법은 없지만 JSON_STORAGE_SIZE() 와 JSON_STORAGE_FREE()를 이용하면 대략 알 수 있음

JSON_STORAGE_FREE() 결과값의 차이가 나는데 필드 값이 변경되면서 10 바이트를 사용하다가 5바이트가 비워져서 생긴 현상임

```SQL
-- 필드값을 10바이트 이상의 값으로 변경하고 확인해보
UPDATE TB_JSON
SET FD = JSON_SET(FD, '$.USER_ID', "12345678901")
WHERE ID = 2;

SELECT ID, FD,
	JSON_STORAGE_SIZE(FD),
    JSON_STORAGE_FREE(FD)
FROM TB_JSON;
```
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/cc9cbf3f-7edb-4307-9030-c5038ab9f4ab)

이경우 JSON_SET 를 사용했지만 부분 없데이트 방식으로 처리되지 못함

최초의 10바이트 공간이 부족하기 때문에 JSON 컬럼 또는 두번째 레코드를 다른위치로 복사해서 저장했기 때문

<br>

부분 업데이트는 특정 조건에서 매우 빠른 업데이트 성능을 보여줌

내부적으로 BLOB(LONGBLOB)으로 저장되는 JSON은 최대 4GB까지 값을 가질 수 있음

1MB만 저장해도 16KB 페이지 64개나 사용하게 되는데

부분 업데이트의 경우 64개의 페이지중 하나만 변경하면 되지만 

부분 업데이트를 못하면 64개의 페이지를 디스크에 다시 기록해야함

```SQL
-- 성능 차이를 보자
DROP TABLE TB_JSON;

CREATE TABLE TB_JSON(
	ID INT, FD JSON, PRIMARY KEY(ID)
);

INSERT INTO TB_JSON (ID, FD) VALUES
	(1, JSON_OBJECT('name', 'Matt', 'visits', 0, 'data', REPEAT('a', 10 * 1000 * 1000))),
    (2, JSON_OBJECT('name', 'Matt', 'visits', 0, 'data', REPEAT('b', 10 * 1000 * 1000))),
    (3, JSON_OBJECT('name', 'Matt', 'visits', 0, 'data', REPEAT('c', 10 * 1000 * 1000))),
    (4, JSON_OBJECT('name', 'Matt', 'visits', 0, 'data', REPEAT('d', 10 * 1000 * 1000)));

INSERT INTO TB_JSON(ID, FD) SELECT ID + 5, FD FROM TB_JSON;
INSERT INTO TB_JSON(ID, FD) SELECT ID + 10, FD FROM TB_JSON;

-- 부분 업데이트를 사용하지 못하는 경우
UPDATE TB_JSON
SET FD = JSON_SET(FD, '$.name', 'Matt Lee');
-- 16 rows affected in 39 s 660 ms

-- 부분 업데이트 사용하는 경우
UPDATE TB_JSON
SET FD = JSON_SET(FD, '$.name', 'Kit');
-- 16 rows affected in 5 s 782 ms
```

MySQL은 일반적으로 복제를 사용하기 때문에 MySQL은 JSON 변경 내용을 바이너리 로그에 기록해야함

이때 바이너리 로그에는 여전히 JSON의 데이터를 모두 기록함

하지만 변경된 내용들만 바이너리 로그에 기록되도록 하는 ``binary_row_value_options``와 ``binlog_row_image`` 시스템변수의 설정값을 변경하면 JSON 컬럼의 부분 업데이트의 성능을 더 빠르게 만들 수 있음

```sql
set binlog_format = ROW;
set binlog_row_value_options = PARTIAL_JSON;
set binlog_row_image = MINIMAL;

UPDATE TB_JSON
SET FD = JSON_SET(FD, '$.name', 'Matt Lee');
-- 16 rows affected in 3 s 165 ms

UPDATE TB_JSON
SET FD = JSON_SET(FD, '$.name', 'Kit');
-- 16 rows affected in 2 s 736 ms
```

바이너리 로그의 포맷을 STATEMENT 타입으로 변경해도 비슷한 효과를 얻을 수 있음

> 부분 업데이트 최적화 효과를 얻기 위해서는 ``binary_row_value_options`` 와 ``binlog_row_image``의 변경도 필요하지만<br>
> JSON 커럶이 가지고 있는 PK가 필수적임<br>
> PK가 없으면 업데이트할 레코드를 식별하기 위해 레코드의 모든 컬럼을 필요로함

단순 정수 필드의 값을 변경하는 UPDATE는 항상 부분 업데이트 기능이 적용됨

하지만 문자열 타입의 필드는 저장되는 문자열의 길이에 따라 부분 업데이트가 사용되지 못할 수도 있음

특정 필드가 작은 용량을 가지면서 자주 길이가 변경되면 해당 필드가 가질 수 있는 최대 길이의 값으로 초기해 두거나 패딩해서 고정길이로 저장하는 방법이 있음

<br>

## 15.7.3 JSON 타입 콜레이션과 비교

JSON 컬럼에 저장되는 데이터와 JSON 컬럼으로부터 가공되어 나온 결과값은 모두 uft8mb4 문자 집합과 utf8mb4_bin 콜레이션을 가짐

그래서 대소문자 및 액센트 문자등도 구분해서 비교해야함

```SQL
SET @USER1 = JSON_OBJECT('name', 'Matt');
SELECT CHARSET(@USER1), COLLATION(@USER1);
```
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/09671b49-f058-443d-911f-d6c7cf6f4d5b)

```SQL
SET @USER2 = JSON_OBJECT('name', 'matt');
SELECT @USER1 = @USER2;
```
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/02143da7-699a-4ad9-941d-9563dd35336a)

<br>

## 15.7.4 JSON 컬럼 선택

일반적으로 정규화한 컬럼과 JSON 컬럼중에는 어떤것을 선택해야할까?

```SQL
-- JSON 컬럼으로 구성된 테이블
CREATE TABLE TB_JSON2(
	DOC JSON NOT NULL,
    ID BIGINT AS (DOC ->> '$.ID') STORED NOT NULL,
    PRIMARY KEY (ID)
);

INSERT INTO TB_JSON2(DOC) VALUES
	('{"ID":1, "NAME":"Matt"}'),
    ('{"ID":2, "NAME":"Esther"}');

-- 정규화된 컬럼으로 구성된 테이블
CREATE TABLE TB_COLUMN(
	ID BIGINT NOT NULL,
    NAME VARCHAR(50) NOT NULL,
    PRIMARY KEY (ID)
);

INSERT INTO TB_COLUMN VALUES
	(1, 'Matt'),
	(2, 'Esther');
```

성능적으로 본다면 정규화된 테이블을 추천
> 정규화된 컬럼은 컬럼명을 메타 정보로만 저장하지만<br>
> JSON 컬럼은 필드명을데이터 파일에 매번 저장해야하니 레코드 수가 많아지면 필드명이 차지하는 공간이 커짐

서버의 압축을 사용하면 디스크의 공간을 수일 수는 있지만 메모리 효율을 눂여주진 못함<br>
MySQL은 메모리에 압축된 페이지와 압축 해제된 페이지가 공존해야 해서 메모리 효율과 CPU 효율 모두 떨어뜨릴 수 있음

JSON으로 모두 저장하면 정수값 하나만 참조하더라도 JSON 컬럼에 저장된 도큐먼트를 모두 읽어야함

<br>

레코드가 가지는 속성들이 너무 상이하고 다양하지만 레코드 별로 선택적으로 값을 가지는 경우라면<br>
모든 속성에 대한 컬럼을 생성하는 것보다는 JSON 컬럼을 사용하는 것이 좋음

중요도가 낮은 것일 수록 좋음

> RDBMS 들은 작은 크기의 데이터 처리에 적합하도록 설계되었음<br>
> 그래서 큰 값을 저장하게 되면 예상했던 것보다 느린 성능을 보이기도함.<br>
> JSON 컬러만 안 뽑이도 성능이 달라지기도함
