---
layout: post
date: 2024-08-01 12:00:00 +0000
title: AWS Step Functions
categories: aws
---
# Step Functions

### Intro
- State Machine 이라는 워크플로우를 설계하고 실행할 수 있는 서버리스 서비스
- 리소스들과 Lambda 를 유연하게 연결하게 하는데 도움이 됨

### 구조 및 특징
- 워크플로우를 UI 혹은 코드를 통해 작성할 수 있음

	

- Task, Wait, Choice 등의 요소가 있고 분기처리 및 동시 작업을 처리하게 할 수 있음
- 각 단계 별 에러 처리 및 로깅하기 편함
- Lambda, EventBridge 등과 선택함에서 고민할 수 있음

### 고려해야 할 사항
- 비용
	- 상태 전환 횟수 별로 비용이 발생해서 복잡하거나 자주 실행되는 플로우에서는 고민이 필요
- 제한된 실행 시간
	- 최대 시간이 제한되어있음