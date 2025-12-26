# Astarchia

> 팀/개인을 위한 문서 협업 플랫폼

## 프로젝트 개요

문서 관리 서비스로, 워크스페이스 기반의 팀 협업과 계층형 폴더/문서 구조를 지원합니다.

| 항목 | 내용                                                   |
|------|------------------------------------------------------|
| 개발 기간 | 2025.11 ~ 진행중                                        |
| 개발 인원 | 1인 (백엔드)                                             |
| 라이브 | [astarchia.com](https://astarchia.com)               |
| GitHub | [Repository](https://github.com/your-repo/astarchia) |

## 기술 스택

| 분류 | 기술 | 선택 이유 |
|------|------|----------|
| Language | Java 21 | Record, Pattern Matching 등 최신 문법 활용 |
| Framework | Spring Boot 3.5 | 생산성, 생태계, LTS 지원 |
| Security | Spring Security + Session | 현재 규모에서 간편함 우선, 확장 시 Redis 세션 클러스터링 전환 예정 |
| ORM | Spring Data JPA | 객체 지향적 데이터 접근, QueryDSL 도입 고려 중 |
| Database | MySQL 8.0 (RDS) | 안정성, AWS 통합, 추후 Read Replica 확장 가능 |
| Infra | AWS EC2, Cloudflare | 비용 효율적 구성, 트래픽 증가 시 ALB + Auto Scaling 전환 예정 |
| Email | AWS SES | 이메일 인증용, 비용 효율적 |

## 시스템 아키텍처

```
┌─────────────┐     ┌─────────────┐      ┌─────────────┐
│   Client    │────▶│   Vercel    │────▶ │ Cloudflare  │
│  (Browser)  │     │  (Frontend) │      │  (DNS Only) │
└─────────────┘     └─────────────┘      └─────────────┘
                                               │
                                               ▼
                    ┌─────────────┐     ┌─────────────┐
                    │  AWS SES    │◀────│  AWS EC2    │
                    │  (Email)    │     │  (Backend)  │
                    └─────────────┘     └─────────────┘
                                               │
                                               ▼
                                        ┌─────────────┐
                                        │  AWS RDS    │
                                        │  (MySQL)    │
                                        └─────────────┘
```

### 현재 구성 vs 확장 계획

| 구성 요소 | 현재 | 확장 시 |
|-----------|------|---------|
| 인증 | 세션 (Tomcat 내장) | Redis 세션 클러스터링 또는 JWT |
| 서버 | 단일 EC2 | ALB + Auto Scaling |
| DB | RDS 단일 | Read Replica 분리 |
| 캐싱 | 없음 | Redis 캐싱 (폴더 트리 등) |
| CDN | Cloudflare (DNS Only) | CloudFront 또는 Cloudflare Pro |

## 핵심 기능

### 1. 워크스페이스 & RBAC 기반 권한 관리
- 역할: OWNER → ADMIN → MEMBER → VIEWER
- enum ordinal 기반 권한 비교로 단순화
- 제약: OWNER 권한 양도 미구현 (향후 추가)

### 2. 계층형 폴더 구조
- 자기 참조 관계로 무한 중첩 지원
- 순환 참조 방지 로직 구현
- 향후: Closure Table 또는 Materialized Path 검토 (깊은 트리 조회 최적화)

### 3. 이메일 인증 (AWS SES)
- 6자리 인증번호, 10분 만료
- 인증 완료 후 데이터 삭제 (일회용)

## 프로젝트 구조

```
src/main/java/com/astarchia/
├── domain/
│   ├── auth/           # 인증
│   ├── user/           # 사용자
│   ├── workspace/      # 워크스페이스
│   ├── folder/         # 폴더
│   ├── post/           # 문서
│   └── email/          # 이메일 인증
└── global/
    ├── config/         # Security, SES 등
    ├── security/       # 인증/인가
    ├── exception/      # 예외 처리
    └── aop/            # 로깅
```

## 관련 문서

| 문서 | 내용 |
|------|------|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | 인프라 구성, 기술 선택 근거, 확장 계획 |
| [FEATURES.md](./FEATURES.md) | 핵심 기능의 설계 의도와 트레이드오프 |
| [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) | 문제 해결 사례 |

## 향후 계획

- [ ] 동시 편집 시 동시성 제어 (비관적 락 또는 CRDT 검토)
- [ ] N+1 쿼리 최적화 (fetch join, BatchSize)
- [ ] Redis 세션 클러스터링
- [ ] CI/CD 파이프라인 (GitHub Actions)
- [ ] 테스트 커버리지 확대
