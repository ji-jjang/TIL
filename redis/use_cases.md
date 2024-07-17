# sorted set을 이용한 실시간 리더보드
## 요구 사항
- 리더보드는 기본적으로 사용자의 스코어를 기반으로 데이터를 정렬하는 서비스이기 때문에 **사용자 증가에 따른 가공해야 할 데이터가 몇 배로 증가**한다. 또한, **실시간 반영**이 필요하다.
  
## sorted set
- 유저의 스코어를 sorted set의 가중치로 설정한다면, 스코어 순으로 유저가 정렬되기 때문에 리더보드의 데이터를 읽어오기 위해 **매번 데이터를 정렬할 필요가 없다**.

### 1. 서비스에 일별 리더보드를 도입
```redis
127.0.0.1:6379> ZADD dilay-score:240712 28 player:286
(integer) 1
127.0.0.1:6379> ZADD dilay-score:240712 400 player:234
(integer) 1
127.0.0.1:6379> ZADD dilay-score:240712 45 player:101
(integer) 1
127.0.0.1:6379> ZADD dilay-score:240712 357 player:24
(integer) 1
127.0.0.1:6379> ZRANGE dilay-score:240712 0 -1 withscores
1) "player:286"
2) "28"
3) "player:101"
4) "45"
5) "player:24"
6) "357"
7) "player:234"
8) "400"
```
### 2. 상위 스코어 세 명의 유저를 출력
```redis
127.0.0.1:6379> ZREVRANGE dilay-score:240712 0 2 withscores
1) "player:234"
2) "400"
3) "player:24"
4) "357"
5) "player:101"
6) "45"
```

- sorted set은 중복을 허용하지 않는 자료 구조이므로 **스코어가 다르면 기존 데이터의 스코어만 새로 입력한 스코어로 업데이**트 된다.
- 직접 스코어의 값을 지정해서 변경하지 않고도 ZINCRBY로 업데이트 할 수 있다. 
```redis
127.0.0.1:6379> ZREVRANGE daily-score:240712 0 2 withscores
1) "player:286"
2) "200"

127.0.0.1:6379> ZINCRBY daily-score:240712 300 player:286
"300"

127.0.0.1:6379> ZREVRANGE daily-score:240712 0 2 withscores
1) "player:286"
2) "500"
```

### 3. 랭킹 합산
- 매주 월요일마다 초기화된다면? **관계형 데이터베이스에서 주간 누적 랭킹을 구현하려면 하나의 테이블에서 일자에 해당하는 데이터를 모두 가져온 뒤 선수별로 합치고, 다시 정렬하는 작업이 필요**하다. redis에서는 **ZUNIONSTORE** 커맨드를 이용해 간단하게 구현할 수 있다.
```redis
127.0.0.1:6379> ZADD daily-score:240712 100 player:101 200 player:102 300 player:103
(integer) 3
127.0.0.1:6379> ZADD daily-score:240713 150 player:101 250 player:104 350 player:105
(integer) 3
127.0.0.1:6379> ZADD daily-score:240714 120 player:106 220 player:107 320 player:108
(integer) 3

127.0.0.1:6379> ZUNIONSTORE weekly-score:2407-2 3 daily-score:240712 daily-score:240713 daily-score:240714
(integer) 9

127.0.0.1:6379> ZREVRANGE weekly-score:2407-2 0 -1 withscores
 1) "player:286"
 2) "500"
 3) "player:105"
 4) "350"
 5) "player:108"
 6) "320"
 7) "player:103"
 8) "300"
 9) "player:104"
10) "250"
11) "player:101"
12) "250"
13) "player:107"
14) "220"
15) "player:102"
16) "200"
17) "player:106"
18) "120"
```
- **데이터를 합칠 때 스코어에 가중치**를 줄 수도 있다.
```redis
127.0.0.1:6379> ZUNIONSTORE weekly-scores:2407-2 3 daily-score:240712 daily-score:240713 daily-score:240714 weights 5 1 1

(integer) 9
127.0.0.1:6379> ZREVRANGE weekly-scores:2407-2 0 -1 withscores
 1) "player:286"
 2) "2500"
 3) "player:103"
 4) "1500"
 5) "player:102"
 6) "1000"
 7) "player:101"
 8) "650"
 9) "player:105"
10) "350"
11) "player:108"
12) "320"
13) "player:104"
14) "250"
15) "player:107"
16) "220"
17) "player:106"
18) "120"
```

