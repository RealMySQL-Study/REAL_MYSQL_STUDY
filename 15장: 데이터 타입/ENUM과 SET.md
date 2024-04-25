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
  > ,ALGORITHM=COPY;
  > -- ,ALGORITHM=INPLACE;
  > -- 책에서는 INSTANT 로 되어있는데 그러면 에러가 떴음
  > -- 
  > ```
+ 

> [단점 참고](https://gompro.postype.com/post/8253823)
