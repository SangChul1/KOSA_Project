# PXC 3노드 + Redis Sentinel 구축 가이드

> **대상 환경**
> - 노트북 1대 (정원 PC, VLAN 30)
> - VMware Workstation/Fusion, Bridged 네트워크
> - Ubuntu 22.04 LTS Server
> - Percona XtraDB Cluster 8.0
> - Redis 7.x + Sentinel

---

## 📋 전체 로드맵

```
Phase 1. VM 6대 준비 (공통 베이스)
Phase 2. PXC 3노드 클러스터 구축 (SSL 포함)
Phase 3. Redis Master + Replica 2대 구성
Phase 4. Sentinel 3대 구성
Phase 5. 장애/페일오버 테스트
Phase 6. App Tier에서 접근 검증
```

---

## 🖥️ 서버 정보

### IP 할당표

| Hostname | IP | 역할 | vCPU | RAM | Disk |
|---|---|---|---|---|---|
| pxc-01 | 192.168.75.74 | PXC 노드1 (부트스트랩) | 2 | 2GB | 30GB |
| pxc-02 | 192.168.75.24 | PXC 노드2 | 2 | 2GB | 30GB |
| pxc-03 | 192.168.75.79 | PXC 노드3 | 2 | 2GB | 30GB |
| redis-01 | 192.168.75.81 | Redis Master + Sentinel | 1 | 1GB | 10GB |
| redis-02 | 192.168.75.82 | Redis Replica + Sentinel | 1 | 1GB | 10GB |
| redis-03 | 192.168.75.83 | Redis Replica + Sentinel | 1 | 1GB | 10GB |

> Redis IP는 예시입니다. 실제 환경에 맞게 조정하세요.

### 사용 포트

| 서비스 | 포트 | 용도 |
|---|---|---|
| MySQL | 3306 | 클라이언트 접속 |
| Galera | 4567 | 클러스터 통신 |
| IST | 4568 | Incremental State Transfer |
| SST | 4444 | State Snapshot Transfer |
| Redis | 6379 | Redis 서버 |
| Sentinel | 26379 | Sentinel 통신 |

### 계정 및 비밀번호 (학습용, 운영 시 변경)

| 계정 | 비밀번호 | 용도 |
|---|---|---|
| root | `Root!2026` | MySQL 관리자 |
| sstuser | `SSTPass!2026` | SST 복제용 |
| appuser | `App!2026` | 애플리케이션 |
| Redis | `Redis!2026` | Redis 인증 |

---

# Phase 1. VM 6대 준비

## 1-1. VMware VM 스펙

| 항목 | PXC VM | Redis VM |
|---|---|---|
| Memory | 2 GB | 1 GB |
| Processors | 2 | 1 |
| Hard Disk | 30 GB | 10 GB |
| Network | **Bridged** (NAT 금지) | **Bridged** |
| OS | Ubuntu 22.04 LTS | Ubuntu 22.04 LTS |

> **중요**: Network Adapter는 반드시 **Bridged**로 설정해야 다른 VM과 통신 가능합니다. NAT 모드는 사용 불가.

## 1-2. 베이스 VM 1개 생성 후 클론 (권장)

VM 하나를 완성한 뒤 VMware의 **Clone (Full Clone)** 기능으로 5개 복제하면 시간을 크게 절약할 수 있습니다.

```
pxc-01 (template) → Full Clone → pxc-02, pxc-03, redis-01, redis-02, redis-03
```

## 1-3. 공통 작업 (6대 모두)

### hostname 설정

```bash
# 각 VM에서 자기 hostname
sudo hostnamectl set-hostname pxc-01    # VM별로 변경
```

### /etc/hosts 등록

```bash
sudo tee -a /etc/hosts << 'EOF'
192.168.75.74  pxc-01
192.168.75.24  pxc-02
192.168.75.79  pxc-03
192.168.75.81  redis-01
192.168.75.82  redis-02
192.168.75.83  redis-03
EOF
```

### 시스템 업데이트 및 필수 도구

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget vim net-tools htop gnupg2 lsb-release
```

### 시간 동기화 (필수)

```bash
sudo apt install -y chrony
sudo systemctl enable --now chrony
chronyc sources
```

### 방화벽 설정

```bash
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing

# SSH
sudo ufw allow 22/tcp

# DB Tier 내부 통신 (VLAN 30 전체 허용)
sudo ufw allow from 192.168.75.0/24

