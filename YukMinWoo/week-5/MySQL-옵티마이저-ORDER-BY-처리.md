# 2025-05-28 Deep Dive - MySQL 옵티마이저 ORDER BY 처리

## **정렬 처리**

- **실행 계획의 `Extra` 컬럼으로 어떤 방식을 적용했는지 확인 가능**

### **Using index**

- `INSERT`, `UPDATE`, `DELETE` 쿼리가 실행될 때 이미 인덱스가 정렬되어 있어서 순서대로 읽기만 하면 되므로 매우 빠름
- `INSERT`, `UPDATE`, `DELETE` 작업 시 부가적인 인덱스 추가/삭제 작업이 필요하므로 느림
- 인덱스 때문에 디스크 공간이 더 많이 필요함
- 인덱스의 개수가 늘어날수록 InnoDB 버퍼 풀을 위한 메모리가 많이 필요함

### **Using filesort**

- 인덱스를 생성하지 않아도 되므로 인덱스를 이용할 때의 단점이 장점으로 바뀜
- 정렬해야 할 레코드가 많지 않으면 메모리에서 Filesort가 처리되므로 충분히 빠름
- 정렬 작업이 쿼리 실행 시 처리되므로 레코드 대상 건수가 많아질수록 쿼리의 응답 속도가 느림

---

### Test

```sql
CREATE TABLE tests (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50),
    email VARCHAR(50),
    INDEX idx_email (email)
);

-- email 컬럼에만 index 설정
```

```bash
mysql> SELECT COUNT(*) FROM tests;
+----------+
| COUNT(*) |
+----------+
|  3000000 |
+----------+
1 row in set (0.22 sec)

-- 데이터 300만개
```

```bash
-- 대략적인 실행 시간 측정

mysql> SHOW PROFILES;
+----------+------------+--------------------------------------------------------+
| Query_ID | Duration   | Query                                                  |
+----------+------------+--------------------------------------------------------+
|        1 | 0.00021300 | SET SESSION profiling_history_size = 100               |
|        2 | 0.00022300 | SELECT DATABASE()                                      |
|        3 | 0.00311000 | show databases                                         |
|        4 | 0.00173200 | show tables                                            |
|        5 | 0.21561600 | SELECT COUNT(*) FROM tests                             |
|        6 | 2.33318800 | SELECT * FROM tests ORDER BY name                      |
|        7 | 2.44952500 | SELECT * FROM tests ORDER BY email                     |
|        8 | 1.83069900 | SELECT name FROM tests ORDER BY name                   |
|        9 | 0.42359100 | SELECT email FROM tests ORDER BY email                 |
|       10 | 0.00195900 | EXPLAIN SELECT * FROM tests ORDER BY name              |
|       11 | 0.00024700 | EXPLAIN SELECT * FROM tests ORDER BY email             |
|       12 | 0.00021200 | EXPLAIN SELECT name FROM tests ORDER BY name           |
|       13 | 0.00023900 | EXPLAIN SELECT email FROM tests ORDER BY email         |
|       14 | 2.16183200 | EXPLAIN ANALYZE SELECT * FROM tests ORDER BY name      |
|       15 | 2.76256700 | EXPLAIN ANALYZE SELECT * FROM tests ORDER BY email     |
|       16 | 1.83411700 | EXPLAIN ANALYZE SELECT name FROM tests ORDER BY name   |
|       17 | 0.32023700 | EXPLAIN ANALYZE SELECT email FROM tests ORDER BY email |
+----------+------------+--------------------------------------------------------+
17 rows in set, 1 warning (0.00 sec)

-- SELECT COUNT(*) FROM tests             -> 0.21561600 sec

-- SELECT * FROM tests ORDER BY name      -> 2.33318800 sec
-- SELECT * FROM tests ORDER BY email     -> 2.44952500 sec

-- SELECT name FROM tests ORDER BY name   -> 1.83069900 sec
-- SELECT email FROM tests ORDER BY email -> 0.42359100 sec
```

---

```bash
mysql> EXPLAIN SELECT * FROM tests ORDER BY name;
+----+-------------+-------+------------+------+---------------+------+---------+------+---------+----------+----------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra          |
+----+-------------+-------+------------+------+---------------+------+---------+------+---------+----------+----------------+
|  1 | SIMPLE      | tests | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 2992104 |   100.00 | Using filesort |
+----+-------------+-------+------------+------+---------------+------+---------+------+---------+----------+----------------+
```

```bash
mysql> EXPLAIN SELECT * FROM tests ORDER BY email;
+----+-------------+-------+------------+------+---------------+------+---------+------+---------+----------+----------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra          |
+----+-------------+-------+------------+------+---------------+------+---------+------+---------+----------+----------------+
|  1 | SIMPLE      | tests | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 2992104 |   100.00 | Using filesort |
+----+-------------+-------+------------+------+---------------+------+---------+------+---------+----------+----------------+

-- email에 index가 있지만 index를 통해 SELECT의 모든 데이터를 가져올 수 없어서(커버링 인덱스 X) filesort가 적용됨
```

