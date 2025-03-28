## 4.2 InnoDB 스토리지 엔진 아키텍처
InnoDB는 MySQL에서 사용할 수 있는 스토리지 엔진 중 거의 유일하게 레코드 기반의 잠금을 제공하며, <br>
그 때문에 높은 동시성 처리가 가능하고 안정적이며 성능이 뛰어나다. <br>

<img width="600" alt="image" src="https://github.com/user-attachments/assets/e11ac2af-33b3-474d-be05-bdf507288038"> <br>

<br>

### 4.2.1 프라이머리 키에 의한 클러스터링
InnoDB의 모든 테이블은 기본적으로 프라이머리 키를 기준으로 클러스터링되어 저장된다. <br>
즉, 프라이머리 키 값의 순서대로 디스크에 저장된다는 뜻이며, <br>
모든 세컨더리 인덱스는 레코드의 주소 대신 프라이머리 키의 값을 논리적인 주소로 사용한다.

- 프라이머리 키가 클러스터링 인덱스이기 때문에 프라이머리 키를 이용한 레인지 스캔은 상당히 빨리 처리될 수 있다. <br>
  결과적으로 쿼리의 실행 계획에서 프라이머리 키는 기본적으로 다른 보조 인덱스에 비해 비중이 높게 설정된다.
- MyISAM 스토리지 엔진에서는 클러스터링 키를 지원하지 않기 때문에 프라이머리 키와 세컨더리 인덱스는 구조적으로 아무런 차이가 없다. <br>
  또한, 프라이머리 키를 포함한 모든 인덱스는 물리적인 레코드의 주소 값(ROWID)을 가진다.

<br>

### 4.2.2 외래 키 지원
부모 테이블과 자식 테이블 모두 해당 컬럼에 인덱스 생성이 필요하고, <br>
변경 시에는 반드시 부모 테이블이나 자식 테이블에 데이터가 있는지 체크하는 작업이 필요하다.

- InnoDB 스토리지 엔진 레벨에서 지원하는 기능으로 MyISAM이나 MEMORY 테이블에서는 사용할 수 없다.
- 잠금이 여러 테이블로 전파되고, 그로 인해 데드락이 발생할 때가 많으므로 외래 키의 존재에 주의하는 것이 좋다.
- `foreign_key_checks` 시스템 변수를 `OFF`로 설정하면 외래 키 관계에 대한 체크 작업을 일시적으로 멈출 수 있다. <br>
  하지만 반드시 부모 테이블과 자식 테이블의 일관성을 맞춰준 후 다시 외래 키 체크 기능을 활성화하도록 하자.

<br>

### 4.2.3 MVCC(Multi Version Concurrency Control)
일반적으로 레코드 레벨의 트랜잭션을 지원하는 DBMS가 제공하는 기능이며, 잠금을 사용하지 않는 일관된 읽기를 제공한다. <br>
InnoDB는 **Undo Log**를 이용해 이 기능을 구현한다. <br>

<br>

```
e.g.

데이터 insert 작업 후 commit O, update 작업 후 commit X
update 문장이 실행되면 InnoDB의 버퍼 풀은 새로운 값을 가지게 되고, 변경 전 값은 언두 로그로 복사한다.
이때 데이터를 조회하면? => 트랜잭션 격리 수준에 따라 결과가 달라진다.

`READ_UNCOMMITTED`인 경우에는 InnoDB 버퍼 풀이 현재 가지고 있는 변경된 데이터를 읽어서 반환한다.
`READ_COMMITTED`, `REPEATABLE_READ`, `SERIALIZABLE` 인 경우에는 변경되기 이전의 내용을 보관하고 있는 언두 영역의 데이터를 반환한다.

이 상태에서 commit을 실행하면 InnoDB는 지금의 상태를 영구적인 데이터로 만든다.
하지만 rollback을 실행하면 InnoDB는 언두 영역에 있는 백업된 데이터를 InnoDB 버퍼 풀로 다시 복구하고, 언두 영역의 내용을 삭제한다. 
```

