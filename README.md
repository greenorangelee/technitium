# Technitium DNS Server 설치 기록

Proxmox LXC 위에 Docker + Dockhand를 사용해서 Technitium DNS Server를 구축한 과정을 기록한 레포입니다.

---

## 환경

| 항목 | 내용 |
|------|------|
| Proxmox 버전 | 9.x |
| CPU | AMD Ryzen 7 5700X |
| Dockhand LXC IP | 10.100.100.111 |
| Technitium LXC IP | 10.100.100.121 |
| 네트워크 | UniFi UCG Fiber + 관리형 스위치 + VLAN 구성 |

---

## 레포 구조

```
technitium-dns-setup/
├── hawser/
│   ├── docker-compose.yml   # Hawser 에이전트 (Dockhand 원격 연결)
│   └── .env                 # Hawser 환경변수
├── technitium/
│   ├── docker-compose.yml   # Technitium DNS Server
│   └── .env                 # Technitium 환경변수
├── .gitignore
└── README.md
```

---

## 전체 아키텍처

```
UniFi UCG Fiber
    └── DNS 쿼리
        └── Technitium LXC (10.100.100.121)
                └── Docker
                        ├── technitium-dns  (DNS 서버, 포트 53/5380)
                        └── hawser          (Dockhand 원격 에이전트)

Dockhand LXC (10.100.100.111)
    └── Docker
            ├── dockhand    (컨테이너 관리 UI, 포트 3000)
            └── postgres    (Dockhand DB)
```

---

## 1단계 — Proxmox LXC 생성

### Technitium 전용 LXC 생성

Proxmox 웹 UI에서 새 LXC 컨테이너 생성:

| 항목 | 값 |
|------|------|
| 템플릿 | ubuntu-24.04-standard |
| vCPU | 2코어 |
| RAM | 2048 MB |
| 디스크 | 20 GB |
| 네트워크 | 고정 IP (10.100.100.121/24) |
| DNS VLAN | 해당 VLAN tag 지정 |

### Nesting 활성화 (필수)

Docker를 LXC 안에서 실행하려면 반드시 필요합니다.

**방법 1 — 웹 UI:**
LXC 선택 → `Options` → `Features` → `Nesting` 체크

**방법 2 — Proxmox 호스트 쉘:**
```bash
pct set <CTID> --features nesting=1
```

### systemd-resolved 비활성화

Ubuntu 24.04는 기본적으로 `systemd-resolved`가 53번 포트를 점유합니다.
Technitium이 53번 포트를 사용하려면 반드시 비활성화해야 합니다.

```bash
systemctl stop systemd-resolved
systemctl disable systemd-resolved
rm /etc/resolv.conf

# Technitium이 올라오기 전까지 임시 DNS 설정
echo "nameserver 1.1.1.1" > /etc/resolv.conf
```

---

## 2단계 — Docker 설치

Technitium LXC 콘솔에서 실행:

```bash
apt update && apt upgrade -y
curl -fsSL https://get.docker.com | sh
docker --version  # 설치 확인
```

---

## 3단계 — Hawser 에이전트 설치

Dockhand에서 Technitium LXC를 원격으로 관리하기 위한 에이전트입니다.

### Dockhand 웹 UI에서 토큰 발급

1. Dockhand 웹 UI (`http://10.100.100.111:3000`) 접속
2. `Environments` → `+` 클릭
3. Connection type: `Hawser agent (edge)` 선택
4. 표시된 토큰 복사
5. Public IP: `10.100.100.121` 입력

### Hawser 컨테이너 실행

Technitium LXC 콘솔에서:

```bash
mkdir -p /home/docker/hawser/stacks
cd /home/docker/hawser
```

`hawser/.env` 파일을 참고해서 `.env` 파일 작성:

```bash
nano .env
```

```env
DOCKHAND_SERVER_URL=ws://10.100.100.111:3000/api/hawser/connect
HAWSER_TOKEN=Dockhand에서_발급받은_토큰
```

`hawser/docker-compose.yml` 파일 복사 후 실행:

