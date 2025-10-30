- 로드 밸런서(Load Balancer) - 트래픽을 여러 서버에 분산
- 탄력적 IP(Elastic IP) - 서버 재시작해도 안바뀌는 고정 IP
- Auto Scaling - 트래픽 많으면 서버 자동 추가, 적으면 자동 제거 비용 절약
- 전용 호스트(Dedicated Host) - AWS 물리 서버 1대를 독점 임대 (비싸지만 라이선스/보안 규정 필요할 때)
- 보안그룹(Security Group) - 어떤 포트/IP에서 들어오는 트래픽을 허용/차단할지 결정하는 방화벽
- 배치그룹(Placement Group) - 서버들을 물리적으로 가까이(빠름) 또는 멀리(안전하게) 배치하는 전략
- 스냅샷 (Snapshot) - 서버 디스크 전체를 특정 시점에 사진 찍듯 백업 (Git commit과 비슷)
- 키페어(Key Pair) - 서버에 접속하기 위한 전자 열쇠 (.pem 파일 = 비밀번호 대신 쓰는 키)
- Capacity Reservations  - 대규모 이벤트 대비해 특정 인스턴스 타입을 미리 예약해두는 것 (거의 안 씀)
- EBS 볼륨(Elastic Block Store) - EC2 인스턴스 1개에만 연결되는 전용 하드디스크
- S3 (Simple Storage Service) -어디서든 접근 가능한 무제한 용량 저장소 (Google Drive 같은 개념, EBS와 다름)


```
[인터넷]
 ↓ 
[Route 53] - example.com 
 ↓ 
[Elastic Load Balancer] 
 ↓ 
┌───[VPC (10.0.0.0/16)]─────────────┐ 
│ │ │ [Public Subnet (10.0.1.0/24)] │ 
│ ├─ Internet Gateway 
│ │ ├─ jump-node (SSH용) │ │ └─ NAT Gateway │ │ ↓ │ │ [Private Subnet (10.0.2.0/24)] │ │ ├─ yuhan-prod-1 │ │ ├─ yuhan-prod-2 │ │ └─ yuhan-test-1 │ │ │ │ [Security Group] │ │ - SSH: jump-node만 허용 │ │ - HTTP/HTTPS: 로드밸런서만 허용 │ └────────────────────────────────────┘
```
