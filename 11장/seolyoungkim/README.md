# Introduction
- DDL(Data Definition Language) : 데이터베이스나 테이블의 구조를 변경하기 위한 문장 
- DML(Data Manipulation Language) : 데이터를 조작(읽기/쓰기)하기 위한 문장

# 11.1 쿼리 작성과 연관된 시스템 변수 
## 11.1.1 SQL 모드 
- `sql_mode` 시스템 변수 : MySQL이 SQL을 처리하는 방법을 제어하는 데 사용
  - SQL 작성과 결과에 영향을 미침  
  - 가능한 변경하지 않는 것이 좋음
- 기본값 
  - `ONLY_FULL_GROUP_BY` 
    - GROUP BY 절에 명시되지 않은 컬럼을 SELECT 절이나 HAVING 절에서 사용할 수 없도록 함 
      - 적용하지 않으면 사용할 수 있음 
  - `STRICT_TRANS_TABLES`
    - InnoDB와 같이 트랜잭션을 지원하는 스토리지 엔진에만 엄격한 모드를 적용  
    - Insert, Update, Delete 시에 잘못된 데이터가 발생하면 오류가 발생함 (select는 경고 발생)
      - 적용하지 않으면 경고만 발생 
  - `NO_ZERO_IN_DATE` & `NO_ZERO_DATE`
    - DATE 또는 DATETIME 컬럼에 잘못된 날짜를 저장하는 것을 불가능하게 함 
  - `ERROR_FOR_DIVISION_BY_ZERO`
  - `NO_ENGINE_SUBSTITUTION`


## 11.1.2 영문 대소문자 구분 
- OS에 따라 테이블명의 대소문자를 구분함 
  - Windows : 대소문자 구분하지 않음 
  - Unix 계열 OS : 대소문자 구분함
- `lower_case_table_names`: 이를 1로 설정하면 모두 소문자로만 저장되며, 대소문자를 구분하지 않게 해줌 
  - default는 0임 
- 가능한 경우 대문자 또는 소문자로만 통일해서 사용하자 

## 11.1.3 MySQL 예약어 
- https://dev.mysql.com/doc/refman/8.0/en/keywords.html 참고 
- 테이블을 생성할 때 예약어를 사용하면 오류가 발생함 
  - `로 테이블명을 감싸면 예약어를 사용할 수 있으나, 이렇게 하진 말자 

---

---

# 11.2 매뉴얼의 SQL 문법 표기를 읽는 법 
- `[]`: 해당 키워드나 표현식 자체가 선택 사항임 
- `|` : 앞과 뒤의 키워드나 표현식 중에서 단 하나만 선택해서 사용할 수 있음 
- `{}` : 괄호 내의 아이템 중에서 반드시 하나를 사용해야 함 
- `...` : 표현식이나 키워드를 반복해서 사용할 수 있음

---

---

# 11.3 MySQL 연산자와 내장 함수 
- 될 수 있으면 ANSI 표준을 따르는 것이 좋음

## 11.3.1 리터럴 표기법 문자열 
### 문자열 
- 작은 따옴표(`'`)로 감싸서 표현
  - MySQL은 `"` 사용 가능
- `''`는 문자열 내에서 작은 따옴표를 표현할 때 사용

### 숫자 
- 작은 따옴표(`'`)로 감싸지 않고 숫자 값을 입력 
- 주의할 점 
  - 숫자와 문자열은 서로 자동으로 타입 변환이 발생함 
  - MySQL은 숫자 타입을 우선시함
  - 예시 
    - number_col = '100' → 문자열 '100'을 숫자로 변환해서 비교
    - string_col = 100 → string_col의 값을 숫자로 변환해서 비교 → 인덱스 못탐 (or 비교가 실패 할수도 있음)
- 타입 변환이 발생하지 않도록 주의하자 

