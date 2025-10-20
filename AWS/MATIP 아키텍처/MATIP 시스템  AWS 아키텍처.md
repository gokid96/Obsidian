
### Single VPC + Active-Standby 구성

```
VPC (10.0.0.0/16)

Public Subnet (10.0.1.0/24) - AZ-A
├─ NAT Gateway A
└─ Bastion Host (관리용)

Public Subnet (10.0.2.0/24) - AZ-B
└─ NAT Gateway B

Private Subnet (10.0.11.0/24) - AZ-A 
├─ Application Server A (Active) 
│ ├─ Ruby 애플리케이션 실행중 
│ ├─ 포트 350/351 Listen 
│ └─ 항공사와 연결됨 
└─ Redis Primary

Private Subnet (10.0.12.0/24) - AZ-B
├─ Application Server A (Standby)
│ ├─ Ruby 애플리케이션 대기중 
│ ├─ 포트 Listen 안함 
│ └─ Primary 장애시에만 활성화
└─ Redis Replica 

Database Subnet (10.0.21.0/24) - Multi-AZ
└─ RDS PostgreSQL (Primary + Standby)
```
### 메시지 처리 계층

```
Application Servers:
├─ Ruby 애플리케이션 서버 (코드 기준)
├─ Auto Scaling 없음 (상태 보존 필요)
├─ 수동 스케일링
└─ 로드밸런서 없음 (단일 연결)

Redis Cluster:
├─ ElastiCache Redis
├─ 메시지 큐잉
├─ 세션 상태 저장
└─ Primary-Replica 구성
```

### 데이터베이스

```
RDS PostgreSQL:
├─ db.r5.large (메모리 집약적)
├─ Multi-AZ 배포 필수
├─ 자동 백업 30일
└─ 읽기 전용 복제본 1개
```

## 네트워크 보안

### Security Groups

```
Application-Server-SG: 
├─ Inbound: 350/351 from 항공사 IP
├─ Inbound: 22 from Bastion-SG 
├─ Outbound: 6379 to Redis-SG 
└─ Outbound: 5432 to DB-SG

Database-SG:
├─ Inbound: 5432 from App/MATIP SG만
└─ Outbound: None
```

### 접근 제어

```
Systems Manager Session Manager:
├─ SSH 키 없이 서버 접근
├─ 모든 세션 로깅
└─ IAM 기반 권한 관리

VPN 연결 (온프레미스):
├─ Site-to-Site VPN
├─ BGP 라우팅
└─ 법무부 시스템 연결
```

## 고가용성 구성

### 페일오버 메커니즘

```
CloudWatch + Lambda:
├─ MATIP 연결 상태 모니터링
├─ Primary 장애시 자동 감지
├─ Standby 서버 자동 활성화
└─ DNS 레코드 업데이트

Elastic IP 기반 페일오버: 
├─ Primary 서버에 EIP 할당 
├─ 장애시 EIP를 Standby로 이동 
└─ 네트워크 레벨 페일오버
```

### 데이터 동기화

```
Real-time Sync:
├─ Redis Primary → Replica 복제
├─ RDS Multi-AZ 동기 복제
└─ 애플리케이션 로그 S3 저장
```

## 모니터링 및 알람

### 필수 모니터링

```
MATIP 특화 메트릭:
├─ 연결 상태 (Connected/Disconnected)
├─ 메시지 처리 속도 (msg/sec)
├─ 세션 타임아웃 횟수
└─ 큐 대기 메시지 수

시스템 메트릭:
├─ CPU/Memory/Network
├─ Redis 연결 수
├─ DB 연결 풀 상태
└─ 디스크 I/O
```

### 알람 설정

```
Critical 알람:
├─ MATIP 연결 끊김 (즉시)
├─ Primary 서버 다운 (1분)
├─ DB 연결 실패 (즉시)
└─ 메시지 처리 지연 (5분)
```

## 백업 및 재해복구

### 백업 전략

```
데이터 백업:
├─ RDS 자동 백업 (30일)
├─ Redis AOF 백업 (일간)
├─ 애플리케이션 로그 S3 (90일)
└─ EC2 AMI 스냅샷 (주간)

설정 백업:
├─ Infrastructure as Code (Terraform)
├─ 애플리케이션 설정 파일
└─ 보안 그룹/IAM 정책
```