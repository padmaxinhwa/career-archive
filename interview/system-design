# System Design Interview – Based on Real Experience

---

## 1. 대규모 기획전 서비스 설계해보세요

### 요구사항

- 기획전 상품 노출
- 재고 기반 필터링
- 사이트별 우선 노출 정책
- 품절 상품 하단 정렬
- 높은 트래픽 (시즌성 피크)

---

### High-Level Architecture

Client  
→ API Gateway  
→ Promotion Service  
→ Filter Service (재고/사이트 정책)  
→ Product Unit Service  
→ Mongo Index  
→ Cache (Redis)

---

### 설계 핵심 포인트

#### 1️⃣ 조회 최적화

- Write-heavy 영역과 Read-heavy 영역 분리
- 인덱스 컬렉션 별도 운영
- 배치 + Consumer 기반 동기화

#### 2️⃣ 정합성 보장

- Filter → Unit → Post Validation 단계
- 재고 API와 노출 API 분리
- 최종 노출 직전 재검증 단계 도입

#### 3️⃣ 확장성

- MSA 분리
- Stateless API
- 수평 확장 가능 구조

#### 4️⃣ 장애 대응

- 서킷 브레이커
- 타임아웃 설정
- 캐시 fallback 전략

---

## 2. WebFlux vs Virtual Thread 선택 기준은?

### WebFlux 적합한 경우

- 매우 높은 동시성
- Backpressure 중요
- 완전 비동기 파이프라인 필요

### Virtual Thread 적합한 경우

- 팀 유지보수성 중요
- IO-bound 중심 서비스
- 기존 동기 코드 기반 확장

### 내 선택 기준

> “팀의 이해 가능성과 유지보수성”을 가장 우선함

---

## 3. 인덱스 설계는 어떻게 했나요?

### 문제

- 직접 Mongo 조회 → 비효율
- 여러 API에서 동일 쿼리 반복

### 해결

- 조회 전용 인덱스 컬렉션 설계
- 배치 기반 데이터 적재
- 변경 이벤트 Consumer 반영

### Trade-off

- 약간의 지연 허용
- 대신 조회 성능 확보

---

## 4. 운영 디버깅 자동화 시스템 설계해보세요

### 기존 방식

- 로그 확인
- API 수기 호출
- DB 직접 조회

### 개선 설계

- API 체인 흐름 재현 시스템
- 단계별 파라미터 자동 생성
- 최종 유효성 검사 자동 수행

### 핵심 설계 원칙

- 사람이 반복하는 작업은 시스템화
- 실제 서비스 로직과 동일한 흐름 재현

---

## 5. 아키텍처 규칙을 어떻게 강제하나요?

### 선택

ArchUnit 도입

### 이유

- 코드리뷰는 사람이 실수함
- 규칙은 테스트로 강제해야 함

### 설계 포인트

- 계층 간 직접 의존 금지
- 어노테이션 기반 강제성
- 패키지 독립성 유지
