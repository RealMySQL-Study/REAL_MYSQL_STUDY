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





