### 날짜 
- 정해진 형태의 날짜 포맷으로 표기하면 MySQL 서버가 자동으로 DATE나 DATETIME 값으로 변환함 
  - `'2023-11-23'` → DATE로 변환 

### 불리언 
- `BOOL`, `BOOLEAN`은 사실 `TINYINT` 타입에 대한 동의어일 뿐이다 (0 또는 1)
  - `TRUE`, `FALSE`와 비교하거나 값을 저장할 수 있다. 
    - `TRUE` → 1
    - `FALSE` → 0


## 11.3.2 MySQL 연산자 
### 동등 비교 
- `=` : 두 값이 동일한지 비교할 때 사용
  - `NULL`과 `NULL`, `NULL`과 다른 값을 비교하면 `NULL`을 반환함  
  - `NULL`과 비교할 때는 `IS NULL`을 사용해야 함
- `<=>` : NULL을 포함한 두 값이 동일한지 비교할 때 사용
  - MySQL에서 제공하는 연산자 

### 부정 비교 
- `<>` 또는 `!=` 사용 

### NOT 연산자
- `NOT` : TRUE, FALSE 연산의 결과를 반대로 만드는 연산자 
  - `NOT (a = b)`와 `(a <> b)`는 동일함

### AND, OR 연산자
- `AND`, `OR` : 논리 연산자 
  - `AND`는 `OR`보다 우선순위가 높음 
  - `AND`와 `OR`를 같이 사용할 때는 괄호를 사용해서 우선순위를 명확하게 해주는 것이 좋음

### 나누기(`/`, `DIV`)와 나머지(`%`, `MOD`) 연산자 
- `/` : 나누기 연산자 
- `DIV` : 몫을 구하는 연산자
- `%` or `MOD` : 나머지 연산자

### REGEXP 연산자
- `REGEXP` : 정규 표현식을 사용해서 문자열을 비교할 때 사용 
  - `LIKE`와 달리 정규 표현식을 사용할 수 있음 
  - `RLIKE`는 `REGEXP`와 동일함
- 인덱스 레인지 스캔 사용 불가 

### LIKE 연산자
- 어떤 상수 문자열의 유무를 판단하는 연산자 
- `%` : 0 or 1개 이상의 모든 문자에 일치 (문자 내용 관계 x)
- `_` : 정확히 1개의 문자에 일치 (문자 내용 관계 x)
- `a%` or `a_` : 인덱스 레인지 스캔 사용 가능. 인덱스는 앞쪽을 비교하기 때문에, 앞이 고정되어 있어야 함 
  - `%a` or `_a` : 인덱스 레인지 스캔 사용 불가

### BETWEEN 연산자
- `BETWEEN` : 두 값 사이에 있는지 확인하는 연산자 
  - `a BETWEEN b AND c` : `a >= b AND a <= c`와 동일함
  - `NOT BETWEEN` : `BETWEEN`의 반대
- 인덱스 사용시 주의점 
  ```mysql
  -- pk = (dept_no, emp_no)
  SELECT * FROM dept_emp WHERE dept_no BETWEEN 'd003' AND 'd005' AND emp_no = 10001;
  ```
  - 위 쿼리는 d003부터 d005까지 모두 조회한다. `emp_no`는 비교 범위를 줄이는 역할을 하지 못한다. 
    - 실제 조회되는 데이터가 1건이라고 해도, d003~d005까지의 모든 데이터를 스캔한다.
  - 위 쿼리는 아래와 같이 개선할 수 있다.
    ```mysql
    SELECT * FROM dept_emp WHERE dept_no IN ('d003', 'd004', 'd005') AND emp_no = 10001;
    ```
    - 동등 비교를 수행함으로써 모든 범위를 스캔하지 않게 만든다. 
  - 인덱스 앞쪽에 있는 컬럼의 선택도가 떨어질 때는 IN으로 변경하는 것이 성능을 개선할 수도 있음.
  - ![img.png](img.png)


