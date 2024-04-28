# 15.4 ENUM과 SET

둘 다 문자열 값을 MySQL 내부적으로 숫자 값으로 매핑해서 관리하는 타입.

숫자나 문자열 값으로 상태를 나타내면 의미를 바로 파악하기 쉽지 않음.<br>
=> 이런 단점을 보완할 수 있는 타입

## 15.4.1 ENUM

ENUM은 테이블의 구조(메타 데이터)에 나열된 목록 중 하나의 값을 가질 수 있음

ENUM의 가장 큰 용도는 코드화된 값을 관리하는 것임

```sql
create table tb_enum(
	fd_enum ENUM('PROCESSING', 'FAILURE', 'SUCCESS')
);

INSERT INTO TB_ENUM VALUES ('PROCESSING'), ('FAILURE');
```
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/30790d8a-4bcf-47be-af80-57c7791b106c)



#### ENUM 타입의 특성
+ ENUM이나 SET 타입의 컬럼에 대해 숫자 연산을 수행하면 매핑된 문자열 값이 아닌 내부적으로 저장된 숫자 값으로 연산함<br>
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/2f02c9cd-9cee-4ba5-9b9e-ed3527074e4a)


+ 문자열처럼 비교하거나 저장할 수 있지만 실제로 값을 저장할 때는 문자열이 아니라 정수값을 사용<br>(문자열이 아니라 정수값으로 INSERT 해도 들어감)<br>
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/60a89286-8b91-4e1d-b38e-c15be5e293fe)<br>
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/bea3abdb-fbf2-49cd-b944-8db0981e8fec)

+ 최대 아이템 개수는 65,535개이며 아이템 개수가 255개 미만이면 1바이트를 사용하고 이상하면 2바이트를 사용
+ 매핑된 정수값은 테이블 정의에 나열된 순서대로 1부터 할당되며 빈문자열은 항상 0이 매핑

#### ENUM의 단점

+ 새로운 값을 추가해야 할 때 테이블의 구조를 변경해야됨
  > 5.6버전 전에는 ENUM 타입이 추가되면 언제나 테이블을 리빌드 해야했음<br>
  > 5.6버전부터는 새로 추가하는 아이템이 ENUM의 제일 마지막에 추가되는 형태라면 테이블의 구조(메타데이터) 변경만으로 완료됨
  > ```SQL
  > ALTER TABLE TB_ENUM
  > MODIFY FD_ENUM ENUM('PROCESSING', 'FAILURE', 'SUCCESS', 'REFUND')
  > ,ALGORITHM=INSTATN;
  > -- ,ALGORITHM=COPY;
  > -- ,ALGORITHM=INPLACE;
  > -- 책에서는 INSTANT 로 되어있는데 그러면 에러가 떴음
  > -- 아마도 lower_case_table_names 변수값이 1로 되어있어서 메타데이터를 변경하는 와중에
  > -- 대소문자를 구분해서 문제가 생긴것 같음.(테이블의 컬럼을 FD_ENUM 으로 변경하고 INSTANT 로 돌렸을때는 됐음)
  > -- lower_case_table_names 변수는 구축 초기에만 설정할 수 있어서 변경해서 테스트 해보지는 못함
  > ```
  > 순서 변경이나 중간에 아이템을 추가하려면 COPY 알고리즘에 읽기 잠금까지 필요
+ 

> [단점 참고](https://gompro.postype.com/post/8253823)


#### 정렬

ENUM 타입의 컬럼으로 정렬 시 문자열 값이 아닌 매핑된 코드 값으로 정렬됨

ENUM 타입 컬럼으로 정렬하지 않는 것이 좋지만 (문자열값으로)해야한다면 CAST() 함수를 통해 정렬할 수 밖에 없음

CAST() 함수를 사용하면 인덱스를 이용한 정렬을 사용할 수 없을 수도 있음

```SQL
SELECT FD_ENUM * 1 AS REAL_VALUE
	, FD_ENUM
FROM TB_ENUM
ORDER BY FD_ENUM;
```
> ![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/56211e78-110c-4a9c-bb79-a43ab0200f03)

```SQL
SELECT FD_ENUM * 1 AS REAL_VALUE
	, FD_ENUM
FROM TB_ENUM
ORDER BY CAST(FD_ENUM AS CHAR);
```
> ![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/c0a3be8e-691e-49fa-8020-e7b1da7456ea)


#### 장점

+ 테이블 구조에 정의된 코드 값만 사용할 수 있게 강제한다
+ (레코드 건수가 많아지면)데이터베이스 서버의 디스크 저장 공간의 크기를 줄여줌
  > 디스크의 데이터는 InnoDB 버퍼풀에 적재되어야 사용할 수 있는데<br>
  > 디스크의 데이터가 크면 메모리도 많이 필요해진다는 의미임
+ 디스크 사용량이 적으면 백업과 복구 시간을 줄일 수 있음

<br>

## 15.4.2 SET

SET 타입도 테이블 구조에 정의된 아이템을 정수값으로 매핑해서 저장하는 방식은 똑같음

SET은 하나의 컬럼에 1개 이상의 값을 저장할 수 있음

내부적으로 BIT-OR 연산을 거쳐 1개 이상의 선택된 값을 저장

각 아이템 값에 매핑되는 정수값은 1씩 증가되는 값이 아니라 2n의 값을 갖게 됨

아이템의 멤버 수가 8개 이하면 1바이트의 공간을 사용하고, 9~16개 이면 2바이트를 사용. 최대 8바이트 까지 사용

```sql
CREATE TABLE TB_SET(
	FD_SET SET('TENNIS', 'SOCCER', 'GOLF', 'TABLE-TENNIS', 'BASKETBALL', 'BILLIARD')
);
-- 여러 값을 저장할 때는 , 로 구분하여 하나의 문자열로 넣으면 됨
INSERT INTO TB_SET (FD_SET) VALUES ('SOCCER'), ('GOLF,TENNIS');

SELECT * FROM TB_SET;
```
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/a214bb3f-6c43-48e0-8181-0e67ee7ab9e7)

