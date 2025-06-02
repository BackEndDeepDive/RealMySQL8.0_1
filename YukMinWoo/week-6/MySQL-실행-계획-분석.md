# 2025-06-02 Deep Dive - MySQL 실행 계획 분석

### 문제 상황

- 회원들의 이메일 도메인 별 팔로워 수와 팔로잉 수 통계를 보고 싶은 상황
- 유저수 1만명, 관계수 500만개인 상황(유저당 평균 팔로잉, 팔로워 수 500명)

### DB 스키마

```SQL
CREATE TABLE test_user (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nickname VARCHAR(50),
    email VARCHAR(100) NOT NULL UNIQUE
)

CREATE TABLE test_follow (
    follower_id INT,
    followee_id INT,
    PRIMARY KEY (follower_id, followee_id),
    FOREIGN KEY (follower_id) REFERENCES test_user(id) ON DELETE CASCADE,
    FOREIGN KEY (followee_id) REFERENCES test_user(id) ON DELETE CASCADE
)
```

```bash
mysql> DESC test_user;
+----------+--------------+------+-----+---------+----------------+
| Field    | Type         | Null | Key | Default | Extra          |
+----------+--------------+------+-----+---------+----------------+
| id       | int          | NO   | PRI | NULL    | auto_increment |
| nickname | varchar(50)  | YES  |     | NULL    |                |
| email    | varchar(100) | NO   | UNI | NULL    |                |
+----------+--------------+------+-----+---------+----------------+
3 rows in set (0.00 sec)

mysql> DESC test_follow;
+-------------+------+------+-----+---------+-------+
| Field       | Type | Null | Key | Default | Extra |
+-------------+------+------+-----+---------+-------+
| follower_id | int  | NO   | PRI | NULL    |       |
| followee_id | int  | NO   | PRI | NULL    |       |
+-------------+------+------+-----+---------+-------+
2 rows in set (0.01 sec)
```

### 레코드 수

```bash
mysql> SELECT COUNT(*)  FROM test_user;
+----------+
| COUNT(*) |
+----------+
|    10000 |
+----------+
1 row in set (0.00 sec)

# 유저 1만명

mysql> SELECT COUNT(*)  FROM test_follow;
+----------+
| COUNT(*) |
+----------+
|  5000000 |
+----------+
1 row in set (0.34 sec)

# 관계 수 500만개
```

### 쿼리

- 이메일 도메인별 유저 수와 각 도메인 별 유저들의 평균 팔로잉 팔로워 수 구하는 쿼리

```SQL
SELECT
    SUBSTRING_INDEX(u.email, '@', -1) AS domain,
    COUNT(DISTINCT u.id) AS user_count,
    ROUND(AVG(follower_counts.cnt), 2) AS avg_followers,
    ROUND(AVG(following_counts.cnt), 2) AS avg_followings
FROM test_user u
LEFT JOIN (
    SELECT followee_id AS user_id, COUNT(*) AS cnt
    FROM test_follow
    GROUP BY followee_id
) AS follower_counts ON u.id = follower_counts.user_id
LEFT JOIN (
    SELECT follower_id AS user_id, COUNT(*) AS cnt
    FROM test_follow
    GROUP BY follower_id
) AS following_counts ON u.id = following_counts.user_id
GROUP BY domain
ORDER BY user_count DESC;
```

```bash
+-------------+------------+---------------+----------------+
| domain      | user_count | avg_followers | avg_followings |
+-------------+------------+---------------+----------------+
| example.org |       3365 |        499.83 |         499.86 |
| example.net |       3326 |        500.29 |         499.78 |
| example.com |       3309 |        499.89 |         500.36 |
+-------------+------------+---------------+----------------+
3 rows in set (0.88 sec)

# 3개의 도메인 유저 수 합 1만
```

### 실행 계획

```bash
+----+-------------+-------------+------------+-------+---------------------+-------------+---------+--------------+---------+----------+----------------------------------------------+
| id | select_type | table       | partitions | type  | possible_keys       | key         | key_len | ref          | rows    | filtered | Extra                                        |
+----+-------------+-------------+------------+-------+---------------------+-------------+---------+--------------+---------+----------+----------------------------------------------+
|  1 | PRIMARY     | u           | NULL       | index | email               | email       | 402     | NULL         |    9621 |   100.00 | Using index; Using temporary; Using filesort |
|  1 | PRIMARY     | <derived2>  | NULL       | ref   | <auto_key0>         | <auto_key0> | 4       | pollock.u.id |     492 |   100.00 | NULL                                         |
|  1 | PRIMARY     | <derived3>  | NULL       | ref   | <auto_key0>         | <auto_key0> | 4       | pollock.u.id |     492 |   100.00 | NULL                                         |
|  3 | DERIVED     | test_follow | NULL       | index | PRIMARY,followee_id | PRIMARY     | 8       | NULL         | 4742182 |   100.00 | Using index                                  |
|  2 | DERIVED     | test_follow | NULL       | index | PRIMARY,followee_id | followee_id | 4       | NULL         | 4742182 |   100.00 | Using index                                  |
+----+-------------+-------------+------------+-------+---------------------+-------------+---------+--------------+---------+----------+----------------------------------------------+
5 rows in set, 1 warning (0.00 sec)
```

