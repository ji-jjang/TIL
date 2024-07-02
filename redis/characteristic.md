# Redis (Remote Dictionary Server)
- 고성능 키-값 인메모리 NoSQL 데이터베이스

## 특징
### 1. 인메모리 데이터베이스(In-Memory Database)
### 2. 단순성
- 키 값 형태로 저장 (string, Hash, set ...)
- 메인 스레드 1개와 별도의 스레드 3개, 총 4개의 스레드로 동작하지만 클라이언트 **커맨드를 처리 하는 부분은 이벤트 루프를 이용한 싱글 스레드로 동작**
### 3. 고가용성
- 복제를 통해 데이터를 여러 서버에 분산 시킬 수 있으며, 센티널을 이용해 마스터에 장애가 발생하더라도 페일오버가 완료되어 정상화된 마스터 노드 사용 가능
### 4. 확장성
- 손쉬운 수평적 확장 가능, **데이터는 클러스터 내에서 자동으로 샤딩된 후 저장되며, 여러 개의 복제본**이 생성될 수 있음
- 데이터 분리는 데이터베이스 레이어에서 처리되며 애플리케이션에서는 대상 데이터가 어떤 샤드에 있는지 신경 쓰지 않아도 됨

## 도커로 Redis 실행하기
```bash
docker run --name my-redis -d -p 6379:6379 redis
docker exec -it my-redis redis-cli
```
- redis 설정 정보 확인하기
```bash
127.0.0.1:6379> info
```
- redis.conf 파일을 이용해서 비밀번호, 포트, 바인딩 등을 설정하여 레디스를 실행할 수 있다.
- 아래와 같은 간단한 redis파일을 만들고 실행해보자.
```bash
// redis.conf 
bind 127.0.0.1
port 6379
loglevel warning

// Docker command
docker run --name my-redis -v ./redis.conf:/usr/local/etc/redis/redis.conf -d -p 6379:6379 redis redis-server /usr/local/etc/redis/redis.conf
docker exec -it my-redis redis-cli

// Redis command
127.0.0.1:6379> config get loglevel
1) "loglevel"
2) "warning"
```
- redis.conf 파일을 읽어 loglevel을 warning으로 변경한 것을 확인할 수 있다.

# 참고자료
- 개발자를 위한 레디스 - 김가림