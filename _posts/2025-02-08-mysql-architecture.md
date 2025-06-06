---
title: MySQL Achitecture
date: 2025-02-09 17:00:00 +0900
categories: [DB]
math: true
mermaid: true
---

### 4.1 MySQL 전체 아키텍처  
- MySql 엔진 
  - SQL 인터페이스, SQL 파서, SQL 옵티마이저, 캐시/버퍼 
- 스토리지 엔진 (핸들러)
  - InnoDB, MyISAM, Memory 

### 4.2 InnoDB 엔진 
- 레코드 기반 잠금 
- pk 기준으로 클러스터링 됨 (ordering)
- 다른 인덱스는 레코드 주소 대신 pk 값을 논리적인 주소로 사용
  - 실행 계획시 pk 가 우선순위가 높다 
- 참고로 MyISAM 에서는 pk 인덱스와 타 인덱스 구조에 차이가 없다 
- fk 는 InnoDB 에서만 사용 가능 
- MVCC (Multi Version Concurrency Control) 지원하여 일관된 읽기 제공
  - Undo 로그 사용하는데 트랜잭션이 길어지면 Undo 로그 공간을 많이 사용하여 문제가 될 수 있다  
- 자동 데드락 감지 스레드가 교착상태 트랜잭션을 찾아서 한개를 강제 종료 시킨다 
  - Undo 로그 적은 쪽을 종료시킴 
- InnoDB 버퍼 풀은 데이터, 인덱스를 캐시해두는 공간이다 
  - 일반적으로 전체 메모리의 50% 를 할당해서 필요하면 조정하는걸 추천 
  - 데이터가 변경되면 Redo 로그에 내용이 기록되고 그 내용이 버퍼풀에 반영된다 
  - 버퍼 풀에는 이미 실제 값이 변경된 Dirty 데이터가 캐시 되어 있을 수 있다 
  - 변경된 버퍼 풀의 데이터는 디스크에 동기화 되는데 성능에 이슈가 발생할 수 있다 
- Adaptive Hash Index (AHI) 를 자동으로 생성하여 자주 사용되는 데이터는 B-Tree 인덱스 사용전에 AHI 를 먼저 활용한다 
   
### 5.2 엔진의 잠금
- 스토리지 레벨의 잠금과 엔진 레벨의 잠금으로 나눌 수 있다 
- 글로벌 락
  - 대부분의 DDL/DML 에서 전체 서버에 영향, 백업관련 작업에서 발생 
- 테이블 락 
  - InnoDB 기준 스키마 변경시 발생 
- 네임드 락 
  - 특정 문자열에 락을 설정
  - 분산락을 처리할때 활용될 수 있겠다 
- 메타데이터 락 
  - 데이터베이스 객체의 이름이나 구조를 변경할때 발생 
  - 기존 테이블에 설계 변화가 커서 테이블을 새로 만들어서 마이그레이션할때 데이터 이관후 신규 테이블을 RENAME 하면 다운타임 최소화 
- InnoDB 스토리지 엔진 잠금 
  - 레코드 락
    - 레코드 기반 잠금 방식, 레코드가 아닌 인덱스의 레코드를 잠궈서 잠김의 범위를 최소화 
  - 갭 락 
    - 레코드와 인접한 레코드 사이의 간격을 잠궈서 insert 제어 
  - 넥스트 키 락 
    - 레코드 락과 갭 락을 합쳐놓은 형태 
  - 자동 증가 락 
    - auto_increment 에 대한 락 
   
### 5.4 격리 수준 
- READ UNCOMMITTED 
- READ COMMITTED 
  - 커밋 된거만 읽는다 
  - 트랜잭션이 긴경우 하나의 트랜잭션 내에서 누군가 커밋해논 내용이 없었다가 생길수 있다 
  - 하나의 트랜잭션 내에서 select 에 대한 정합성이 유지되지 않는다 
- REPEATABLE READ
  - InnoDB 기본 격리 수준 
  - 하나의 트랜잭션 내에서는 undo 를 읽기 때문에 개별 레코드에 대해서 일관된 읽기를 제공 
  - undo 가 너무 커지면 부하가 있기 때문에 트랜잭션은 짧게 가져가자 
  - 트랜잭션 내에서 특정 범위로 select for update 구문을 사용해도 특정범위 전체에 lock 이 걸리지 않는다 
    - undo 에 잠금을 걸 수 없기 때문이다 
    - 특정 범위 내에서 다른 트랜잭션이 커밋한 데이터가 보였다가 안보였다 할 수 있다 
