| 계층       | 직접 구성                   | AWS 관리형              |
| -------- | ----------------------- | -------------------- |
| 로드밸런서    | Nginx EC2에 직접 설치        | ALB                  |
| 오토스케일링   | 스크립트로 EC2 수동 관리         | Auto Scaling Group   |
| DB 고가용성  | MySQL Replication + MHA | RDS Multi-AZ         |
| DB 읽기 분산 | MySQL Replica 직접 구성     | RDS Read Replica     |
| 캐시 고가용성  | Redis Sentinel 직접 구성    | ElastiCache Multi-AZ |
| 컨테이너 관리  | Docker + 직접 배포          | ECS / EKS            |