### IN 연산자
- `IN` : 여러 개의 값에 대해 동등 비교 연산을 수행 
  - 여러 번의 동등 비교로 실행 → 일반적으로 빠르게 처리 
- `IN (?, ?, ?)` : 상수가 사용된 경우
- `IN (SELECT ...)` : 서브쿼리가 사용된 경우
- `NOT IN` : 부정형 비교이기 때문에 인덱스 풀 스캔을 하는 경우가 많음


## 11.3.3 MySQL 내장 함수
### NULL 값 비교 및 대체 
- `IFNULL(expr1, expr2)`
  - expr1: NULL인지 아닌지 비교하려는 컬럼 혹은 표현식. NULL이 아닐 경우 expr1을 반환함
  - expr2: expr1이 NULL인 경우 대체할 값이나 컬럼. NULL이 아닐 경우 expr2를 반환함 
- `ISNULL(expr)` : expr이 NULL이면 1을 반환하고, 그렇지 않으면 0을 반환


### 현재 시각 조회 
- `NOW()` : 현재 시각을 반환함
  - 하나의 SQL 문장 내의 NOW() 함수는 모두 같은 값을 가짐 
- `SYSDATE()` : 현재 시각을 반환함
  - 하나의 SQL 문장 내에서도 호출 시점에 따라 결괏값이 달라짐 
  - 레플리카 서버에서 안정적으로 복제되지 못함 
  - 인덱스를 효율적으로 사용하지 못함 
    - 호출 시점에 따라 값이 달라지기 때문에, 상수가 아님. 인덱스 스캔 시에도 매번 비교되는 레코드마다 함수가 실행돼야 함.
  - 그러니 왠만하면 사용 ㄴㄴ


### 날짜와 시간 포맷 
- `DATE_FORMAT(date, format)` : 날짜를 원하는 포맷으로 변환함
  - date: 날짜 값
  - format: 날짜 포맷 (ex: `%Y-%m-%d`)
- `STR_TO_DATE(str, format)` : 문자열을 날짜로 변환함
  - str: 날짜 형식의 문자열
  - format: 날짜 포맷


### 날짜와 시간 연산 
- `DATE_ADD(date, INTERVAL expr unit)` : 날짜에 일정 기간을 더함
  - date: 날짜 값
  - expr: 더하고자 하는 기간
  - unit: 기간의 단위 (ex: DAY, MONTH, YEAR)
- `DATE_SUB(date, INTERVAL expr unit)` : 날짜에 일정 기간을 뺌
  - date: 날짜 값
  - expr: 빼고자 하는 기간
  - unit: 기간의 단위 (ex: DAY, MONTH, YEAR)


### 타임스탬프 연산
- `UNIX_TIMESTAMP()` : 현재 시각을 타임스탬프로 반환함
  - `1970-01-01 00:00:00`부터 현재 시각까지의 경과된 초를 반환
  - 4바이트 숫자 타입이기 때문에 `1970-01-01 00:00:00` ~ `2038-01-19 03:14:07`까지만 표현 가능
- `FROM_UNIXTIME(unix_timestamp)` : 타임스탬프를 날짜로 변환함


### 문자열 결합 
- `CONCAT(str1, str2, ...)` : 여러 개의 문자열을 결합함
  - str1, str2, ...: 결합할 문자열


### 값의 비교 및 대체 
- `CASE WHEN ... THEN ... END`
  - `SWITCH` 구문과 같은 역할을 함 


---

---

# 11.4 SELECT
## 11.4.1 SELECT 절의 처리 순서 
```mysql
SELECT s.emp_no, COUNT(DISTINCT e. first_name) AS cnt  -- SELECT 절
FROM salaries s  -- FROM 절
INNER JOIN employees e ON e.emp_no=s. emp_no
WHERE s.emp_no IN (100001, 100002)  -- WHERE 절
GROUP BY s.emp_no  -- GROUP BY 절
HAVING AVG(s.salary) > 1000  -- HAVING 절
ORDER BY AVG(s.salary)  -- ORDER BY 절
LIMIT 10;  -- LIMIT 절
```
- 각 쿼리 절의 실행 순서 
  - ![img_1.png](img_1.png)
  - 인덱스를 이용해 처리할 때는 GROUP BY, ORDER BY 자체가 불필요하므로 생략됨 
