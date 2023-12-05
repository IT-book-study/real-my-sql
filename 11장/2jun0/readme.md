# 11.1 쿼리 작성과 연관된 시스템 변수

## 11.1.2 영문 대소문자 구분

- MySQL 서버는 운영체제에 따라 테이블명의 대소문자를 구별 할 수도 있고 / 아닐 수도 있음.
- 이는 테이블을 디렉터니라 파일로 매핑하기 때문임. (OS의 대소문자 구분 여부였던 것임!)
- 따라서 가!능!한! 테이블 대문자 또는 소문자로 통일하자.

# 11.3 MySQL 연산자와 내장 함수

## 11.3.1 리터럴 표기법 문자열

### 문자열

- SQL "표준"에서 문자열은 항상 '를 이용한다.

### 숫자

- WHERE 절에 string_column=123 이런 경우, 타입 변환이 이루어진다.
- 숫자가 우선이라 string_column이 숫자로 변환되니, 숫자값은 숫자 타입의 칼럼에만 저장하자.

### 날짜

- 날짜 타입은 문자열로 해도 된다.
  - `SELECT \* FROM dept_emp WHERE from_date='2023-09-14'`
  - DATE_FORMAT? 이런 이상한 함수 쓰지 말자. 퉤퉤

### 불리언

- BOOL이나 BOOLEAN은 TINYINT다.

## 11.3.2 MySQL 연산자

### LIKE

- `a%`, `a_`는 인덱스 레인지 스캔 활용 가능
- `%a`, `_a`는 인덱스 레인지 스캔 못씀

### BETWEEN

- BETWEEN은 범위 검색을 위한 것.
- 반면에 IN은 동등 비교 연산과 같다.
- 따라서, (index_a, index_b)와 같은 인덱스를 사용할때는 IN이 더 좋을 수 있다. (범위검색은 index_a먼저 검색한 뒤 index_b를 검색하기 때문)

### IN

- NOT IN은 부정형 비교라서 인덱스 풀 스캔으로 처리한다.

# 11.4 SELECT

## 11.4.2 WHERE 절과 GROUP BY 절, ORDER BY 절의 인덱스 사용

### 인덱스를 사용하기 위한 기본 규칙

- 인덱스된 칼럼의 값 자체를 변환하지 않고 그대로 사용해야 함.
  - 인덱스 값을 가공하면 안된다.
    - ex) `SELECT * FROM salaries WHERE salary * 10 > 150000;`
  - 문자열 컬럼에 숫자 리터럴을 사용해도 문제가 된다.
    - ex) `SELECT * FROM tb WHERE str_col = 12;`
    - 인덱스 풀 스캔 한 뒤, 모든 인덱스를 숫자로 형변환한다..

### WHERE 절의 인덱스 사용

- WHERE 절에서 동등 비교 / IN 조건에 사용된 칼럼들이 얼마나 일치하는가에 따라 달라짐.
- WHERE 절의 조건 순서를 어떻게 하든, 옵티마이저는 최적화 해준다.
- OR 연산자를 이용해 인덱스 + 일반 컬럼의 조건을 사용하면 인덱스 최적화가 없어진다. (풀 테이블 스캔 한번으로 해결하려고 한다.)
  - ex) `SELECT * FROM tb WHERE index_a='hi' OR col_1='hello'`

### GROUP BY 절의 인덱스 사용

- GROUP BY 절에 명시된 칼럼의 순서가 인덱스를 구성하는 칼럼의 순서와 같으면 인덱스 사용가능.
  - 인덱스가 (a, b, c)면, (a, b), (a,b,c) 사용가능
  - (a, c), (a, b, c, d) 사용 불가능
- WHERE 절에 인덱스를 구성하는 일부컬럼의 동등 비교가 있으면 해당 칼럼은 생략 가능하다.
  - 인덱스가 (a, b, c)일때, `WHERE a='상수' GROUP BY b, c` 사용가능
  - 인덱스가 (a, b, c)일때, `WHERE a='상수' AND b='상수' GROUP BY c` 사용가능

### ORDER BY 절의 인덱스 사용

