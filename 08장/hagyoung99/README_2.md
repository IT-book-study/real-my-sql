# 08_인덱스_4주차

## 8.8 클러스터링 인덱스
- 여러 개를 하나로 묶는다는 의미로 클러스터링 인덱스는 레코드를 비슷한 것(프라이머리 키 기준)들 끼리 묶어서 저장하는 형태로 구현한다.

### 8.8.1 클러스터링 인덱스
- 테이블의 프라이머리 키에 대해서만 적용되는 것으로 프라이머리 키가 비슷한 레코드 끼리 묶어 저장하는 것을 말한다.
- 클러스터링 인덱스는 **프라이머리 키 값에 의해 레코드의 저장 위치가 결정** 되고, 프라이머리 키 값이 **변경되면 레코드의 물리적인 위치도 변경** 된다.
  - 저장 위치가 달라지기 때문에 인덱스 알고리즘 보다는 테이블 레코드의 저장 방식이라 볼 수 있다.
  - 그래서 '클러스터링 인덱스'와 '클러스터링 테이블'은 동의어로 사용되기도 한다.
  - 프라이머리 키를 클러스터링 키 라고 표현하기도 한다.
- 프라이머리 키 값에 대한 의존도가 높기 때문에 프라이머리 키 결정에 신중해야한다.
- 프라이머리 키를 기준으로 레코드가 저장되기 때문에 프라이머리 키 기반의 검색이 매우 빠르지만 레코드의 저장이나 프라이머리 키의 변경 시 상대적으로 느리다.


- 일단 B-Tree 구조와 다르게 클러스터링 인덱스의 리프 노드에는 레코드의 모든 칼럼이 저장되어 있다.
- 즉, **클러스터링 테이블은 그 자체가 하나의 거대한 인덱스 구조로 관리된다**
- 프라이머리 키가 변경되면 데이터 레코드에서의 물리적 변경도 일어난다.


#### 프라이머리 키가 없는 InnoDB 테이블의 구성
> 1. 프라이머리 키
> 2. NOT NULL 옵션의 유니크 인덱스 중 첫 번째 인덱스
> 3. 자동으로 유니크한 값을 가지도록 증가되는 컬럼을 내부적으로 추가

- 프라이머리 키가 없다면 위의 순서대로 클러스터링 기준으로 선택한다.
- 적절한 키가 없다면 3번처럼 내부적으로 레코드의 일련번호 칼럼을 생성하는데 이는 아무 의미 없는 숫자 값으로 클러스터링 되는 것으로 인덱스 혜택을 사용할 수 없게 된다.
- 따라서 클러스터링 인덱스를 사용할 경우 프라이머리 키를 명시적으로 생성하는게 좋다.


### 8.8.2 세컨더리 인덱스에 미치는 영향
- MyISAM 이나 MEMORY 테이블에서는 데이터 레코드가 저장된 주소는 내부적인 레코드 아이디 역할을 하고 프라이머리 키나 세컨더리 인덱스의 각 키는 그 주소를 이용해 실제 데이터 값을 찾아온다.
- 인덱스가 실제 레코드의 저장 주소를 가지고 있다면 클러스터링 키 값이 변경될 때 마다 레코드의 주소가 변경되어야하고, 그때 마다 모든 인덱스에 저장된 주소값을 변경해야하는데 이런 오버헤드를 제거하기 위해 InnoDB 테이블의 모든 세컨더리 인덱스는 프라이머리 키 값을 저장하게 구현되어있다.
- MyISAM: 인덱스 검색으로 레코드 주소 확인 후 레코드 주소로 최종 레코드를 가져온다.
- InnoDB: 인덱스를 검색해 레코드의 프라이머리 키 확인 후 프라이머리 키 인덱스를 검색해 최종 레코드를 가져온다.

### 8.8.3 클러스터링 인덱스의 장점과 단점

- 장점
  - 프라이머리 키로 검색할 때 처리 성능이 빠르다. 특히, 프라이머리 키를 범위 검색에 사용할 경우 매우 빠르다.
  - 테이블의 모든 세컨더리 인덱스가 프라이머리 키를 가지고 있어 인덱스만으로 처리될 수 있는 경우가 많다.(커버링 인덱스)
- 단점
  - 클러스터링 키 값의 크기가 클 경우 테이블의 세컨더리 인덱스가 해당 키를 가져 인덱스의 크기가 커진다.
  - 세컨더리 인덱스를 통해 검색할 때 프라이머리 키로 다시 검색해야해 처리 성능이 느리다.
  - INSERT 할때 프라이머리 키를 기준으로 레코드를 저장해 처리 성능이 느리다.
  - 프라이머리 키 변경 시 레코드 삭제 후 저장하는 작업이 필요해 처리 성능이 느리다.


### 8.8.4 클러스터링 테이블 사용 시 주의사항

#### 8.8.4.1 클러스터링 인덱스 키의 크기

- 클러스터링 테이블의 경우 모든 세컨더리 인덱스가 프라이머리 키 값을 포함하는데 프라이머리 키의 크기가 커지면 세컨더리 인덱스도 함게 커진다.
- 인덱스가 커질수록 같은 성능을 내기 위해서는 메모리가 더 필요해지므로 프라이머리 키는 신중하게 선택해야한다.

#### 8.8.4.2 프라이머리 키는 AUTO-INCREMENT 보다 업무적인 칼럼으로 생성(가능한 경우)

- 프라이머리 키에 의해 레코드 위치가 결정되기 때문에 칼럼의 크기ㅏㄱ 크더라도 업무적으로 해당 레코드를 대표할 수 있다면 그 칼럼을 프라이머리 키로 설정하는게 좋다.