- GROUP BY 절이 없이 ORDER BY 절만 사용된 쿼리에서는 아래와 같이 실행 순서가 적용될 수 있음 
  - ![img_2.png](img_2.png)

## 11.4.2 WHERE 절, GROUP BY 절, ORDER BY 절의 인덱스 사용 
### 인덱스를 사용하기 위한 기본 규칙 
- 인덱스된 컬럼 값 자체를 변환하지 않고 그대로 사용해야 함 
- WHERE 절의 비교 조건에서, 두 비교 대상 값은 데이터 타입이 일치해야 함  
  - 저장이든 비교든 타입은 일치시키자. 

### WHERE 절의 인덱스 사용 
- 동등 비교 or IN 으로 구성된 조건에 사용된 컬럼들이 인덱스의 컬럼 구성과 좌측에서 비교했을 때 얼마나 일치하는가에 따라 달라짐
- ![img_3.png](img_3.png)
  - 위 예제는 AND 조건에 대한 예제이다. 
  - 인덱스 순서와 WHERE 조건절의 순서는 실제 인덱스 사용 여부와 무관함 
  - COL3이 범위 비교 조건으로 사용됐기 때문에, COL4는 체크 조건으로 사용된다. 
    - 결국, 인덱스 컬럼 순서와 비교 조건이 중요하지, WHERE 조건절의 순서는 중요하지 않다.
 
- WHERE절에 OR가 있으면 주의해야 함 
  - AND로 연결되면 읽어와야 할 레코드 건수를 줄이는 역할을 하지만,
  - OR로 연결되면 읽어서 비교해야 할 레코드가 더 늘어나기 때문 

### GROUP BY 절의 인덱스 사용
- GROUP BY 절에 명시된 컬럼의 순서가 인덱스를 구성하는 컬럼의 순서와 같으면 인덱스를 사용할 수 있음
- 인덱스를 구성하는 컬럼 중, 뒤쪽에 있는 컬럼은 GROUP BY 절에 명시되지 않아도 인덱스를 사용할 수 있음
  - 즉, 다중 컬럼 인덱스 중에서 일부만 명시되어도 되지만, 뒤쪽의 인덱스만 제외될 수 있음   
  - 따라서, 인덱스의 앞쪽에 있는 컬럼은 명시되지 않으면 인덱스를 사용할 수 없음
- GROUP BY 절에 명시된 컬럼이 하나라도 인덱스에 없으면 전혀 인덱스를 이용할 수 없음
- 예제 
  - ex1: 인덱스가 (a, b, c)일 때, GROUP BY a, b, c, d, e, f는 인덱스를 사용할 수 없음
  - ex2: 인덱스가 (a, b, c)일 때, GROUP BY b, c는 인덱스를 사용할 수 없음
  - ex3: 인덱스가 (a, b, c)일 때, GROUP BY a, c는 인덱스를 사용할 수 없음
  - ex4: 인덱스가 (a, b, c)일 때, GROUP BY a는 인덱스를 사용할 수 있음
  - ex5: 인덱스가 (a, b, c)일 때, GROUP BY a, b는 인덱스를 사용할 수 있음
  - ex6: 인덱스가 (a, b, c)일 때, GROUP BY a, b, c는 인덱스를 사용할 수 있음

### ORDER BY 절의 인덱스 사용
- GROUP BY와 상당히 흡사함 
- 정렬되는 각 컬럼의 ASC, DESC 옵션이 인덱스와 같거나 정반대인 경우에만 사용 가능 

### WHERE 절과 ORDER BY(또는 GROUP BY) 절의 인덱스 사용