- 정렬되는 각 칼럼의 오름차순 및 내림차순 옵션이 인덱스와 같거나 정반대 인 경우에만 사용가능.
  - 인덱스가 (a, b, c)면, `ORDER BY a, b, c`, `ORDER BY a DESC, b DESC, c DESC` 가능
  - 인덱스가 (a, b, c)면, `ORDER BY a, c`, `ORDER BY a, b DESC, c`, 불가능

### WHERE + (ORDER BY | GROUP BY)의 인덱스 사용

- 다음과 같은 3가지 방식 중 한가지로 인덱스를 이용함
  - WHERE 절과 ORDER BY 절이 동시에 같은 인덱스를 사용 (BEST)
  - WHERE 절만 인덱스를 이용
    - ORDER BY는 filesort 방식으로 처리된다.
    - WHERE 절에서 필터링된 레코드 건수가 적을때 좋다.
  - ORDER BY 절만 인덱스 이용
    - ORDER BY 절의 순서대로 인덱스를 읽으면서 레코드 한 건씩 WHERE 절의 조건에 일치하는지 비교한다.
    - 아무 많은 레코드를 정렬해야 할때 이렇게 하기도 함.

### GROUP BY + ORDER BY의 인덱스 사용

- 하나의 인덱스를 사용해서 처리하려면, GROUP BY와 ORDER BY에 명시된 칼럼의 순서와 내용이 "모두 같아야 한다."
  - ex) `GROUP BY col_1, col_2 ORDER BY col_1, col_2`
- 둘 중에 하나가 인덱스를 사용하지 못하면 모두 인덱스를 이용하지 못한다. (연대 책임)
  - ex) `GROUP BY col_1, col_3 ORDER BY col_1, col_2`
- MySQL 8.0 부터는 GROUP BY가 정렬까지 보장하지 않는다. 정렬이 필요하면 ORDER BY도 같이 쓰자.

### WHERE + ORDER BY + GROUP BY

- 다음과 같은 3개의 질문을 기본으로 해서 판단한다.
  1. WHERE 절이 인덱스를 사용할 수 있나?
  2. GROUP BY 절이 인덱스를 사용할 수 있나?
  3. GROUP BY 절과 ORDER BY 절이 동시에 인덱스를 사용할 수 있는가?
- 그냥 아래 참고하면 정리가 된다.

  ```mermaid
  graph LR
  A[WHERE]-->|Yes| B[GROUP BY]
  A-->|No| C[GROUP BY]

  B-->|Yes| D[ORDER BY]
  B-->|No| F2[WHERE만 인덱스 사용]
  D-->|Yes| F1[모두 인덱스 사용]
  D-->|No| F2

  C-->|Yes| E[ORDER BY]
  C-->|No| F4[인덱스 못씀]
  E-->|Yes| F3[GROUP BY + ORDER BY 인덱스 사용]
  E-->|No| F4
  ```

### WHERE 절의 비교 조건 사용 시 주의사항

- 인덱스에 NULL도 넣을 수 있다. 근데, 이건 표준이 아님!

  - SQL 표준에서 NULL는 비교할 수 없는 값이다.
  - 그래서 NULL과 연산하거나 비교하면 **결과도 항상 NULL**이 나온다.
  - NULL인지 확인하고 싶다면 `IS NULL`, `<=>`연산자를 사용하자.

- DATE vs DATETIME

  - DATETIME이 우선이다. DATE가 DATETIME으로 변환된다.
  - NOW 주의, DATE만 비교하고 싶으면 형변환 해줘야 한다.
    - `SELECT * FROM t WHERE date_col>DATE(NOW())`

- DATETIME / TIMESTAMP

  - TIMESTAMP는 숫자 값에 불과함으로, DATETIME 칼럼과 비교하려 들지 말자.
  - 형 변 환! 이 세글자를 꼭 기억해죠

- Short-Circuit Evaluation

  - 선행 표현식에 결과에 따라 후행 표현식을 평가할지 결정하는 최적화
  - 의사코드

    ```python
    in_transaction: bool

    # in_transaction이 False라면 뒤 함수는 실행되지 않는다.
    if in_transaction and has_modified():
        commit()
    ```

  - WHERE 절도 이런 최적화를 이용한다. 조건의 순서에 주의하자.

