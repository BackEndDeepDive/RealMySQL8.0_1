<div align="center">

# Deep Dive  
Deep Dive into Real My SQL 8.0(1)

--- 
</div>

</div>

## Contents

* [Lost Update 문제](#lost-update-문제)
* [비관적 락 (Pessimistic Lock)](#비관적-락-pessimistic-lock)
* [낙관적 락 (Optimistic Lock)](#낙관적-락-optimistic-lock)
* [CAS 연산 (Compare-And-Swap)](#cas-연산-compare-and-swap)

---

### Lost Update 문제

동시성 제어 없이 단순히 읽기·쓰기만 수행할 때 발생할 수 있는 Lost Updates(분실 갱신) 문제의 예시입니다.
 
```sql
CREATE TABLE account (
  id INT PRIMARY KEY,
  balance INT
);
INSERT INTO account VALUES (1, 1000);
```

```
time
|
|
v
        transaction A                                       transaction B

        SET autocommit=0;                                   SET autocommit=0;

        SELECT balance FROM account WHERE id=1; → 1000      SELECT balance FROM account WHERE id=1; → 1000

        UPDATE account SET balance = 1000 - 200 WHERE id=1; 
        A: balance→800
                                                            UPDATE account SET balance = 1000 - 300 WHERE id=1; -- B: balance→700
                                                            COMMIT;  -- B commits last
        COMMIT;  -- A commits earlier

        SELECT balance FROM account WHERE id=1; → 700  -- A의 감소(800) 반영 누락
```

결과: 두 건의 출금(200+300=500) 중 마지막 커밋만 반영되어 `balance=700`이 됩니다.

---

### 비관적 락 (Pessimistic Lock)

데이터베이스 레이어에서 직접 행·테이블 락을 이용해 Lost Update를 방지하는 방법입니다.

```sql
-- 트랜잭션 A
START TRANSACTION;
SELECT balance FROM account WHERE id=1 FOR UPDATE;  -- 행 락 획득
UPDATE account SET balance = balance - 200 WHERE id=1;
-- 트랜잭션 B는 A 커밋 전까지 FOR UPDATE 대기
COMMIT;

-- 트랜잭션 B
START TRANSACTION;
SELECT balance FROM account WHERE id=1 FOR UPDATE;  -- 이제 락 해제 후 balance 읽음 (800)
UPDATE account SET balance = balance - 300 WHERE id=1;
COMMIT;
```

* **장점**: 트랜잭션을 시작할때 락을 획득해야만 접근할 수 있어 확실하게 동시성을 제어할 수 있습니다.
* **단점**: 동시성이 높은 환경에서는 락 대기로 인한 블로킹, 데드락 위험, 커널 콜 오버헤드 발생합니다.

---

### 낙관적 락 (Optimistic Lock)

애플리케이션/ORM 레이어에서 버전 비교를 통해 충돌을 감지하고 재시도하는 방법입니다.

```sql
ALTER TABLE account ADD COLUMN version INT DEFAULT 0;
```

1. 트랜잭션 A, B가 동시에 `SELECT balance, version` → `(1000, 0)`
2. A:

```sql
UPDATE account
 SET balance = 1000 - 200,
     version = version + 1
 WHERE id=1 AND version = 0;
-- affected_rows = 1 → 성공, balance=800, version=1
COMMIT;
```

3. B:

```sql
UPDATE account
 SET balance = 1000 - 300,
     version = version + 1
 WHERE id=1 AND version = 0;
-- affected_rows = 0 → 충돌 감지 → ROLLBACK 또는 재시도
```

* **장점**: 락 대기 없이 높은 동시성
* **단점**: 충돌 빈도 높으면 재시도 오버헤드

---

### CAS 연산 (Compare-And-Swap)

낙관적 락은 **버전(version) 비교 후 갱신**하는 방법입니다. 이 비교-갱신을 원자적으로 처리해 주는 연산이 CAS입니다.

#### CAS 동작 원리

> CAS(addr, expected, new\_value) → Boolean

```
1. old = *addr;
2. if old == expected:
     *addr = new_value;
     return true;
   else:
     return false;
```

* **비교(Compare)**: 메모리에서 `old` 값을 읽어 `expected`와 비교
* **교체(Swap)**: 일치하면 `new_value`로 교체하고 성공 반환, 불일치 시 실패 반환

#### 예시 1

```c
int version = 5;          // 현재 버전
int expected = 5;         // 예상 버전
int new_value = 6;        // 갱신할 버전
bool success = CAS(&version, expected, new_value);
// success=true, version==6

// 다른 스레드가 먼저 바꿨다면
version = 6;
bool fail = CAS(&version, expected, new_value);
// expected(5)와 달라 fail=false 
```

#### 예시 2

```java
import java.util.concurrent.atomic.AtomicInteger;

AtomicInteger counter = new AtomicInteger(0);
int oldVal = counter.get();                // 0 읽음
boolean okA = counter.compareAndSet(oldVal, oldVal + 1); // okA=true, counter==1
boolean okB = counter.compareAndSet(oldVal, oldVal + 1); // okB=false, counter stays 1
```

* **특징**: 커널 호출 없이 사용자 공간에서 원자적 업데이트
* **ABA 문제**: 값이 A→B→A 후 다시 A가 되면 변경 사실 인지 못함, 해결 위해 버전 태깅 사용
* **활용 예**: Java `AtomicInteger.compareAndSet`, InnoDB buffer latch 등

