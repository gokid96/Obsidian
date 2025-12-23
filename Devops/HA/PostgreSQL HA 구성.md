## 필수 구성요소

#### 1. Patroni — 클러스터 관리
- 자동 failover, Primary/Replica 상태 관리, 리더 선출
#### 2. DCS (Distributed Configuration Store) — 클러스터 상태 저장
- **etcd**: 가볍고 단순, PostgreSQL HA에 가장 많이 사용
- **Consul**: 서비스 디스커버리 기능 포함, 다른 인프라와 통합 시 유리
- **ZooKeeper**: Kafka 등 이미 사용 중이면 재활용 가능, 단독으로는 무거움
#### 3. Proxy — Primary로 트래픽 라우팅
- **HAProxy**: L4/L7 로드밸런서, health check로 Primary 자동 감지
- **PgBouncer**: connection pooling 특화, 라우팅 기능은 제한적
- **Keepalived + VIP**: 가상 IP 방식, 단순하지만 failover 속도 느림


