# 커넥션 제어
- 레디스 이전 버전에서는 각 인스턴스에 하나의 패스워드만 설정할 수 있었음. 하지만 버전 6에서 도입된 ACL 기능을 이용하면 유저를 생성한 뒤 각 유저별로 다른 패스워드를 설정할 수 있으며, 다른 권한도 설정할 수 있음

## bind
- 레디스 인스턴스가 실행 중인 서버는 여러 개의 네트워크 인터페이스를 가질 수 있음
- bind 설정은 레디스 서버의 여러 IP 중 어떤 IP를 통해 들어오는 연결을 받아들일 것인지 결정
- bind 설정값을 주석처리하거나 0.0.0.0 또는 *로 설정할 경우 모든 연결 허용
- 특정한 IP로 설정하여 외부에서 연결을 방지하는 보안 설정 권장

## 패스워드
### 노드에 직접 패스워드 지정하는 방식
```redis
CONFIG SET requirepass password

redis-cli -a password # 패스워드가 설정된 노드에 접속

AUTH password # 인증
```
- `requirepass` 커맨드로 레디스에 기본 패스워드 설정
- 커맨드라인에서 직접 패스워드를 입력할 경우 안전하지 않다는 경고. `--no-auth-warning`으로 경고 표시 제거

## Protected mode
- 레디스를 운영 용도로 사용한다면 설정하는 것을 권장
- 직접적으로 기능을 변경하진 않지만 다른 설정값을 제어하는 역할
- protected mode가 yes일 때 레디스 인스턴스에 패스워드를 설정하지 않았다면 레디스는 127.0.0.1 IP를 이용해 로컬에서 들어오는 연결만 허용. bind 설정을 이용해 다른 네트워크 인터페이스를 이용해 들어온 커넥션을 허용한다고 설정했을 경우에도 마찬가지
- 패스워드 없이 레디스 인스턴스를 사용하고 싶다면 protected mode를 no로 변경 (아니면 로컬에서만 접근 가능)

# 커맨드 제어
## 커맨드 이름 변경
- `rename-command` 특정 커맨드를 다른 이름으로 변경하거나 커맨드를 비활성화할 수 있는 설정. 이 설정을 사용하면 레디스 커맨드를 커스터마이징하거나 보안을 강화하는 데 유용
- `redis.conf` 파일에서만 변경할 수 있으며, 실행 중 동적으로 변경할 수 없음
- 센티널을 사용한다면 변경한 커맨드를 센티널에도 적용(sentinel.conf)해주어야 한다.

## 커맨드 실행 환경 제어
- 특정 커맨드를 실행하는 환경 제어
- 실행 중일때 변경하면 위험할 수 있는 커맨드는 변경할 수 없도록 차단
```redis
enable-protected-configs no  # 레디스 기본 경로 설정인 dir 및 백업 파일의 경로를 지정하는 dbfile 등의 옵션을 CONFIG 커맨드로 수정하는 것을 차단
enable-debug-command no # DEBUG 커맨드 차단
enable-module-command # MODULE 커맨드 수행 차단
```
- 이 세가지 설정 파일은 일반적으로 레디스를 운영 목적으로 사용할 때에는 잘 사용되지 않는 커맨드이기에 레디스가 악성 공격에 노출될 가능성을 줄이기 위해 도입
- 세 가지 설정은 아래 값으로 변경 가능
  - no: 모든 연결에 대해서 명령어 수행 차단
  - yes: 모든 연결에 대해 명령어 수행 허용
  - local:ㅁ로컬 연결에 대해서만 명령어 수행 허용

# ACL
- 각 유저별로 실행 가능한 커맨드와 접근 가능한 키를 제한하는 기능
## 유저의 생성과 삭제
```redis
> ACL SETUSER juny on >password ~cached:* &* +@all -@dangerous
```
- 패스워드는 passowrd로 설정. 유저가 접근 가능한 키는 cached:의 프리픽스를 가진 모든 키이며, 이 외의 키에는 접근할 수 없음. 제한 없이 모든 채널에 pub/sub할 수 있음(&*). 위험한 커맨드를 제외한 전체 커맨드 사용 가능(-@dangerous)
- `ACL GETUSER`로 유저 권한 확인 
- `ACL DELUSER`로 유저 삭제 가능 
- `ACL LIST`로 레디스에 생성된 모든 유저 확인
### 기본 유저
- 유저 이름: default
- 유저 상태: on(활성)
- 유저 패스워드: 패스워드 없음
- 유저가 접근할 수 있는 키: ~*(전체 키)
- 유저가 접근할 수 있는 채널; &*(전체 채널)
- 유저가 접근할 수 있는 커맨드: +@all(전체 커맨드)

## 유저 상태 제어
- 유저의 활성 상태는 on과 off로 제어
- `acl setuser <username> on`으로 on 상태로 변경
## 패스워드
- `>password` 키워드로 패스워드 지정. 1개 이상 지정할 수 있으며, `<password` 키워드로 지정한 패스워드 삭제 가능
- 유저에 `resetpassword` 권한을 부여하면 유저에 저장된 모든 패스워드 삭제. nopass 상태도 제거. 즉 다른 패스워드나 nopass 권한이 부여되기 전까지 그 유저에 접근할 수 없음
## 패스워드 저장 방식
- 내부적으로 SHA256 방식으로 암호화
- 다른 사용자가 예측할 수 없도록 복잡한 패스워드 사용 권장. `ACL GENPASS`로 난수 생성 가능
  