sudo ufw --force enable
sudo ufw status
```

### SSH 키 등록 (선택, 편의용)

```bash
# 관리 PC에서
ssh-copy-id kosa@192.168.75.74
```

---

# Phase 2. PXC 3노드 클러스터 구축

## 2-1. Percona 설치 (3노드 모두)

```bash
# 저장소 등록
wget https://repo.percona.com/apt/percona-release_latest.$(lsb_release -sc)_all.deb
sudo dpkg -i percona-release_latest.$(lsb_release -sc)_all.deb
sudo percona-release setup pxc-80

# 설치 (root 비밀번호 입력창이 뜸 - 3노드 모두 동일하게 Root!2026)
sudo apt update
sudo apt install -y percona-xtradb-cluster
```

## 2-2. 자동 실행된 MySQL 중지 (3노드 모두)

```bash
sudo systemctl stop mysql
sudo systemctl disable mysql
```

## 2-3. SSL 인증서 생성 (pxc-01에서만)

PXC 8.0은 기본으로 노드 간 통신에 SSL을 요구합니다. 정공법으로 SSL 키를 만듭니다.

```bash
# 인증서 디렉토리 생성
sudo mkdir -p /etc/mysql/certs

# 인증서 자동 생성
sudo mysql_ssl_rsa_setup --datadir=/etc/mysql/certs --uid=mysql

