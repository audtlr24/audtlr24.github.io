---
layout: post
title: Kafka Streams
date: 2024-08-08 12:00:00 +0000
categories: backend
---
## Kafka Streams?

Apache Kafka 에서 수집하는 데이터 (topic-partition)를 실시간으로 처리할 수 있는 스트리밍 라이브러리

### 스트림 데이터란?
- 연속적인 데이터 흐름
- 데이터가 실시간으로 지속적으로 들어오는 것
- 스트림으로 처리하면 데이터를 발생하자마자 처리하기 때문에 저지연성, 지속성의 특징이 있음

## Kafka - Kafka Streams

### Kafka Streams 는 데이터를 어떻게 가져오는가?
Kafka Streams 는 Kafka 의 consumer 역할을 하며 데이터를 지속적으로 polling 하여 읽어옴
- Kafka 브로커로부터 새로운 데이터를 감지하여 이를 가져옴
- 가져온 이후에는 Streams 의 Processor Topology라는 것을 통해서 처리함

### Processor Topology
- Source Processor
	- Kafka 토픽에서 데이터를 읽어와서 스트림으로 변환함
- Stream Processor
	- 데이터를 처리하고 변환
	- 필터링, 매핑, aggregating 등의 작업을 수행할 수 있음
- Sink Processor
	- 처리된 데이터를 다른 Kafka 토픽 혹은 외부시스템에 기록함


## Kfaka Streams DSL
- 고수준 DSL(Domain Specific Language)을 제공하고 이를 통해 스트림을 쉽게 처리 할 수 있음
- 주요 구성은 KStream, KTable

### KStream
- 토픽에서 읽은 데이터의 스트림
- 각 레코드는 독립적인 이벤트로 처리되며, 실시간 데이터 처리에 적합

### KTable
- 상태를 갖는 데이터의 스냅샷
- 각 키는 하나의 최신값을 유지하고 새로운 상태를 계속 반영함

KStream 과 KTable 은 서로 쉽게 변환이 가능하고 각각 집계를 하거나 다시 이벤트 스트림을 생성하는데 자유롭다는 의미로 데이터 처리 로직을 쉽게 작성할 수 있게 함

### KTable 주요 기능
Windowing
- window 기반으로 데이터를 집계 하는 기능
- 특정 기간 내의 데이터만을 고려해서 집계를 수행할 수 있음

Join
- 다른 KTable, KStream 간의 join 을 지원함

Aggregation
- 데이터 집계를 지원하며 데이터 요약이 가능함
- Count, Sum, Average 등


## KStreams 의 특징 & 이점

### Parellel Processing
KStrams 는 하나의 토픽에 대해 여러 프로세서를 병렬로 처리할 수 있는 기능을 제공
- 파티션 기반 병렬 처리
	- Kafka 에서 토픽을 여러 파티션으로 나누어 저장하는데 Kafka Streams 는 이를 개별적으로 소비할 수 있고 여러 스레드가 각각 다른 파티션을 처리할 수 있음
- 스레드 수준 병렬 처리
	- 여러 스레드를 설정할 수 있으며 서로 다른 파티션의 데이터를 병렬로 처리 할 수 있음
- 프로세스 토폴로지
	- 프로세서는 여러개로 구성될 수 있음

### Exactly-once Guarantee
메시지를 중복 없이 정확히 한번만 처리한다는 특징
- 트랜잭션
	- Producer - Consumer 간의 작업을 하나의 트랜잭션으로 묶음
- Atomic Source - Sink
	- Source Processor 에서의 레코드는 Sink Processor 에 의해 쓰여질때 까지 커밋되지 않음
- 상태 저장소 (state store) 를 통해 프로세싱 상태를 기록하며 필요할때 복구에 이용 가능

### One-record-at-a-time Processing
레코드를 개별적으로 처리하는 방식
- 낮은 지연시간, 순차적 처리의 특징

