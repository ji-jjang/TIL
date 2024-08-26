# 설정 (HomeBrew)
## 시작과 종료 
```bash
❯ brew services list
Name    Status  User      File
mysql   started jijunhyuk ~/Library/LaunchAgents/homebrew.mxcl.mysql.plist
unbound none
vault   none

❯ brew services stop mysql
Stopping `mysql`... (might take a while)
==> Successfully stopped `mysql` (label: homebrew.mxcl.mysql)

❯ brew services start mysql
==> Successfully started `mysql` (label: homebrew.mxcl.mysql)


❯ brew services restart mysql
Stopping `mysql`... (might take a while)
==> Successfully stopped `mysql` (label: homebrew.mxcl.mysql)
==> Successfully started `mysql` (label: homebrew.mxcl.mysql)
```

- MySQL 서버가 시작되거나 종료될 때는 버퍼 풀 내용을 백업하고, 복구하는 과정 존재. 실제 버퍼 풀의 내용을 백업하는 게 아니라 버퍼 풀에 적재돼 있던 데이터 파일의 데이터 페이지에 대한 **메타 정보를 백업** 하기에 용량이 크지 않음. 하지만, **MySQL 서버가 새로 실행될 때는 디스크에서 데이터 파일들을 모두 읽어서 적재하므로 상당한 시간이 걸림**

## 연결
```bash
# 소켓으로 연결
❯ mysql -h 127.0.0.1 -u root -p --socket /tmp/mysql.sock
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.

# TCP/IP로 연결
❯ mysql -h 127.0.0.1 -u root -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.

# 기본값으로 호스트는 localhost, 소켓으로 연결
❯ mysql -u root -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
```

### 서버 설정
```bash
# my.cnf 위치 
❯ mysql --help
...
Default options are read from the following files in the given order:
/etc/my.cnf /etc/mysql/my.cnf /opt/homebrew/etc/my.cnf ~/.my.cnf
...
```
- MySQL 서버는 단 하나의 설정 파일(my.cnf)를 사용하지만, 설정 파일이 위치한 디렉터리는 여러 곳일 수 있다. 위 명령어를 통해 **어떤 디렉토리에 있는 설정 파일이 우선 적용했는지 파악 가능**

#### 설정 파일 구성
```ini
[mysqld_safe]
malloc-lib = /opt/lib/libtcmalloc_minimal.so

[mysqld]
socket = /user/local/mysql/tmp/mysql.sock
port = 3306

[mysql]
default-character-set = utf8mb4
socket = /usr/local/mysql/tmp/mysql.sock
port = 3306

[mysqldump]
default-character-set = utf8mb4
socket = /user/local/mysql/tmp/mysql.sock
port = 3305
```

- 설정 파일이 MySQL 서버만을 위한 설정 파일이라면 `[mysqld]` 그룹만 명시해도 충분. 그러나 MySQL 클라이언트나 백업을 위한 mysqldump 프로그램이 실행될 때도 이 설정 파일을 공용으로 사용하고 싶다면 `[mysql]` 또는 `[msqldump]` 등의 그룹 함께 설정 가능
- **설정 파일의 각 그룹은 같은 파일을 공유하지만 서로 무관하게 적용**

#### MySQL 시스템 변수