```bash
nano docker-compose.yml  # 레포의 hawser/docker-compose.yml 내용 붙여넣기
docker compose up -d
docker compose logs -f   # Connected to Dockhand 메시지 확인
```

### Dockhand에서 환경 추가 완료

Dockhand 웹 UI에서 `+ Add` 버튼 클릭 후 환경이 `Online` 상태로 바뀌면 성공입니다.

---

## 4단계 — Technitium DNS Server 배포

### 디렉토리 생성

Technitium LXC 콘솔에서:

```bash
mkdir -p /home/docker/dockhand/data/stacks/technitium-dns/config
mkdir -p /home/docker/dockhand/data/stacks/technitium-dns/logs
```

### Dockhand에서 스택 배포

1. Dockhand 웹 UI → Technitium 환경 선택
2. `Stacks` → `+ Create`
3. Stack name: `technitium-dns`
4. `technitium/docker-compose.yml` 내용 붙여넣기
5. `technitium/.env` 내용 참고해서 환경변수 입력:

```env
DNS_SERVER_DOMAIN=dns.yourdomain.com   # 본인 도메인으로 변경
DNS_SERVER_ADMIN_PASSWORD=비밀번호      # 반드시 변경
```

6. `Create & Start` 클릭

### 접속 확인

브라우저에서 `http://10.100.100.121:5380` 접속
- Username: `admin`
- Password: `.env`에 설정한 비밀번호

---

## 5단계 — Technitium 초기 설정

### 재귀 리졸버 설정

`Settings` → `Proxy & Forwarders` 탭:
- Forwarders: **비워두기** (재귀 리졸버로 동작하게 하려면 비워야 함)

`Settings` → `Recursion` 탭:
- Recursion: `Allow Recursion Only For Private Networks` 선택
- `QNAME Minimization` 체크 ✅
- `Randomize Name` 체크 ✅
- **Save** 클릭

### 블록리스트 설정

`Settings` → `Blocking` 탭:
- `Enable Blocking` 체크 ✅
- `Allow TXT Blocking Report` 체크 ✅
- Blocking Type: `NX Domain (recommended)` 선택

`Allow / Block List URLs`에 아래 URL 추가:

```
https://adguardteam.github.io/HostlistsRegistry/assets/filter_1.txt
https://adguardteam.github.io/HostlistsRegistry/assets/filter_2.txt
https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts
https://github.com/List-KR/List-KR/raw/master/filter.txt
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/pro.plus.txt
```

**Save** 클릭 → 블록리스트 자동 다운로드 시작

### Query Logs 앱 설치

`Apps` → `Get Apps` → `Query Logs (Sqlite)` → `Install`

설치 후 `Logs` → `Query Logs`에서 DNS 쿼리 로그 확인 가능

---

## 6단계 — UniFi DNS 설정

UniFi 웹 콘솔 → `Networks` → 해당 네트워크 선택 → `DNS Server` → `Edit`:

```
DNS server 1: 10.100.100.121
```

**Apply Changes** 클릭

### Technitium LXC DNS 업데이트

Technitium이 안정적으로 동작하는 것을 확인한 후:

```bash
echo "nameserver 10.100.100.121" > /etc/resolv.conf
```

---

## 추후 계획

- [ ] Technitium Secondary LXC 구성 (이중화)
- [ ] UniFi DNS 2순위 추가 (`10.100.100.122`)
- [ ] Traefik 리버스 프록시 연동
- [ ] DoH/DoT 암호화 프로토콜 활성화
- [ ] DNSSEC 설정

---

## 참고

- [Technitium DNS Server 공식 문서](https://technitium.com/dns/)
- [Technitium Docker 환경변수](https://github.com/TechnitiumSoftware/DnsServer/blob/master/DockerEnvironmentVariables.md)
- [Dockhand 공식 문서](https://github.com/Finsys/dockhand)
- [Hawser 에이전트](https://github.com/Finsys/hawser)
- [HaGeZi DNS 블록리스트](https://github.com/hagezi/dns-blocklists)
- [List-KR 한국어 필터](https://github.com/List-KR/List-KR)
