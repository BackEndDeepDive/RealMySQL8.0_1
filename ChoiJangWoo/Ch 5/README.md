<div align="center">

# Deep Dive  
Deep Dive into Real My SQL 8.0(1)

--- 
</div>

</div>

## Contents
- [개요](#개요)
- [쿼리 실행 절차](#쿼리-실행-절차)
- [1. SQL Parsing](#1-sql-parsing)
- [2. Optimization](#2-optimization)
- [3. Execution](#3-execution)
--- 

### 개요
MySQL 옵티마이저는 Parsing → Optimization → Execution의 세 단계로 쿼리를 처리합니다.
각 단계마다 내부에서 무슨 일이 벌어지는지, 왜 중요한지, 그리고 우리가 어떻게 플랜을 해석해야 하는지를 사례와 함께 살펴보겠습니다. 

### 쿼리 실행 절차 
| 단계 번호 | 단계 이름                                        | 담당 모듈                   | 주요 역할                           |
| ----- | -------------------------------------------- | ----------------------- | ------------------------------- |
| 1     | **SQL 파싱 (Parsing)**                         | SQL Parser              | SQL 문법 검사, 파스 트리(AST) 생성        |
| 2     | **최적화 & 실행 계획 수립 (Optimization & Planning)** | 옵티마이저(Optimizer)        | 조건·조인순서 단순화, 인덱스 선택, 실행 계획 생성   |
| 3     | **실행 (Execution)**                           | 스토리지 엔진(Storage Engine) | 실제 레코드 읽기/쓰기, 정렬·집계 등 물리적 연산 수행 |

하나의 쿼리문을 SQL에서 어떻게 실행 계획을 가지는 지 확인해보겠습니다.

SQL의 employees 테이블을 아래의 쿼리로 실행해보겠습니다.

```SQL
SELECT e.emp_no,
       e.first_name,
       s.salary
  FROM employees AS e
  JOIN salaries  AS s ON e.emp_no = s.emp_no
 WHERE e.hire_date > '2000-01-01'
 ORDER BY s.salary DESC
 LIMIT 10;
```

### 1. SQL Parsing

####  역할
* **문법 검사(Syntax Check)**
  입력된 SQL 문이 MySQL 문법에 맞는지 검사하여, 잘못된 구문은 이 단계에서 즉시 오류를 발생시킵니다.
* **파스 트리(AST) 생성**
  토큰화(lexing) → 문장 구조 분석(parsing)을 거쳐 AST(Abstract Syntax Tree)를 만듭니다.
  이 트리가 이후 옵티마이저가 쿼리를 이해하고 변형하는 기초가 됩니다.

#### 내부 처리 흐름

1. **토큰화(Tokenization)**

   ```txt
   SELECT → 키워드
   e.emp_no → 식별자(Identifier)
   ,       → 구분자(Comma)
   …
   ```
2. **구문 분석(Parsing)**

   * AST의 노드(node)로 `SELECT`, `JOIN`, `WHERE`, `ORDER BY`, `LIMIT` 등을 계층 구조로 구성
3. **구조 검증(Validation)**

   * 테이블·칼럼 존재 여부, 권한 체크 등


#### 파스 트리 확인 방법

```sql
EXPLAIN FORMAT=TREE
SELECT …
\G
```

> ```sql
> EXPLAIN FORMAT=TREE
> SELECT e.emp_no, e.first_name, s.salary
>   FROM employees AS e
>   JOIN salaries  AS s ON e.emp_no = s.emp_no
>  WHERE e.hire_date > '2000-01-01'
>  ORDER BY s.salary DESC
>  LIMIT 10\G
> ```
>
> ```
> -> Limit: 10 row(s)
>     -> Sort: s.salary DESC, limit input to 10 row(s) per chunk
>         -> Stream results  (cost=228212 rows=965634)
>             -> Nested loop inner join  (cost=228212 rows=965634)
>                 -> Filter: (e.hire_date > DATE'2000-01-01')  (cost=30894 rows=99872)
>                     -> Table scan on e  (cost=30894 rows=299645)
>                 -> Index lookup on s using PRIMARY (emp_no=e.emp_no)  (cost=1.01 rows=9.67)
> ```

* **첫 줄**(`-> Limit`)부터 **가장 안쪽**(`Table scan on e`)까지 AST 기반으로 쿼리 구조가 논리적으로 드러납니다.


#### 왜 중요한가?

* **오타·구문 오류 조기 발견**
* **복잡한 VIEW나 서브쿼리 디버깅**
* **미들웨어·프록시에서 쿼리 리라이팅 검증**
* 이후 단계(Optimization/Execution)의 “입력”이 되는 핵심 자료구조임을 인지하게됩니다,


### 2. Optimization

#### 역할

* **Logical Rewrite**: 조건·조인 순서 재배치하고 불필요한 연산 제거합니다.
* **Plan Enumeration**: 가능한 물리 연산(풀 스캔, 인덱스 스캔, Nested-Loop/Hash Join 등) 후보를 생성합니다.
* **Costing**: 각 후보에 대해 통계 기반 비용(cost) 계산합니다.
* **Plan Selection**: 최소 비용(plan\_cost)을 갖는 실행 계획을 선택합니다.
#### 확인 방법
```SQL
mysql> EXPLAIN FORMAT=JSON
    -> SELECT e.emp_no, e.first_name, s.salary
    ->   FROM employees AS e
    ->   JOIN salaries  AS s ON e.emp_no = s.emp_no
    ->  WHERE e.hire_date > '2000-01-01'
    ->  ORDER BY s.salary DESC
    ->  LIMIT 10;

```

#### JSON 플랜 주요 구조

```json
"query_block": {
  "select_id": 1,
  "cost_info": {
    "query_cost": "1193846.70"
  },
  "ordering_operation": {
    "using_temporary_table": true,
    "using_filesort": true,
    "cost_info": {
      "sort_cost": "965634.26"
    },
    "nested_loop": [
      {
        "table": {
          "table_name": "e",
          "access_type": "ALL",
          "possible_keys": ["PRIMARY"],
          "rows_examined_per_scan": 299645,
          "rows_produced_per_join": 99871,
          "filtered": "33.33",
          "cost_info": {
            "read_cost": "20906.33",
            "eval_cost": "9987.17",
            "prefix_cost": "30893.50",
            "data_read_per_join": "12M"
          },
          "used_columns": ["emp_no","first_name","hire_date"],
          "attached_condition": "(`employees`.`e`.`hire_date` > DATE'2000-01-01')"
        }
      },
      {
        "table": {
          "table_name": "s",
          "access_type": "ref",
          "possible_keys": ["PRIMARY"],
          "key": "PRIMARY",
          "used_key_parts": ["emp_no"],
          "key_length": "4",
          "ref": ["employees.e.emp_no"],
          "rows_examined_per_scan": 9,
          "rows_produced_per_join": 965634,
          "filtered": "100.00",
          "cost_info": {
            "read_cost": "100755.52",
            "eval_cost": "96563.43",
            "prefix_cost": "228212.45",
            "data_read_per_join": "14M"
          },
          "used_columns": ["emp_no","salary","from_date"]
        }
      }
    ]
  }
}
```


#### 주요 필드 

| JSON 경로                                      | 설명                                            |
| -------------------------------------------- | --------------------------------------------- |
| `query_block.cost_info.query_cost`           | **총 예상 비용**(클수록 비싼 쿼리)                        |
| `ordering_operation.using_temporary_table`   | 정렬 시 **임시 테이블** 사용 여부 (`true`면 디스크 I/O 발생 가능) |
| `ordering_operation.using_filesort`          | **FILESORT** 사용 여부 (인덱스만으로 ORDER BY 불가 시)     |
| `ordering_operation.cost_info.sort_cost`     | **정렬 비용**                                     |
| `nested_loop[].table.table_name`             | 처리 대상 테이블                                     |
| `nested_loop[].table.access_type`            | 접근 방식 (`ALL`, `range`, `ref`, `eq_ref` 등)     |
| `nested_loop[].table.possible_keys`          | 고려된 인덱스 목록                                    |
| `nested_loop[].table.rows_examined_per_scan` | 이 테이블에서 **스캔할 것으로 예측된 행(row) 수**              |
| `nested_loop[].table.rows_produced_per_join` | **조인 후 반환**될 것으로 예측된 행(row) 수                 |
| `nested_loop[].table.filtered`               | WHERE 조건 적용 후 **남는 비율**(%)                    |
| `nested_loop[].table.cost_info.read_cost`    | I/O 비용                                        |
| `nested_loop[].table.cost_info.eval_cost`    | CPU(조건 평가) 비용                                 |
| `nested_loop[].table.cost_info.prefix_cost`  | 누적(plan prefix) 비용                            |


#### 결과 해석

1. **전체 비용(≈1.19M)**
   – 이 쿼리가 처리하는 데 예상되는 상대비용이 1.19×10⁶들게 됩니다.
2. **정렬 비용(≈965K)**
   – `ORDER BY … LIMIT`에도 불구하고 `using_temporary_table`·`using_filesort`가 `true`이므로,
   Top-N 최적화가 적용되지 않은 상태임을 의미합니다.
3. **조인 방식: Nested-Loop**

   * 먼저 `e`를 풀 스캔(`access_type = ALL`, ≈300K건) → 필터 후 ≈100K건 남깁니다.
   * 각 `e.emp_no`에 대해 `s` PK 인덱스 조회(`ref`, ≈9건 스캔) → 최종 ≈965K건 조인하게 됩니다.

이처럼 Optimization 단계에서는 JSON 플랜의 비용과 통계 정보를 활용해 쿼리별 예상 비용을 비교·분석하고, 인덱스나 힌트를 통해 물리 계획을 직접 튜닝합니다.

### 3. Execution

#### 역할

* **물리 연산 수행(Physical Operations)**
  옵티마이저가 선택한 실행 계획을 바탕으로 스토리지 엔진(InnoDB 등)에서 실제 레코드를 읽고, 필터·조인·정렬·집계 등을 수행합니다.
* **버퍼 풀 활용(Buffer Pool)**
  디스크 I/O를 최소화하기 위해 메모리 캐시에서 페이지를 읽고, 필요 시 플러시하거나 디스크에서 로드합니다.
* **락·트랜잭션 관리(Locking & Transactions)**
  ACID 보장을 위해 필요한 수준의 락을 걸고 트랜잭션을 처리합니다.


#### 내부 처리 흐름

1. **테이블·인덱스 오픈**

   * 파일 핸들 열기, 메타데이터가 로드됩니다.
2. **레코드 스캔 & 필터**

   * InnoDB 버퍼 풀 → 디스크 페이지
   * `WHERE`/`ON` 조건이 적용됩니다.
3. **조인 수행**

   * **Nested-Loop**: 외부 테이블 행마다 내부 인덱스 조회합니다.
   * **Hash Join**(8.0.18+): 내부 테이블 해시 테이블을 빌드하게 됩니다.
4. **정렬·집계**

   * 메모리 정렬 시 부족하면 임시 테이블디스크가 발생됩니다.
5. **결과 반환**

   * 클라이언트에 프로토콜 패킷으로 전송

#### 실행 확인

```sql
mysql> EXPLAIN ANALYZE
    -> SELECT e.emp_no, e.first_name, s.salary
    ->   FROM employees AS e
    ->   JOIN salaries  AS s ON e.emp_no = s.emp_no
    ->  WHERE e.hire_date > '2000-01-01'
    ->  ORDER BY s.salary DESC
    ->  LIMIT 10;
```


#### 실행 결과 및 요약

```txt
-> Table scan on e            (cost=30894 rows=299645)   (actual time=4.76..88.2 rows=300024 loops=1)
-> Filter: e.hire_date > …     (cost=30894 rows=99872)    (actual time=29.6..102 rows=12 loops=1)
-> Index lookup on s           (cost=1.01 rows=9.67)      (actual time=0.379..0.38 rows=2.75 loops=12)
-> Nested loop inner join      (cost=228212 rows=965634)  (actual time=30.9..107 rows=33 loops=1)
-> Sort + Limit                (using_temporary_table;   (actual time=107..107 rows=10 loops=1)
                                 using_filesort)
-> Limit: final output        (actual time=107..107 rows=10 loops=1)
```

| 단계                | 추정 비용 & 행수               | 실제 시간(ms)    | 실제 행수   | loops |
| ----------------- | ------------------------ | ------------ | ------- | ----- |
| Table scan on e   | cost=30894, rows=299645  | 4.76 … 88.2  | 300,024 | 1     |
| Filter            | cost=30894, rows=99872   | 29.6 … 102   | 12      | 1     |
| Index lookup on s | cost=1.01, rows=9.67     | 0.379 … 0.38 | 2.75    | 12    |
| Nested loop join  | cost=228212, rows=965634 | 30.9 … 107   | 33      | 1     |
| Sort + Limit      | —                        | 107 … 107    | 10      | 1     |

#### 실제 수행 시간 해석

1. **Table scan on e (4.76 – 88.2 ms, 300 024 rows, loops=1)**

   * 이 단계에서 300K건을 한 번에 읽어오는 데 최대 88 ms가 소요되었습니다.
   * 상대적으로 긴 시간 구간(4.76→88.2 ms)이 걸린 것은, 버퍼 풀에 없는 페이지를 디스크에서 로드했거나, 캐시 부하가 있었음을 알 수 있습니다.

2. **Filter (29.6 – 102 ms, 12 rows, loops=1)**

   * 필터 후 실제 반환된 행이 12건으로, 옵티마이저 추정치(≈99K)와 큰 차이가 있습니다.

3. **Index lookup on s (0.379 – 0.380 ms × 12 loops)**

   * 루프당 0.38 ms, 총 12회 수행되어 약 4.56 ms가 이 단계에 쓰였습니다.
   * PK 인덱스를 이용한 조회는 매우 빠르지만, 루프가 많으면 누적 비용이 발생합니다.

4. **Nested loop inner join (30.9 – 107 ms, 33 rows, loops=1)**

   * 조인 전체를 완료하는 데 최대 107 ms, 최소 30.9 ms가 소요되었습니다.

5. **Sort + Limit (107 ms, 10 rows, loops=1)**

   * 최종 정렬과 LIMIT 적용에 107 ms가 걸렸습니다.
   * Top-N 최적화가 적용됐더라도 전체 결과 중 상위 10건을 위해 여전히 메모리 정렬 작업이 발생한 것으로 보입니다.

