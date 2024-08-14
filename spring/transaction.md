# 트랜잭션

- 데이터베이스의 상태를 변환시키는 하나의 논리적인 기능을 수행하는 작업 단위, 한꺼번에 모두 수행되어야 할 연산

## 성질

- 원자성(Atomicity): 트랜잭션의 연산은 데이터베이스에 모두 반영되든지 아니면 전혀 반영되지 않아야 함
- 일관성(Consistency): 트랜잭션이 그 실행을 성공적으로 완료하면 언제나 일관성 있는 데이터베이스 상태로 변환해야 함
- 독립성(Isolation): 둘 이상의 트랜잭션이 동시에 병행 실행되는 경우 어느 하나의 트랜잭션 실행 중 다른 트랜잭션의 연산이 끼어들 수 없음
- 영속성(Durability): 성공적으로 완료된 트랜잭션의 결과는 시스템이 고장나더라도 영구적으로 반영되어야 함

## 상태

- 활동(Active): 트랜잭션이 실행 중인 상태
- 실패(Failed): 트랜잭션 실행에 오류가 발생하여 중단된 상태
- 철회(Aborted): 트랜잭션이 비정상적으로 종료되어 Rollback 연산을 수행한 상태
- 부분 완료(Partially Committed): 트랜잭션의 마지막 연산까지 실행했지만, Commit 연산이 실행되기 직전의 상태
- 완료(Committed): 트랜잭션이 성공적으로 종료되어 Commit 연산을 실행한 후의 상태

## @Transactional

- 트랜잭션에 대한 전파, 격리, 시간 초과, 읽기 전용 및 롤백 조건을 설정

## 전파 속성(propagation)

- 메서드 호출 시 기존 트랜잭션의 존재 여부에 따라 새로운 트랜잭션을 시작할지, 기존 트랜잭션에 참여할지 등을 결정

### 종류

### 1. REQUIRED(기본값)

- 현재 트랜잭션이 존재하면 참여하고, 없으면 새로운 트랜잭션을 시작
- `@Transactional(propagation = Propagation.REQUIRED)`

### 2. REQUIRES_NEW

- 항상 새로운 트랜잭션을 시작하고, 기존 트랜잭션은 일시 중지
- `@Transactional(propagation = Propagation.REQUIRES_NEW)`

### 3. MANDATORY

- 현재 트랜잭션이 존재하지 않으면 예외를 발생
- `@Transactional(propagation = Propagation.MANDATORY)`

### 4. SUPPORTS

- 현재 트랜잭션이 존재하면 참여하고, 없으면 트랜잭션 없이 실행
- `@Transactional(propagation = Propagation.SUPPORTS)`

### 5. NOT_SUPPORTED

- 트랜잭션 없이 실행하고, 현재 트랜잭션은 일시 중지
- `@Transactional(propagation = Propagation.NOT_SUPPORTED)`

### 6. NEVER

- 현재 트랜잭션이 존재하면 예외를 발생
- `@Transactional(propagation = Propagation.NEVER)`

### 7. NESTED

- 중첩된 트랜잭션을 시작
- `@Transactional(propagation = Propagation.NESTED)`

## 격리 수준(isolation)

- 동시에 실행되는 트랜잭션들이 서로에게 미치는 영향을 제어

### 종류

### 1. DEFAULT

- 기본 데이터베이스의 격리 수준을 따름
- `@Transactional(isolation = Isolation.DEFAULT)`

### 2. READ_UNCOMMITTED

- 가장 낮은 격리 수준, 변경 중인 데이터를 다른 트랜잭션에서 읽기 가능
- `@Transactional(isolation = Isolation.READ_UNCOMMITTED)`

### 3. READ_COMMITTED

- 커밋된 데이터만 읽기 가능
- `@Transactional(isolation = Isolation.READ_COMMITTED)`

### 4. REPEATABLE_READ

- 트랜잭션이 시작된 후 변경된 데이터는 다른 트랜잭션에서 읽을 수 없음
- `@Transactional(isolation = Isolation.REPEATABLE_READ)`

### 5. SERIALIZABLE

- 가장 높은 격리 수준, 모든 트랜잭션을 순차적으로 실행
- `@Transactional(isolation = Isolation.SERIALIZABLE)`

### Dirty Read, Non-repeatable Read, Phantom Read
- Dirty Read는 트랜잭션이 아직 커밋되지 않은 다른 트랜잭션의 변경사항을 읽는 경우
- Non-Repeatable Read는 같은 트랜잭션 내에서 동일한 쿼리를 두 번 실행했을 때, 그 결과가 다른 경우
- PhanTom Read는 트랜잭션이 동일한 범위 쿼리를 두 번 이상 실행할 때, 다른 트랜잭션이 데이터를 삽입하거나 삭제함으로써 결과 집합이 달라지는 현상
- Read Uncommitted: Dirty Read 허용, Non-repeatable Read 허용.
- Read Committed: Dirty Read 방지, Non-repeatable Read 허용.
- Repeatable Read: Dirty Read 방지, Non-repeatable Read 방지.
- Serializable: Dirty Read 방지, Non-repeatable Read 방지, Phantom Read 방지.


## Timeout

- 트랜잭션이 설정된 시간 내에 완료되지 않으면 자동으로 롤백
- 초 단위로 설정
- `@Transactional(timeout = 30)`

## ReadOnly

- 트랜잭션이 데이터 읽기만 할 때 최적화를 위해 사용
- 읽기 전용 트랜잭션은 데이터베이스의 특정 최적화를 활용
- `@Transactional(readOnly = true)`

## rollbackFor / noRollbackFor

- 특정 예외가 발생했을 때 트랜잭션을 롤백하거나 롤백하지 않도록 설정
- rollbackFor
    - 지정된 예외 발생 시 롤백
    - `@Transactional(rollbackFor = CustomException.class)`
- noRollbackFor
    - 지정된 예외 발생 시 롤백하지 않음
    - `@Transactional(noRollbackFor = NotCriticalException.class)`
