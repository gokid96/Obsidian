

#### EC2 인스턴스 초기 사양

| EC2/RDS | t3.small | db.t4g.micro |
| ------- | -------- | ------------ |
| 메모리     | 2GB      | 1 GB         |
| CPU     | 2vCPU    | 2 vCPU       |

- **지금**: JVM 메모리 튜닝 + 불필요 프로세스 정리 + RDS idle 커넥션 정리 VS t3.medium(4GB)으로 업그레이드 + 스왑 추가

- **트래픽 늘면**: 세션을 Redis로 전환 → Auto Scaling 구성 또는 아직 Redis 사용하JWT 토큰 방식 로그인으로 전환할지

- **더 커지면**: RDS도 db.t4g.small(2GB)로 업그레이드