- SERIALIZABLE
  - 한 트랜잭션에서 읽고 쓰는 모든 레코드를 잠금 

### 8.2 인덱스 
- 역할별로는 pk와 secondary key 로 나눈다 
- B-Tree 
  - 루트 - 브랜치 - 리프 노드로 구성 
  - 정렬된 상태, insert/update/delete 에 대한 비용이 있지만 select 에 강점 
  - 유니크한 값의 수가 많은가? 기수성(cardinallity) 높아야 성능 향상 
  - 전체레코드의 20~25% 를 읽으면 손익분기점을 못넘는다 인덱스를 안쓰는게 좋은 상황 
  - index range scan 
  - index full scan 
  - loose index scan 
    - 조건에 만족하지 않는 레코드 알아서 스킵 
  - index skip scan 
    - 결합인덱스 중에 앞에 있는 컬럼을 활용안하고 두번째 컬럼에 대한 인덱스만 검색 
- R-Tree 
  - 2차원 공간값에 대한 인덱스 
- Full Text 
  - MeCab 사용한 어근 분석, n-gram 분석   
- function index 
  - 인덱싱할 값을 함수로 계산하여 그값을 내부적 B-Tree 인덱스와 유사하게 구성 
- multi value index 
  - json 형태에 데이터를 인덱싱하여 그 json 내부 값을 인덱스 조건으로 활용가능 

### 8.8 클러스터링 인덱스 
- pk 에 대해서만 적용 
- 값이 비슷한 레코드끼리 묶어서 저장 
- pk 가 없는 테이블은 적절한 조건에 부합하는 다른 컬럼이 클러스터링 키로 선택됨 
- 세컨더리 인덱스는 데이터 주소 없이 pk 값을 가지도록 구현됨 
- 유니크 인덱스는 제약조건이고 실제 인덱스가 아니다 
- fk 는 자동으로 인덱스를 생성한다 

### 9.2 기본 데이터 처리 
- 소트 버퍼 
  - order by (using filesort) 할때 사용되는데 크기가 크다고 해서 성능이 빨라지지는 않는다 
  - 56kb ~ 1MB 적당, 각 세션별로 메모리를 소비하기 때문이다 
- 정렬 방식 
  - 싱글 패스, select 하는 모든 컬럼을 소트 버퍼에 담고 정렬 
  - 투 패스, 정렬 대상 컬럼과 pk 만 소트 버퍼에 담아서 정렬하고 결과로 다시 select 수행 
- 실행 계획의 extra 에 정렬 방법이 3가지중 하나로 표시됨 아래로 갈수록 느린 방법 
  - 표시 없을때, 인덱스 사용한 정렬  
  - 조인에서 드라이빙 테이블만 정렬할때, Using filesort 
  - 조인 결과를 임시테이블로 저장 후 정렬, Using temporary;Using filesort
- 가능하면 인덱스사용 정렬만 되도록하고 그렇지 못하면 드라이빙 테이블만 정렬하는 형태로 튜닝하자 
- 인덱스 스캔을 사용하는 group by 
  - 드라이빙 테이블의 컬럼만으로 grouping 할때 이미 인덱스가 있는 경우 
- 루스 인덱스 스캔을 이용한 group by 
  - 예를 들어 A + B 컬럼 두개 결합인덱스 일때 where B = 1 조건이 있지만 group by A 라면 
  - extra 컬럼에 Using where; Using index for group by 루스 인덱스 스캔이 발생 
- 임시 테이블 사용하는 group by 
  - 인덱스를 전혀 활용하지 못할때 extra 정보에 Using temporary
- 임시 테이블은 몇몇 조건을 만족하면 메모리가 아닌 디스크에 만들어진다 
  - 512바이트 이상 컬럼이 group by, distinct, union select 에서 사용되는 경우 
  - 메모리 임시 테이블 크기가 시스템 변수보다 큰 경우 

### 9.3 고급 최적화 
- block nested loop 
  - 드리븐 테이블의 풀스캔, 인덱스풀스캔을 피할 수 없을때 한번 읽은 드리븐 테이블 내용을 메모리 (join buffer) 에 담아서 조인 