## 커맨드 권한 제어
- +@all 혹은 allcommand는 모든 커맨드 수행 권한 부여. -@all 혹은 nocommans는 아무런 커맨드 수행할 수 없음
- 특정 카테고리 권한을 추가하려면 +@<category>, 제외하려면 -@<category>를 사용. 개별 커맨드의 권한을 추가, 제외하려면 @이 없이 바로 사용
```redis
ACL SETUER user1 +@all -@admin +bgsave +slowlog|get
```
- 모든 커맨드 수행 권한 부여. admin 카테고리 커맨드 수행 권한 제외. bgsave 커맨드와 slowlog 커맨드 중 get이라는 서브 커맨드에 대한 수행 권한만 추가로 다시 부여
- `ACL CAT`으로 레디스에 미리 정의돼 있는 카테고리 커맨드 list 확인
- `dangerous 카테고리`에는 사용하면 위험할 수 있는 커맨드가 포함
- `admin 카테고리`는 dangerous 카테고리에서 장애 유발 커맨드를 제외한 커맨드
- `fast 카테고리`는 O(1)로 수행되는 커맨드를 모아 놓은 카테고리
- `slow 카테고리`는 fast에 속하지 않은 커맨드가 속한 카테고리
- `keyspace 카테고리`는 키와 관련된 커맨드가 포함된 카테고리
- `read 카테고리`는 데이터를 읽어오는 커맨드가 포함된 카테고리
- `write 카테고리`는 데이터를 쓰는 커맨드가 포함된 카테고리

## 키 접근 제어
- 유저가 접근할 수 있는 키도 제어 가능. 레디스에서 프리픽스를 이용해 키를 생성하는 것이 일반적. 프리픽스 규칙을 미리 정해뒀다면 특정한 프리픽스를 가지고 있는 키에만 접근할 수 있도록 제어 가능
- ~* 혹은 allkeys 키워드는 모든 키에 대한 접근 가능. ~<pattern>을 이용해 접근 가능한 키 정의
```redis
ACL SETUSER loguser ~log:* %R~mail:* %R~sms:*
```
- log: 프리픅스에 대한 모든 접근 권한 부여. mail이나 sms에는 읽기 접근 권한만 부여

## 셀렉터
- 버전 7에서 새로 추가된 개념으로, 더 유연한 ACL 규칙을 위해 도입
```redis
ACL SETUSER loguser resetkeys ~log:* (+GET ~mail:*)
```
- 괄호 안에 정의된 것이 셀렉터. loguser 에 정의된 모든 키를 리셋하고 log:에 대한 모든 접근 권한을 부여한 뒤, mail:에 대해서는 get만 가능하도록 설정한 것

## 유저 초기화
- reset 커맨드를 이용해 유저에 대한 모든 권한 회수 및 기본 상태로 변경 가능
- resetpass, resetkeys, resetchannels, off, -@all 상태로 변경돼 ACL SETUSER를 한 직후와 동일

## ACL 규칙 파일로 관리하기
- redis.conf에 저장. ACL 파일을 따로 관리해 유저 정보만 저장하는 것도 가능
- /etc/redis/users.acl 파일로 ACL 파일을 관리하고 싶다면 redis.conf에 아래 커맨드를 추가한다. `aclfile /etc/redis/users.acl`
- ACL 파일을 사용하지 않을 때에는 `CONFIG REWRITE`로 레디스의 모든 설정값과 ACL룰을 한 번에 redis.conf에 저장
- ACL 파일을 따로 관리할 경우 `ACL LOAD`나 `ACL SAVE`로 유저 데이터를 레디스로 로드하거나 저장하는 것이 가능해지기에 조금 더 유연한 관리가 가능

# SSL/TLS
## 레디스에서 SSL/TLS 사용하기
```redis
make BUILD_TILS=yes
```
- 기본적으로 레디스에서 SSL/TLS 설정은 비활성화. SSL/TLS 프로토클을 사용하기 위해서는 처음 빌드할 때 위와 같이 정의
- 레디스 인스턴스와 클라이언트 간 동일한 인증서 사용
```redis
tls-port <포트번호>
tls-cert-file /path/to/redis.crt
tls-key-file /path/to/redis.key
tls-ca-cert-file /path/to/ca.crt
```
- 연결 시 인증서 입력
## SSL/TLS를 사용한 HA 구성
- SSL/TLS 사용하는 마스터와 TLS 연결을 이용한 복제를 하기 위해서는 복제본도 tls 설정을 추가, `tls-replication` 값을 yes로 설정
- 센티널에서도 복제 연결을 할 때와 마찬가지로 센티널 구성 파일에 tls 설정 추가
- 클러스터 구성에서도 `tls-cluster yes` 및 tls 설정 추가