---
layout: post
title: Github Action Basic
date: 2024-08-01 12:00:00 +0000
categories: cicd
---

## github action

### Intro

github 에서 제공하는 CI 와 CD 를 위한 서비스
- 코드 저장소 (repository) 에 어떤 이벤트가 발생했을 때 주기적으로 다른 작업을 실행시킬 수 있게 하는 서비스
- marketplace 가 있고 이를 yml 파일에 몇 줄 사용하는 것 만으로도 쉽게 사용할 수 있어서 강력함
- project 의 상단 directory 에 .github/workflows 에 yml 파일을 생성하여 사용
	- 여러개의 yml 을 생성할 수 있고 각각이 실행되는 구조
- on, jobs, steps, actions 의 기본 구성요소


### On
```yaml
on:  
  push:  
    branches:  
      - master  
      - develop  
    tags:  
      - release/v*  
    paths-ignore:   
      - "docker/Dockerfile 
```

- 언제 해당 workflow 가 실행 될지 표기하는 부분
- 위의 예시는 master, develop branch 혹은 release/v* 로 표기되는 태그 브랜치에 push 될때 동작한다는 의미
- paths-ignore 를 통해 접근하지 않는 파일을 지정할수도 있음

### Jobs
```yaml
jobs:  
  publish:  
    if: github.event_name == 'push'  
    runs-on:  
      group: example-runners  
      labels: 4-core-ubuntu  
    outputs:  
      server-gitlog: ${{ steps.commit-msg.outputs.log }}
  publish_gql_schema:  
    needs: [ publish ]  
    runs-on: ubuntu-latest
```

- 실행의 가장 기본이 되는단위이며 여러 job 들을 정의 할수 있음
- runs-on 을 통해서 어떤환경에서 실행될지 정할 수 있는데, `ubuntu-latest` 처럼 github 에서 제공하는 환경을 이용할 수 있고 직접 환경을 구현하고 지정할 수 있음
- 예시에서는 publish, publish_gql_schema 라는 2개의 job 을 정의 하고 있음
	- publish 는 자체로 구현한 환경 (example-runners) 에서 실행
	- publish_gql_schema 는 publish job 이 완료되고 github 에서 제공하는 ubuntu 환경에서 실행

### steps
```yaml
steps:
	- uses: actions/checkout@v4  
  
	- name: Set build env for prod  
	  if: ${{ startsWith(github.ref, 'refs/tags/release/v') }}  
	  run: |  
	    echo "BUILD_ENV=prod" >> $GITHUB_ENV  
	    echo "ENV_FILE=prod.env" >> $GITHUB_ENV  
	    echo "ENV_S3_BUCKET=prod" >> $GITHUB_ENV  
	    echo "ECS_TASK_DEFINITION=example-server" >> $GITHUB_ENV  
	    echo "ECS_CLUSTER=example-cluster" >> $GITHUB_ENV
	- name: Set up Docker  
		  uses: docker/setup-buildx-action@v3  
  
	- name: Docker meta  
	  id: meta  
	  uses: docker/metadata-action@v5  
	  with:  
	    images: ${{ steps.login-ecr.outputs.registry }}/example-server  
	    tags: |  
	      type=sha,format=long
```

- github action 에서 처리할 명령들을 순서대로 나열
- job 안에서 실제 실행 할 명령들을 명시하는 곳
- action 을 사용하기도 하고 run 을 통해서 커맨드나 스크립트를 실행하기도 함

### actions
- 빈번하게 사용되는 step 을 미리 지정해서 쓰는 것
- github 에서 직접 제공되는 것도 있고 공유되고 있는 [github marketplace](https://github.com/marketplace?type=actions) 에 올라온 것들을 사용할 수 도 있음