- 세미 조인 
  - 실제 조인 하지 않고 일치하는 조건이 있는지만 체크 
- table pull-out 
  - subquery 에 사용된 테이블을 outer 쿼리로 재생성 
- first match 
  - in subquery 형태의 세미 조인을 exists subquery 형태로 실행 
- loose scan 
  - 서브 쿼리쪽에 루스 인덱스 스캔이 사용할 수 있는 경우 먼저 읽고 outer 테이블처럼 사용하데 먼저읽어서 드리븐할 수 있도록 함 
- materialization (구체화)
  - 서브 쿼리를 임시테이블로 생성하여 활용 
- duplicated weed-out 
  - 세비 조인 서브쿼리를 inner join 으로 바꾸고 중복을 제거하는 형태 
- skip scan 
  - A + B 컬럼 결합인덱스는 있는데 B 컬럼에 대한 조건만 있는 경우 A 컬럼값을 가져와서 활용 
- hash join 
  - nl 보다 항상 빠른건 아니다 
  - 첫 레코드 찾는데는 느리지만 최종 레코드 찾는데 까지는 빠르다 
  - 메모리에 해시 테이블 생성하여 조인
- index extention
  - 세컨더리 인덱스에 자동으로 pk 를 추가한 형태의 인덱스를 사용할 수 있게 해줌 
- index merge intersection
  - 두 조건이 각각 인덱스가 있는데 and 연산자인 경우 최적화  
- index merge union
  - 두 조건이 각각 인덱스가 있는데 or 연산자로 연결된 경우 최적화 
- index merge union sort 
  - index merge union 후 정렬   
- index condition push down 
  - 컬럼 앞부분 like 검색 같은 범위 조건이 있을때 컬럼에 인덱스는 있지만 활용못하는 조건이다 
  - 이 경우 디스크 i/o 가 많이 발생하는게 정상이지만 그렇지 않고 인덱스만 읽어서 해당 조건을 처리하도록 최적화 한다 
  - extra 정보에 Using index condition 이라고 표시됨    

### 9.4 쿼리 힌트 
- straght_join
  - 조인 순서를 고정, from 절에 명시된 테이블 순서대로 
  - 순서를 명시하는 힌트도 있다 join_order, join_prefix, join_suffix
- use index, force index, ignore index 
  - 힌트를 준다고 해서 무조건 따르지는 않는다 
  - use index for join, for order by, for group by 처럼 특정 용도로 힌트를 줄 수도 있다 
- hashjoin, no_hashjoin 
- merge 
  - subquery 를 임시테이블로 생성한다, 이 힌트가 없어도 보통 그렇게 된다 
  - no_merge 를 사용하면 임시테이블 생성하지 않는다 
- index_merge, no_index_merge 
- no_icp
  - index push down 비활성화 
- skip_scan, no_skip_csan 
- index, no_index 

### 10.2 실행 계획 분석 
- id 
  - 단위 select 별로 부여되는 값 
- select type 
  - simple 
    - union, subquery 사용하지 않는 단순 select 단위 쿼리 
  - primary 
    - union, subquery 가지는 쿼리에서 가장 바깥에 있는 단위 쿼리 
  - union
    - union 결합되는 쿼리 중에 두번째 이후 단위 쿼리 
  - dependent union
    - union 쿼리중에 외부의 값을 참조하는 쿼리 
  - union result 
    - union 결과 담아두는 테이블 
  - subquery
    - from 절 이외에서 사용되는 서브쿼리 
    - 이 중에 바깥의 컬럼을 참조하면 dependent subquery 가 된다 
  - derived 
    - 단위 쿼리가 임시 테이블로 변환되어 생성된 경우 
    - 인덱스가 자동으로 설정된다 
    - 외부의 값을 참조하면 dependent derived 라고 표시된다 
  - uncacheable subquery 
    - subquery 결과는 보통 캐시되는데 안되는 경우에 이렇게 표시된다 
  - materialized 
    - from, in 에 사용된 subquery 를 임시테이블로 생성함 
- table 
  - 말그대로 테이블 
- partitions 
  - 파티션 정보 