```bash
mysql> EXPLAIN SELECT name FROM tests ORDER BY name;
+----+-------------+-------+------------+------+---------------+------+---------+------+---------+----------+----------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra          |
+----+-------------+-------+------------+------+---------------+------+---------+------+---------+----------+----------------+
|  1 | SIMPLE      | tests | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 2992104 |   100.00 | Using filesort |
+----+-------------+-------+------------+------+---------------+------+---------+------+---------+----------+----------------+
```

```bash
mysql> EXPLAIN SELECT email FROM tests ORDER BY email;
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+---------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key       | key_len | ref  | rows    | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+---------+----------+-------------+
|  1 | SIMPLE      | tests | NULL       | index | NULL          | idx_email | 203     | NULL | 2992104 |   100.00 | Using index |
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+---------+----------+-------------+

-- Using index가 적용되어 조회 성능 향상
```

```bash
mysql> EXPLAIN ANALYZE SELECT * FROM tests ORDER BY name;
+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| EXPLAIN                                                                                                                                                                                         |
+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| -> Sort: tests.`name`  (cost=310959 rows=2.99e+6) (actual time=1934..2065 rows=3e+6 loops=1)
    -> Table scan on tests  (cost=310959 rows=2.99e+6) (actual time=0.215..434 rows=3e+6 loops=1)
 |
+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (2.16 sec)

mysql> EXPLAIN ANALYZE SELECT * FROM tests ORDER BY email;
+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| EXPLAIN                                                                                                                                                                                         |
+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| -> Sort: tests.email  (cost=308728 rows=2.99e+6) (actual time=2191..2326 rows=3e+6 loops=1)
    -> Table scan on tests  (cost=308728 rows=2.99e+6) (actual time=0.0577..420 rows=3e+6 loops=1)
 |
+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (2.76 sec)

mysql> EXPLAIN ANALYZE SELECT name FROM tests ORDER BY name;
+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| EXPLAIN                                                                                                                                                                                        |
+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| -> Sort: tests.`name`  (cost=308728 rows=2.99e+6) (actual time=1719..1780 rows=3e+6 loops=1)
    -> Table scan on tests  (cost=308728 rows=2.99e+6) (actual time=0.24..345 rows=3e+6 loops=1)
 |
+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (1.84 sec)

mysql> EXPLAIN ANALYZE SELECT email FROM tests ORDER BY email;
+-------------------------------------------------------------------------------------------------------------------------+
| EXPLAIN                                                                                                                 |
+-------------------------------------------------------------------------------------------------------------------------+
| -> Covering index scan on tests using idx_email  (cost=308728 rows=2.99e+6) (actual time=0.052..268 rows=3e+6 loops=1)
 |
+-------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.32 sec)
```

---

## **소트 버퍼(Sort Buffer)**

- MySQL이 정렬을 수행하기 위해 할당 받는 별도의 메모리 공간으로 쿼리 실행이 완료되면 즉시 시스템으로 반납
- 소트 버퍼의 크기는 정렬해야 할 레코드의 크기에 따라 가변적으로 증가하지만 최대 사용 가능한 공간은 `sort_buffer_size`라는 시스템 변수로 설정 가능
- 소트 버퍼의 크기가 커져도 처리 성능이 항상 빨리지는 것은 아니며 소트 버퍼는 세션 메모리 영역이어서 너무 크게 설정하면 여러 커넥션이 동시에 실행될 때 메모리 부족 현상이 생길 수 있음
- 정렬해야 할 레코드의 건수가 소트 버퍼로 할당된 공간보다 크면 **임시 저장을 위해 디스크를 사용**
- 정렬해야 할 레코드의 건수가 소트 버퍼로 할당된 공간보다 크면 임시 저장을 위해 디스크를 사용하며 레코드를 여러 조각으로 나누어 소트 버퍼에서 정렬을 수행하고, 그 결과를 임시로 디스크에 기록해두며 다음 레코드를 가져와서 다시 정렬하고 임시 저장하는 과정을 반복
- 정렬된 레코드를 다시 병합하며 정렬을 수행하고 이 병합 작업을 **멀티 머지(multi merge)**라고 함

