# KOSA 인프라 — pfSense HA 구축 완전 가이드

> kosa1, kosa2 초기 세팅부터 pfSense 이중화(HA)까지 한 번에

---

## 목차

1. [전체 아키텍처](#1-전체-아키텍처)
2. [사전 준비](#2-사전-준비)
3. [Phase 1: 관리형 스위치 트렁크 설정](#phase-1-관리형-스위치-트렁크-설정)
4. [Phase 2: kosa1 Proxmox 네트워크 설정](#phase-2-kosa1-proxmox-네트워크-설정)
5. [Phase 3: kosa1 pfSense VM 생성 및 설치](#phase-3-kosa1-pfsense-vm-생성-및-설치)
6. [Phase 4: pfSense 기본 설정 (Setup Wizard)](#phase-4-pfsense-기본-설정-setup-wizard)
7. [Phase 5: VLAN 인터페이스 생성](#phase-5-vlan-인터페이스-생성)
8. [Phase 6: DHCP 서버 설정](#phase-6-dhcp-서버-설정)
9. [Phase 7: 방화벽 규칙 (Zone 분리)](#phase-7-방화벽-규칙-zone-분리)
10. [Phase 8: 영구 적용 (Hook Script)](#phase-8-영구-적용-hook-script)
11. [Phase 9: kosa2 환경 구성](#phase-9-kosa2-환경-구성)
12. [Phase 10: HA 이중화 (CARP + pfsync + XMLRPC)](#phase-10-ha-이중화-carp--pfsync--xmlrpc)
13. [Phase 11: 검증 및 페일오버 테스트](#phase-11-검증-및-페일오버-테스트)
14. [부록: 트러블슈팅](#부록-트러블슈팅)

---

## 1. 전체 아키텍처

### 네트워크 토폴로지

```
[인터넷]
   ↓
[라우터 192.168.21.1] (192.168.21.0/24)
   │
   ├── kosa4 (192.168.21.5, 직결)
   │
   └── [관리형 스위치] (Trunk 분배)
        │
        ├── Port 1: 라우터 (Trunk)
        ├── Port 2: kosa1 eno1 (Trunk)
        ├── Port 3: kosa2 eno1 (Trunk)
        ├── Port 4: kosa3 eno1 (Trunk)
        └── Port 5: 비관리형 스위치 (Access VLAN 40)
                ↓
              [관리자 노트북들]
```

### VLAN 설계 (보안 영역 분리)

| VLAN | 용도 | 네트워크 | pfSense GW (kosa1) | pfSense GW (kosa2) | CARP VIP |
|---|---|---|---|---|---|
| 10 | Public (퍼블릭 서버) | 172.16.21.0/24 | 172.16.21.2 | 172.16.21.3 | **172.16.21.1** |
| 20 | DMZ | 172.16.22.0/24 | 172.16.22.2 | 172.16.22.3 | **172.16.22.1** |
| 30 | Internal (내부망) | 172.16.23.0/24 | 172.16.23.2 | 172.16.23.3 | **172.16.23.1** |
| 40 | Mgmt (관리망) | 172.16.24.0/24 | 172.16.24.2 | 172.16.24.3 | **172.16.24.1** |

> VM/노트북의 게이트웨이는 **CARP VIP (.1)** 사용. Master/Backup 페일오버 시에도 .1은 살아있음.

### 노드별 IP 매핑

| 노드 | vmbr0 (1G 관리/WAN) | vmbr1 (10G Ceph) |
|---|---|---|
| kosa1 | 192.168.21.2/24 | 10.10.10.31/24 |
| kosa2 | 192.168.21.3/24 | 10.10.10.32/24 |
| kosa3 | 192.168.21.4/24 | 10.10.10.33/24 |
| kosa4 | 192.168.21.5/24 (라우터 직결) | — |

---

## 2. 사전 준비

### 필요한 것

- Proxmox VE 4대 설치 완료 (kosa1~4)
- 관리형 스위치 (Trunk 지원)
- TP-Link Omada 라우터 (또는 동급)
- pfSense ISO (https://www.pfsense.org/download/)
- 노트북 (라우터 직결, 192.168.21.x DHCP 수신)

### 작업 전 백업

각 Proxmox 노드에서:

```bash
ssh root@192.168.21.2   # kosa1
cp /etc/network/interfaces /etc/network/interfaces.bak.$(date +%s)
```

kosa2, kosa3도 동일.

---

## Phase 1: 관리형 스위치 트렁크 설정

### 목표

각 Proxmox 노드 연결 포트를 Trunk로 변경, VLAN 10/20/30/40 트렁크 통과.

### Step 1-1. 802.1Q VLAN 활성화

스위치 웹UI 접속 → **VLAN Management → 802.1Q VLAN**:

```
802.1Q VLAN enabled: ● Enable → Apply
```

### Step 1-2. Port Type 설정

| Port | 연결 | Type |
|---|---|---|
| 1 | 라우터 | **Trunk** |
| 2 | kosa1 | **Trunk** |
| 3 | kosa2 | **Trunk** |
| 4 | kosa3 | **Trunk** |
| 5 | 비관리형 스위치 (관리망) | **Access** |

→ Apply.

### Step 1-3. VLAN 추가

| VLAN ID | Name | Untagged Ports | Tagged Ports |
|---|---|---|---|
| 1 | LAN | 1, 2, 3, 4 | — |
| 10 | Public | — | 1, 2, 3, 4 |
| 20 | DMZ | — | 1, 2, 3, 4 |
| 30 | Internal | — | 1, 2, 3, 4 |
| 40 | Mgmt | 5 | 1, 2, 3, 4 |

→ 각 VLAN 추가 후 Apply.

### 검증

노트북에서 인터넷 정상 동작 확인:

```bash
ping 192.168.21.1   # 라우터
ping 8.8.8.8         # 인터넷
```

---

## Phase 2: kosa1 Proxmox 네트워크 설정

### 목표

vmbr0를 VLAN-aware로 변경하여 트렁크 트래픽 처리.

### Step 2-1. SSH 접속

```bash
ssh root@192.168.21.2
```

### Step 2-2. interfaces 파일 작성

```bash
nano /etc/network/interfaces
```

**전체 내용**:

```
auto lo
iface lo inet loopback

auto eno1
iface eno1 inet manual

auto enp1s0f0
iface enp1s0f0 inet manual
    mtu 9000

iface enp1s0f1 inet manual

##############################
# 1G — VLAN-aware (WAN + LAN 통합)
##############################
auto vmbr0
iface vmbr0 inet static
    address 192.168.21.2/24
    gateway 192.168.21.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 1-4094
    up bridge vlan add vid 2-4094 dev vmbr0 self

##############################
# 10G — Ceph 직결망
##############################
auto vmbr1
iface vmbr1 inet static
    address 10.10.10.31/24
    bridge-ports enp1s0f0
    bridge-stp off
    bridge-fd 0
    mtu 9000

source /etc/network/interfaces.d/*
```

### Step 2-3. 적용 및 검증

```bash
ifreload -a

# 검증
ip -br addr | grep vmbr
ip -d link show vmbr0 | grep vlan_filtering   # → vlan_filtering 1
ping -c 3 192.168.21.1
ping -c 3 8.8.8.8
```

---

## Phase 3: kosa1 pfSense VM 생성 및 설치

### Step 3-1. pfSense ISO 업로드

Proxmox 웹UI → kosa1 → local → ISO Images → Upload (pfSense ISO).

### Step 3-2. VM 생성

**Create VM** 클릭:

#### General
| 항목 | 값 |
|---|---|
| VM ID | **100** |
| Name | pfsense-01 |
| Start at boot | ☑ |

#### OS
| 항목 | 값 |
|---|---|
| ISO image | pfSense ISO |
| Type | Other |

#### System
| 항목 | 값 |
|---|---|
| BIOS | OVMF (UEFI) |
| Machine | q35 |
| Pre-Enroll keys | ☐ |

#### Disks
| 항목 | 값 |
|---|---|
| Disk size | 32 GB |
| Storage | local-lvm |

#### CPU
| 항목 | 값 |
|---|---|
| Cores | 4 |
| Type | host |

#### Memory
| 항목 | 값 |
|---|---|
| Memory | 4096 MB |
| Ballooning | ☐ |

#### Network (NIC 1 = WAN)
| 항목 | 값 |
|---|---|
| Bridge | **vmbr0** |
| Model | VirtIO |
| VLAN Tag | (비움) |

→ Confirm → **Start after created: ☐** → Finish

### Step 3-3. NIC 2 추가 (LAN)

VM 100 → Hardware → Add → Network Device:

| 항목 | 값 |
|---|---|
| Bridge | **vmbr0** ★ (Router-on-a-stick) |
| Model | VirtIO |
| VLAN Tag | **(비움)** |

→ Add.

### Step 3-4. pfSense 설치

VM 시작 → Console:

1. Accept (라이센스)
2. Install pfSense
3. Auto (UFS) BIOS
4. 디스크 선택 → YES
5. Manual Configuration → NO
6. Reboot

→ ISO 분리 (CD/DVD Drive → Do not use any media).

### Step 3-5. 인터페이스 할당

콘솔 메뉴:

```
Should VLANs be set up now? n
WAN interface: vtnet0
LAN interface: vtnet1
Optional: (Enter)
Proceed? y
```

### Step 3-6. WAN/LAN IP 설정

콘솔 메뉴 **2 (Set interface IP)**:

#### WAN (1)
```
DHCP? y
IPv6? n
HTTP web GUI? n
```

#### LAN (2)
```
DHCP? n
LAN IP: 172.16.0.1
Subnet bits: 24
Gateway: (Enter)
IPv6? n
DHCP server? n
HTTP web GUI? n
```

---

## Phase 4: pfSense 기본 설정 (Setup Wizard)

### Step 4-1. 임시 IP 추가 (kosa1)

```bash
ssh root@192.168.21.2
ip addr add 172.16.0.20/24 dev vmbr0
ping -c 3 172.16.0.1
```

### Step 4-2. SSH 터널 (노트북)

```bash
ssh -L 8443:172.16.0.1:443 root@192.168.21.2
```

### Step 4-3. 브라우저 접속

```
https://localhost:8443
```

→ admin / pfsense.

### Step 4-4. Setup Wizard

| 화면 | 입력 |
|---|---|
| Hostname | pfsense-01 |
| Domain | team2.local |
| Primary DNS | 1.1.1.1 |
| Secondary DNS | 8.8.8.8 |
| Override DNS | ☑ |
| Time | kr.pool.ntp.org / Asia/Seoul |
| WAN Type | DHCP |
| **Block private networks** | **☐ 체크 해제** |
| LAN IP | 172.16.0.1/24 |
| Admin Password | (강한 비밀번호) |

→ Reload.

---

## Phase 5: VLAN 인터페이스 생성

### Step 5-1. VLAN 4개 정의

**Interfaces → Assignments → VLANs 탭 → +Add (4번)**:

| Parent | VLAN Tag | Description |
|---|---|---|
| vtnet1 | 10 | Public |
| vtnet1 | 20 | DMZ |
| vtnet1 | 30 | Internal |
| vtnet1 | 40 | Mgmt |

### Step 5-2. 인터페이스 할당

**Interfaces → Assignments → 메인 탭** → Available에서 4개 +Add → OPT2/3/4/5 생성.

### Step 5-3. 각 VLAN IP 부여

**Interfaces → 각 OPT**:

| 인터페이스 | Description | IP |
|---|---|---|
| OPT2 (VLAN10) | VLAN10_Public | **172.16.21.2/24** |
| OPT3 (VLAN20) | VLAN20_DMZ | **172.16.22.2/24** |
| OPT4 (VLAN30) | VLAN30_Internal | **172.16.23.2/24** |
| OPT5 (VLAN40) | VLAN40_Mgmt | **172.16.24.2/24** |

각각:
- Enable: ☑
- IPv4 Configuration Type: Static IPv4
- IPv4 Upstream Gateway: (None)

→ Save → Apply Changes.

---

## Phase 6: DHCP 서버 설정

### VLAN별 정책

| VLAN | DHCP | 이유 |
|---|---|---|
| 10 (Public) | ☐ 끔 | 서버는 Static (DNS A 레코드) |
| 20 (DMZ) | ☐ 끔 | 서버는 Static |
| 30 (Internal) | ☑ 켬 (좁은 풀) | 서버는 Static, VM은 DHCP |
| 40 (Mgmt) | **☑ 켬** | 노트북 자동 할당 |

### Step 6-1. VLAN 30 DHCP

**Services → DHCP Server → VLAN30_Internal 탭**:

| 항목 | 값 |
|---|---|
| Enable | ☑ |
| Range Start | 172.16.23.100 |
| Range End | 172.16.23.200 |

→ Save.

### Step 6-2. VLAN 40 DHCP (관리망 - 필수)

**Services → DHCP Server → VLAN40_Mgmt 탭**:

| 항목 | 값 |
|---|---|
| Enable | ☑ |
| Range Start | 172.16.24.100 |
| Range End | 172.16.24.200 |
| DNS Server 1 | 1.1.1.1 |
| DNS Server 2 | 8.8.8.8 |

→ Save → Apply.

### IP 배분 가이드

각 VLAN 내부:

```
.1                CARP VIP (게이트웨이)
.2                pfSense kosa1
.3                pfSense kosa2 (HA)
.4 ~ .9           예비
.10 ~ .99         Static 서버
.100 ~ .200       DHCP 풀
.201 ~ .254       예비
```

---

## Phase 7: 방화벽 규칙 (Zone 분리)

각 VLAN 탭 (Firewall → Rules):

### VLAN 40 (Mgmt) — 모두 허용

```
Action:      Pass
Interface:   VLAN40_Mgmt
Protocol:    Any
Source:      VLAN40_Mgmt net
Destination: any
```

### VLAN 30 (Internal) — 외부 OK, 내부 VLAN 차단

```
[Rule 1] Pass / Any / VLAN30 net → !RFC1918
[Rule 2] Block / Any / VLAN30 net → 172.16.0.0/16
```

### VLAN 20 (DMZ) — 외부 OK, 내부 차단

```
[Rule 1] Pass / Any / VLAN20 net → !RFC1918
[Rule 2] Block / Any / VLAN20 net → VLAN30 net
[Rule 3] Block / Any / VLAN20 net → VLAN40 net
```

### VLAN 10 (Public) — 외부 OK, 모든 내부 차단

```
[Rule 1] Pass / Any / VLAN10 net → !RFC1918
[Rule 2] Block / Any / VLAN10 net → 172.16.20.0/24
[Rule 3] Block / Any / VLAN10 net → 172.16.23.0/24
[Rule 4] Block / Any / VLAN10 net → 172.16.24.0/24
```

→ 각 탭 Save → Apply Changes.

---

## Phase 8: 영구 적용 (Hook Script)

### Step 8-1. Hook Script 생성 (kosa1)

```bash
ssh root@192.168.21.2

mkdir -p /var/lib/vz/snippets

cat > /var/lib/vz/snippets/vlan-hook.sh << 'EOF'
#!/bin/bash
vmid="$1"
phase="$2"

if [ "$phase" = "post-start" ]; then
    sleep 3
    for tap in $(ls /sys/class/net/ | grep "^tap${vmid}i"); do
        bridge vlan add vid 2-4094 dev $tap 2>/dev/null
        echo "$(date) - VLAN added to $tap" >> /var/log/vlan-hook.log
    done
fi
EOF

chmod +x /var/lib/vz/snippets/vlan-hook.sh
```

### Step 8-2. 모든 VM에 적용

```bash
# 모든 VMID에 hook 적용
for vmid in $(qm list | tail -n +2 | awk '{print $1}'); do
    qm set $vmid --hookscript local:snippets/vlan-hook.sh
    echo "Applied to VM $vmid"
done
```

### Step 8-3. 재부팅 테스트

```bash
qm shutdown 100
sleep 5
qm start 100
sleep 15

bridge vlan show dev tap100i1 | head -3
```

---

## Phase 9: kosa2 환경 구성

### Step 9-1. SSH 접속

```bash
ssh root@192.168.21.3
```

### Step 9-2. interfaces 파일 (kosa1과 동일, IP만 변경)

```bash
cp /etc/network/interfaces /etc/network/interfaces.bak.$(date +%s)
nano /etc/network/interfaces
```

**전체 내용**:

```
auto lo
iface lo inet loopback

auto eno1
iface eno1 inet manual

auto enp1s0f0
iface enp1s0f0 inet manual
    mtu 9000

iface enp1s0f1 inet manual

auto vmbr0
iface vmbr0 inet static
    address 192.168.21.3/24
    gateway 192.168.21.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 1-4094
    up bridge vlan add vid 2-4094 dev vmbr0 self

auto vmbr1
iface vmbr1 inet static
    address 10.10.10.32/24
    bridge-ports enp1s0f0
    bridge-stp off
    bridge-fd 0
    mtu 9000

source /etc/network/interfaces.d/*
```

### Step 9-3. 적용 + 검증

```bash
ifreload -a

ip -br addr | grep vmbr
ip -d link show vmbr0 | grep vlan_filtering
ping -c 3 192.168.21.1
ping -c 3 192.168.21.2     # kosa1
ping -c 3 10.10.10.31      # kosa1 Ceph
```

### Step 9-4. Hook Script (kosa2도)

```bash
mkdir -p /var/lib/vz/snippets

cat > /var/lib/vz/snippets/vlan-hook.sh << 'EOF'
#!/bin/bash
vmid="$1"
phase="$2"

if [ "$phase" = "post-start" ]; then
    sleep 3
    for tap in $(ls /sys/class/net/ | grep "^tap${vmid}i"); do
        bridge vlan add vid 2-4094 dev $tap 2>/dev/null
        echo "$(date) - VLAN added to $tap" >> /var/log/vlan-hook.log
    done
fi
EOF

chmod +x /var/lib/vz/snippets/vlan-hook.sh
```

### Step 9-5. kosa2에 두 번째 pfSense VM 생성

Proxmox 웹UI → kosa2 → Create VM (kosa1과 동일 절차):

- VM ID: **101**
- Name: **pfsense-02**
- 나머지 모두 kosa1과 동일

NIC 2개 모두 vmbr0 (VLAN Tag 비움).

### Step 9-6. pfSense 설치 (kosa2)

설치 절차 kosa1과 동일.

### Step 9-7. kosa2 pfSense 인터페이스 할당 + IP

콘솔에서:

```
Should VLANs be set up now? n
WAN: vtnet0
LAN: vtnet1
Optional: (Enter)
Proceed? y
```

#### IP 설정

WAN: DHCP

LAN:
```
DHCP? n
LAN IP: 172.16.0.2     ← kosa1은 .1, kosa2는 .2
Subnet bits: 24
Gateway: (Enter)
DHCP server? n
```

> **VLAN 인터페이스는 kosa2에 직접 만들지 않음**. XMLRPC Sync로 자동 복제됨.

### Step 9-8. VM에 hook 적용

```bash
qm set 101 --hookscript local:snippets/vlan-hook.sh
```

---

## Phase 10: HA 이중화 (CARP + pfsync + XMLRPC)

### 10-1. SYNC 인터페이스 추가 (양쪽 pfSense)

**Proxmox 웹UI → 각 pfSense VM → Hardware → Add Network Device**:

| 항목 | 값 |
|---|---|
| Bridge | vmbr0 |
| VLAN Tag | **99** (pfsync 전용) |
| Model | VirtIO |

→ kosa1, kosa2 양쪽 다 추가 후 VM 재시작.

### 10-2. SYNC 인터페이스 활성화 (각 pfSense)

#### kosa1 pfSense 웹UI

**Interfaces → Assignments**:
- Available: vtnet2 → Add → OPT6 생성

**Interfaces → OPT6**:
- Enable: ☑
- Description: SYNC
- IPv4: Static **10.10.99.1/30**
- Gateway: (None)

→ Save → Apply.

#### kosa2 pfSense 웹UI

같은 방식, IP만:
- IPv4: Static **10.10.99.2/30**

### 10-3. SYNC 방화벽 (Allow All)

**Firewall → Rules → SYNC 탭**:

```
Action:   Pass
Protocol: Any
Source:   any
Dest:     any
```

→ 양쪽 다 추가.

### 10-4. CARP VIP 만들기 (kosa1만)

**Firewall → Virtual IPs → +Add** (각 인터페이스마다):

| Interface | Type | Address | VHID | Skew | Password |
|---|---|---|---|---|---|
| WAN | CARP | 192.168.21.10/24 | 1 | 0 | (공통) |
| VLAN10_Public | CARP | **172.16.21.1/24** | 10 | 0 | (공통) |
| VLAN20_DMZ | CARP | **172.16.22.1/24** | 20 | 0 | (공통) |
| VLAN30_Internal | CARP | **172.16.23.1/24** | 30 | 0 | (공통) |
| VLAN40_Mgmt | CARP | **172.16.24.1/24** | 40 | 0 | (공통) |

→ 각각 Save → Apply.

> **VHID는 VLAN별 다르게**, **Password는 모두 동일**.

### 10-5. VLAN 인터페이스 IP 변경 (.2 → 그대로, .3 추가는 kosa2)

#### kosa1 (그대로)
- VLAN10: 172.16.21.2 (이미)
- VLAN20: 172.16.22.2
- VLAN30: 172.16.23.2
- VLAN40: 172.16.24.2

#### kosa2 (XMLRPC 동기화 후 변경)
- VLAN10: 172.16.21.3
- VLAN20: 172.16.22.3
- VLAN30: 172.16.23.3
- VLAN40: 172.16.24.3

> **VM의 게이트웨이는 CARP VIP (.1)** 사용.

### 10-6. pfsync 설정 (양쪽)

**System → High Avail. Sync → State Synchronization Settings**:

#### kosa1
| 항목 | 값 |
|---|---|
| Synchronize States | ☑ |
| Synchronize Interface | **SYNC** |
| pfsync Synchronize Peer IP | **10.10.99.2** |

#### kosa2
| 항목 | 값 |
|---|---|
| Synchronize States | ☑ |
| Synchronize Interface | **SYNC** |
| pfsync Synchronize Peer IP | **10.10.99.1** |

→ 각각 Save.

### 10-7. XMLRPC Sync (kosa1만)

**System → High Avail. Sync → Configuration Synchronization Settings**:

| 항목 | 값 |
|---|---|
| Synchronize Config to IP | **10.10.99.2** |
| Remote System Username | admin |
| Remote System Password | (kosa2 admin 비밀번호) |
| 동기화 항목 | **모두 체크** (Users, Auth, Certs, Rules, NAT, IPsec, OpenVPN, DHCP, Virtual IPs, ...) |

→ Save.

> **kosa2는 XMLRPC Sync 설정 안 함** (피드백 루프 방지).

### 10-8. NAT Outbound 변경 (CARP VIP 사용)

**Firewall → NAT → Outbound**:

- Mode: **Manual** 또는 **Hybrid**
- 각 룰의 Translation IP를 **WAN CARP VIP (192.168.21.10)** 로 지정

→ 페일오버 시 NAT 매핑 안정.

### 10-9. VM의 게이트웨이 변경

각 VM/VLAN의 게이트웨이를 CARP VIP로:

| VLAN | 변경 전 (kosa1만) | 변경 후 (CARP VIP) |
|---|---|---|
| 10 | 172.16.21.2 | **172.16.21.1** |
| 20 | 172.16.22.2 | **172.16.22.1** |
| 30 | 172.16.23.2 | **172.16.23.1** |
| 40 | 172.16.24.2 | **172.16.24.1** |

→ DHCP 서버에서 Gateway 자동 적용.

---

## Phase 11: 검증 및 페일오버 테스트

### 11-1. 기본 상태 확인

#### kosa1 pfSense → Status → CARP

```
WAN VIP        MASTER ✅
VLAN10 VIP     MASTER ✅
VLAN20 VIP     MASTER ✅
VLAN30 VIP     MASTER ✅
VLAN40 VIP     MASTER ✅
```

#### kosa2 pfSense → Status → CARP

```
WAN VIP        BACKUP ✅
VLAN10 VIP     BACKUP ✅
...
```

### 11-2. VLAN 통신 검증

각 VLAN에 VM 만들어 ping:

```bash
# VM 안에서
ping 172.16.21.1   # CARP VIP (게이트웨이)
ping 8.8.8.8       # 인터넷
```

### 11-3. 페일오버 테스트

#### Master 다운 시뮬레이션

```bash
# kosa1 pfSense 정지
ssh root@192.168.21.2
qm shutdown 100
```

#### kosa2 자동 인계 확인

- kosa2 pfSense → Status → CARP → 모두 **MASTER** 로 변경
- VM에서 인터넷/통신 끊김 없이 유지 (pfsync 덕분)

#### 복구

```bash
qm start 100
```

→ kosa1 다시 살아나면 자동으로 MASTER 회수.

### 11-4. 노트북 관리망 이동

비관리형 스위치 → Port 5 (VLAN 40) 측에 노트북 연결:

```bash
# 노트북에서 ipconfig
# IP: 172.16.24.x (DHCP) ← VLAN 40
# Gateway: 172.16.24.1 (CARP VIP)

# pfSense 직접 접속
https://172.16.24.2 (kosa1)
https://172.16.24.3 (kosa2)
```

→ SSH 터널 더 이상 불필요.

---

## 부록: 트러블슈팅

### A. ping 실패 시 (Destination Host Unreachable)

```bash
# 호스트에서
bridge vlan show dev vmbr0
brctl show vmbr0

# tap에 VLAN 추가
for tap in $(brctl show vmbr0 | grep tap); do
    bridge vlan add vid 2-4094 dev $tap
done
```

### B. SSH 끊김 시

콘솔(IPMI/모니터)에서:

```bash
cp /etc/network/interfaces.bak.<숫자> /etc/network/interfaces
ifreload -a
```

### C. pfSense 웹UI 접속 안 됨

```bash
# kosa1에서
ip addr add 172.16.0.20/24 dev vmbr0

# 노트북에서 SSH 터널
ssh -L 8443:172.16.0.1:443 root@192.168.21.2

# 브라우저
https://localhost:8443
```

### D. CARP가 BACKUP에서 안 바뀜

확인 사항:
- VHID가 VLAN별로 다른지
- Password가 양쪽 동일한지
- pfsync IP가 서로 reachable한지
- Skew: kosa1=0(Master), kosa2=100(Backup)

### E. 게이트웨이 Pending 상태

- **Block private networks** 체크 해제 (Interfaces → WAN)
- 1~2분 대기 (Monitor IP 응답 시간)

---

## 완료 체크리스트

### 인프라 기본
- [ ] 관리형 스위치 트렁크 설정 완료
- [ ] kosa1 vmbr0 VLAN-aware
- [ ] kosa1 vmbr1 Ceph IP
- [ ] kosa2 vmbr0 VLAN-aware
- [ ] kosa2 vmbr1 Ceph IP

### pfSense 단일 (kosa1)
- [ ] pfSense VM 생성 + 설치
- [ ] WAN/LAN 인터페이스 할당
- [ ] VLAN 10/20/30/40 인터페이스
- [ ] DHCP 서버 (VLAN 30, 40)
- [ ] 방화벽 규칙 (Zone 분리)
- [ ] Hook script 적용

### pfSense HA (kosa2)
- [ ] kosa2 pfSense VM 생성 + 설치
- [ ] SYNC 인터페이스 (10.10.99.1/2)
- [ ] CARP VIP (5개)
- [ ] pfsync 설정 (양쪽)
- [ ] XMLRPC Sync (kosa1)
- [ ] NAT Outbound (CARP VIP)

### 검증
- [ ] CARP 상태 (Master/Backup)
- [ ] VM 통신 (각 VLAN)
- [ ] 페일오버 테스트
- [ ] 노트북 관리망 이동

---

## 다음 단계 (선택)

- **Ceph 클러스터링** — 6대 노드 별도 구성
- **Kubernetes 클러스터** — VLAN 30에 K8s 노드
- **ArgoCD GitOps** — CI/CD 파이프라인
- **Prometheus/Grafana** — 모니터링
- **AWS Site-to-Site VPN** — 하이브리드 클라우드

---

## 참고

### 주요 IP 요약

| 항목 | IP |
|---|---|
| 라우터 | 192.168.21.1 |
| kosa1 (관리) | 192.168.21.2 |
| kosa2 (관리) | 192.168.21.3 |
| kosa3 (관리) | 192.168.21.4 |
| kosa4 (관리) | 192.168.21.5 |
| kosa1 Ceph | 10.10.10.31 |
| kosa2 Ceph | 10.10.10.32 |
| pfSense kosa1 SYNC | 10.10.99.1 |
| pfSense kosa2 SYNC | 10.10.99.2 |
| WAN CARP VIP | 192.168.21.10 |
| VLAN10 CARP VIP | 172.16.21.1 |
| VLAN20 CARP VIP | 172.16.22.1 |
| VLAN30 CARP VIP | 172.16.23.1 |
| VLAN40 CARP VIP | 172.16.24.1 |

### 중요 파일

- `/etc/network/interfaces` — Proxmox 네트워크
- `/var/lib/vz/snippets/vlan-hook.sh` — VM 시작 시 VLAN 자동 적용
- `/var/log/vlan-hook.log` — Hook 실행 로그
- pfSense 설정 백업: **Diagnostics → Backup & Restore**

---

> **작성일**: 2026-05-08
> **대상**: KOSA 인프라 프로젝트 4인 팀
> **메인 목표**: pfSense HA + VLAN 라우팅 + 관리망 분리
