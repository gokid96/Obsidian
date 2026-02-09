
## 목차

### 읽기 패턴
- [[01. Cache-Aside (Lazy Loading)]]
- [[02. Read-Through Cache]]

### 쓰기 패턴
- [[03. Write-Through]]
- [[04. Write-Behind (Write-Back)]]
- [[05. Write-Around]]

### 무효화 패턴
- [[06. Cache Invalidation (캐시 무효화)]]

### 고급 패턴
- [[07. Cache Stampede 방지]]
- [[08. Multi-Layer Cache (다계층 캐시)]]
- [[09. Cache Warming (캐시 워밍)]]

---

> [!info] 실무 가이드
> - 가장 일반적인 조합: **Cache-Aside + TTL**
> - 일관성 중요: **Write-Through**
> - 쓰기 성능 중요: **Write-Behind**
> - 대규모 트래픽: **Multi-Layer Cache + Cache Stampede 방지**