#### 검색과 INDEX 사용

```SQL
-- 검색 시에 FIND_IN_SET() 함수를 사용
SELECT * FROM TB_SET WHERE FIND_IN_SET('GOLF', FD_SET);
```

![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/746b4ac2-5e49-4e0a-80a5-530342e97567)

```SQL
-- 검색시 LIKE 도 사용 가능
SELECT * FROM TB_SET WHERE FD_SET LIKE '%GOLF%';
```
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/7627758d-118c-4932-bf34-d2af13dee43b)


SET 타입 컬럼은 동등비교를 하려면 컬럼에 저장된 순서대로 문자열을 나열해야만 검색 가능

SET 타입 컬럼에 인덱스가 있더라도 FIND_IN_SET 이나 LIKE 를 사용한 쿼리는 INDEX를 사용할 수 없음

```SQL
SELECT * FROM TB_SET WHERE FD_SET = 'TENNIS,GOLF';
```
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/6f6057ad-5650-4290-bc98-3603bb95d1ab)

```SQL
SELECT * FROM TB_SET WHERE FD_SET = 'GOLF,TENNIS';
```
![image](https://github.com/RealMySQL-Study/REAL_MYSQL_STUDY/assets/92290312/5fa0d377-ce32-4196-9165-a4fda3525c69)


만약 SET 타입 컬럼이 특정 값을 포함하고 있는 지를 검색하기 위해 FIND_IN_SET 함수를 자주 사용한다면<br>
SET타입 컬럼을 정규화해서 별도로 인덱스를 가진 자식 테이블을 생성하는 것이 좋다.

#### 추가 및 수정

ENUM과 마찬가지로 SET도 정의된 아이템 중간에 새로운 아이템을 추가하는 경우 테이블의 읽기 잠금과 리블드 작업이 필요함

```SQL
ALTER TABLE TB_SET 
MODIFY FD_SET SET('TENNIS', 'SOCCER', 'GOLF', 'TABLE-TENNIS', 'BASKETBALL', 'e-SPORTS','BILLIARD'),
ALGORITHM=COPY, LOCK=SHARED;
```

하지만 마지막에 새로운 아이템을 추가하는 작업은 INSTANT 알고리즘으로 메타정보만 변경하고 즉시 완료됨.

```SQL
ALTER TABLE TB_SET
MODIFY FD_SET SET('TENNIS', 'SOCCER', 'GOLF', 'TABLE-TENNIS', 'BASKETBALL','BILLIARD', 'e-SPORTS'),
ALGORITHM=INSTANT;
```

하지만 아이템의 개수가 8개를 넘어서 9개로 바뀔 때(저장공간의 크기가 바뀔때) 읽기 잠금과 테이블 리빌드가 필요

```SQL
ALTER TABLE TB_SET
MODIFY FD_SET SET('TENNIS', 'SOCCER', 'GOLF', 'TABLE-TENNIS'
				, 'BASKETBALL','BILLIARD', 'e-SPORTS', 'SCUBA-DIVING'
                , 'SWIMMING'),
ALGORITHM=INSTANT;

/*
11:21:04 ALTER TABLE TB_SET MODIFY FD_SET SET('TENNIS', 'SOCCER', 'GOLF', 'TABLE-TENNIS', 'BASKETBALL','BILLIARD', 'e-SPORTS', 'SCUBA-DIVING', 'SWIMMING'), ALGORITHM=INSTANT
Error Code: 1846. ALGORITHM=INSTANT is not supported. Reason: Cannot change column type INPLACE. Try ALGORITHM=COPY/INPLACE.	0.016 sec
*/

ALTER TABLE TB_SET
MODIFY FD_SET SET('TENNIS', 'SOCCER', 'GOLF', 'TABLE-TENNIS'
				, 'BASKETBALL','BILLIARD', 'e-SPORTS', 'SCUBA-DIVING'
                , 'SWIMMING'),
ALGORITHM=COPY, LOCK=SHARED;
```
