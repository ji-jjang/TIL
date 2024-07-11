
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [Redis 자료 구조](#redis-자료-구조)
  - [1. string](#1-string)
    - [1) NX](#1-nx)
    - [2) XX](#2-xx)
    - [3) INCR, INCRBY](#3-incr-incrby)
    - [4) MSET, MGET](#4-mset-mget)
  - [2. list](#2-list)
    - [1) LPUSH](#1-lpush)
    - [2) RPUSH](#2-rpush)
    - [3) LRANGE](#3-lrange)
    - [4) LPOP](#4-lpop)
    - [5) LTRIM](#5-ltrim)
    - [6) LINSERT](#6-linsert)
    - [7) LSET](#7-lset)
    - [8) LINDEX](#8-lindex)
  - [3. Hash](#3-hash)
    - [1) HSET](#1-hset)
    - [2) HGET, HMGET, HGETALL](#2-hget-hmget-hgetall)
  - [4. Set](#4-set)
    - [1) SADD](#1-sadd)
    - [2) SMEMBERS](#2-smembers)
    - [3) SREM, SPOP](#3-srem-spop)
    - [4) SUNION(합집합), SINTER(교집합), SDIFF(차집합)](#4-sunion합집합-sinter교집합-sdiff차집합)
  - [5. Sorted Set](#5-sorted-set)
    - [1) ZADD](#1-zadd)
    - [2) ZRANGE](#2-zrange)
  - [6. 비트맵](#6-비트맵)
    - [SETBIT, GETBIT, BITFIELD, BITCOUNT](#setbit-getbit-bitfield-bitcount)
  - [7. Hyperloglog](#7-hyperloglog)
    - [PFADD, PFCOUNT](#pfadd-pfcount)
  - [8. Geospatial](#8-geospatial)
    - [GEOADD, GEOPOS, GEODIST, GEOSEARCH, BYRADIUS, BYBOX](#geoadd-geopos-geodist-geosearch-byradius-bybox)
- [레디스에서 키를 관리하는 법](#레디스에서-키를-관리하는-법)
  - [키의 자동 생성과 삭제](#키의-자동-생성과-삭제)
    - [1. 키가 존재하지 않을 때 아이템을 넣으면 아이템을 삽입하기 전 빈 자료구조 생성](#1-키가-존재하지-않을-때-아이템을-넣으면-아이템을-삽입하기-전-빈-자료구조-생성)
    - [2. 모든 아이템을 삭제하면 키도 자동으로 삭제(stream은 예외)](#2-모든-아이템을-삭제하면-키도-자동으로-삭제stream은-예외)
    - [3. 키가 없는 상태에서 키 삭제, 아이템 삭제, 자료 구조 크기 조회 같은 읽기 전용 커맨드를 수행하면 에러 대신 키가 있으나 아이템이 없는 것처럼 동작](#3-키가-없는-상태에서-키-삭제-아이템-삭제-자료-구조-크기-조회-같은-읽기-전용-커맨드를-수행하면-에러-대신-키가-있으나-아이템이-없는-것처럼-동작)
  - [키와 관련한 커맨드](#키와-관련한-커맨드)
    - [키 조회](#키-조회)
      - [1. EXISTS](#1-exists)
      - [2. KEYS](#2-keys)
      - [3. SCAN](#3-scan)
      - [4. SORT](#4-sort)
      - [5. RENAME / RENAMENX](#5-rename--renamenx)
      - [6. COPY](#6-copy)
      - [7. TYPE](#7-type)
      - [8. OBJECT](#8-object)
    - [키 삭제](#키-삭제)
      - [1. FLUSHALL](#1-flushall)
      - [2. DEL](#2-del)
      - [3. UNLINK](#3-unlink)
    - [키 만료](#키-만료)
      - [1. EXPIRE](#1-expire)
      - [2. EXPIREAT](#2-expireat)
      - [3. EXPIRETIME](#3-expiretime)
      - [4. TTL](#4-ttl)

<!-- /code_chunk_output -->



# Redis 자료 구조
## 1. string
- 최대 512MB 문자열 데이터 저장
- 모든 종류의 문자열이 **binary-safe**하게 저장되어 JPEG 이미지와 같은 바이트 값, HTTP 응답값 등 다양한 데이터 저장
- **key, value가 일대일로 연결되는 유일한 자료 구조**, 다른 자료 구조에서는 하나의 키에 여러 개의 아이템이 연결
```redis
> SET hello world
OK

> GET hee
(nil)

> GET hello
"world"
```
- **hello라는 키에 다른 값이 연결되었다면, 기존 값은 새로 입력된 값으로 교체**

### 1) NX
- **지정한 키가 없을 때에만 새로운 키를 저장**
```redis
> SET hello newWorld NX // 이미 등록된 키는 저장되지 않음
(nil)
```

### 2) XX
- **지정한 키가 있을 때에만 새로운 값으로 덮어 쓰며 새로운 키를 생성하지 않음**
```redis
> SET hello newWorld XX
OK

> GET hello
"newWorld"
```

### 3) INCR, INCRBY
- **string 자료 구조에 저장된 숫자를 원자적으로 조작**
```redis
> Set counter 100
OK

> INCR counter // counter 값을 1 증가
101

> INCRBY counter 100 // counter 값을 100 증가
201
```

### 4) MSET, MGET
- **한 번에 여러 키를 조작**
- 성능이 중요한 대규모 시스템에서는 밀리세컨드 단위의 속도 향상도 서비스 전체의 속도 향상
```redis
> MSET a 1 b 2 c 3
OK

> MGET a b c
1) "1"
2) "2"
3) "3" 
```

## 2. list
- **순서를 가지는 문자열의 목록**
- 하나의 list에는 최대 42억여 개의 아이템을 저장할 수 있다.
- **배열처럼 인덱스를 이용해 데이터에 직접 접근하거나 스택과 큐로서 사용**할 수도 있다.

### 1) LPUSH
- **list의 왼쪽(head)에 데이터를 추가**
```redis
> LPUSH mylist E
(integer) 1
```

### 2) RPUSH
- **list의 오른쪽(head)에 데이터 추가**
```redis
> RPUSH mylist B
(integer) 2
```

### 3) LRANGE
- **시작과 끝 인덱스를 인수로 받아 list에 들어 있는 데이터 조회**
```redis
> LPUSH mylist A B C D E
(integer) 7

> LRANGE mylist 0 -1
1) "E"
2) "D"
3) "C"
4) "B"
5) "A"
6) "E"
7) "B"

> LRANGE mylist 0 3
1) "E"
2) "D"
3) "C"
4) "B"
```

### 4) LPOP
- **list에 저장된 첫 번째 아이템을 반환하는 동시에 list에서 삭제**한다. 숫자와 함께 사용하면 지정한 숫자만큼의 아이템을 반복해서 반환한다.
```redis
> LPOP mylist
"E"
LPOP mylist 2
1) "D"
2) "C"
```

### 5) LTRIM
- **시작과 끝 아이템의 인덱스를 인자로 전달받아 지정한 범위에 속하지 않은 아이템은 모두 삭제**하지만, LPOP과 같이 삭제되는 아이템을 반환하지는 않는다.
```redis
> LRANGE mylist 0 -1
1) "B"
2) "A"
3) "E"
4) "B"
> LTRIM mylist 0 1
OK
> LRANGE mylist 0 -1
1) "B"
2) "A"
```
- LPUSH와 LTRIM 커맨드를 함께 사용하면 고정된 길이의 큐를 쉽게 유지할 수 있다.
```redis
> LPUSH logdata <data>
> LTRIM logdata 0 999
```
- 로그
-그 데이터를 일단 쌓은 뒤 주기적으로 배치 처리를 이용해서 삭제하는 것보다 위와 같은 방식으로 삭제하는 것이 훨씬 효율적이다. 그 이유는 매번 큐의 마지막 데이터만 삭제되기 때문이다. list에서 tail을 삭제하는 것은 O(1)로 동작한다.
- 인덱스나 데이터를 이용해 list의 중간 데이터에 접근할 때에는 O(n)으로 처리된다.

### 6) LINSERT
- **원하는 데이터의 앞이나 뒤에 데이터를 추가**할 수 있다. 데이터의 앞에 추가하려면 BEFORE, 뒤에 추가하려면 AFTER 옵션을 사용한다. 만약 지정한 데이터가 없으면 오류를 반환한다.
```redis
> LRANGE mylist 0 -1
1) "B"
2) "A"
> LINSERT mylist BEFORE B E
(integer) 3
> LRANGE mylist 0 -1
1) "E"
2) "B"
3) "A"
> LINSERT mylist AFTER B E
(integer) 4
> LRANGE mylist 0 -1
1) "E"
2) "B"
3) "E"
4) "A"
```  

### 7) LSET
- **지정한 인덱스의 데이터를 신규 입력하는 데이터로 덮어 쓴다**. 만약 list의 범위를 벗어난 인덱스를 입력하면 에러를 반환한다.
```redis
> LSET mylist 2 F
OK
> LRANGE mylist 0 -1
1) "E"
2) "B"
3) "F"
4) "A"
```

### 8) LINDEX
- **원하는 인덱스의 데이터를 확인**할 수 있다.
```redis
> LINDEX mylist 3
"A"
```

## 3. Hash
- 레디스에서 **hash는 필드-값 쌍을 가진 아이템의 집합**이다.
- **필드는 하나의 hash 내에서 유일**하며, **필드와 값 모두 문자열 데이터로 저장**된다.
- **hash는 객체를 표현하기 적절한 자료구조**이기에 관계형 데티어베이스의 테이블 데이터로 변환하는 것도 간편하다.
- 칼럼이 고정된 관계형 데이터베이스 테이블과 달리 **hash에서 필드를 추가하는 것은 간단**하다.
- hash에서는 각 아이템마다 다른 필드를 가질 수 있으며 **동적으로 다양한 필드를 추가**할 수 있다.
- 관계형 데이터베이스 테이블에 데이터를 저장할 때에는 미리 합의된 칼럼 데이터를 저장할 수 밖에 없는데, **hash에는 새로운 필드에 데이터를 저장**할 수 있기에 유연한 개발이 가능하다.

### 1) HSET
- **hash에 아이템을 저장**한다. 한 번에 여러 필드-값 쌍을 저장할 수도 있다.
```redis
> HSET Space:123 Name "Happy"
(integer) 1

> HSET Space:123 TypeID 123
(integer) 1

> HSET Space:123 Version 2024
(integer) 1

> HSET Space:345 Name "HOHO" TypeID 345
(integer) 2
```

### 2) HGET, HMGET, HGETALL
- **HGET 커맨드로 키와 아이템 필드를 함께 입력**하여 hash에 저장된 데이터를 가져올 수 있다.
- **HMGET 커맨드로 하나의 hasㅇ내에서 다양한 필드의 값**을 가져올 수 있다.
- **HGETALL 커맨드는 hash 내의 모든 필드-값 쌍을 차례로 반환**한다.
```redis
> HGET Space:123 TypeID
"123"

> HMGET Space:123 Name TypeID
1) "Happy"
2) "123"

> HMGET Space:345 Name TypeID
1) "HOHO"
2) "345"

> HGETALL Space:123
1) "Name"
2) "Happy"
3) "TypeID"
4) "123"
5) "Version"
6) "2024"
```

## 4. Set
- **정렬되지 않은 문자열 모음**이다.
- 하나의 Set에서 아이템은 중복해서 저장되지 않으며, 교집합, 합집합, 차집합 등의 집합 연산과 관련한 커맨드를 제공한다. 객체 간의 관계를 계산하거나 유일한 원소를 구해야 할 경우 유용하다.

### 1) SADD
- **Set에 아이템을 저장하며, 한 번에 여러 개의 아이템을 저장**할 수 있다.
- 저장되는 실제 아이템 수를 반환한다. 
```redis
> SADD myset A
(integer) 1

> SADD myset A A A A A B B B B C C C D D E
(integer) 4
```

### 2) SMEMBERS
- **Set 자료 구조에 저장된 전체 아이템을 저장된 순서와 관계없이 출력**한다. 
```redis
> SMEMBERS myset
1) "A"
2) "B"
3) "C"
4) "D"
5) "E"
```

### 3) SREM, SPOP
- **SREM 커맨드는 Set에서 원하는 데이터를 삭제**할 수 있다.
- **SPOP 커맨드는 Set 내부의 아이템 중 랜덤으로 하나의 아이템을 반환하는 동시에 set에서 그 아이템을 삭제**한다.
```redis
> SREM myset B
(integer) 1

> SPOP myset
"A"
```

### 4) SUNION(합집합), SINTER(교집합), SDIFF(차집합)
```redis
> SADD 111 A B C D E
(integer) 5
> SADD 222 D E F G H
(integer) 5

> SINTER 111 222
1) "D"
2) "E"

> SUNION 111 222
1) "A"
2) "B"
3) "C"
4) "D"
5) "E"
6) "F"
7) "G"
8) "H"

> SDIFF 111 222
1) "A"
2) "B"
3) "C"
```

## 5. Sorted Set
- **스코어 값에 따라 정렬되는 고유한 문자열의 집합**이다.
- 모든 아이템은 스코어-값 쌍을 가지며, 저장될 때부터 스코어 값으로 정렬돼 저장된다.
- 같은 스코어를 가진 아이템은 데이터의 사전 순으로 정렬돼 저장된다.
- **데이터가 중복 없이 저장되므로 Set과 유사**하며, **각 아이템은 스코어라는 데이터에 연결되어 있어 Hash와도 유사**하다. 또한 **모든 아이템은 스코어 순으로 정렬되어 list처럼 인덱스를 이용해 접근**할 수도 있다.
- **redis에서 list 접근 시간 복잡도는 O(n)이고, sorted set은 O(logn)이다.**

### 1) ZADD
- sorted set에 스코어-값 쌍으로 아이템을 저장한다.
- **한 번에 여러 아이템을 저장할 수 있고, 저장되는 동시에 스코어 값으로 정렬**된다.
```redis
> ZADD score:220817 100 user:A
(integer) 1

> ZADD score:220817 150 user:B 150 user:C 200 user:F 300 user:E
(integer) 4
```
- 다양한 옵션을 지원한다.
    - **XX**: 아이템이 이미 존재할 때만 스코어를 업데이트한다.
    - **NX**: 아이템이 존재하지 않을 때에만 신규 삽입하며, 기존 아이템의 스코어를 업데이트하지 않는다.
    - **LT**: 업데이트하고자 하는 스코어가 기존 아이템의 스코어보다 작을 때에만 업데이트한다. 기존에 아이템이 존재하지 않을 때에는 새로운 데이터를 삽입한다.
    - **GT**: 업데이트하고자 하는 스코어가 기존 아이템의 스코어보다 클 때에만 업데이트한다. 기존에 아이템이 존재하지 않을 때에는 새로운 데이터를 삽입한다.

### 2) ZRANGE
- **ZRANGE 커맨드로 sorted set에 저장된 데이터를 조회**할 수 있으며, start와 stop 범위를 항상 입력해야 한다.
```redis
ZRANGE key start stop [BYSCORE | BYLEX] [REV] [LIMIT offset count] [WITHSCORES]
```
```redis
> ZRANGE score:220817 0 -1 WITHSCORES
 1) "user:A"
 2) "100"
 3) "user:B"
 4) "150"
 5) "user:C"
 6) "150"
 7) "user:F"
 8) "200"
 9) "user:E"
10) "300"

> ZRANGE score:220817 0 -1 WITHSCORES REV
 1) "user:E"
 2) "300"
 3) "user:F"
 4) "200"
 5) "user:C"
 6) "150"
 7) "user:B"
 8) "150"
 9) "user:A"
10) "100"
```
- **BYSCORE 옵션을 사용하면 스코어를 이용해 데이터를 조회**할 수 있다.
```redis
> ZRANGE score:220817 100 150 BYSCORE WITHSCORES
1) "user:A"
2) "100"
3) "user:B"
4) "150"
5) "user:C"
6) "150"
```
- 인수로 전달하는 스코어에 **( 문자를 추가하면 해당 스코어를 포함하지 않는 값**만 조회한다.
```redis
> ZRANGE score:220817 (100 150 BYSCORE WITHSCORES
1) "user:B"
2) "150"
3) "user:C"
4) "150"
```
- 스코어의 최솟값과 최대값을 표현하기 위해 **infinity를 의미하는 -inf, +inf** 값을 사용한다.
- **아이템을 역순으로 조회하고 싶다면 REV** 커맨드를 쓸 수 있으며, **이때 최소값과 최대값 스코어의 전달 순서는 변경**해야 한다.
```redis
> ZRANGE score:220817 200 +inf  BYSCORE WITHSCORES
1) "user:F"
2) "200"
3) "user:E"
4) "300"

> ZRANGE score:220817 +inf 200 BYSCORE WITHSCORES REV
1) "user:E"
2) "300"
3) "user:F"
4) "200"
```
- 스코어가 같으면 데이터는 사전 순으로 정렬된다. **스코어가 같을 때 BYLEX 옵션을 사용하면 사전식 순서를 이용해 특정 아이템을 조회**할 수 있다.

```redis
> ZADD mySortedSet 0 apple 0 banana 0 candy 0 dream 0 egg 0 frog
(integer) 6

> ZRANGE mySortedSet 0 -1 WITHSCORES
 1) "apple"
 2) "0"
 3) "banana"
 4) "0"
 5) "candy"
 6) "0"
 7) "dream"
 8) "0"
 9) "egg"
10) "0"
11) "frog"
12) "0"

> ZRANGE mySortedSet (b (f BYLEX
1) "banana"
2) "candy"
3) "dream"
4) "egg"
```
- 입력한 문자열을 **포함하려면 ( 을, 포함하지 않으려면 [** 문자를 사용한다.
- **사전식 문자열의 가장 처음은 - 문자로, 가장 마지막은 + 문자로 대체할 수 있기에 ZRANGE <KEY> - + BYLEX 커맨드는 sorted set에 저장된 모든 데이터를 조회**한다.

## 6. 비트맵
- 비트맵(Bitmap)은 독자적인 자료 구조는 아니며 string 자료 구조에 bit 연산을 수행할 수 있도록 확장한 형태이다.
- string 자료 구조가 binary safe하고 최대 512MB 값을 저장할 수 있기 때문에 $2^{32}$ 비트를 가지고 있는 비트맵 형태라고 볼 수 있다.
- 비트맵을 사용할 때 가장 큰 장점은 저장 공간을 획기적으로 줄일 수 있다. 

### SETBIT, GETBIT, BITFIELD, BITCOUNT
- SETBIT: 비트를 저장할 수 있다. 설정되기 전의 값을 반환한다.
- GETBIT: 비트를 조회한다.
- BITFIELD: 한 번에 여러 비트를 저장한다.
```redis
> SETBIT mybitmap 2 1
(integer) 0

> SETBIT mybitmap 2 0
(integer) 1

> SETBIT mybitmap 2 1
(integer) 0

> GETBIT mybitmap 2
(integer) 1

> BITFIELD mybitmap SET u1 6 1 SET u1 10 1 SET u1 14 1
1) (integer) 0
2) (integer) 0
3) (integer) 0

> BITCOUNT mybitmap
(integer) 4
```

## 7. Hyperloglog
- 집합의 원소 개수인 **카디널리티를 추정**할 수 있는 자료 구조이다.
- **대량 데이터에서 중복되지 않는 고유한 값을 집계**할 때 유용하게 사용할 수 있다.
- Set과 달리 입력되는 데이터 그 자체를 저장하지 않고, 자체적인 방법으로 데이터를 변경해 처리한다. 따라서 저장되는 데이터 개수에 구애받지 않고 일정한 메모리를 유지할 수 있으며, 중복되지 않는 유일한 원소의 개수를 계산할 수 있다.
- 하나의 hyperloglog 자료 구조는 최대 12KB 크기를 가지며, 카디널리티 추정 오차는 0.81%로 비교적 정확하게 데이터를 추정할 수 있고, $2^{64}$개의 아이템을 저장할 수 있다.

### PFADD, PFCOUNT
- PFADD: **아이템을 저장**한다.
- PFCOUNT: **저장된 아이템의 개수, 카디널리티를 추정**한다.
```redis
> PFADD members 123
(integer) 1

> PFADD members 1234
(integer) 1

> PFADD members 12345
(integer) 1

> PFCOUNT members
(integer) 3
```

## 8. Geospatial
- **경도, 위도 데이터 쌍의 집합으로 간편하게 지리 데이터를 저장할 수 있는 자료 구조**이다.
- 내부적으로 sorted set으로 저장되며, 하나의 자료 구조 안에 키는 중복되어 저장되지 않는다.

### GEOADD, GEOPOS, GEODIST, GEOSEARCH, BYRADIUS, BYBOX
- GEOADD: **경도, 위도, member 순서로 저장**된다.
- GEOPOS: **저장된 위치 데이터를 조회**한다.
- GEODIST: **두 아이템 사이의 거리를 반환**한다.
```redis
> GEOADD travel 14.3999 50.09999 prague
(integer) 1

> GEOADD travel 127.001 37.564 seoul -122.434 37.785 SanFrancisco
(integer) 2

> GEOPOS travel prague
1) 1) "14.39989775419235229"
   2) "50.09998925201490749"

> GEODIST travel seoul prague KM
"8252.9117"
```

# 레디스에서 키를 관리하는 법

## 키의 자동 생성과 삭제

### 1. 키가 존재하지 않을 때 아이템을 넣으면 아이템을 삽입하기 전 빈 자료구조 생성

### 2. 모든 아이템을 삭제하면 키도 자동으로 삭제(stream은 예외)

### 3. 키가 없는 상태에서 키 삭제, 아이템 삭제, 자료 구조 크기 조회 같은 읽기 전용 커맨드를 수행하면 에러 대신 키가 있으나 아이템이 없는 것처럼 동작

## 키와 관련한 커맨드

### 키 조회
#### 1. EXISTS 
- 키가 존재하면 1, 존재하지 않으면 0을 반환.

#### 2. KEYS 
- 레디스에 저장된 키를 모두 조회하는 커맨드(GLOB).
- **레디스는 싱글 스레드로 동작하기에 실행 시간이 오래 걸리는 커맨드를 수행하는 동안 다른 커맨드 차단**

#### 3. SCAN 
- cursor [MATCH pattern] [COUNT count] [TYPE type]
- SCAN은 KEYS를 대체해 키를 조회할 때 사용하는 커맨드
- 특정 범위의 키만 조회할 수 있기 때문에 비교적 안전
```redis
127.0.0.1:6379> SCAN 0
1) "25"
2)  1) "222"
    2) "b"
    3) "score:220817"
    4) "mybitmap"
    5) "mySortedSet"
    6) "hellod"
    7) "111"
    8) "myset"
    9) "travel"
   10) "counter"

127.0.0.1:6379> SCAN 25
1) "31"
2)  1) "members"
    2) "Space:123"
    3) "product:234"
    4) "product:123"
    5) "mylist"
    6) "a"
    7) "mybm"
    8) "hello"
    9) "Space:345"
   10) "c"

127.0.0.1:6379> SCAN 31
1) "0"
2) (empty array)

127.0.0.1:6379> SCAN 0 MATCH *m*
1) "25"
2) 1) "mybitmap"
   2) "mySortedSet"
   3) "myset"

127.0.0.1:6379> SCAN 0 TYPE zset
1) "25"
2) 1) "score:220817"
   2) "mySortedSet"
   3) "travel"
```
- 기본적으로 한 번에 반환되는 키의 개수는 10개 정도, COUNT 옵션을 사용하면 이 개수를 조정할 수 있다. **하지만 데이터는 정확하게 지정한 개수만큼 출력되지 않는데, 레디스는 메모리를 스캔하며 저장된 형상에 따라 몇 개의 키를 더 읽는 것이 효율적이라고 판단하면 더 읽은 뒤 함께 반환**한다.
- SCAN과 비슷한 명령어로는 SSCAN, HSCAN, ZSCAN이 있고 각각 set, hash, sorted set에서 아이템을 조회하기 위해 사용되는 SMEMBERS, HGETALL, ZRANGE WITHSCORE를 대체해서 안전하게 호출할 수 있는 커맨드다.
  
#### 4. SORT 
- key [BY pattern] [LIMIT offset count] [GET pattern [GET pattern ...]] [ASC | DESC] [ALPHA] [STORE destination]
- list, set, sorted set에서만 사용할 수 있는 커맨드로, 키 내부의 아이템을 정렬해 반환한다.
- LIMIT 옵션을 사용하면 일부 데이터만 조회할 수 있고, ASC/DESC를 통해 정렬 순서를 변경할 수 있다. 
- 정렬 대상이 문자열인 경우 ALPHA 옵션을 사용한다.

```redis
127.0.0.1:6379> LPUSH mylist a
(integer) 1

127.0.0.1:6379> LPUSH mylist b
(integer) 2

127.0.0.1:6379> LPUSH mylist c
(integer) 3

127.0.0.1:6379> LPUSH mylist abcde
(integer) 4

127.0.0.1:6379> SORT mylist ALPHA
1) "a"
2) "abcde"
3) "b"
4) "c"
```

#### 5. RENAME / RENAMENX
- RENAME, RENAMENX 커맨드는 모두 키의 이름을 변경하는 커맨드다. RENAMENX는 오직 변경할 키가 존재하지 않을 때에만 동작한다.
```redis
127.0.0.1:6379> SET a apple
OK

127.0.0.1:6379> SET b banana
OK

127.0.0.1:6379> RENAME a aa
OK

127.0.0.1:6379> GET aa
"apple"

127.0.0.1:6379> RENAMENX aa cc
(integer) 1

127.0.0.1:6379> GET cc
"apple"
```

#### 6. COPY
- source destination [DB destination-db] [REPLACE]
- Source에 저장된 키를 destination 키에 복사한다. Destination에 지정한 키가 이미 있는 경우 에러가 반환된다. 이때, REPLACE 옵션을 사용하면 destination 키를 삭제한 뒤 값을 복사하기 때문에 에러가 발생하지 않는다.
```redis
127.0.0.1:6379> SET b banana
OK

127.0.0.1:6379> COPY b bb
(integer) 1

127.0.0.1:6379> get bb
"banana"

127.0.0.1:6379> SET b bababa
OK

127.0.0.1:6379> COPY b bb REPLACE
(integer) 1

127.0.0.1:6379> get bb
"bababa"
```

#### 7. TYPE 
- key
- 지정한 키의 자료 구조 타입을 반환한다.

#### 8. OBJECT 
- <subcommand> [<arg> [value] [opt] ...]
- 키에 대한 상세 정보를 반환한다.
- subcommand 옵션으로는 ENCODING, IDLETIME 등이 있으며 키가 내부적으로 어떻게 저장됐는지,ㅋ혹은ㄴ키가 호출되지 않은 시간이 얼마나 됐는지 등을 확인할 수 있다.

### 키 삭제
#### 1. FLUSHALL 
- [ASYNC | SYNC]
- **레디스에 저장된 모든 키를 삭제**한다. 기본적으로 **동기적으로 동작**하여 커맨드가 실행되는 도중 다른 처리를 할 수 없다. ASYNC 옵션을 사용하면 백그라운드로 실행된다. flush는 백그라운드로 실행되고, 커맨드가 수행됐을 때 존재했던 키만 삭제해서 flush되는 중 새로 생성된 키는 삭제되지 않는다.
- lazyfree-lazy-user-flush 옵션이 yes인 경우 ASYNC 옵션 없이 FLUSHALL 커맨드를 사용하더라도 백그라운드로 키 삭제 작업이 동작한다. 기본값은 no이다.

#### 2. DEL 
- **키와 키에 저장된 모든 아이템을 삭제하는 커맨드. 기본적으로 동기적으로 동작**한다.

#### 3. UNLINK
- **백그라운드에서 다른 스레드에 의해 처리되며, 우선 키와 연결된 데이터의 연결을 끊는다**.
- set, sorted set과 같이 하나의 키에 여러 개의 아이템이 저장된 자료 구조의 경우 1개의 키를 삭제하는 DEL 커맨드를 수행하는 것은 레디스 인스턴스에 영향을 끼칠 가능성이 있다. 100만 개의 아이템이 저장돼 있는 sorted set 키를 DEL 커맨드로 삭제하는 것은 전체 키가 100만 개 있는 레디스에서 동기적인 방식으로 FLUSH ALL을 수행하는 것과 같다. 따라서 키에 저장된 값이 많은 경우 DEL이 아니라 UNLINK를 사용하는 것이 좋다.
- lazyfree-lazy-user-del 옵션이 yes인 경우 모든 DEL 커맨드는 UNLINK로 동작해 백그라운드 키를 삭제한다. 기본값은 no이다.

### 키 만료
#### 1. EXPIRE 
- 키가 만료될 시간을 초 단위로 정의할 수 있다.
- NX: 해당 키에 만료 시간이 정의돼 있지 않을 경우에만 커맨드 수행
- XX: 해당 키에 만료 시간이 정의돼 있을 때에만 커맨드 수행
- GT: 현재 키가 가지고 있는 만료 시간보다 새로 입력한 초가 더 클 때에만 수행
- LT: 현재 키가 가지고 있는 만료 시간보다 새로 입력한 초가 더 작을 때에만 수행

#### 2. EXPIREAT
- key unix-time-second 
- 키가 특정 유닉스 타임스탬프에 만료될 수 있도록 키의 만료 시간을 직접 지정한다.

#### 3. EXPIRETIME 
- key
-ㅇ키가 삭제되는 유닉스 타임스탬프를 초 단위로 변환한다. 키가 존재하지만 만료 시간이 설정돼 있지 않은 경우에는 -1을, 키가 없을 때에는 -2를 반환한다.

#### 4. TTL 
- key
- 키가 몇초 뒤에 만료되는지 반환한다. 키가 존재하지만 만료 시간이 설정돼 있지 않은 경우에는 -1을, 키가 없을 때에는 -2를 반환한다.

> PEXPIRE, PEXPIREAT, PEXPIRETIME, PTTL은 밀리초 단위로 계산된다는 점만 다르다.