- type 
  - 어떤 방식으로 테이블을 읽었는지 표시 
  - system 
    - record 한건 혹은 0 건인 테이블 참조함 
  - const 
    - pk 혹은 uk 참조하는 조건 있고 1건 이상 결과가 존재 
  - eq_ref 
    - join 쿼리에서 드라이빙 테이블 컬럼이 드리븐 테이블의 pk 나 uk 일때 
  - ref 
    - 인덱스 종류 상관없이 equal 조건으로 조인될때 
  - fulltext 
    - 전문검색 할때 
  - ref_or_null
    - ref 와 같은데 null 비교 조건이 추가된 형태 
  - unique_subquery 
    - where 절에 in subquery 를 이용 하는데 중복없는 값이 반환되는 경우 
  - index_subquery
    - in subquery 에서 중복된 결과가 있을때 
  - range 
    - 범위 검색 
  - index_merge 
    - 2 개 이상의 인덱스로 결과를 만들고 병합하는 경우     
  - index 
    - 인덱스 full scan 
  - all
    - 테이블 full scan 
- possible_keys 
  - 사용을 고려한 인덱스, 실제로 사용된게 아닐 수 있음 
- keys 
  - 최종적으로 사용된 인덱스 
- key_len 
  - 인덱스 각 레코드에서 몇 바이트 사용했는지 알려줌 
- ref 
  - 어떤 조건을 참조 했는지 표시 
- rows 
  - 하나의 처리방식이 얼마정도 레코드를 읽어야하는지 예측 
- filtered 
  - 빌터링되고 남은 레코드의 비율 예측치 
- extra 
  - 성능에 관련한 주요 표시 사항 
  - const row not found 
    - const 접근 했는데 결과 1건도 없음 
  - deleting all rows 
    - 삭제 쿼리 인데 조건절 없음 
  - distinct
    - join 한다음 distinct 할때 필요 없는 row 데이터는 무시함 
  - first match 
    - 첫번째 일치 한 값만 활용 
  - full scan on null key 
    - in query 중에 null 을 만나면 그냥 full scan 한다 
  - impossible having 
    - having 절 조건 만족하는게 없음 
  - impossible where
    - where 절 항상 false 
  - loose scan 
    - loose scan 이 발생함 
    - group by, min, max 등 필요없는 인덱스 값 무시하고 읽음 
    - 참고로 skip scan 은 결합 인덱스에서 두번째 조건만 활용하는 경우 
  - not exists 
    - A 테이블에 있는데 B 테이블에 없는 값을 join 하여 조회하는경우 
    - B 테이블에 데이터 있는지만 체크함 
    - outer join, not in, not exists 등 에서 사용가능 
  - range checked for each record 
    - 컬럼 값에 따라서 join 하는 대상을 full scan 할지 range scan 할지 유동적으로 결정함 
  - rematerialize 
    - lateral join 하는 테이블을 임시테이블로 생성하는 경우
    - lateral 은 mysql 8.0 부터 도입, subquery 에서 메인 쿼리 컬럼을 참조가능 
  - select tableds optimized away
    - min, max 가 인덱스로 한건만 읽을 때 
  - started temporary, end temporary
    - duplicated weed-out 이 발생했을때   
  - unique row not found 
    - pk, uk 로 outer join 했으나 일치하는 레코드 없음 
  - using filesort
    - order by 하는데 인덱스 활용 못함 
  - using index 
    - 커버링 인덱스가 동작함 
  - using index condition
    - 인덱스 컨디션 푸시다운이 발생 
  - using join buffer 
    - 드리븐 테이블에 인덱스가 있어야 좋지만 없을때 block nl 혹은 hash join 이 발생한다 
    - 이경우 join buffer 라고 표시된다 
  - using MRR(multi range read)
    - MRR 은 여러개의 키값을 한번에 스터ㅗ리지 엔진으로 넘겨서 처리 
  - using sort_union, union, intersect 
    - index merge 가 발생했을때 두 인덱스로 부터 읽은 결과를 어떻게 병합했는지 표시 
  - using temporary
    - 임시 테이블 활용됨 
  - using where 
    - 스토리지 엔진 레벨에서 데이터를 많이 읽었는데 그중 mysql 엔진 레벨에서 필터링이 발생한 경우 
  - zero limit 
    - 쿼리 결과 없이 메타 정보만 반환됨   