```bash
 1. -> Sort: user_count DESC  (actual time=1105..1105 rows=3 loops=1)
 2.     -> Stream results  (cost=177e+9 rows=938307) (actual time=1096..1105 rows=3 loops=1)
 3.         -> Group aggregate: avg(following_counts.cnt), avg(follower_counts.cnt), count(distinct u.id)  (cost=177e+9 rows=938307) (actual time=1096..1105 rows=3 loops=1)
 4.             -> Nested loop left join  (cost=88.6e+9 rows=880e+9) (actual time=1092..1103 rows=10000 loops=1)
 5.                 -> Nested loop left join  (cost=10.5e+6 rows=93.3e+6) (actual time=540..546 rows=10000 loops=1)
 6.                     -> Sort: domain  (cost=1057 rows=9621) (actual time=11.8..12.1 rows=10000 loops=1)
 7.                         -> Covering index scan on u using email  (cost=1057 rows=9621) (actual time=0.201..3.46 rows=10000 loops=1)
 8.                     -> Index lookup on follower_counts using <auto_key0> (user_id=u.id)  (cost=960118..960241 rows=493) (actual time=0.0533..0.0533 rows=1 loops=10000)
 9.                         -> Materialize  (cost=960117..960117 rows=9698) (actual time=529..529 rows=10000 loops=1)
10.                             -> Group aggregate: count(0)  (cost=959148 rows=9698) (actual time=2.47..525 rows=10000 loops=1)
11.                                 -> Covering index scan on test_follow using followee_id  (cost=484929 rows=4.74e+6) (actual time=2.35..420 rows=5e+6 loops=1)
12.                 -> Index lookup on following_counts using <auto_key0> (user_id=u.id)  (cost=960091..960214 rows=493) (actual time=0.0555..0.0556 rows=1 loops=10000)
13.                     -> Materialize  (cost=960091..960091 rows=9436) (actual time=551..551 rows=10000 loops=1)
14.                         -> Group aggregate: count(0)  (cost=959148 rows=9436) (actual time=0.984..547 rows=10000 loops=1)
15.                             -> Covering index scan on test_follow using PRIMARY  (cost=484929 rows=4.74e+6) (actual time=0.957..451 rows=5e+6 loops=1)


# 전체 쿼리 실행 시간 1.11sec
# 7번에서 email의 index로 커버링 인덱스 스캔이 적용되어 성능 향상이 있지만 유저 수 1만명에서 큰 의미 없음 (커버링 인덱스 스캔이 적용되지 않았을 경우 향후 성능 개선 포인트)
# 8번, 10번에서 팔로워 그룹 집계에 약 500ms, 팔로잉 그룹 집계에 약 500ms 정도로 전체 쿼리의 90% 이상 소요 (성능 개선 포인트)
```

### 해결 방법

- 매번 지금처럼 집계에서 제공
  -> 팔로잉, 팔로워 수는 아주 정확한 데이터까지는 필요하지 않아서 장기적으로 보면 안좋아 보임
- 매번 집계 함수로 통계를 내는건 자원 소요가 많으므로 주기적으로 통계를 내고 결과를 반환(정확한 값이 크게 중요하지 않을 때)
  -> 구현이 간단한 장점은 있지만 실시간성이 떨어짐
- 팔로잉 팔로워 수 집계 테이블을 만들고 팔로우 언팔로우 기점에 집계 테이블 갱신 및 조회시 집계 테이블과 조인(동기화 문제, 트랜잭션 처리 로직 추가)
  -> 구현이 좀 더 복잡해지지만 확장성이 가장 좋아 보임

```SQL
CREATE TABLE user_follow_count (
    user_id BIGINT PRIMARY KEY,
    follower_count INT NOT NULL DEFAULT 0,
    following_count INT NOT NULL DEFAULT 0
);
```

```SQL
SELECT
    SUBSTRING_INDEX(u.email, '@', -1) AS domain,
    COUNT(*) AS user_count,
    ROUND(AVG(IFNULL(ufc.follower_count, 0)), 2) AS avg_followers,
    ROUND(AVG(IFNULL(ufc.following_count, 0)), 2) AS avg_followings
FROM test_user u
LEFT JOIN user_follow_count ufc ON u.id = ufc.user_id
GROUP BY domain
ORDER BY user_count DESC;
```

```bash
+-------------+------------+---------------+----------------+
| domain      | user_count | avg_followers | avg_followings |
+-------------+------------+---------------+----------------+
| example.org |       3365 |        499.83 |         499.86 |
| example.net |       3326 |        500.29 |         499.78 |
| example.com |       3309 |        499.89 |         500.36 |
+-------------+------------+---------------+----------------+
3 rows in set (0.02 sec)
```
