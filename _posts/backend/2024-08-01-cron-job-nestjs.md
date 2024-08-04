---
layout: post
title: Nestjs Cronjob with ECS
date: 2024-08-01 12:00:00 +0000
categories: backend
---
### Nestjs 에서 Cronjob

기본적으로는 Cron 이라는 @nestjs/schedule 의 decorator 를 통해서 크론을 등록할 수 있지만 자유도가 떨어지는 문제가 있음
- 특정 조건에서만 Cron 이 등록되게 하고 싶을때
- 실행되는 Cronjob 의 개수를 제한하고 싶을때

### ScheduleRegistry 사용

@nestjs/schedule 에 있는 scheduleRegistry.addCronJob 을 이용하면 코드 안에서 크론잡을 추가 할 수 있고 job.start() 를 통해 실행이 가능함

```typescript
addCronJob(name: string, expression: string, fn: () => Promise<void>, onlyDev: boolean = false) {  
  if (onlyDev && process.env.NODE_ENV !== 'dev') return;  
  
  const job = new CronJob(  
    expression,  
    async () => {  
      const cronJobPromise = fn();  
      this.activeCronJobs.push(cronJobPromise);  
      try {  
        await cronJobPromise;  
      } finally {  
        this.activeCronJobs = this.activeCronJobs.filter((p) => p !== cronJobPromise);  
      }  
    },  
    null,    true,    'Asia/Seoul',  
  );  
  
  this.schedulerRegistry.addCronJob(name, job);  
  job.start();  
  console.log(`[CronJob] ${name} added.`);  
}
```
- NODE_ENV 처럼 조건에 따라 등록 여부 결정 가능
- activeCronJobs 에 실행되고 있는 Cronjob 관리 가능

### 리더 선출 알고리즘

Nestjs 인스턴스를 여러개 띄우는 서비스의 경우 CronJob 이 여러 인스턴스에 동시에 등록되면서 실행되는 불상사가 생길수 있음
이를 방지하고 반드시 하나의 서비스에서만 CronJob 이 등록되게 하는 방법으로 redis 를 이용한 리더 선출 알고리즘을 사용할 수 있음

```typescript
async onModuleInit() {  
  if (!this.cronRedisClient) {  
    return;  
  }  
  
  await this.tryElectLeader();  
}

private async tryElectLeader() {  
  await this.electLeader();  
  
  if (!this.isLeader && !this.isShuttingDown) {  
    // 종료 상태가 아닌 경우에만 재시도  
    setTimeout(() => this.tryElectLeader(), 5000); // 5초 후 재시도  
  }  
}

private async electLeader() {  
  try {  
    const result = await this.cronRedisClient.set(this.lockKey, this.lockValue, 'EX', this.lockExpiration, 'NX');  
    if (result) {  
      this.isLeader = true;  
      this.addCronJobs();  
      this.keepLockAlive();  
    }  
  } catch (error) {  
    console.error('[CronJob] Failed to elect leader:', error);  
  }  
}

private keepLockAlive() {  
  const interval = setInterval(  
    async () => {  
      if (this.isShuttingDown) {  
        clearInterval(interval);  
        return;      }  
  
      if (this.isLeader) {  
        try {  
          const result = await this.cronRedisClient.set(this.lockKey, this.lockValue, 'EX', this.lockExpiration, 'XX');  
          if (!result) {  
            console.error('[CronJob] Lost leadership, re-electing leader...');  
            this.isLeader = false;  
            await this.tryElectLeader();  
          }  
        } catch (error) {  
          console.error('[CronJob] Failed to keep lock alive:', error);  
          this.isLeader = false;  
          await this.tryElectLeader();  
        }  
      }  
    },  
    (this.lockExpiration / 2) * 1000,  
  );  
}
```
- redis 를 이용해서 lock 을 잡는 방식으로 구성
- redis 의 특정 키를 점유하고 해당 key 가 있을때만 cronJob 등록을 실행
- key 가 없을때는 일정 주기로 재시도 하고 key를 점유한 경우에는 주기적으로 key의 expiration 업데이트

### graceful shutdown

배포 상황에서 이전에 CronJob 을 실행중이던 서비스는 서비스를 다 완료하고 종료되어야 함
이에 따라 graceful shutdown 을 적용

```typescript

async beforeApplicationShutdown(signal?: string) {    
  this.isShuttingDown = true;  
  await this.handleShutdown();  
}

private async handleShutdown() {  
  if (this.isLeader && this.cronRedisClient) {  
    try {  
      await Promise.all(this.activeCronJobs);  
      const result = await this.cronRedisClient.del(this.lockKey);  
      if (result) {  
        console.log('[CronJob] Leader lock released');  
      } else {  
        console.log('[CronJob] Failed to release leader lock');  
      }  
    } catch (error) {  
      console.error('[CronJob] Failed to release leader lock:', error);  
    }  
  }  
}

```

- 기존에 추가해둔 activeCronJobs 를 이용해서 모든 Cronjob 서비스 호출이 끝났는지 판단