## DISTINCT

- DISTINCT 남용하지 말자!

### LIMIT n

- DISTINCT에 LIMIT을 붙이면 최적화가 진행된다.
  - `SELECT DISTINCT col FROM tb LIMIT 0, 10;`
  - 유니크한 그룹을 만들기 위해 풀 테이블 스캔을 한다.
  - 그 동시에 중복 제거 작업을 하면서 임시 테이블에 넣는다.
  - 임시테이블에 LIMIT 건수 만큼 채워지면 작업을 종료한다.
- LIMIT의 인자가 너무 클때, WHERE 절을 넣어서 위치를 찾아주자.

  - before `SELECT * FROM tb ORDER BY col LIMIT 1000000000, 10`
  - after `SELECT * FROM tb WHERE col>=1000000000 ORDER BY col LIMIT 0, 10`

## COUNT

- COUNT(\*)쿼리에 ORDER BY 절을 사용하면 옵티마이저가 무시한다. (애초에 쓴 사람이 바보)
- COUNT(col)은 col이 NULL이 아닌 레코드 건수를 반환한다.
- 게시물 목록을 구현할때 주의사항
  ```sql
  -- // 게시물 건수 확인
  -- // 인덱스를 이용하면 커버링 인덱스로 빠른 처리 가능하다.
  SELECT COUNT(*) FROM articles WHERE board_id=1
  -- // 하지만 여러 조건이 추가되면 오래걸릴 수 있다.
  SELECT COUNT(*) FROM articles WHERE board_id=1 AND deleted=0 AND hidden=0
  -- // 그냥 전체건수는 보여주지 말자!
  ```
  -

## JOIN

### 드라이빙 vs 드리븐

```sql
SELECT *
FROM employees e, dept_emp de
WHERE e.emp_no=de.emp_no
```

- 두 테이블 칼럼에 모두 인덱스 있음
  - 통계정보 활용, 처리후 레코드 건수가 적은 쪽이 드라이빙
- 한 테이블만 인덱스 있음
  - 인덱스 없는 쪽을 드라이빙 테이블로 선택
  - 이렇게 해야 인덱스를 활용할 수 있다. (역으로 하면 드리븐 테이블을 풀 태이블 스캔 하게 된다.)
    ![Alt text](image.png)
- 모두 인덱스 없음
  - 레코드 건수가 적은 걸 선택
  - 풀 태이블 스캔은 막을 수 없다.
- 주의!
  - 조인 조건의 두 칼럼의 타입이 다르면 풀 테이블 스캔 하게 된다.

### OUTER JOIN 주의사항

```sql
SELECT *
FROM employee e
LEFT JOIN dept_emp de ON de.emp_no=e.emp_no
LEFT JOIN departments d ON d.dept_no=de.dept_no AND d.dept_name='Development';
```

- 옵티마이저가 조인 순서를 최적화 할수 없다!

  - employees를 항상 드라이빙 테이블로 선택할 수 밖에 없다.

- OUTER로 조인되는 테이블에 WHERE 조건을 쓰면 INNER JOIN으로 바뀐다.
  ```sql
  -- // 조인되는 테이블에 WHERE 조건을 써버렸다!
  SELECT *
  FROM em e
    LEFT JOIN demanger mgr ON mgr.e_no=e.e_no
  WHERE mgr.e_no='d001';
  -- // 옵티마이저가 이렇게 바꾼다. mgr.e_no가 NULL일 수 없기 때문.
  SELECT *
  FROM em e
    INNER JOIN demanger mgr ON mgr.e_no=e.e_no
  WHERE mgr.e_no='d001';
  -- // 조인 조건에 넣어줍시다.
  SELECT *
  FROM em e
    LEFT JOIN demanger mgr ON mgr.e_no=e.e_no AND mgr.e_no='d001';
  ```

### JOIN + FOREIGN KEY

