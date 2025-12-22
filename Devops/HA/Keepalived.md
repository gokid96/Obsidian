
## 사전 설정 사항
###### 가상 IP 선택:
- 사용 가능한 IP 확인 (예: 192.168.14.100) 
```bash
ping 192.168.14.100 
```
응답이 없어야 함 (사용되지 않는 IP) 
- 또는 네트워크 스캔 
```bash
nmap -sn 192.168.14.0/24
```

###### 방화벽 설정 (두 서버 모두)
```bash
# VRRP 프로토콜 허용
sudo iptables -A INPUT -p vrrp -j ACCEPT
sudo iptables -A OUTPUT -p vrrp -j ACCEPT

# 멀티캐스트 주소 허용
sudo iptables -A INPUT -d 224.0.0.18 -j ACCEPT
sudo iptables -A OUTPUT -d 224.0.0.18 -j ACCEPT
```

###### 네트워크 인터페이스 이름 확인 ip addr show
```bash
# 네트워크 인터페이스 이름 확인 
ip addr show
```

###### 호스트 간 통신 확인
```bash
# 32번 메인서버에서 35번 백업서버로
ping 192.168.14.35

# 35번 백업서버에서 32번 메인서버로  
ping 192.168.14.32

# SSH 연결도 확인
ssh user@192.168.14.35
```


## 1. Keepalived 설치 (두 서버 모두)

```bash
# 두 서버 모두에서 
sudo apt update && sudo apt install keepalived 
# 또는 
sudo yum install keepalived
```

## 2. 설정 파일 생성

**마스터 서버 (192.168.14.32)에서:**
```bash
sudo vi /etc/keepalived/keepalived.conf
```

다음 내용 입력:
```bash
vrrp_instance VI_1 {        # VRRP 인스턴스 이름 (임의 지정 가능, 관리용)
    state MASTER            # 초기 상태 (MASTER 또는 BACKUP 가능)
    interface enp1s0        # VRRP를 동작시킬 네트워크 인터페이스
    virtual_router_id 10    # 가상 라우터 ID (0~255 범위, 동일 그룹 내에서 동일해야 함)
    priority 110            # 우선순위 (0~255, 값이 높을수록 MASTER 우선)
    advert_int 1            # VRRP Advertisement 패킷 주기 (초 단위, 기본 1초)
    authentication {        # VRRP 인증 설정 블록
        auth_type PASS      # 인증 방식 (PASS: 평문 암호, AH: IPSEC AH)
        auth_pass testpass123  # 인증 비밀번호 (MASTER/BACKUP 모두 동일해야 함)
    }
    virtual_ipaddress {     # 가상 IP 주소 블록 (공유할 VIP 지정)
        192.168.14.100      # 실제로 서비스에 사용할 가상 IP (Failover 시 이동됨)
    }
}```

**백업 서버 (192.168.14.35)에서:**
```bash
sudo vi /etc/keepalived/keepalived.conf
```
다음 내용 입력
```bash
vrrp_instance VI_1 {       # VRRP 인스턴스 이름 (MASTER와 동일해야 동일 그룹으로 인식)
    state BACKUP           # 초기 상태 (BACKUP: MASTER 대기 모드)
    interface enp1s0       # VRRP를 동작시킬 네트워크 인터페이스
    virtual_router_id 10   # 가상 라우터 ID (MASTER와 동일해야 함)
    priority 100           # 우선순위 (MASTER보다 낮게 설정 → MASTER 장애 시 승격됨)
    advert_int 1           # VRRP Advertisement 패킷 주기 (MASTER와 동일해야 함)
    authentication {       # VRRP 인증 설정 블록
        auth_type PASS     # 인증 방식 (PASS: 평문 암호, AH: IPSEC AH)
        auth_pass testpass123  # 인증 비밀번호 (MASTER와 반드시 동일해야 함)
    }
    virtual_ipaddress {    # 가상 IP 주소 블록 (MASTER와 동일해야 함)
        192.168.14.100     # 공유할 가상 IP (MASTER 죽으면 BACKUP이 가져옴)
    }
}
```

- `virtual_router_id` 와 `auth_pass` 는 같은 VRRP 그룹 내 모든 서버에서 동일해야 함.
- `priority` 가 가장 높은 서버가 MASTER 가 됨 (동점 시 IP 주소가 높은 쪽이 MASTER).
- `virtual_ipaddress` 는 실제 서비스에 붙는 IP라, 클라이언트는 이 IP로 접속.
- **MASTER** 와 **BACKUP** 은 같은 `virtual_router_id`, `auth_pass`, `virtual_ipaddress` 를 가져야 됨
- `priority` 값은 MASTER가 더 높아야 함 (여기서는 MASTER=110, BACKUP=100).
- MASTER 장애 시 BACKUP이 VIP(`192.168.14.100`)를 인계받아 서비스가 끊기지 않음

## 4. Keepalived 시작

**두 서버에서 동시에:**
```bash
sudo systemctl start keepalived
sudo systemctl enable keepalived
```

## 5. 테스트 확인

**마스터 서버 (192.168.14.32)에서:**

```bash
ip addr show enp1s0
```

→ 가상 IP `192.168.14.100`이 보여야 함

**백업 서버 (192.168.14.35)에서:**

```bash
ip addr show enp1s0
```

→ 가상 IP가 보이지 않아야 함

**외부에서 테스트:**
```bash
ping 192.168.14.100
```

## 6. 장애 테스트

**마스터 서버에서 keepalived 중지:**

```bash
sudo systemctl stop keepalived
```

그 후 백업 서버에서 확인:
```bash
ip addr show enp1s0
```

→ 가상 IP가 백업 서버로 이동했는지 확인
