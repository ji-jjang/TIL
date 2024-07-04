- [Redis (Remote Dictionary Server)](#redis-remote-dictionary-server)
  - [특징](#특징)
    - [1. 인메모리 데이터베이스(In-Memory Database)](#1-인메모리-데이터베이스in-memory-database)
    - [2. 단순성](#2-단순성)
    - [3. 고가용성](#3-고가용성)
    - [4. 확장성](#4-확장성)
  - [레디스 환경](#레디스-환경)
    - [1. maxclients](#1-maxclients)
    - [2. THP(Transparent Huge Page)](#2-thptransparent-huge-page)
    - [3. vm.overcommit\_memory = 1](#3-vmovercommit_memory--1)
    - [4. somaxconn(socket max connection), syn\_backlog](#4-somaxconnsocket-max-connection-syn_backlog)
    - [5. redis.conf](#5-redisconf)
      - [port](#port)
      - [bind](#bind)
      - [protected-mode](#protected-mode)
      - [daemonize](#daemonize)
      - [dir](#dir)
  - [Redis 실행](#redis-실행)
    - [1. 프로세스 시작과 종료](#1-프로세스-시작과-종료)
    - [2. 레디스 접속](#2-레디스-접속)
    - [3. 도커로 Redis 실행하기](#3-도커로-redis-실행하기)
- [참고자료](#참고자료)

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

## 레디스 환경
### 1. maxclients
- 레디스의 기본 maxclients 설정값은 10'000이다.이는 **레디스 프로세스에서 받아들일 수 있는 최대 클라이언트의 개수**를 의미한다. 
- 레디스 프로세스 내부적으로 사용하기 위해 예약한 파일 디스크립터 수는 32개 이므로 **레디스의 최대 클라이언트 수를 기본값인 10'000으로 지정하고 싶으면 서버의 파일 디스크립터 수를 최소 10'032 이상으로 지정**해야 한다.
- redis linux에 접속하여 파일 디스크립터 수를 확인하면 충분히 설정되어 있다.
```bash
$ docker exec -it my-redis bash
$ ulimit -n 
// 1048576
```
- 만약에 해당 값이 작게 설정되어 있다면(AWS Ubuntu의 경우 기본값 1024) `/etc/security/limits.conf`에서 파일디스크립터 수를 늘려주자.
```bash
# limits.conf
* hard nofile 100000
* soft nofile 100000
```
- 이렇게 설정하면 파일디스크립터 수가 늘어난 것을 확인할 수 있다.

### 2. THP(Transparent Huge Page)
- 리눅스는 메모리를 페이지 단위로 관리하며 기본 페이지는 4096바이트로 고정되어 있다. 
- 메모리 크기가 커질수록 페이지를 관리하는 테이블인 TLB의 크기도 커지기에 페이지를 크게 만든 뒤 자동으로 관리하는 THP 기능이 도입되었다.
- **하지만, 레디스 같은 데이터베이스 애플리케이션에는 해당 기능을 사용할 때 퍼포먼스가 떨어지고, 레이턴시가 올라가는 현상**이 발생한다.

> 항상 THP를 끄는 게 권장되었지만, 최근에는 데이터를 디스크에 영구적으로 저장하는 경우에 한해서 필요하다고 한다.

- THP 일시적 비활성화
```bash
echo > never > /sys/kernel/mm/transparent_hugepage/enabled
```
- THP 영구적 비활성화
```bash
$ vim /etc/rc.local

if test -f /sys/kernel/mm/transparent_hugepage/enabled; then
  echo never > /sys/kernel/mm/transparent_hugepage/enabled
fi 
```
- 부팅 중 rc.local 파일이 자동으로 실행되도록 설정
```bash
$ chmod +x /etc/rc.d/rc.local
```

### 3. vm.overcommit_memory = 1
```bash
$ cat /proc/sys/vm/overcommit_memory // 기본적으로 0으로 설정되어 있다.

$ vim /etc/sysctl.conf
vm.overcommit_memory=1

$ sysctl overcommit_memory=1 // 재부팅 없이 설정 적용
```
- 레디스는 **디스크에 파일을 저장할 때 fork()를 이용해 백그라운드 프로세스를 만든다**. 
- 이때 **COW(Copy On Write)**로 동작하는데, 부모 프로세스와 자식 프로세스가 동일한 메모리 페이지를 공유하다가 레디스의 데이터가 변경될 때마다 메모리 페이지를 복사하기 때문에 데이터 변경이 많이 발생하면 메모리 사용량이 빠르게 증가할 수 있다.
- 메모리를 순간적으로 초과해 할당해야 하는 상황이 발생할 수 있으며 이를 위해 해당 옵션을 1로 설정하는 것이다. 0으로 설정하면 필요한 메모리를 할당받지 못해 제대로 동작하지 못할 수 있다. 물론 이 경우 linux OutOfMemory Killer 가 동작하는 것을 인지해야 한다.

### 4. somaxconn(socket max connection), syn_backlog
- 레디스 설정 파일의 tcp-backlog 파라미터는 레디스 인스턴스가 클라이언트가 통신할 때 사용하는 tcp backlog 큐의 크기를 지정한다.
- 이때 redis.conf에서 지정한 tcp-backlog 값은 서버의 somaxconn과 syn_backlog 값보다 클 수 없다.
```bash
127.0.0.1:6379> CONFIG GET tcp-backlog
1) "tcp-backlog"
2) "511"
```
- 기본 값은 511이므로 서버 설정이 최소 이 값보다 크게 한다. 현재 설정 값은 아래와 같이 확인할 수 있다.
```bash 
$ sudo sysctl -a | grep syn_backlog
net.ipv4.tcp_max_syn_backlog = 128

$ sudo sysctl -a | grep somaxconn
net.core.somaxconn = 4096
```
- 영구적으로 해당 설정을 적용할 수 있다.
```bash
$ vim /etc/sysctl.conf 
net.ipv4.tcp_max_syn_backlog = 1024
net.core.somaxconn = 1024

// 재부팅 없이 바로 설정 적용하기
sysctl net.ipv4.tcp_max_syn_backlog==1024
sysctl net.core.somaxconn=1024
```

### 5. redis.conf
- 레디스 설정파일은 간단한 형태로 구성된다.
```bash
keyword argument1 argument2
```
#### port
- 기본값: 6379
- 커넥션이 지정된 포트로 레디스 서버에 접속할 수 있도록 허용

#### bind
- 기본값: 127.0.0.1 -::1

#### protected-mode
- 기본값: yes
- 패스워드를 설정해야만 레디스에 접근할 수 있으며 **설정되어 있지 않다면 로컬에서만 연결할 수 있다**.

#### daemonize
- 기본값: no
- 레디스 프로세스를 데몬으로 실행시키려면 yes로 변경해야 한다. 데몬으로 실행하면 pid 파일이 생성된다. 기본값은 /var/run/redis_6379.pid 이다.

#### dir
- 기본값: ./ 
- 로그 파일이나 백업 파일 등 인스턴스를 실행하면서 만들어지는 파일이 저장되는 위치이다.

## Redis 실행
### 1. 프로세스 시작과 종료
- daemonize를 yes로 해야 아래와 같이 레디스를 실행할 수 있다.
```bash
$ bin/redis-server redis.conf

$ bin/redis-cli shutdown
```

### 2. 레디스 접속
```bash
$ export PATH=$PATH:/home/centos/redis/bin

$ redis-cli -h <ip주소> -p <port> -a <password>

$ redis-cli

$ PING // 사용자가 연결을 끊을 때까지 계속 레디스 서버에 접속된 상태를 유지한다.

$ redis-cli PING // 레디스 서버에 특정 커맨드를 수행시킨 뒤 종료시키고 싶다면 커맨드라인 모드로 사용한다.

```

### 3. 도커로 Redis 실행하기
```bash
$ docker run --name my-redis -d -p 6379:6379 redis
$ docker exec -it my-redis redis-cli
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
$ docker run --name my-redis -v ./redis.conf:/usr/local/etc/redis/redis.conf -d -p 6379:6379 redis redis-server /usr/local/etc/redis/redis.conf
$ docker exec -it my-redis redis-cli

// Redis command
$ 127.0.0.1:6379> config get loglevel
1) "loglevel"
2) "warning"
```
- redis.conf 파일을 읽어 loglevel을 warning으로 변경한 것을 확인할 수 있다.


# 참고자료
- 개발자를 위한 레디스 - 김가림
- https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/#latency-induced-by-transparent-huge-pages