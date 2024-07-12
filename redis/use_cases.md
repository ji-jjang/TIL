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