# 권한 설정
sudo chown -R mysql:mysql /etc/mysql/certs
sudo chmod 600 /etc/mysql/certs/*-key.pem
sudo chmod 644 /etc/mysql/certs/*-cert.pem /etc/mysql/certs/ca.pem

# 확인
sudo ls -la /etc/mysql/certs/
```

생성 결과:
```
ca.pem              ca-key.pem
server-cert.pem     server-key.pem
client-cert.pem     client-key.pem
```

## 2-4. 인증서 pxc-02, pxc-03에 복사

**중요**: 3노드 모두 **같은 인증서**를 사용해야 합니다.

### pxc-01에서 전송

```bash
sudo cp /etc/mysql/certs/*.pem /tmp/
sudo chmod 644 /tmp/*.pem

scp /tmp/*.pem kosa@192.168.75.24:/tmp/
scp /tmp/*.pem kosa@192.168.75.79:/tmp/
```

### pxc-02, pxc-03 각각에서 배치

```bash
sudo mkdir -p /etc/mysql/certs
sudo mv /tmp/*.pem /etc/mysql/certs/
sudo chown -R mysql:mysql /etc/mysql/certs
sudo chmod 600 /etc/mysql/certs/*-key.pem
sudo chmod 644 /etc/mysql/certs/*-cert.pem /etc/mysql/certs/ca.pem
```

## 2-5. my.cnf 설정 (3노드 각각)

### 설정 파일 백업

```bash
sudo cp /etc/mysql/mysql.conf.d/mysqld.cnf /etc/mysql/mysql.conf.d/mysqld.cnf.bak
```

### pxc-01 설정

파일 전체를 아래 내용으로 **덮어쓰세요**:

```bash
sudo tee /etc/mysql/mysql.conf.d/mysqld.cnf > /dev/null << 'EOF'
[client]
socket = /var/run/mysqld/mysqld.sock

[mysqld]
# 일반 설정
pid-file  = /var/run/mysqld/mysqld.pid
socket    = /var/run/mysqld/mysqld.sock
datadir   = /var/lib/mysql
log-error = /var/log/mysql/error.log
bind-address = 0.0.0.0

# InnoDB
default_storage_engine         = InnoDB
innodb_buffer_pool_size        = 256M
innodb_flush_log_at_trx_commit = 2
innodb_autoinc_lock_mode       = 2

# 바이너리 로그
binlog_format              = ROW
binlog_expire_logs_seconds = 604800

######## wsrep / Galera ########
wsrep_provider         = /usr/lib/galera4/libgalera_smm.so
wsrep_provider_options = "socket.ssl_key=/etc/mysql/certs/server-key.pem;socket.ssl_cert=/etc/mysql/certs/server-cert.pem;socket.ssl_ca=/etc/mysql/certs/ca.pem"
wsrep_cluster_name     = pxc-cluster
wsrep_cluster_address  = gcomm://192.168.75.74,192.168.75.24,192.168.75.79
wsrep_node_name        = pxc-01
wsrep_node_address     = 192.168.75.74
wsrep_slave_threads    = 4
wsrep_log_conflicts
wsrep_sst_method       = xtrabackup-v2
wsrep_sst_auth         = "sstuser:SSTPass!2026"
pxc_strict_mode        = ENFORCING

######## SST용 SSL ########
[sst]
encrypt  = 4
ssl-ca   = /etc/mysql/certs/ca.pem
ssl-cert = /etc/mysql/certs/server-cert.pem
ssl-key  = /etc/mysql/certs/server-key.pem
EOF
```

### pxc-02 설정

pxc-01과 동일하되 **두 줄만 변경**:

```ini
wsrep_node_name    = pxc-02
wsrep_node_address = 192.168.75.24
```

### pxc-03 설정

역시 **두 줄만 변경**:

```ini
wsrep_node_name    = pxc-03
wsrep_node_address = 192.168.75.79
```

## 2-6. 설정 검증 (각 노드에서)

```bash
sudo mysqld --verbose --help --user=mysql 2>&1 | grep -iE "\[ERROR\]" | head -5
```

ERROR 라인이 **안 나오면** OK입니다.

## 2-7. pxc-01 부트스트랩

### 좀비 프로세스 정리 (중요)

```bash
sudo systemctl stop mysql 2>/dev/null
sudo systemctl stop mysql@bootstrap.service 2>/dev/null
sudo pkill -9 mysqld 2>/dev/null
sudo pkill -9 mysqld_safe 2>/dev/null
sleep 5

# 포트 확인 (비어있어야 정상)
sudo ss -tlnp | grep -E "3306|4567|4568|4444"

# 소켓 파일 정리
sudo rm -f /var/run/mysqld/mysqld.sock
sudo rm -f /var/run/mysqld/mysqld.pid
```

### grastate.dat 확인

```bash
sudo cat /var/lib/mysql/grastate.dat 2>/dev/null
```

파일이 있고 `safe_to_bootstrap: 0`이면 `1`로 변경:

```bash
sudo sed -i 's/safe_to_bootstrap: 0/safe_to_bootstrap: 1/' /var/lib/mysql/grastate.dat
```

### 부트스트랩 실행

```bash
sudo systemctl start mysql@bootstrap.service
sudo systemctl status mysql@bootstrap.service --no-pager
```

**`active (running)`** 이 나오면 성공.

### 클러스터 상태 확인

```bash
sudo mysql -uroot -p -e "SHOW STATUS LIKE 'wsrep_cluster%';"
```

기대 결과:
```
wsrep_cluster_size       1
wsrep_cluster_status     Primary
wsrep_ready              ON
```

## 2-8. SST 계정 생성 (pxc-01에서만)

```bash
sudo mysql -uroot -p
```

```sql
CREATE USER 'sstuser'@'localhost' IDENTIFIED BY 'SSTPass!2026';
GRANT BACKUP_ADMIN, RELOAD, LOCK TABLES, PROCESS, REPLICATION CLIENT ON *.* TO 'sstuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

## 2-9. pxc-02, pxc-03 순차 합류

### pxc-02에서

```bash
sudo systemctl stop mysql 2>/dev/null
sudo pkill -9 mysqld 2>/dev/null
sleep 3

sudo systemctl start mysql
sudo systemctl status mysql --no-pager
```

### pxc-01에서 합류 확인

```bash
sudo mysql -uroot -p -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
# 결과: 2
```

### pxc-03도 동일하게

```bash
sudo systemctl stop mysql 2>/dev/null
sudo pkill -9 mysqld 2>/dev/null
sleep 3

sudo systemctl start mysql
```

### 최종 확인

```bash
sudo mysql -uroot -p -e "SHOW STATUS LIKE 'wsrep_cluster%';"
```

기대 결과:
```
wsrep_cluster_size       3
wsrep_cluster_status     Primary
wsrep_connected          ON
wsrep_ready              ON
```

## 2-10. 복제 동작 테스트

### pxc-01에서 DB와 데이터 생성

```sql
CREATE DATABASE appdb;
CREATE USER 'appuser'@'%' IDENTIFIED BY 'App!2026';
GRANT ALL PRIVILEGES ON appdb.* TO 'appuser'@'%';
FLUSH PRIVILEGES;

USE appdb;
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
INSERT INTO users (name) VALUES ('alice'), ('bob'), ('charlie');
```

### pxc-02, pxc-03에서 즉시 읽기 검증

```bash
sudo mysql -uroot -p -e "SELECT * FROM appdb.users;"
```

3명(alice, bob, charlie)이 보이면 **동기 복제 정상 동작**.

### 양방향 쓰기 테스트

pxc-02에서 쓰기:
```sql
INSERT INTO appdb.users (name) VALUES ('david');
```

pxc-01, pxc-03에서 즉시 조회 → david 포함 확인.

---

# Phase 3. Redis Master + Replica 구성

## 3-1. Redis 설치 (3노드 모두)

```bash
sudo apt update
sudo apt install -y redis-server

# 방화벽
sudo ufw allow from 192.168.75.0/24 to any port 6379 proto tcp
sudo ufw allow from 192.168.75.0/24 to any port 26379 proto tcp

# 자동 시작 중지 (수동 제어)
sudo systemctl stop redis-server
```

## 3-2. redis-01 (Master) 설정

```bash
sudo nano /etc/redis/redis.conf
```

다음 항목 수정:

```conf
bind 0.0.0.0
protected-mode yes
port 6379
requirepass Redis!2026
masterauth Redis!2026

maxmemory 512mb
maxmemory-policy allkeys-lru

# 영속성
appendonly yes
appendfsync everysec
save 900 1
save 300 10
save 60 10000

# 로그
logfile /var/log/redis/redis-server.log
dir /var/lib/redis

# 보안
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command CONFIG "CONFIG_5f8e1a"
```

시작:

```bash
sudo systemctl start redis-server
sudo systemctl enable redis-server
redis-cli -a 'Redis!2026' ping
# → PONG
```

## 3-3. redis-02, redis-03 (Replica) 설정

Master와 동일하게 설정 후 **맨 아래에 다음 추가**:

```conf
replicaof 192.168.75.81 6379
replica-read-only yes
```

시작:

```bash
sudo systemctl start redis-server
sudo systemctl enable redis-server
```

## 3-4. 복제 상태 확인

### Master(redis-01)에서

```bash
redis-cli -a 'Redis!2026' INFO replication
```

기대 출력:
```
role:master
connected_slaves:2
slave0:ip=192.168.75.82,port=6379,state=online,...
slave1:ip=192.168.75.83,port=6379,state=online,...
```

### Replica(redis-02)에서

```bash
redis-cli -a 'Redis!2026' INFO replication
```

기대:
```
role:slave
master_host:192.168.75.81
master_link_status:up
```

## 3-5. 복제 동작 테스트

```bash
# Master에서 쓰기
redis-cli -a 'Redis!2026' SET test:key "hello"

# Replica에서 읽기
redis-cli -a 'Redis!2026' GET test:key
# → "hello"

# Replica에서 쓰기 시도 (실패해야 정상)
redis-cli -a 'Redis!2026' SET fail "should-fail"
# → (error) READONLY You can't write against a read only replica.
```

---

# Phase 4. Sentinel 3대 구성

## 4-1. Sentinel 설정 (3노드 모두 동일)

```bash
sudo cp /etc/redis/sentinel.conf /etc/redis/sentinel.conf.bak

sudo tee /etc/redis/sentinel.conf > /dev/null << 'EOF'
port 26379
bind 0.0.0.0
daemonize no
pidfile /var/run/redis/redis-sentinel.pid
logfile /var/log/redis/redis-sentinel.log
dir /var/lib/redis

sentinel monitor mymaster 192.168.75.81 6379 2
sentinel auth-pass mymaster Redis!2026
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 10000
sentinel parallel-syncs mymaster 1

sentinel deny-scripts-reconfig yes
EOF

sudo chown redis:redis /etc/redis/sentinel.conf
```

## 4-2. Sentinel 서비스 시작 (3노드 모두)

```bash
sudo systemctl start redis-sentinel
sudo systemctl enable redis-sentinel
sudo systemctl status redis-sentinel --no-pager
```

## 4-3. Sentinel 상태 확인

```bash
redis-cli -p 26379 SENTINEL masters
```

기대 출력의 핵심:
```
"num-other-sentinels" → "2"   ← 나머지 2대 인식 확인
"quorum"              → "2"
"num-slaves"          → "2"
```

```bash
# 복제본 확인
redis-cli -p 26379 SENTINEL replicas mymaster

# 다른 Sentinel 확인
redis-cli -p 26379 SENTINEL sentinels mymaster
```

---

# Phase 5. 장애/페일오버 테스트

## 5-1. PXC 장애 테스트

### 한 노드 정지 → 나머지 2노드 서비스 유지

```bash
# pxc-02 정지
sudo systemctl stop mysql

# pxc-01에서 쓰기 시도 (정상 동작해야 함)
sudo mysql -uroot -p -e "INSERT INTO appdb.users (name) VALUES ('ghost');"

# 클러스터 사이즈 확인
sudo mysql -uroot -p -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
# → 2

# pxc-02 재시작 → 자동 합류
sudo systemctl start mysql

# 재합류 및 데이터 동기화 확인
sudo mysql -uroot -p -e "SELECT * FROM appdb.users;"
```

### 정족수 깨짐 테스트 (2노드 동시 정지)

```bash
# pxc-02, pxc-03 정지
# pxc-01은 split-brain 방지로 읽기/쓰기 차단됨
sudo mysql -uroot -p -e "INSERT INTO appdb.users (name) VALUES ('x');"
# → ERROR 1047: WSREP has not yet prepared node for application use

# 복구: 두 노드 재시작
sudo systemctl start mysql   # pxc-02
sudo systemctl start mysql   # pxc-03
```

## 5-2. Redis Sentinel 페일오버 테스트

### Master 강제 종료

```bash
# redis-01(Master) VM에서
sudo systemctl stop redis-server

# 다른 VM에서 Sentinel 로그 실시간 관찰
sudo tail -f /var/log/redis/redis-sentinel.log
```

### 기대 로그 흐름

```
+sdown master mymaster ...          # 주관적 다운 감지
+odown master mymaster ...          # 객관적 다운 확정
+try-failover master mymaster ...
+elected-leader master mymaster ... # 리더 Sentinel 선출
+promoted-slave slave 192.168.75.82 # 승격
+switch-master mymaster 192.168.75.81 6379 192.168.75.82 6379
```

### 새 Master 확인

```bash
redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster
# → "192.168.75.82"

redis-cli -h 192.168.75.82 -a 'Redis!2026' INFO replication
# role:master
# connected_slaves:1
```

### 원 Master 복구

```bash
# redis-01 재시작
sudo systemctl start redis-server

# 자동으로 Replica로 재편입됨
redis-cli -h 192.168.75.81 -a 'Redis!2026' INFO replication
# role:slave
# master_host:192.168.75.82
```

## 5-3. 페일오버 시간 측정

```bash
# 클라이언트에서 초당 SET 반복
while true; do
  redis-cli -h 192.168.75.82 -a 'Redis!2026' SET ping $(date +%s.%N)
  sleep 0.5
done

# 다른 터미널에서 Master kill → 복구까지 시간 측정
```

`down-after-milliseconds`(5초) + `failover-timeout`(10초) 기준 약 **10~15초 내 복구**.

---

# Phase 6. App Tier에서 접근 검증

## 6-1. 네트워크 연결 확인

App Tier(혜윤 PC, VLAN 20) VM에서:

```bash
ping -c 3 192.168.75.74    # pxc-01
ping -c 3 192.168.75.81    # redis-01

nc -zv 192.168.75.74 3306     # PXC
nc -zv 192.168.75.81 6379     # Redis
nc -zv 192.168.75.81 26379    # Sentinel
```

## 6-2. Python 종단 간 테스트

```bash
sudo apt install -y python3-pip
pip3 install redis pymysql
```

`test_db_redis.py`:

```python
import redis
from redis.sentinel import Sentinel
import pymysql

# === Redis Sentinel ===
sentinel = Sentinel([
    ('192.168.75.81', 26379),
    ('192.168.75.82', 26379),
    ('192.168.75.83', 26379),
], socket_timeout=0.5, password='Redis!2026')

master = sentinel.master_for('mymaster', password='Redis!2026')
replica = sentinel.slave_for('mymaster', password='Redis!2026')

master.set('hello', 'world from app tier')
print('Redis 읽기:', replica.get('hello'))

# === PXC ===
conn = pymysql.connect(
    host='192.168.75.74',
    user='appuser',
    password='App!2026',
    database='appdb',
)
with conn.cursor() as cur:
    cur.execute("SELECT id, name FROM users LIMIT 5")
    print('PXC 읽기:', cur.fetchall())
conn.close()
```

```bash
python3 test_db_redis.py
```

---

# 🔧 트러블슈팅

## 문제 1: SSL 에러 "ssl-ca, ssl-cert, and ssl-key must all be defined"

**원인**: PXC 8.0은 기본으로 SSL 강제.

**해결**:
- Phase 2-3 SSL 키 생성을 수행했는지 확인
- `wsrep_provider_options`에 SSL 경로가 올바르게 설정되어 있는지 확인
- 3노드 모두 **같은 인증서**를 사용하는지 확인

## 문제 2: 포트 점유 "Address already in use"

**원인**: 이전 mysqld 프로세스가 좀비로 남아있음.

**해결**:
```bash
sudo pkill -9 mysqld
sudo pkill -9 mysqld_safe
sleep 5
sudo ss -tlnp | grep -E "3306|4567"   # 비어있어야 함
```

그래도 안 되면 리부팅.

## 문제 3: "safe_to_bootstrap: 0"

**원인**: 이전 실패 흔적.

**해결**:
```bash
sudo sed -i 's/safe_to_bootstrap: 0/safe_to_bootstrap: 1/' /var/lib/mysql/grastate.dat
```

## 문제 4: pxc-02/03 합류 실패 (SST 에러)

**원인**:
- sstuser 계정이 pxc-01에 없음
- `wsrep_sst_auth` 비밀번호와 실제 sstuser 비밀번호 불일치
- 4444/4568 포트 방화벽 차단

**해결**:
- pxc-01에서 sstuser 재생성 (Phase 2-8)
- 방화벽 규칙 확인

## 문제 5: OOM (메모리 부족)

**원인**: `innodb_buffer_pool_size`가 VM 메모리 대비 너무 큼.

**해결**: VM RAM 2GB일 때는 `innodb_buffer_pool_size = 128M` 이하 권장.

## 문제 6: Sentinel이 서로 인식 못함 (num-other-sentinels=0)

**원인**:
- 26379 포트 방화벽 차단
- `sentinel monitor`의 Master IP가 노드마다 다름
- `bind` 설정이 127.0.0.1

**해결**:
- `bind 0.0.0.0` 확인
- 3노드 모두 같은 `sentinel.conf` 사용

## 문제 7: 데이터 디렉토리 완전 초기화 (최후의 수단)

```bash
sudo systemctl stop mysql@bootstrap.service
sudo pkill -9 mysqld
sudo rm -rf /var/lib/mysql/*
sudo mysqld --initialize --user=mysql --datadir=/var/lib/mysql
sudo grep 'temporary password' /var/log/mysql/error.log
sudo systemctl start mysql@bootstrap.service
```

---

# ✅ 체크리스트

진행 상황 추적용:

- [ ] VM 6대 생성 및 Bridged 네트워크 설정
- [ ] hostname/IP/hosts 파일 설정
- [ ] 모든 VM 시간 동기화 (chrony)
- [ ] 방화벽 규칙 적용
- [ ] Percona 3노드 설치
- [ ] SSL 인증서 생성 (pxc-01)
- [ ] SSL 인증서 pxc-02, pxc-03 배포
- [ ] my.cnf 3노드 설정 (node_name, node_address만 다름)
- [ ] pxc-01 부트스트랩 성공
- [ ] sstuser 계정 생성
- [ ] pxc-02 합류 (cluster_size=2)
- [ ] pxc-03 합류 (cluster_size=3)
- [ ] 데이터 동기 복제 검증
- [ ] Redis 3노드 설치
- [ ] Master-Replica 구성 및 복제 확인
- [ ] Sentinel 3대 기동
- [ ] num-other-sentinels=2 확인
- [ ] PXC 장애 테스트 (노드 정지/복구)
- [ ] Redis Sentinel 페일오버 테스트
- [ ] App Tier에서 DB/Redis/Sentinel 접근 확인

---

# 📚 참고 자료

- [Percona XtraDB Cluster 8.0 공식 문서](https://docs.percona.com/percona-xtradb-cluster/8.0/)
- [Redis Sentinel 공식 문서](https://redis.io/docs/management/sentinel/)
- [Galera Cluster 문서](https://galeracluster.com/library/documentation/)

---

# 🎯 다음 단계

Phase 6까지 완료되면:

1. **HAProxy/ProxySQL 추가**: 앱이 PXC 3노드에 로드밸런싱 접근
2. **모니터링 연동**: Prometheus + Grafana + exporter 연결
3. **백업 자동화**: mariabackup cron, Redis RDB 백업
4. **장애 시나리오 고도화**: 네트워크 분할, 디스크 풀 테스트

---

**작성일**: 2026-04-19
**환경**: Ubuntu 22.04, Percona XtraDB Cluster 8.0, Redis 7.x