- Q. 두 테이블을 조인 하고 싶은데요 외래키를 꼭 만들어야 하나요?
- A. 아니오, 외래키를 생성하는 주 목적은 데이터의 무결성을 보장하기 위해서 입니다.
  - 데이터 모델링을 할때는 관게를 그려넣습니다.
  - 하지만 그 데이터 모델을 DB에 생성할때는 외래키를 안쓸때가 더 많아요.
- Q. 네? 무슨소리인지 하나도 모르겠어요. 다시 설명해주세요 외래키 쓰라고 하던데..
- A. (...님이 채팅방을 나갔습니다.)

### Delayed JOIN

- GROUP BY / ORDER BY를 먼저 하고 JOIN하는게 더 이득이다!
  - 조인은 대체로 하면 할수록 레코드 건수가 늘어나기 때문이다.

```sql
-- // 서브 쿼리로 명시적으로 GROUP BY / ORDER BY를 수행한 임시 테이블을 만들고, 조인을 한다.
-- // 책에서는 서브 쿼리로 했지만.. 그냥 이러지 말고 쿼리를 두번 날리는게 좋을듯
SELECT e.*
FROM
  (SELECT s.emp_no
   FROM salaries s
   GROUP BY s.emp_no
   ORDER BY SUM(s.salary) DESC
   LIMIT 10) x,
  employees e
WHERE e.emp_no=x.emp_no
```

### 조인과 정렬

- 네스티드-루프 방식의 조인은 드라이빙 테이블에서 읽은 레코드의 순서가 유지된다.
- 하지만 해시 조인을 사용하면, 그렇지 않을 수 있다.
- 그냥 정렬이 필요하면 ORDER BY를 해주자.

## ORDER BY

- 인덱스가 아닌 칼럼에 ORDER BY를 사용하면 filesort를 사용한다는 그런 이야기..

## CTE(Common Table Expression)

- 임시테이블로서, 재귀적 반복 실행 여부를 기준으로 Non-recursive / Recursive CTE로 구분된다.
- 하하하 미안해요 패스

## 잠금을 사용하는 SELECT

- InnoDB에선 레코드를 SELECT할때 잠금을 사용하지 않는다. (잠금 없는 읽기)
- 하지만 레코드를 잠그고 싶다면 아래와 같은 방법이 있다.
  - `FOR SHARE`: 읽어들인 레코드를 읽기 잠금한다.
  - `FOR UPDATE`: 읽어들인 레코드를 쓰기 잠금한다.
- 주의 상황
  - 잠금되었어도 `FOR SHARE / UPDATE`절을 가지지 않는 단순 SELECT 쿼리는 아무 대기없이 실행된다.

### 잠금 테이블 선택

- 쿼리에 등장하는 모든 테이블을 잠금하지 않고 일부만 잠그고 싶다?
  - OF를 붙이자. `FOR UPDATE OF tb`

### NOWAIT & SKIP LOCKED

- NOWAIT: 잠금이 걸려있다면 에러를 반환하며 쿼리가 종료됨
- SKIP LOCKED: 잠금이 걸려있다면 잠금이 걸리지 않은 레코드만 가져온다.
  - 쿠폰 발급 서비스에 유용하다고 함.
  - 이해는 했지만.. 잘 모르겠다!

# 11.5 INSRERT

## 11.5.1 고급 옵션

- 이걸 써도 될까?

### INSERT IGNORE

- 유니크 칼럼에 중복이 생겨도 오류를 경고 수준으로 내리고 무시한다.
- 데이터 타입이 일치하지 않아도 무시

### INSERT ... ON DUPLICATED KEY UPDATE

- 유니크 인덱스의 중복이 발생하면 UPDATE를 수행하게 해준다.
- REPLACE는 DELETE + INSERT의 조합이라서 다르다. (쓰지않기로)

## 11.5.2 LOAD DATA 주의 사항

- LOAD DATA 명령은 MySQL 엔진과 스토리지 엔진의 호출 횟수를 최소화하고, 스토리지 엔진이 직접 데이터를 적재함.
  - INSERT 보다 매우 빠르다.