#### 8.8.4.3 프라이머리 키는 반드시 명시할 것

- 프라이머리 키가 없는 테이블의 경우 내부적으로 일련번호 칼럼을 추가하는데 해당 칼럼은 AUTO_INCREMENT 와 기능이 동일하다.
- 내부적으로 추가된 컬럼은 사용자가 접근하지 못하기 때문에 프라이머리 키로 지정할 칼럼이 없다면 AUTO_INCREMENT를 추가해 설정하는게 좋다.

#### 8.8.4.4 AUTO-INCREMENT 칼럼을 인조 식별자로 사용할 경우

- 여러 개의 칼럼이 복합으로 프라이머리 키가 만들어지는 경우 프라이머리 키가 길어질 수 있는데
- 프라이머리 키의 크기가 길어도 세컨더리 인덱스 불필요 시 프라이머리 키를 사용
- 세컨더리 인덱스 사용, 프라이머리 키의 크기가 길다면 AUTO_INCREMENT 를 추가해 프라이머리 키로 설정
  - 프라이머리 키 대체 키를 인조 식별자라고 한다.
- 조회성 보다는 INSERT 위주의 테이블에 인조 식별자를 프라이머리 키로 사용하는것이 성능 향상에 도움이 된다.

## 8.9 유니크 인덱스

- 유니크는 테이블이나 인덱스에 같은 값이 2개 이상 저장될 수 없음을 의미하는데 인덱스보다는 제약조건에 가깝다.
- 유니크 인덱스는 NULL이 저장될 수 있는데 NULL은 특정 값이 아니기 때문에 2개 이상 저장될 수 있다.

### 8.9.1 유니크 인덱스와 일반 세컨더리 인덱스의 비교

#### 8.9.1.1 인덱스 읽기

- 유니크 하지 않은 세컨더리 인덱스는 중복된 값이 허용되어 읽어야할 레코드가 많아져서 느릴 뿐 인덱스 자체의 특성 때문에 느리지지는 않기 때문에 두 인덱스의 성능 차이는 거의 없다.
- 읽어야할 레코드 건수가 같다면 성능상 차이는 미미하다.

#### 8.9.1.2 인덱스 쓰기

- 유니크 인덱스는 중복된 값이 저장되지 못하기 때문에 키 값을 쓸 때 중복된 값이 있는지 없는지 체크하는 과정이 필요해 중복이 허용되는 세컨더리 인덱스의 쓰기보다는 느리다.
- MySQL 에서 유니크 인덱스에 중복 체크를 할 때는 읽기 잠금, 쓰기를 할때는 쓰기 잠금을 하는데 이 때 데드락이 자주 발생하고, 인덱스 키와는 다르게 유니크 인덱스는 중복체크를 해야해 작업을 버퍼링 하지 못하기 때문에 유니크 인덱스는 일반 세컨더리 인덱스보다 변경 및 쓰기 작업이 느리다.

### 8.9.2 유니크 인덱스 사용 시 주의사항

- 유니크 인덱스를 사용한다고 성능이 좋아지지는 않기 때문에 불필요하게 유니크 인덱스를 생성하지 않는게 좋다.
- 유니크 인덱스와 일반 세컨더리 인덱스의 역할은 동일하기 때문에 인덱스를 중복해 만들 필요는 없다.
- 같은 컬럼에 유니크와 프라이머리키 인덱스를 동일하게 생성하는 경우도 인덱스의 중복이기 때문에 주의해햐앟낟.
- 유니트 인덱스는 쿼리의 실행 계획이나 테이블의 파티션에 미치는 경우가 있다 → 해당 내용은 10장과 13장에 나온다.
- 결론적으로 유일성이 꼭 필요한 컬럼이 아니라면 유니크 컬럼보다 세컨더리 인덱스를 생성하는 방법이 좋을 수 있다.

## 8.10 외래키

- 외래키는 InnoDB 스토리지 엔진에서만 생성 가능하며 외래키 제약이 설정되면 연관되는 테이블 칼럼에 인덱스도 자동으로 생성된다.
- 생성된 인덱스는 외래키가 제거되지 않는 상태에서 삭제할 수 없다.

#### 외래키 관리의 특징

- 테이블의 변경이 발생하는 경우 잠금 대기가 발생한다.
- 외래키와 연관없는 칼럼의 변경은 최대한 잠금 대기를 발생시키지 않는다.

### 8.10.1 자식 테이블의 변경이 대기하는 경우

- 자식 테이블의 외래키 칼럼의 변경은 부모 테이블의 확인이 필요하기 때문에 부모 테이블에서 해당 레코드에 쓰기 잠금이 걸려있다면 해제할 때까지 기다린다.
- 자식테이블의 외래키가 아닌 칼럼은 외래키로 인한 잠금 확장이 발생하지 않는다.

### 8.10.2 부모 테이블의 변경 작업이 대기하는 경우

- 부모 키를 참조하는 자식 테이블의 레코드를 변경하면 자식 테이블에 대해 쓰기 잠금이 걸린다. 쓰기 잠금이 걸리게 되면 부모 테이블에서 자식 테이블이 참조하고 있는 키의 레코드를 삭제하는 경우 자식 테이블의 잠금이 해제될 때 까지 기다린다.
- 기다리는 이유는 외래키 생성 시 정의된 부모 레코드와 연관된 자식레코드는 함께 삭제되어야하기 때문

- 데이터베이스에서 외래키를 물리적으로 생성하려면 잠금 경합까지 고려해 모델링을 진행해야한다.
