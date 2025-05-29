<div align="center">

# Deep Dive  
Deep Dive into Real My SQL 8.0(1)

--- 
</div>

## Contents
#### 1주차(2025.04.21 ~ 2025.04.27)
- **범위** 
    - 1.1 MySQL 소개 ~ 2.4.6 'my.conf' 파일 

- **세미나**
    - 장우:[MySQL 8.0에서 my.cnf(default)는 어떻게 작성되었을까?](https://github.com/BackEndDeepDive/RealMySQL8.0_1/blob/main/ChoiJangWoo/Ch%201/README.md)
    - 민우:[MySQL 서버 연결](https://github.com/BackEndDeepDive/RealMySQL8.0_1/blob/main/YukMinWoo/MySQL-%EC%84%9C%EB%B2%84-%EC%97%B0%EA%B2%B0.md) 
    - 현재:[InnoDB에 대한 고찰](https://github.com/BackEndDeepDive/RealMySQL8.0_1/blob/main/kimhyeonjae/1%EC%A3%BC%EC%B0%A8.%20InnoDB%EC%97%90%20%EB%8C%80%ED%95%9C%20%EA%B3%A0%EC%B0%B0.md)
    - 재현: [MySQL 서버 업그레이드](https://github.com/BackEndDeepDive/RealMySQL8.0_1/blob/main/KimJaeHyun/1주차.MySQL_서버_업그레이드.md) 

#### 2주차(2025.04.28 ~ 2025.05.06)
- **범위** 
    - 4.1 MySQL 엔진 아키텍처 ~ 4.4 MySQL 로그 파일

- **세미나**
    - 장우: [MVCC & Non-locking Consistent Read](https://github.com/BackEndDeepDive/RealMySQL8.0_1/blob/choijangwoo/ChoiJangWoo/Ch%202/README.md)
    - 민우: [MVCC](https://github.com/BackEndDeepDive/RealMySQL8.0_1/blob/main/YukMinWoo/week-2/MVCC.md)
    - 현재: [트랜잭션과 격리 수준](https://github.com/BackEndDeepDive/RealMySQL8.0_1/blob/main/kimhyeonjae/2%EC%A3%BC%EC%B0%A8.%20%ED%8A%B8%EB%9E%9C%EC%9E%AD%EC%85%98%EA%B3%BC%20%EA%B2%A9%EB%A6%AC%20%EC%88%98%EC%A4%80.md)
    - 재현: [MySQL 전체 워크플로우](https://github.com/BackEndDeepDive/RealMySQL8.0_1/blob/main/KimJaeHyun/2주차.MySQL_전체_워크플로우.md) 

#### 3주차(2025.05.07 ~ 2025.05.14)
- **범위** 
    - 3 사용자 및 권한, 5 트랜잭션과 잠금, 6 데이터 압축

- **세미나**
    - 장우: [Lost updates 와 Lock 근데 CAS를 곁들인](https://github.com/BackEndDeepDive/RealMySQL8.0_1/tree/choijangwoo/ChoiJangWoo/Ch%203)
    - 민우: [Lock](https://github.com/BackEndDeepDive/RealMySQL8.0_1/blob/main/YukMinWoo/week-3/Lock.md)
    - 현재: [MySQL InnoDB의 인덱스 잠금 메커니즘 테스트 및 분석](https://github.com/BackEndDeepDive/RealMySQL8.0_1/blob/main/kimhyeonjae/3%EC%A3%BC%EC%B0%A8.%20InnoDB%20%EC%9D%B8%EB%8D%B1%EC%8A%A4%20%EC%9E%A0%EA%B8%88%20%ED%85%8C%EC%8A%A4%ED%8A%B8.md)
    - 재현: [낙관적 락과 비관적 락](https://github.com/BackEndDeepDive/RealMySQL8.0_1/blob/main/KimJaeHyun/3주차.낙관적_락과_비관적_락.md) 

#### 4주차(2025.05.14 ~ 2025.05.21)
- **범위** 
    - 7 데이터 암호화, 8 인덱스

- **세미나**
    - 장우: [은총알은 없다](https://github.com/BackEndDeepDive/RealMySQL8.0_1/tree/choijangwoo/ChoiJangWoo/Ch%204)
    - 민우: [Index](https://github.com/BackEndDeepDive/RealMySQL8.0_1/blob/main/YukMinWoo/week-4/Clustered%20Index%20%26%20Non-Clustered%20Index.md)
    - 현재: [MySQL vs Elasticsearch 검색 비교](https://github.com/BackEndDeepDive/RealMySQL8.0_1/blob/main/kimhyeonjae/4%EC%A3%BC%EC%B0%A8.%20MySQL%20vs%20Elasticsearch%20%EA%B2%80%EC%83%89%20%EB%B9%84%EA%B5%90.md) 
    - 재현: [쿼리 실행 계획](https://github.com/BackEndDeepDive/RealMySQL8.0_1/blob/main/KimJaeHyun/4주차.쿼리_실행_계획.md) 

#### 5주차(2025.05.21 ~ 2025.05.28)
- **범위** 
    - 9 옵티마이저와 힌드 

- **세미나**
    - 장우: 
    - 민우: 
    - 현재:  
    - 재현: [인덱스 스킵 스캔](https://github.com/BackEndDeepDive/RealMySQL8.0_1/blob/main/KimJaeHyun/5주차.인덱스_스킵_스캔.md) 

## Conventions
1. 본인이 정리한 부분은 본인의 이름 브랜치에 기록한다.
    - 브랜치는 영어(소문자)로 설정

    ```
    main(default)
    choijangwoo
    yukminwoo
    kimjaehyun
    kimhyunjae
    ```

2. 각 주차 **세미나** 부분의 개인 링크를 수정할 때는 main에서 브랜치를 분기하여 readme를 수정 후 PR을 작성한 뒤 분기한 브랜치를 삭제한다.

    - 세미나 부분의 링크는 아래와 같은 형식으로 작성한다.
    ```
    장우:[담당 부분 제목](자료 링크)
    ```

---

## Contributors

|<img src="https://github.com/choijw1004.png" width="150" height="150"/>|<img src="https://github.com/FickleBoBo.png" width="150" height="150"/>|<img src="https://github.com/KimJ4ehyun.png" width="150" height="150"/>|<img src="https://github.com/Kguswo.png" width="150" height="150"/>|
|:-:|:-:|:-:|:-:|
|최장우<br/>[@choijw1004](https://github.com/choijw1004)|육민우<br/>[@minwoo_Yuk](https://github.com/FickleBoBo)|김재현<br/>[@KimJ4ehyun](https://github.com/KimJ4ehyun)|김현재<br/>[@Kguswo](https://github.com/Kguswo)|