- 근데 단점이 있다.
  - 단일 스레드로 실행
    - 적재해야 할 데이터 파일이 너무 크면 시간이 오래걸릴 수 있고, DB서버의 성능이 전체적으로 떨어질 수 있다.
  - 단일 트랜잭션으로 실행
    - LOAD DATA 문장이 시작한 시점부터 언두 로그가 삭제되지 못하는 이슈가 있음.
    - 레코드를 읽는 다른 쿼리들이 더 오래걸리게 된다.
- 데이터 파일을 나눠서 LOAD DATA명령 여러개로 처리하자!

### 11.5.3 성능을 위한 테이블 구조

- 대부분의 INSERT 성능 이슈는 테이블 설계를 잘 못해서 일어난다.
  - 하지마!: INSERT를 튜닝할 생각
  - 해!: 테이블 구조를 잘 설계할 생각

### 대량 INSERT의 성능

- 하나의 INSERT 문장으로 대량의 레코드를 INSERT한다면 미리 PK를 기준으로 정렬해서 INSERT 문장을 구성하는게 성능에 좋다.
  - PK 클러스터링 인덱스 페이지를 랜덤한 위치에서 읽어와야 하기 때문에 버퍼 풀을 잘 활용하지 못하기 때문이다.
  - 반면에 PK가 정렬된 순서로 INSERT를 하게 되면, 항상 마지막 페이지만 참조하게 되므로 개이득이다.
- 또, 아래와 같은 이유 때문에 성능에 문제가 생긴다.
  - 세컨더리 인덱스가 많을 수록
  - 테이블이 클 수록

### Auto-Increment

- Auto-Increment는 SELECT보다는 INSERT에 최적화된 방법이다.
  - 단조 증가라서 항상 마지막 페이지만 참조하게 된다.
- innodb_autoinc_lock_mode 옵션
  - 0: 항상 AUTO-INC 잠금을 건다.
    1. 한번에 한개씩
    2. 연속된 번호를 보장
  - 1(Consecutive mode): 한 건씩 INSERT하는 쿼리에서는 뮤텍스를 이용해 가볍게 처리한다.
    - 여러개를 INSERT 하면, AUTO-INC 잠금을 걸고, 필요한 만큼의 값을 한꺼번에 가져와 사용한다.
    1. 한번에 여러개씩
    2. 연속된 번호를 보장
  - 2(Interleaved mode): AUTO-INC는 더 이상 사용하지 않는다.
    - MySQL의 기본값
    - 소스 서버와 레플리카 서버의 자동 증가값이 동기화 되지 않을 수 있다.
    1. 잠금 없음.
    2. 연속된 번호 보장 안함

## 11.6 UPDATE와 DELETE

### 11.6.1 UPDATE ... ORDER BY ... LIMIT n

- 특정 칼럼으로 정렬해서 상위 몇 건만 수정하는 작업이다.
  - 너무 많은 레코드를 변경 및 삭제하는 작업은 서버에 과부하를 유발할 수 있다.
  - 이때, LIMIT을 이용해 나눠서 처리할 수 있다.
- 바이너리 로그 포맷이 ROW일때는 괜찮지만, STATEMENT일때는 경고가 나온다.
  - ORDER BY로 정렬하더라도 "중복된 값의 순서"가 소스 서버와 레플리카 서버에서 차이를 보일 수 있기 때문이다.
  - 물론 유니크하면 상관없다. (그래도 경고는 나옴)

### 11.6.2 JOIN UPDATE

- 조인된 결과 레코드를 변경 & 삭제 하는 작업이다.
  - 조인된 테이블 중에서 특정 테이블의 칼럼값을 다른 테이블의 칼럼값으로 수정하고 싶을때 쓸 수 있다.
  - 양쪽 테이블에 공통으로 존재하는 레코드만 수정할때도 사용할 수 있다.
- 읽기 참조만 되는 테이블은 읽기 잠금, 칼럼이 변경되는 테이블은 쓰기 잠금이 걸린다.
  - 웹 서비스 같은 OLTP 환경에서는 데드락을 유발할 수도 있다.
  - 배치 프로그램이나 통계용 UPDATE 문장에서는 유용하다고 함.
- 예제
  ```sql
  -- // t2의 col 칼럼 값을 t1의 col 칼럼에 복사함
  UPDATE tb1 t1, tb2 t2
    SET t1.col = t2.col
  WHERE t1.no = t2.no;
  ```
