# Backend Engineer | Architecture & System Design

## Summary

대규모 전시 서비스에서  
레거시 구조를 MSA로 전환하고,  
Reactive 기반 구조를 Virtual Thread로 재설계하며,  
운영 디버깅을 자동화한 백엔드 엔지니어입니다.

저는 단순 기능 구현보다  
"왜 이 구조가 필요한지"를 먼저 고민하고 설계합니다.

---

## 1. MSA 전환 – 확장성과 정합성 문제 해결

### Why
레거시 인덱스 구조는:
- 중복 데이터 타입 존재
- 직접 Mongo 조회 기반 로직
- 확장 시 코드 변경 범위 과다

### What I Did
- Mongo 인덱스 컬렉션 재설계
- 일배치 + Consumer 기반 동기화 구조 설계
- 데이터 타입 2종 → 1종 통합
- 3개 이상 서비스 영역에 적용

### Result
- 확장 가능한 구조 확보
- 인덱스 로직 단일화
- 유지보수 범위 축소

---

## 2. API Chain 재설계 – 정합성 문제 해결

### Problem
Mongo 직접 조회 기반 상품 노출 구조 →  
품절/미노출 이슈 발생

### Approach
- Filter API → Unit API → Post Validation 구조 설계
- 5개 이상 API 재설계
- 정합성 재검증 단계 도입

### Outcome
- 상품 노출 오류 감소
- 호출 흐름 표준화
- 정책 기반 구조 확립

---

## 3. Reactive → Virtual Thread 전환

### Question 면접에서 반드시 나옴:
"왜 Reactive를 버렸나요?"

### My Decision
- 이벤트 기반 구조의 복잡성 증가
- 디버깅 및 에러 추적 어려움
- 실제 트래픽 패턴은 CPU-bound보다 IO-bound 비중 높음

### Action
- Java 21 Virtual Thread 도입
- 블로킹 모델 기반 단순화
- 저사용 고비용 기능 제거

### Result
- 코드 가독성 개선
- 디버깅 효율 향상
- 구조 단순화

---

## 4. Architecture Governance – ArchUnit

### Problem
패키지 의존성 규칙이 코드리뷰에만 의존

### Solution
- 6개 이상 주요 패키지 대상 ArchUnit 도입
- 계층 간 직접 의존 차단
- @BizComponent 강제 규칙 적용

### Impact
- 구조 위반 테스트 단계 차단
- 커뮤니케이션 비용 감소
- 신규 개발 시 구조 안정성 확보

---

## 5. Operational Automation – Internal Diagnostic Tool

### Why
MSA 전환 이후:
- 재고 동기화 이슈 증가
- 수기 로그 분석 필요
- 운영 대응 시간 증가

### What I Built
- 기획전 마스터/구분자 자동 조회
- 재고 필터 API 파라미터 자동 생성
- 상품 유닛 호출 흐름 시각화
- 자동 유효성 검사 및 동기화 기능

### Result
- 수기 디버깅 프로세스 제거
- 원인 분석 자동화
- 운영 대응 속도 개선

---

## Engineering Philosophy

- 기술은 복잡하게 만드는 게 아니라 단순하게 만들기 위해 선택
- 구조는 코드리뷰가 아니라 테스트로 지켜야 한다
- 운영 문제는 사람을 늘려 해결하는 게 아니라 시스템화해야 한다
