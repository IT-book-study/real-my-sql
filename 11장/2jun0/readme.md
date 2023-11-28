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
