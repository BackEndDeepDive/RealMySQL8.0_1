<div align="center">

# Deep Dive  
Deep Dive into Real My SQL 8.0(1)

--- 
</div>

## Contents
- [MVCC & Non-locking Consistent Read](#mvcc--non-locking-consistent-read)
- [MVCC](#mvcc)
- [Non-locking Consistent Read](#non-locking-consistent-read)
- [Pros, Cons](#pros-cons)
---

### MVCC & Non-locking Consistent Read
Real My SQL 4.2.4 잠금 없는 일관된 읽기(Non-locking Consistent Read) 부분에서 InnoDB 스토리지 엔진의 MVCC를 다음과 같이 설명합니다. 
> InnoDB 스토리지 엔진은 MVCC 기술을 이용해 잠금을 걸지 않고 읽기 작업을 수행한다. 

그렇다면 MVCC가 무엇이고 InooDB는 왜 MVCC를 사용하고 있으며 Locking을 사용하는 READ 작업과는 어떤 차이가 있는 지 알아보겠습니다. 

#### MVCC
My SQL에서 MVCC는 다음과 같이 작성되어있습니다.
> MVCC (Multi-Version Concurrency Control) is the mechanism InnoDB uses to provide non-locking consistent reads and high concurrency.

innoDB에서 MVCC는 아래와 같이 동작합니다.
1. MVCC는 각 트랜잭션 ```INSERT```, ```UPDATE```,```DELETE```을 실행할 때 변경 전의 snapshot을 언두 로그(undo log)에 남기게 됩니다.
2. 레코드 헤더에 본 트랜잭션 ID와 언두 로그 포인터를 연결하여 버전 체인을 구성합니다.
3. 트랜잭션, SQL이 시작될 때 스냅샷 타임포인트를 부여받고 이 시점까지 커밋된 버전만 '각 트랜잭션만 확인하는 데이터'로 간주하고 이후 변경된 커밋이나 커밋되지 않은 사항은 확인할 수 없습니다.
4. **Consistent Read** ```SELECT``` 시 언두 로그에 저장된 버전 체인을 따라가며 스냅샷 시점에 맞는 버전을 재구성합니다.
5. 락을을 전혀 걸지 않으면서 과거 시점의 일관된 데이터 뷰를 제공합니다.

따라서 MVCC는 
- READ 작업이 락 없이 이루어지기 때문에 빠르고 동시성이 높습니다.
- WRITE는 이전 버전을 언두 로그에 남기므로 커밋이 충돌되는 경우 롤백이 가능합니다.
- 하지만 장기간 언두 로그가 디스크와 메모리에 남아있게 되면 오버헤드가 발생하게 됩니다.


#### Non-locking Consistent Read
락을 사용하지 않고 여러가지 케이스를 확인해보겠습니다.

Case 1. 기본 snapshot consistent reat(REPEATALBE READ)

트랜잭션 시작 시점의 스냅샷만 참조하여 이후 다른 트랜잭션이 삽입·수정·삭제하고 커밋한 변경은 절대 보지 않습니다. (Stale Read)

한계: 스냅샷 이후 커밋된 최신 데이터가 반영되지 않아 실시간성 조회가 필요한 경우 적용할 수 없는 문제가 있습니다.

```
time
|
|
v
        transaction A              transaction B

        SET autocommit=0;         SET autocommit=0;


        SELECT * FROM t;
        empty set
                                  
                                  INSERT INTO t VALUES (1, 2);

        SELECT * FROM t;
        empty set
                                  COMMIT;

        SELECT * FROM t;
        empty set

        COMMIT;

        SELECT * FROM t;
        ---------------------
        |    1    |    2    |
        ---------------------
```

Case 2. READ COMMITTED + Non-locking Read

READ COMMITTED 격리 수준에서 같은 트랜잭션 내라도 매 조회마다 새 스냅샷을 반영합니다.

한계: 반복 조회 시점마다 결과가 달라지는 Non-repeatable Read 및 Phantom Read 발생합니다.

```
time
|
|
v
        transaction A                         transaction B

        SET SESSION TRANSACTION             SET autocommit=0;
        ISOLATION LEVEL READ COMMITTED;
        SET autocommit=0;

        SELECT * FROM t;                    ← 첫 조회 (스냅샷 A₁)
        → 결과: empty set

                                            INSERT INTO t VALUES (1,2);
                                            COMMIT;   ← (1,2) 커밋

        SELECT * FROM t;                     ← 두 번째 조회 (스냅샷 A₂)
        → 결과: (1,2)
 
                                            INSERT INTO t VALUES (3,4);
                                            COMMIT;   ← (3,4) 커밋
        SELECT * FROM t;                    ← 세 번째 조회 (스냅샷 A₃)
        → 결과: (1,2), (3,4)

```
Case 3. Non-locking Read만 사용한 계좌 이체 (Lost Update)

락을 사용하지 않는 경우 데이터의 정합성이 중요한 계좌 이체, 재고 관리에서 두 트랜잭션이 동시에 동일 잔액을 읽고 갱신하여 마지막 커밋이 앞선 커밋을 덮어쓰는 Lost Update가 발생합니다.

```
time
|
|
v
        transaction A                               transaction B
        
        SET autocommit=0;                           SET autocommit=0;

        현재 잔액 확인                      
        SELECT balance FROM account WHERE id=1;
        → 1000                                     
                                                    동시에 잔액 확인
                                                    SELECT balance FROM account WHERE id=1;
                                                    → 1000
                                                    200원 출금 
                                                    UPDATE account 
                                                    SET balance = 1000 - 200 
                                                    WHERE id=1;
                                                    COMMIT;  B 커밋 후 잔액 800
        200원 출금                  
        UPDATE account 
        SET balance = 1000 - 200   
        WHERE id=1;
        COMMIT;  -- A 커밋 후 잔액 800

        최종 잔액 조회                             
        SELECT balance FROM account WHERE id=1;
        → 800  (실제 두 건의 출금액 400원 중 200원만 반영됨)
```

Case 4. Non-locking Read만 사용한 재고 관리 (Over-sell)
락을 사용하지 않는 경우 데이터 정합성이 중요한 재고 관리에서 두 트랜잭션이 동시에 동일 재고 수량을 읽고 갱신하여 마지막 커밋이 앞선 커밋을 덮어쓰면서 재고가 음수가 되는 Over-sell이 발생합니다.

```
time
|
|
v
        transaction A                               transaction B

        SET autocommit=0;                           SET autocommit=0;

        재고 확인                            
        SELECT stock FROM products WHERE id=1;      
        -> stock = 1                              

                                                    동시에 재고 확인
                                                    SELECT stock FROM products WHERE id=1;
                                                    -> stock = 1
                                                    주문 수락
                                                    UPDATE products 
                                                    SET stock = stock - 1 
                                                    WHERE id=1;
                                                    COMMIT;   재고 = 0
     
        주문 수락                             
        UPDATE products 
        SET stock = stock - 1 
        WHERE id=1;
        COMMIT;  재고 = -1  (음수 재고 오버셀 발생)
   
        최종 재고 조회                             
        SELECT stock FROM products WHERE id=1;
        stock = -1  (실제 주문 2건이 모두 처리됨)

```

#### Pros Cons
MVCC, Non-locking Consistent Read의 동작 원리와 대표적인 Anomaly(Stale Read, Non-repeatable Read/Phantom Read, Lost Update, Over-sell 등)를 살펴보았습니다.
- 장점: 읽기 시 락이 전혀 걸리지 않으므로 수많은 클라이언트가 동시 조회를 수행해도 블로킹 없이 응답 속도를 낼 수 있습니다.
- 단점: 핵심 비즈니스 로직(계좌 이체, 재고 차감, 예약 시스템 등)에서는 데이터 정합성이 꺠질 수 있어 격리 수준을 더 강하게 조정하거나 락을 사용해야합니다.
