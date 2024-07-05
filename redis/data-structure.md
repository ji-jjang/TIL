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

### NX
- 지정한 키가 없을 때에만 새로운 키를 저장
```redis
> SET hello newWorld NX // 이미 등록된 키는 저장되지 않음
(nil)
```

### XX
- 지정한 키가 있을 때에만 새로운 값으로 덮어 쓰며 새로운 키를 생성하지 않음
```redis
> SET hello newWorld XX
OK

> GET hello
"newWorld"
```

### INCR, INCRBY
- string 자료 구조에 저장된 숫자를 원자적으로 조작
```redis
> Set counter 100
OK

> INCR counter // counter 값을 1 증가
101

> INCRBY counter 100 // counter 값을 100 증가
201
```

### MSET, MGET
- 한 번에 여러 키를 조작
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
- 순서를 가지는 문자열의 목록
- 하나의 list에는 최대 42억여 개의 아이템을 저장할 수 있다.
- **배열처럼 인덱스를 이용해 데이터에 직접 접근하거나 스택과 큐로서 사용**할 수도 있다.

### LPUSH
- list의 왼쪽(head)에 데이터를 추가
```redis
> LPUSH mylist E
(integer) 1
```

### RPUSH
- list의 오른쪽(head)에 데이터 추가
```redis
> RPUSH mylist B
(integer) 2
```

### LRANGE
- 시작과 끝 인덱스를 인수로 받아 list에 들어 있는 데이터 조회
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