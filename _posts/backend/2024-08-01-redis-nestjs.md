---
layout: post
title: Redis (with Nestjs)
date: 2024-08-01 12:00:00 +0000
categories: backend
---
## Redis

오픈 소스 인메모리 데이터 구조 저장소로 데이터베이스, 캐시, 메시지 브로커, 스트리밍 엔진 등으로 사용됨

### Redis 주요 특징

인메모리 데이터 저장소
- 모든 데이터를 메모리에 저장
- 매우 빠른 성능을 보장

고성능 & 고가용성
- 빠른 응답시간 제공
- 대량의 동시 요청 처리 가능
- 마스터 - 슬레이브 복제를 지원하여 데이터의 고사용성과 부하 분산을 제공

### Redis 활용

캐싱
- 데이터베이스 쿼리 결과, 세션 데이터 등을 캐싱
- 정적데이터 캐싱

세션 저장소
- 사용자 세션 데이터를 저장하고 관리하는데 적합

메시지 브로커
- Pub/Sub 모델을 지원하여 메시지 큐와 브로커로 사용이 가능
- Redis Streams 를 사용하여 복잡한 메시징 요구 사항을 처리 할 수 있음

### AWS 에서 Redis

EC2 인스턴스에 직접 설치해서 관리하는 방법도 있지만
Amazon ElasticCache 를 이용하면 다양한 기능을 사용할 수 있음
- 여러 노드를 구성할 수 있음
- 리전 내 replica 설정이 가능

### Redis in Nestjs

ioredis 패키지를 통해서 nestjs 에서는 redis 와 편하게 통신이 가능

redis service 예시
```typescript
// redis/redis.service.ts

import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import * as Redis from 'ioredis';

@Injectable()
export class RedisService implements OnModuleInit, OnModuleDestroy {
  private client: Redis.Redis;

  onModuleInit() {
    // Redis 클라이언트 초기화
    this.client = new Redis({
      host: 'localhost', // Redis 서버 주소
      port: 6379,        // Redis 포트
      // 옵션: password, db 등을 추가할 수 있습니다.
    });

    this.client.on('connect', () => {
      console.log('Connected to Redis');
    });

    this.client.on('error', (err) => {
      console.error('Redis error:', err);
    });
  }

  onModuleDestroy() {
    // 애플리케이션 종료 시 Redis 연결 종료
    this.client.quit();
  }

  async set(key: string, value: any) {
    await this.client.set(key, value);
  }

  async get(key: string) {
    return this.client.get(key);
  }

  async del(key: string) {
    await this.client.del(key);
  }

  async incr(key: string) {
    return this.client.incr(key);
  }

  // 다른 Redis 명령어를 추가로 정의할 수 있습니다.
}

```