- 주의!
  - JOIN이라서 어떤 테이블을 드라이빙 테이블로 할지에 따라서 성능이 좌우된다. -> 실행계획을 확인하자.
  - JOIN UPDATE 문장에서는 GROUP BY나 ORDER BY 절을 사용할 수 없다.

### 11.6.3 여러 레코드 UPDATE

- 원래 UPDATE로 여러 레코드를 수정할때 동일한 값으로만 업데이트 할 수 있다.
  - ex) `UPDATE departments SET emp_count=10`
  - ex) `UPDATE departments SET emp_count=emp_count + 10`
- MySQL 8.0부터는 레코드별 다른 값을 업데이트 해줄 수 있다.
  ```sql
  -- // VALUE ROW(...), ROW(...)는 문장 내에서 임시 테이블을 생성하는 효과를 낸다.
  -- // 여기선, (1, 1), (2, 4)를 가지는 2건의 레코드를 가지는 임시 테이블을 생성했다.
  UPDATE user_level ul
    INNER JOIN (VALUES ROW(1, 1), ROW(2, 4)) new_user_level (user_id, user_lv)
      ON new_user_level.user_id=ul.user_id
    SET ul.user_lv=ul.user_lv + new_user_level.user_lv;
  -- // 이게 뭐야 -_-;;
  ```

### 11.6.4 JOIN DELETE

- JOIN DELETE 문장에서는 DELETE할 레코드의 테이블을 명시해야 한다.

  ```sql
  DELETE e -- // 이렇게!
  FROM employees e, dept_emp de, department d
  WHERE e.emp_no=de.emp_no AND de.dept_no=d.dept_no AND d.dept_no='d001';

  DELETE e, de, d -- // 여러개도 가능함
  FROM employees e, dept_emp de, department d
  WHERE e.emp_no=de.emp_no AND de.dept_no=d.dept_no AND d.dept_no='d001';
  ```

## 11.7 스키마 조작(DDL)

- DDL: 서버의 모든 오브젝트를 생성하거나 변경하는 쿼리

### 11.7.1 온라인 DDL

- 온라인 DDL: 스키마를 변경하는 와중에도 다른 커넥션에서 데이터를 변경하거나 조회하는 작업을 가능하게 해준다.
- 버전 이슈
  - MySQL 5.5 이전 버전까지는 MySQL서버에서 테이블의 구조를 변경하는 동안에는 다른 커넥션에서 DML을 실행할 수 없었다.
  - MySQL 8.0 부터는 "온라인 DDL"을 사용할 수 있게 되었다.

#### 온라인 DDL 알고리즘

`ALTER TABLE`을 실행하면 아래와 같은 순서로 스키마 변경에 적합한 알고리즘을 찾는다.

1. ALGORITHM=INSTANT로 스키마 변경이 가능한가?
2. ALGORITHM=INPLACE로 스키마 변경이 가능한가?
3. ALGORITHM=COPY 선택

알고리즘은 다음과 같다.

- INSTANT
  - 테이블의 데이터는 변경하지 않고, 메타데이터만 변경한다.
  - 스키마 변경 도중, 쓰기 잠금이 걸린다.
  - 변경이 굉장히 빠르게 처리되기 때문에 다른 커넥션의 성능에 큰 영향이 없다.
- INPLACE
  - 임시 테이블로 데이터를 복사하지 않고 스키마 변경을 실행
  - 레코드의 복사는 없지만, 모든 레코드를 리빌드 한다. (테이블 크기에 따라 시간 소요될 수 있음)
  - 최초 시작 시점과 마지막 종료 시점에만 쓰기 잠금이 걸린다.
  - 잠금 시간은 매우 짧아서 다른 커넥션에 영향이 거의 없다.
- COPY
  - 변경된 스키마를 적용한 임시 테이블을 만들고, 레코드를 모두 복사한다.
  - 읽기 잠금이 걸리고, DML(INSERT, UPDATE, DELETE)는 실행 불가능 하다. # 대기 한다는 것인가? 실행 불가능 하다는 것인가?

#### 잠금 수준