### 4. 최근 검색 기록
- 요구사항은 아래와 같다.
  1. 유저별로 다른 키워드 노출
  2. 검색 내역은 중복 제거
  3. 가장 최근 검색한 5개의 키워드만 사용자에게 노출
- 관계형 데이터베이스 테이블에 저장한다면 다음과 비슷한 쿼리문이 필요하다.
```sql
SELECT * ¸
FROM keyword 
WHERE user_id = 123 
ORDER BY reg_date DESC LIMIT 5;
```
- 유저가 최근에 검색했던 테이블에서 최근 5개의 데이터를 조회한다. 테이블에 데이터를 저장할 때는 기존에 사용자가 같은 키워드를 검색했는지 여부를 확인한 뒤 업데이트해야 하며 데이터가 무제한으로 쌓이는 것을 방지하기 위해 스케줄러나 배치 작업이 필요할 수 있다.
- 데이터를 가져올 때 검색한 시점을 기준으로 정렬해야 하기에 사용자와 검색 기록이 늘어날수록 많은 데이터를 테이블에서 관리해야 한다.
- redis sorted set을 이용하면, ID가 123인 유저가 2024년 7월 15일 21시 10분 10초에 지니라는 키워드로 검색했다면 아래와 같이 저장된다.
```redis
127.0.0.1:6379> ZADD search-keyword:123 20240715211010 지니
(integer) 1

127.0.0.1:6379> ZREVRANGE search-keyword:123 0 4 withscores
호랑
20240716211017
맛집
20240715212531
짜장면
20240715211532
복숭아D
20240715211030
복숭아
20240715211030

127.0.0.1:6379> ZADD search-keyword:123 20240716212017 복숭아
0
127.0.0.1:6379> ZREVRANGE search-keyword:123 0 4 withscores
복숭아
20240716212017
호랑
20240716211017
맛집
20240715212531
짜장면
20240715211532
복숭아D
20240715211030
```
- 한글이 깨진다면 `docker exec -it my-redis redis-cli --raw` 이렇게 접속하면 된다.
- **유저별로 최근 5개 검색어를 제외하고 삭제하는 작업을 진행할 때, 음수 인덱스를 이용하면 간편하게 처리할 수 있다. 음수 인덱스는 아이템의 제일 마지막 값을 -1로 시작해서 역순으로 증가하는 값**이다. 데이터가 6개 저장되었을 때 가장 오래된 데이터는 -6 인덱스다. 
- ZREMRANGEBYRANK 커맨드를 이용하면 인덱스 범위로 아이템을 삭제할 수 있다.
```redis
127.0.0.1:6379> ZADD search-keyword:123 20240716214017 우동
1

127.0.0.1:6379> ZREMRANGEBYRANK search-keyword:123 -6 -6
1

127.0.0.1:6379> ZREVRANGE search-keyword:123 0 -1 withscores
우동
20240716214017
복숭아
20240716212017
호랑
20240716211017
맛집
20240715212531
짜장면
20240715211532
```

### 5. 태그 기능
- 관계형 데이터베이스에서 태그 기능을 사욯아려면 적어도 2개의 테이블이 추가되어야 한다. 첫번째로는 태그 테이블, 두 번째로는 태그-게시물 테이블이다.

```redis
127.0.0.1:6379> SADD post:47:tags IT REDIS DATASTORE
3

127.0.0.1:6379> SADD post:22:tags IT PYTHON
2
```
- 각 포스트가 사용하는 태그를 레디스의 set을 이용해 저장한 내용을 나타낸다. ID 47인 포스트에서 사용하는 태그는 IT, REDIS, DATASTORE 이렇게 3개인 것이다. 
- **태그 기능을 사용하는 이유 중 하나는 특정 게시물이 어떤 태그와 연관돼 있는지 확인하는 것 뿐만 아니라 특정한 태그를 포함한 게시물들만 확인**하기 위해서일 수 있다. **데이터를 저장할 때 포스트를 기준으로 하는 set 그리고, 태그를 기준으로 하는 set에 각각 데이터를 넣어주면 그 기능을 쉽게 구현**할 수 있다.
```redis
127.0.0.1:6379> SADD post:47:tags IT REDIS DATASTORE
3
127.0.0.1:6379> SADD post:22:tags IT PYTHON
2
127.0.0.1:6379> SADD post:53:tags DATASTORE IT MYSQL
3

...
127.0.0.1:6379> SADD tag:DATASTORE:posts 53
1
127.0.0.1:6379> SADD tag:IT:posts 53
1
127.0.0.1:6379> SADD tag:MYSQL:posts 53
```
- `SMEMBERS`로 특정 태그를 갖고 있는 포스트를 쉽게 확인할 수 있고, `SINTER`로 특정 set의 교집합을 확인할 수 있다.
```redis
127.0.0.1:6379> SMEMBERS tag:IT:posts
22
47
53

127.0.0.1:6379> SINTER tag:IT:posts tag:DATASTORE:posts
47
53
```
- 만약 이 기능을 관계형 데이터베이스에서 구현하라면 태그-포스트 관계형 테이블을 만드는 것이 일반적이다. 아래와 비슷한 쿼리가 만들어지고, 이는 테이블 크기에 따라 DB에 부하를 줄 수 있다.
```sql
SELECT post_id 
FROM tag_post 
WHERE tag_id IN (1,3) 
GROUP BY post_id HAVING COUNT(tag_id) <= 2;
```

