<div align="center">

# Deep Dive  
Deep Dive into Real My SQL 8.0(1)

--- 
</div>

</div>

## Contents
- [인덱스](#인덱스)
- [은총알은 없다](#은-총알은-없다)
- [MySQL - Employees](#my-sql---employees)
- [Employees](#employees)
- [Case 1. 대량 INSERT 시 B-Tree 갱신 오버헤드](#case-1-대량-insert-시-b-tree-갱신-오버헤드)
- [Case 2. 대량 UPDATE 시 인덱스 갱신](#case-2-대량-update-시-인덱스-갱신) 

### 인덱스
RealMySQL 8.0 책의 인덱스 부분을 공부하던 중 DB 인덱스는 데이터베이스 성능 향상에 필수적이지만 부작용도 존재한다고 확인할 수 있었습니다. 가장 큰 부작용은 데이터 삽입, 삭제, 수정 시 인덱스 유지 작업으로 인한 성능 저하와 추가적인 디스크 I/O 발생입니다. 또한, 인덱스가 많아질수록 전체 데이터베이스 성능 부하가 증가하고, 잘못된 쿼리 실행 시에는 인덱스 효과가 나타나지 않을 수 있어 잘 활용해야합니다.

### 은 총알은 없다
추가적으로 네이버 D2 블로그의 [성능 향상을 위한 SQL 작성법](https://d2.naver.com/helloworld/1155?utm_source=chatgpt.com)에는 인덱스 사용 시 성능 저하가 발생할 수 있어 고려해야할 내용을 명시해놓은 것을 확인할 수 있었습니다. 
![alt text](image.png)

따라서 기존에 있던 대용량 데이터를 가지고 몇 가지의 사례를 통해 인덱스가 어떻게 성능 저하를 불러일으키는 지 확인해보겠습니다.

### My SQL - Employees
MySQL에서는 학습용 대규모 데이터 [test_db](https://github.com/datacharmer/test_db)를 지원하고 있어 이 중 400만 건의 데이터를 가지고 있는 ```Employees``` 데이터베이스를 사용해보겠습니다.

### Employees

MySQL의 공식 학습용 `Employees` 데이터베이스는 실제 운영 환경과 유사한 **약 400만 건** 규모의 샘플 스키마입니다.  
주요 테이블과 관계, 데이터 규모를 간단히 요약하면 다음과 같습니다.

![Employees ERD](image-1.png)

```Employees```를 요약하자면 다음과 같습니다.

| 테이블         | 설명                             | 레코드 수       |
|---------------|--------------------------------|---------------|
| **employees**   | 직원 기본 정보 (사번, 이름, 성별, 입사일 등) | 약 300,000건  |
| **salaries**    | 직원별 연봉 이력 (`from_date`~`to_date`)    | 약 2,800,000건 |
| **titles**      | 직원별 직함 이력 (`from_date`~`to_date`)    | 약   440,000건 |
| **departments** | 부서 정보 (부서코드 PK, 부서명)             | 9건           |
| **dept_emp**    | 직원–부서 매핑 이력 (`from_date`~`to_date`)  | 약   330,000건 |
| **dept_manager**| 부서 관리자 이력                          | 24건          |


### Case 1. 대량 INSERT 시 B-Tree 갱신 오버헤드
읽기 성능을 위해 만든 first_name, last_name의 인덱스가 있다고 가정하고 신입사원 공채로 대량 INSERT가 발생하였을 때 TPS와 지연시간을 비교해보겠습니다.

먼저 원본 테이블을 복사하여 벤치마크 전용 테이블로 만들어보겠습니다.
![alt text](image-2.png)

읽기 성능을 위하여 `first_name` 과 `last_name`에 인덱스를 걸고 sysbench를 통하여 TPS와 Latency를 측정해보겠습니다.

sysbench 스크립트는 다음과 같습니다.
```
sysbench \
  --db-driver=mysql \
  --mysql-user=root \
  --mysql-password=1069 \
  --mysql-db=employees \
  --tables=1 \
  --threads=16 \
  --time=60 \
  --report-interval=60 \
  ~/oltp_insert_emp.lua run
```

이제 인덱스를 추가하여 sysbench를 다시 돌려보도록하겠습니다.

```SQL
  ALTER TABLE emp_bench
  ADD INDEX idx_first_name (first_name),
  ADD INDEX idx_last_name  (last_name);
```

```
[ 60s ] thds: 16 tps: 54147.51 qps: 54147.51 (r/w/o: 0.00/54147.51/0.00) lat (ms,95%): 0.00 err/s: 0.00 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            0
        write:                           3248889
        other:                           0
        total:                           3248889
    transactions:                        3248889 (54139.88 per sec.)
    queries:                             3248889 (54139.88 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          60.0090s
    total number of events:              3248889

Latency (ms):
         min:                                    0.09
         avg:                                    0.30
         max:                                  128.90
         95th percentile:                        0.00
         sum:                               958442.39

Threads fairness:
    events (avg/stddev):           203055.5625/140.20
    execution time (avg/stddev):   59.9026/0.00
```


```
Threads started!

[ 60s ] thds: 16 tps: 54816.93 qps: 54816.93 (r/w/o: 0.00/54816.93/0.00) lat (ms,95%): 0.00 err/s: 0.00 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            0
        write:                           3289080
        other:                           0
        total:                           3289080
    transactions:                        3289080 (54805.36 per sec.)
    queries:                             3289080 (54805.36 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          60.0137s
    total number of events:              3289080

Latency (ms):
         min:                                    0.09
         avg:                                    0.29
         max:                                  153.15
         95th percentile:                        0.00
         sum:                               958309.36

Threads fairness:
    events (avg/stddev):           205567.5000/223.79
    execution time (avg/stddev):   59.8943/0.00
```
| 설정                     | TPS (transactions/sec) | Avg Latency (ms) |
| ---------------------- | ---------------------: | ---------------: |
| No Index, 16 threads   |              54,139.88 |             0.30 |
| With Index, 16 threads |              54,805.36 |             0.29 |

유의미하다면 할 수 있지만 현재 디바이스의 사양(M3 Pro 16GB RAM, 512 SDD)이 처리하는 데이터의 크기가 작고 인덱스의 개수가 작기 때문에 유의미한 결과를 도출해 낼 수 없었습니다. 다만 실제 배포환경에서의 데이터베이스 스토리지들은 위와 같은 사양이 아니기 때문에 훨씬 체감할 수 있을 것 같습니다.

### Case 2. 대량 UPDATE 시 인덱스 갱신 
`salary_bench` 테이블의 `salary` 칼럼에 인덱스를 걸었을 때
  대량 UPDATE(월급 5% 인상) 성능이 얼마나 저하되는지 측정해보겠습니다.

---

```sql
USE employees;

DROP TABLE IF EXISTS salary_bench;
CREATE TABLE salary_bench LIKE salaries;
INSERT INTO salary_bench SELECT * FROM salaries;

SET SQL_SAFE_UPDATES = 0;

```
인덱스를 걸지않고 TIMESTAMPDIFF를 통하여 쿼리를 찍어보면 

```SQL
USE employees;

SET @t0 = NOW(6);
UPDATE salary_bench
  SET salary = salary * 1.05;
SET @t1 = NOW(6);

SELECT ROUND(
  TIMESTAMPDIFF(MICROSECOND, @t0, @t1) / 1000000
, 3) AS `With IDX (s)`;

```
![alt text](<스크린샷 2025-05-21 19.53.48.png>)

이제 인덱스를 추가한 후 똑같은 쿼리를 찍어보겠습니다. 
```
USE employees;

ALTER TABLE salary_bench
  ADD INDEX idx_salary (salary);
```

![alt text](image-4.png)

- 결론은 다음과 같습니다.

| 설정     | 실행 시간 (초) |
| ------ | --------: |
| 인덱스 없음 |      7.10 |
| 인덱스 적용 |     28.50 |


- 따라서 읽기 성능을 위해 무분별하게 인덱스를 추가하면 대량 쓰기(UPDATE/INSERT)가 많은 시스템에서는 병목이 될 수 있으므로 주의해야 합니다.

