---
layout: post
title: Postgres Vaccum
date: 2024-08-01 12:00:00 +0000
categories: backend
---

# Postgres Vaccum

### Vacuum 이란?
 - Dead Tuple 을 정리하여 빈 공간으로 반환하는 작업 (GC 와 비슷한 역할)
 - Transaction ID Wrapground 방지 & 공간 효율
	 - 쿼리 성능에도 영향을 미칠 수 있음

### Dead Tuple 생기는 이유
- Postgres 의 MVCC
	- 데이터의 업데이트나 삭제에서 새로운 데이터 버젼을 생성하고(xmin, xmax) 이전의 데이터는 남겨진 채로 놔두게 되는 방식을 사용함
	- 트랜잭션의 동시성을 위해 데이터의 락 대신에 이런 방식을 채용한 것
- 결과적으로 이전 튜플은 어디도 참조되지 않게 되는 튜플이 되는데 이를 Dead Tuple 이라고 함

### Vacuum
- Dead Tuple 은 결과적으로 쓸데없는 데이터이고 결국 데이터 쿼리를 할때 page 단위로 읽어오는 작업에서 성능 저하가 발생
- Vacuum 은 이러한 dead tuple 을 정리하고 해당 메모리에 새로운 데이터를 넣을 수 있게 함
- 또한 아래에서 후술할 Transaction ID Wraparound 문제를 해결하기 위한 freeze 의 역할도 함
- AutoVacuum 설정을 통해 이러한 Vacuum 작업을 자동으로 할 수는 있음
	- default 도 on 으로 되어있음

### Transaction ID Wraparound
- Transaction ID 를 모두 소모해서 (약 40억개) 다시 ID가 1부터 시작되고 그러면 기존의 데이터들이 미래 데이터로 취급되면서 모든 데이터가 소실되는 현상이 나타남
- 이를 방지하기 위해 특정 시점에서 과거의 Transaction ID 를 frozen XID 라는 값으로 바꿔버리는데 이를 freeze 혹은 Anti Wraparound Vacuum 이라고 함

### Vacuum Full
- 그러나 단순 Vacuum 은 os 디스크 공간 반환까지는 처리하지 못함
- 공간 반환까지 처리하려면 Vacuum Full 을 해야하는데 Access Exclusive Lock 을 획득해서 운영 상태에는 할 수 없는 작업
- Copy 작업도 들어가기 때문에 디스크 가용량도 필요함