## hash, set, sorted set
###  랜덤 데이터 출력 
- 보통 관계형 데이터베이스에는 랜덤 데이터 추출을 사용할 때 `ORDER BY RAND()` 함수를 주로 사용한다. 이 함수는 쿼리의 결과값을 랜덤하게 정렬하지만, **조건 절에 맞는 모든 행을 읽은 뒤, 임시 테이블에 넣어 정렬한 다음 랜덤으로 limit에 해당할 때까지 데이터를 출력한다. 데이터가 1만 건 이상일 경우 부하가 많이 가는 연산**이다.
- 레디스를 `RANDOMKEY` 커맨드를 사용하면 **$O(1)$에 전체 키 중 하나를 무작위로 반환**한다. 하지만 보통 하나의 레디스 인스턴스에 한 가지 종류의 데이터만 저장하지 않기 때문에 랜덤 키 추출은 크게 의미가 없을 수 있다.
- `HRANDFIELD, SRANDMEMBER, ZRANDMEMBER` 커맨드는 각각 hash, set, sorted set에 저장된 아이템 중 랜덤한 아이템을 추출할 수 있다.

```redis
127.0.0.1:6379> HSET user:hash "ID:123" "juny"
(integer) 1
127.0.0.1:6379> HSET user:hash "ID:456" "jiny"
(integer) 1
127.0.0.1:6379> HRANDFIELD user:hash
"ID:456"
127.0.0.1:6379> HRANDFIELD user:hash 1 WITHVALUES
1) "ID:123"
2) "juny"
127.0.0.1:6379> HRANDFIELD user:hash 1 WITHVALUES
1) "ID:456"
2) "jiny"
127.0.0.1:6379> HRANDFIELD user:hash 1 WITHVALUES
1) "ID:123"
2) "juny"
```
- `COUNT` 옵션을 사용하면 원하는 개수만큼 랜덤 아이템이 반환되며, `WITHVALUES` 옵션을 사용하면 필드에 연결된 값도 함께 반환할 수 있다.
- `COUNT` 옵션을 양수로 설정하면 중복되지 않는 랜덤 데이터가 반환되고, 음수로 설정하면 데이터가 중복해서 반환될 수 있다.
```redis
127.0.0.1:6379> HRANDFIELD user:hash -2 WITHVALUES
1) "ID:123"
2) "juny"
3) "ID:123"
4) "juny"

127.0.0.1:6379> HRANDFIELD user:hash 2 WITHVALUES
1) "ID:123"
2) "juny"
3) "ID:456"
4) "jiny"
```

## set

### 좋아요 처리하기
- 트래픽이 굉장히 많은 사이트라면 하나의 뉴스 댓글에 좋아요가 눌리는 일은 1초에 몇만 개 이상 발생할 수 있으며, **좋아요를 누를 때마다 관계형 데이터베이스 테이블의 특정행에서 좋아요 개수 데이터를 증가시키는 일은 성능에 약영향** 줄 수 있다.
- 댓글 id를 기준으로 set을 생성한 뒤 좋아요를 누른 유저의 id를 set에 저장하면 중복 없이 데이터를 저장할 수 있다.
```redis
127.0.0.1:6379> SADD comment-like:12554 1
(integer) 1
127.0.0.1:6379> SADD comment-like:12554 17
(integer) 1
127.0.0.1:6379> SADD comment-like:12554 1722
(integer) 1

127.0.0.1:6379> SCARD comment-like:12554
(integer) 3
```
- 댓글 id 12554에 좋아요를 누른 유저는 1, 17, 1722로 총 3명이다.

