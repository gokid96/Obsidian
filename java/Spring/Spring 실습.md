

### **현재 프로젝트: JPA + MVC (전통적 Layered Architecture)**
```
✅ 배우는 것:
- Spring Boot 기본
- JPA/Hibernate
- REST API 설계
- DTO 패턴
- 트랜잭션 관리
- 테스트 코드
```

### **두 번째 프로젝트: DDD (Domain Driven Design)**
```
✅ 배우는 것:
- 도메인 중심 설계
- Aggregate, Entity, VO
- 도메인 서비스 vs 응용 서비스
- 이벤트 기반 아키텍처
- 헥사고날 아키텍처 (선택)

→ 중/대규모 프로젝트, 복잡한 비즈니스 로직에 적합
```



현재 프로젝트에서..
1. Entity를 API 응답으로 Controller에서 직접 반환하던 것을 DTO로 변환해서 반환하도록 수정(Entity는 JPA/DB용, DTO는 API 통신용으로 분리)
2. 