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

### 3. 데이터 업데이트
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

### 4. 랭킹 합산
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

### 5. 최근 검색 기록
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

### 6. 태그 기능
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