온라인 DDL 명령은 잠금 수준도 함께 명시할 수 있다.  
아래와 같이! (INSTANT는 명시할 수 없다.)

```sql
ALTER TABLE salaries CHANGE to_date end_date DATE NOT NULL, ALGORITHM=INPLACE, LOCK=NONE;
```

종류는 다음과 같다.

- NONE: 무잠금
- SHARED: 읽기 잠금, 쓰기 불가능
- EXCLUSIVE: 쓰기 잠금, 읽기 쓰기 불가능

#### 온라인 DDL을 지원하는지 확인하는 법

- 메뉴얼을 참고한다.
- `ALTER TABLE`문장에 `LOCK`과 `ALGORITHM`을 명시해서 에러가 나오는지 확인해본다.

  1. ALGORITHM=INSTANT
  2. ALGORITHM=INPLACE, LOCK=NONE (여기까지 무 잠금)
  3. ALGORITHM=INPLACE, LOCK=SHARED (여기서부터는 잠금이 걸린다. 서비스를 멈추고 스키마 점검 ㄱ)
  4. ALGORITHM=COPY, LOCK=SHARED
  5. ALGORITHM=COPY, LOCK=EXCLUSIVE

  ```sql
  -- ALGORITHM=INSTANT
  sql> ALTER TABLE tb DROP PRIMARY KEY, ALGORITHM=INSTANT;
  ERROR 1846: 뭐시기 어쩌고 저쩌고 ... Try ALGORITHM=COPY/INPLACE.

  -- ALGORITHM=INPLACE, LOCK=NONE OR SHARED
  sql> ALTER TABLE tb DROP PRIMARY KEY, ALGORITHM=INPLACE, LOCK=NONE;
  ERROR 1846: 뭐시기 어쩌고 저쩌고 ... Try ALGORITHM=COPY.

  -- ALGORITHM=COPY, LOCK=SHARED
  sql> ALTER TABLE tb DROP PRIMARY KEY, ALGORITHM=INPLACE, LOCK=NONE;
  Query OK, XXX rows affected. -- 성공!
  ```

#### INPLACE의 테이블 리빌드 (Data Reorganizing / Table Rebuild)

- 프라이머리 키를 추가하는 작업은 레코드의 저장 위치가 변경되기 때문에, 리빌드가 필요하다.
- 반면, 칼럼의 이름만 변경하는 경우는 필요 없다.

#### INPLACE 알고리즘

- 테이블 리빌드하는 과정
  1. INPLACE 스키마 변경이 지원되는 스토리지 엔진의 테이블인지 확인
  2. INPLACE 스키마 변경 준비
  - (온라인 DDL 작업 동안 변경되는 데이터를 추적할 준비)
  3. 스키마 변경 및 새로운 DML 로깅
  - (다른 커넥션의 DML은 대기하지 않는다!)
  - (DML는 온라인 변경 로그에 기록함)
  4. 로그 적용
  - (위에서 수집된 DML 로그를 적용)
  5. COMMIT
- 2번과 4번에서 배타적 잠금(Exclusive lock)이 필요하다.
- 대부분의 실행시간은 3번이 차지한다.

### 11.7.2 ~

팁만 뽑았어요!

#### 테이블 생성

- 숫자 타입은 길이를 가질 수 있지만, 단순히 값을 보여줄때 길이를 지정하는 것이다. 그리고 Deprecated되었다.
- DATE, DATETIME, TIMESTAMP는 DEFAULT로 현재시간을 명시할 수 있다. (좋은걸까..?)

#### 테이블 삭제

- 서비스 도중에 테이블 삭제 하지 말자.
- 파일 조각들이 분산되어 있다면 부하가 높아져서 다른 커넥션에 영향을 끼칠 수 있다.
- 어댑티브 해시 인덱스 삭제 작업으로 인해 서버에 부하가 높아질 수도 있다.

#### 칼럼 추가

- 테이블 마지막에 칼럼을 추가하면 INSTANT 알고리즘으로 즉시 처리가 가능하다. (중간에 하지 말자)

#### 칼럼 수정

