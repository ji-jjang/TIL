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

## Hash
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

## Set
- **정렬되지 않은 문자열 모음**이다.
- 하나의 Set에서 아이템은 중복해서 저장되지 않으며, 교집합, 합집합, 차집합 등의 집합 연산과 관련한 커맨드를 제공한다. 객체 간의 관계를 계산하거나 유일한 원소를 구해야 할 경우 유용하다.

### SADD
- **Set에 아이템을 저장하며, 한 번에 여러 개의 아이템을 저장**할 수 있다.
- 저장되는 실제 아이템 수를 반환한다. 
```redis
> SADD myset A
(integer) 1

> SADD myset A A A A A B B B B C C C D D E
(integer) 4
```

### SMEMBERS
- **Set 자료 구조에 저장된 전체 아이템을 저장된 순서와 관계없이 출력**한다. 
```redis
> SMEMBERS myset
1) "A"
2) "B"
3) "C"
4) "D"
5) "E"
```

### SREM, SPOP
- **SREM 커맨드는 Set에서 원하는 데이터를 삭제**할 수 있다.
- **SPOP 커맨드는 Set 내부의 아이템 중 랜덤으로 하나의 아이템을 반환하는 동시에 set에서 그 아이템을 삭제**한다.
```redis
> SREM myset B
(integer) 1

> SPOP myset
"A"
```

### SUNION(합집합), SINTER(교집합), SDIFF(차집합)
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

## Sorted Set
- 스코어 값에 따라 정렬되는 고유한 문자열의 집합이다.
- 모든 아이템은 스코어-값 쌍을 가지며, 저장될 때부터 스코어 값으로 정렬돼 저장된다.
- 같은 스코어를 가진 아이템은 데이터의 사전 순으로 정렬돼 저장된다.
- **데이터가 중복 없이 저장되므로 Set과 유사**하며, **각 아이템은 스코어라는 데이터에 연결되어 있어 Hash와도 유사**하다. 또한 **모든 아이템은 스코어 순으로 정렬되어 list처럼 인덱스를 이용해 접근**할 수도 있다.

### ZADD