```bash
-- p.291

mysql> SELECT name FROM tests ORDER BY name

*************************** 1. row ***************************
                            QUERY: SELECT name FROM tests ORDER BY name
                            TRACE: {
  "steps": [
    {
      "join_preparation": {
        "select#": 1,
        "steps": [
          {
            "expanded_query": "/* select#1 */ select `tests`.`name` AS `name` from `tests` order by `tests`.`name`"
          }
        ] /* steps */
      } /* join_preparation */
    },
    {
      "join_optimization": {
        "select#": 1,
        "steps": [
          {
            "substitute_generated_columns": {
            } /* substitute_generated_columns */
          },
          {
            "table_dependencies": [
              {
                "table": "`tests`",
                "row_may_be_null": false,
                "map_bit": 0,
                "depends_on_map_bits": [
                ] /* depends_on_map_bits */
              }
            ] /* table_dependencies */
          },
          {
            "rows_estimation": [
              {
                "table": "`tests`",
                "table_scan": {
                  "rows": 2992104,
                  "cost": 10289.4
                } /* table_scan */
              }
            ] /* rows_estimation */
          },
          {
            "considered_execution_plans": [
              {
                "plan_prefix": [
                ] /* plan_prefix */,
                "table": "`tests`",
                "best_access_path": {
                  "considered_access_paths": [
                    {
                      "rows_to_scan": 2992104,
                      "access_type": "scan",
                      "resulting_rows": 2.9921e+06,
                      "cost": 309500,
                      "chosen": true,
                      "use_tmp_table": true
                    }
                  ] /* considered_access_paths */
                } /* best_access_path */,
                "condition_filtering_pct": 100,
                "rows_for_plan": 2.9921e+06,
                "cost_for_plan": 309500,
                "sort_cost": 2.9921e+06,
                "new_cost_for_plan": 3.3016e+06,
                "chosen": true
              }
            ] /* considered_execution_plans */
          },
          {
            "attaching_conditions_to_tables": {
              "original_condition": null,
              "attached_conditions_computation": [
              ] /* attached_conditions_computation */,
              "attached_conditions_summary": [
                {
                  "table": "`tests`",
                  "attached": null
                }
              ] /* attached_conditions_summary */
            } /* attaching_conditions_to_tables */
          },
          {
            "optimizing_distinct_group_by_order_by": {
              "simplifying_order_by": {
                "original_clause": "`tests`.`name`",
                "items": [
                  {
                    "item": "`tests`.`name`"
                  }
                ] /* items */,
                "resulting_clause_is_simple": true,
                "resulting_clause": "`tests`.`name`"
              } /* simplifying_order_by */
            } /* optimizing_distinct_group_by_order_by */
          },
          {
            "finalizing_table_conditions": [
            ] /* finalizing_table_conditions */
          },
          {
            "refine_plan": [
              {
                "table": "`tests`"
              }
            ] /* refine_plan */
          },
          {
            "considering_tmp_tables": [
              {
                "adding_sort_to_table": "tests"
              } /* filesort */
            ] /* considering_tmp_tables */
          }
        ] /* steps */
      } /* join_optimization */
    },
    {
      "join_execution": {
        "select#": 1,
        "steps": [
          {
            "sorting_table": "tests",
            "filesort_information": [
              {
                "direction": "asc",
                "expression": "`tests`.`name`"
              }
            ] /* filesort_information */,
            "filesort_priority_queue_optimization": {
              "usable": false,
              "cause": "not applicable (no LIMIT)"
            } /* filesort_priority_queue_optimization */,
            "filesort_execution": [
            ] /* filesort_execution */,
            "filesort_summary": {
              "memory_available": 262144,
              "key_size": 809,
              "row_size": 1015,
              "max_rows_per_buffer": 258,
              "num_rows_estimate": 2992104,
              "num_rows_found": 3000000,
              "num_initial_chunks_spilled_to_disk": 803,
              "peak_memory_used": 294912,
              "sort_algorithm": "std::sort",
              "sort_mode": "<varlen_sort_key, packed_additional_fields>"
            } /* filesort_summary */
          }
        ] /* steps */
      } /* join_execution */
    }
  ] /* steps */
}
MISSING_BYTES_BEYOND_MAX_MEM_SIZE: 0
          INSUFFICIENT_PRIVILEGES: 0
1 row in set (0.01 sec)

-- "sort_algorithm": "std::sort" -> 정렬 알고리즘
-- "sort_mode": "<varlen_sort_key, packed_additional_fields>" -> 정렬 방식

-- "num_rows_estimate": 2992104 -> 옵티마이저가 예상한 레코드 수
-- "num_rows_found": 3000000 -> 발견된 레코드 수

-- "memory_available": 262144 -> 소트 버퍼의 크기
-- "row_size": 1015 -> 각 레코드의 정렬 대상 크기
-- "max_rows_per_buffer": 258 -> 청크당 최대 행 수
-- "num_initial_chunks_spilled_to_disk": 803 -> 디스크에 기록된 청크 수

-- 300만개 중 디스크에 청크 수(803) * 청크당 레코드 수(258) 20만개가 저장된걸로 보이는데 남은 280만개는 소트 버퍼가 아닌 메모리에 위치한다고 함(GPT 피셜)
```