## hash
### 읽지 않은 메시지 수 카운팅하기
- 채팅 메시지가 도착할 때마다 바로 관계형 데이터베이스를 업데이트하는 대신 데이터를 레디스와 같은 인메모리 데이터베이스에 일시적으로 저장한 뒤 필요한 시점에 한꺼번에 업데이트하는 방식을 사용하면 성능을 향상시킬 수 있다.
- 앞서 좋아요 예제와 다르게 채팅의 내용을 확인하거나 중복된 데이터를 고려할 필요 없이 단순히 채널에 새로 추가된 메시지의 개수를 확인하면 된다. 
- **사용자의 ID를 키로 사용하고, 채널의 ID를 아이템의 키로 활용해 숫자 형태의 메시지 카운트를 관리하는 방법**을 고려할 수 있다.
```redis
127.0.0.1:6379> HINCRBY user:234 channel:4234 1
(integer) 1
127.0.0.1:6379> HGET user:234 channel:4234
"1"

127.0.0.1:6379> HINCRBY user:234 channel:4234 50
(integer) 51
127.0.0.1:6379> HGET user:234 channel:4234
"51"

127.0.0.1:6379> HINCRBY user:234 channel:4234 -5
(integer) 46
127.0.0.1:6379> HGET user:234 channel:4234
"46"
```

## bitmap
### DAU(Daily Active User) 구하기
- DAU는 하루 동안 서비스에 방문한 사용자 수를 의미한다. 하루에 여러번 방문했다 하더라도 한 번으로 카운팅된다.
- 만약 하루 1000만 명 이상의 유저가 방문하는 큰 서비스일 때 Set에 저장한다면 하나의 키 안에 너무 많은 아이템이 저장될 수 있으며 이는 곧 성능 저하로 이어진다. (보통 키 하나당 아이템은 최대 200~300만 개까지로 조정할 것을 권장한다)
- **레디스의 비트맵을 이용하면 메모리를 효율적으로 줄이면서도 실시간으로 서비스의 DAU를 확인**할 수 있다. 레디스에서 비트맵은 별개의 자료 구조로 존재하는 것은 아니고, **string 자료 구조에 bit 연산**을 할 수 있도록 지원한다.
- 사용자 ID는 string 자료 구조에서 하나의 비트로 표현될 수 있다. **1천만명의 사용자는 1천만 개의 비트로 나타낼 수 있으며, 이는 대략 1.2MB 크기에 해당한다. 레디스에서 string의 최대 길이는 512MB**이기에 하나의 키를 사용해 1천만 명의 사용자를 카운팅하는 것은 문제없이 가능하다.
```redis
127.0.0.1:6379> SETBIT uv:20240717 14 1
(integer) 0
127.0.0.1:6379> BITCOUNT uv:20240717
(integer) 1

127.0.0.1:6379> SETBIT uv:20240717 142 1
(integer) 0
127.0.0.1:6379> BITCOUNT uv:20240717
(integer) 2

127.0.0.1:6379> SETBIT uv:20240717 22502 1
(integer) 0
127.0.0.1:6379> BITCOUNT uv:20240717
(integer) 3
```
- 2024년 7월 17일 14, 142, 22502 아이디를 가진 유저가 방문했고 BITCOUNT로 방문한 총 유저는 3명인 것을 확인할 수 있다.
- 비트맵에서 `BITOP` 커맨드를 사용하면 **AND, OR, XOR, NOT 연산**을 할 수 있고, 레디스 서버에서 바로 계산된 결과를 가져올 수 있다.

### 3일간 매일 출석한 유저에게 보상 지급하기
```redis
127.0.0.1:6379> SETBIT uv:20240716 14 1
(integer) 0
127.0.0.1:6379> SETBIT uv:20240715 14 1
(integer) 0

127.0.0.1:6379> BITOP AND event:202407 uv:20240715 uv:20240716 uv:20240717
(integer) 2813
127.0.0.1:6379> GETBIT event:202407 14
(integer) 1
127.0.0.1:6379> GETBIT event:202407 22502
(integer) 0
```
- 비트맵의 길이는 가장 긴 입력 비트맵에 의해 결정되며, 바이트 단위로 반환된다. 제일 긴 비트맵은 22502이기 때문에 `22502/8 = 2813`이 나온 걸 확인할 수 있다. 14번 비트를 구하면 14번 유저는 3일간 출석했기 때문에 1로 나온다. 