- 칼럼의 이름 변경: INSTANT 알고리즘으로 빠른 처리 가능
- 칼럼의 타입 변경: COPY, 테이블에 잠금이 걸리게 된다.
- 칼럼의 길이 변경(VARCHAR 같은거): 케바케

##### 칼럼의 길이 변경

- 선행 지식
  - 칼럼의 최대 사이즈는 메타데이터에 저장
  - 값의 실 길이는 데이터 레코드의 칼럼 헤더에 저장

실 길이를 저장하는 칼럼의 헤더 크기 (**길이를 저장하는 헤더의 크기**)에 따라 테이블 리빌드가 발생할 수 있다.

- 최대 길이가 255 바이트 이하인 경우 -> 1바이트 (0<= x < 2^8)
- 256 이상인 경우 -> 2바이트 (2^8<= x < 2^16)

#### 인덱스 추가

- 인덱스 변경도 온라인 DDL처리가 가능하다!
- 전문 검색을 제외하면 모두 INPLACE에 LOCK=NONE으로 처리가 가능하다.

#### 인덱스 이름 변경

- INPLACE, LOCK=NONE
- 테이블 리빌드 없다!

#### 인덱스 칼럼 변경

- `새 인덱스 추가 -> 기존 인덱스 삭제 -> 새 인덱스 이름 변경`과 같은 과정으로 처리 가능
- INPLACE, LOCK=NONE

#### 인덱스 가시성

- 인덱스가 있지만, 마치 없는 것 처럼 할 수 있다. 옵티마이저가 무시한다.
- `sql> ALTER TABLE tb ALTER INDEX ix_col INVISIBLE` (메타 데이터만 변경됨 INSTANT?)
- 반대는 VISIBLE
- `sql> ALTER TABLE tb ALTER INDEX ix_col VISIBLE`
- 인덱스가 많아지면 성능이 악화될 수 있다. INVISIBLE 키워드로 인덱스를 적용할지 말지를 설정할 수 있다.

#### 인덱스 삭제

- 세컨더리 인덱스
  - INPLACE, LOCK=NONE
  - 테이블 리빌드 없다!
- 프라이머리 키
  - COPY, SHARED (세컨더리 인덱스에 있는 pk를 삭제해야 하기 때문에..)

### 11.7.8 프로세스 조회 및 강제 종료

- mysql에 접속한 사용자 목록이나 쿼리 실행 현황을 보고 싶을때
  - `SHOW PROCESSLIST`
  - 스레드 수만큼 표시된다.
- 특정 스레드 / 쿼리를 죽이고 싶을때!
  - `KILL QUERY [스레드 id]`: 쿼리만 종료
  - `KILL [스레드 id]`: 커넥션 강제종료 -> 트랜잭션은 롤백 처리된다.

### 11.7.9 활성 트랜잭션 조회

- `information_schema.innodb_trx` 테이블을 통해 확인 가능.

## 11.8 쿼리 성능 테스트

### 성능에 영향을 미치는 요소

- 운영체제의 캐시
  - InnoDB는 파일 시스템의 캐시나 버퍼를 거치지 않는 Direct I/O를 사용, 운영체제 캐시는 별로 상관없다.
  - MyISAM 마이 아이 삼 같은 경우는 운영체제의 캐시 의존도가 높다.
- InnoDB 버퍼 풀
  - 캐시 대상: 인덱스 페이지, 데이터 페이지, 쓰기 작업을 위한 버퍼링
  - **MySQL 서버가 종료되면 자동으로 덤프되고, 다시 시작할때 자동으로 적재한다.**
  - 덤프되는게 부담스럽고 완전히 새로 시작하고 싶으면..
    - `innodb_buffer_pool_load_at_startup=OFF`
    - `innodb_buffer_pool_down_at_shutdown=OFF`
- 독립된 MySQL서버
  - MySQL 서버를 실행하는 기기가 다른 프로그램을 같이 돌린다? 오오 성능 구려짐.
- 쿼리 테스트 횟수
  - 쿼리의 성능을 테스트 할때는 쿼리를 6~7번 정도 실행해보고 처음 2개는 버리고 나머지의 평균으로 비교하자.
  - 워밍업/콜드 이슈가 있다.
