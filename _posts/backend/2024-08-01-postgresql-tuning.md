---
layout: post
title: Postgresql Parameter Tunning
date: 2024-08-01 12:00:00 +0000
categories: backend
---


# Postgresql Parameter

참고 [AWS Tuning PostgreSQL](https://docs.aws.amazon.com/prescriptive-guidance/latest/tuning-postgresql-parameters/introduction.html)

## Memory Parameter
- 적절한 메모리 parameter 세팅은 query, sorting, indexing, caching 등에 큰 영향을 줌

### shared_buffers
- 자주 접근하는 데이터를 메모리에 캐시하여 디스크 I/O 를 줄여주는 역할
- 쓰기 최적화 - 쓰기 작업이 있을때 shared_buffers에 먼저 기록후 디스크 기록
- 동시성 관리 - 여러 세션이 접근할때 데이터가 메모리에 있어 동시성 제어가 효율적

- 권장 조절
	- 서버 메모리의 25% 정도
	- 워크로드가 I/O 집약적인 경우에는 쫌 더 늘리는 것이 도움이 됨
	- 너무 늘리면 시스템 전체 메모리 부족이 될 수 있음을 주의
- 성능 모니터링
	- pg_stat_activity, pg_stat_database, pg_buffercache
	- pg_stat_activity 에서는 캐시 히트율을 계산할수 있고, 99% 이하라면 증가시키는 것 고려 가능 (현재 prod 기준으로는 99.966)
```sql 
SELECT datname, blks_hit, blks_read, blks_hit * 100.0 / (blks_hit + blks_read) AS hit_ratio FROM pg_stat_database;
```
	
	
- AWS parameter
	- SUM({DBInstanceClassMemory/12038},-50003)
	- 32GB 기준 -> 2.68GB, 64GB 기준 -> 5.4GB

### temp_buffers
- 정렬, 해시 작업등에서 사용하는 임시 테이블이 사용할 수 있는 메모리의 양
- 권장사항은 딱히 없지만, sorting 이 자주있다면 늘리는게 도움이 됨 

- default 는 8MB

### effective_cache_size
- 쿼리 최적화 시 사용할 수 있는 전체 캐시 메모리 크기 예상하는데 사용
- 쿼리 플래너가 인덱스를 사용할지, 시퀀셜 스캔을 사용할지 선택하는데 도움을 줌
- 높은 값을 세팅하면 인덱스를 사용하는 쪽으로 최적화

- 메모리의 절반 또는 3/4 설정이 일반적

- AWS parameter
	- SUM({DBInstanceClassMemory/12038},-50003)
	- 32GB 기준 -> 2.68GB, 64GB 기준 -> 5.4GB

### work_mem
- sort, join 등에서 사용하는 메모리 양

- default 는 4MB
- 보통은 전체 메모리 / (최대커넥션 수 * 16) 값으로 설정하긴 하는데 
  너무 큰 값을 잡으면 Out of Memory 발생하니 work_mem 줄여야함
  
### maintenance_work_mem
- 데이터베이스 유지 작업에 사용 되는 메모리 (vacuum, create index, add foreign key)

- 기본값은 64MB
- 적절값 계산식 
`maintenance_work_mem = (total_memory - shared_buffers) / (max_connections * 5)`
- 512 MB 까지 늘리는 케이스 많음 (64GB 기준)

- AWS parameter
	- GREATEST({DBInstanceClassMemory/63963136 * 1024}, 65536)
	- 32GB 기준 -> 524MB, 64GB 기준 -> 1049MB

### random_page_cost
- 쿼리 플래너가 random page 에 access 할 때의 cost 로 `seq_page_cost` 와 함께 쿼리 플래너 결정에 영향을 줌

- 기본값은 4
- SSD 를 사용한다면 값을 낮추는게 좋을 수 있음 (1.2 ~ 1.5 정도 까지)

### seq_page_cost
- `random_page_cost` 와 반대로 디스크 순차 접근에서 cost 로 쿼리 플래너에 영향을 줌

- 기본값은 1, 보통은 기본값 유지

### track_activity_query_size
- pg_stat_activity 에서 추적할 수 있는 최대 길이를 지정

- 기본값은 1KB (aurora 는 4KB)

### idle_in_transaction_session_timeout
- Idle transaction 이 stop 되기 전 대기하는 시간에 대한 값

- aurora 에서는 86,400,000 ms (24시간)
- 결국 idle 상태의 transaction 이 자원을 소모하게 되긴하는데, 안정성이 있을 수 있음
- 일반적인 웹 어플리케이션인 경우 5분 정도로 설정하는 것으로 충분

### statement_timeout
- 쿼리의 최대 실행시간

- 기본값은 0 (no timeout)
- 제한해놓고 너무 오래걸리는 쿼리는 쿼리를 튜닝하거나 인덱스 추가하는 방식으로 활용하기도 함

### search_path
- SQL 문에서 스키마가 객체를 검색하는 순서

- 기본값은 $user, public
	- 사용자 이름과 일치하는 스키마 다음 public 검색
- public 만 스키마로 사용한다면 굳이 변경 필요 없음

### max_connections
- 데이터베이스 최대 커넥션 수

- 설정시 고려할 것들
	- 각 커넥션이 평균적으로 10M 의 메모리를 사용
	- 커넥션 수가 클 수록 CPU 부하
	- pg_stat_activity -> 접속 수 확인

- AWS parameter
	- LEAST(DBInstanceClassMemory/9531392,5000)
	- 32GB 기준 -> 3604, 64GB 기준 -> 5000