---
layout: post
title: MQTT Broker
date: 2024-08-04 12:00:00 +0000
categories: backend
---
## MQTT (Media Queuing Telemetry Transport)

### MQTT 프로토콜

경량의 메시지 전송 프로토콜로 IoT 환경에서 기기 간 통신을 위해 많이 사용됨
- 저전력, 저대역폭에 최적화
- Publish - Sunscribe 의 모델을 사용하여 메시지를 전송함
- 클라이언트간 직접적인 연결없이 중개 서버로 Broker 를 이용

### MQTT 프로토콜 작동방식

1. 클라이언트 연결
   - MQTT 클라이언트는 브로커에 TCP/IP 연결
2. 메시지 발행
   - 특정 주제 (topic) 으로 메시지를 발행, 브로커는 메시지를 구독하는 모든 클라이언트에게 전달
3. 메시지 구독
   - 구독자는 브로커에게 특정 topic 을 구독하겠다고 요청
   - 구독시 QoS (Quality of Service) 수준 지정 가능 
	   - Qos 0 - 최대 1회 전송, 손실 가능
	   - Qos 1 - 최소 1회 전송, 중복 가능
	   - Qos 2 - 정확히 1회 전송, 메시지가 중복없이 전달
4. 메시지 전달
   - 브로커는 발행된 메시지를 Qos 수준에 따라 구독자에게 전달
5. 연결 해제
   - 작업이 완료되면 브로커와 연결을 해제할수 있음
   - LWT 메시지 설정을 통해 비정상인 연결 종료를 알수 있음

### QoS 에 따른 보장 방식

Qos 0 : At Most Once
- 발행자가 메시지 브로커에 전달하면 구독자에게 그대로 전달
- 센터 데이터 스트림에서 자주 사용

Qos 1 : At Least Once
- 발행자가 메시지를 브로커에 전달하고 브로커로부터 PUBACK 을 받을 때까지 계속 전송을 시도
- 브로커가 메시지를 구독자에게 전달하고 PUBACK 을 받을때까지 전송을 시도

Qos 2 : Exactly Once
- 발행자가 메시지 전송하고 PUBREC를 기다림
- 브로커는 메시지를 구독자에게 전달하고 발행자에게 PUBREL 을 전송
- 구독자는 메시지를 수신하고 PUBCOMP를 브로커에 응답

### MQTT 데이터 구조

MQTT 메시지는 Fixed Header, Variable Header, Payload 로 구성

Fixed Header
- Message Type and Flags
	- 메시지 유형 (CONNECT, PUBLISH, SUBSCRIBE) 를 나타내는 4비트 필드
	- Flag는 QoS, 중복 전송 여부 (DUP), 유지 메시지 여부 (RETAIN) 을 설정하는데 사용
- Remaining Length
	- 가변 헤더와 베이로드 바이트 수

Variable Header
- Packer Identifier
	- 메시지를 추적하고 구분하는 2바이트 식별자
	- PUBLISH, PUBACK, PUBREC, PUBREL, PUBCOMP, SUBSCRIBE, SUBACK, UNSUBSCRIBE, UNSUBACK
- 기타
	- CONNECT 메시시 - 프로토콜 이름, 레벨, 유지 시간 (Keep Alive) 등이 포함됨
	- PUBLISH 메시지 - Topic Name 이 포함

Payload
- 유형별로 다른 내용을 가짐
	- CONNECT - 클라이언트 식별자(Client Identifier), 사용자 이름 및 비밀번호(옵션), Will 메시지(옵션)
	- PUBLISH - 메시지 콘텐츠 자체
	- SUBSCRIBE - 구독할 주제 이름과 QoS 수준
	- SUBACK - 구독 확인 응답, QoS 수준

### MQTT 소프트웨어
- Eclipse Mosquitto
- EMQX
- HiveMQ
- AWS IoT Core