<br>

#### 💡 참고

트랜잭션 격리 수준 | 설명
:--- | :---
READ_UNCOMMITTED | 커밋되지 않은 데이터를 읽는다.
READ_COMMITTED | 커밋된 데이터만 읽을 수 있다.
REPEATABLE_READ | 동일 트랜잭션 내에서 동일한 데이터의 결과를 보장한다.
SERIALIZABLE | 트랜잭션을 직렬화된 순서로 실행한다. SELECT 문도 공유 잠금을 획득한다.

- 격리 수준이 높을수록 데이터 일관성은 높아지지만 성능 저하가 발생할 수 있다.

<br>

### 4.2.4 잠금 없는 일관된 읽기(Non-Locking Consistent READ)
InnoDB 스토리지 엔진은 MVCC 기술을 이용해 잠금을 걸지 않고 읽기 작업을 수행한다. <br>
특정 사용자가 레코드를 변경하고 아직 커밋을 수행하지 않았다 하더라도 이 변경 트랜잭션이 다른 사용자의 SELECT 작업을 방해하지 않는다. <br>
InnoDB에서는 변경되기 전의 데이터를 읽기 위해 언두 로그를 사용한다.

<br>

### 4.2.5 자동 데드락 감지
InnoDB 스토리지 엔진은 내부적으로 잠금이 교착 상태에 빠지지 않았는지 체크하기 위해 잠금 대기 목록을 그래프 형태로 관리한다. <br>
데드락 감지 스레드가 주기적으로 잠금 대기 그래프를 검사해 교착 상태에 빠진 트랜잭션들을 찾아 그 중 하나를 강제 종료한다. <br>

- 어느 트랜잭션을 먼저 강제 종료할 것인지를 판단하는 기준은 언두 로그 양이다. 일반적으로 언두 로그 레코드를 더 적게 가진 트랜잭션이 롤백의 대상이 된다.
- `innodb_table_locks` 시스템 변수를 활성화하면 InnoDB 스토리지 엔진 내부의 레코드 잠금뿐만 아니라 테이블 레벨의 잠금까지 감지할 수 있다.
- `innodb_deadlock_detect` 시스템 변수를 `OFF`로 설정하면 데드락 감지 스레드는 작동하지 않는다.
- `innodb_lock_wait_timeout` 시스템 변수는 초 단위로 설정하며, 잠금을 설정한 시간 동안 획득하지 못하면 쿼리는 실패하고 에러를 반환한다.

<br>

### 4.2.6 자동화된 장애 복구
InnoDB 데이터 파일은 기본적으로 MySQL 서버가 시작될 때 항상 자동 복구를 수행한다.

- 이 단계에서 자동으로 복구될 수 없는 손상이 있다면 자동 복구를 멈추고 MySQL 서버가 종료된다. <br>
  이때는 MySQL 서버의 설정 파일에 `innodb_force_recovery` 시스템 변수를 설정해서 MySQL 서버를 시작해야 한다. <br>
  값이 커질수록 복구 가능성이 적어진다.
  - `1`: 로그 파일 복구만 수행 (기본 복구 수준)
  - `2`: 버퍼 풀에서 수정된 페이지 무시
  - `3`: 일관성 검사 중단 (테이블스페이스 스캔 무시)
  - `4`: 언두 로그 적용 건너뛰기 (롤백 수행 안함)
  - `5`: 복구 시 언두 로그 삭제 (일관성 무시)
  - `6`: 모든 복구 작업 중단 -> 강제 마운트 (읽기 전용)
- 마지막 풀 백업 시점으로부터 장애 시점까지의 바이너리 로그가 있다면 <br>
  InnoDB의 복구를 이용하는 것보다 풀 백업과 바이너리 로그로 복구하는 편이 데이터 손실이 더 적을 수도 있다.
  
