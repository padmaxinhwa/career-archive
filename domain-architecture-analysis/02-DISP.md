# 전시 도메인 (DP)

전시 도메인(MSA/ssg-dp.*, FRONT/ssg-dp.front.api-webapp)의 **사용 기술**, **시스템 디자인**, **잘한 점**, **못한 점**, **다른 구조로 했으면 좋았을 것**을 기술 관점에서 정리했습니다.

---

## 1. 사용 기술 요약

| 영역 | 기술 | 용도 |
|------|------|------|
| **API·서비스** | Spring Boot, Gradle, Kotlin(일부) | ssg-dp.module, corner, planshop, service-shop, brandstore-mvc, internal-verifier, front.api-webapp, consumer.api-webapp |
| **전시 API 진입** | Spring Boot, context-path (/dp/api) | FRONT/ssg-dp.front.api-webapp – 채널 웹앱에 전시(코너·플랜샵·카테고리) 데이터 제공 |
| **배치** | Spring Batch (ssg-dp.batch) | 카테고리 인덱싱 등 |
| **컨슈머** | ssg-dp.consumer | Kafka·RabbitMQ 구독, API 접근 로그·트리립 로그 수집, Elasticsearch 적재 |
| **DB** | Oracle(MyBatis), MongoDB(devdispdb) | 트랜잭션·마스터 데이터 / 설정·비정형·캐시성 데이터 |
| **캐시** | Redis (cache-config, standalone/cluster) | 조회 캐시 |
| **메시지** | Kafka, RabbitMQ (virtual-host: config) | 이벤트·로그 수집, Spring Cloud Config 연동 |
| **검색·로그** | Elasticsearch (RestHighLevelClient 7.x) | 카테고리 인덱싱, API 접근 로그, 트리립 로그, logback-elasticsearch-appender |
| **인증** | JWT (dp-framework 옵션) | API 간·파트너 API |
| **모니터링** | Sentry, Logback, ES Appender | 에러 추적, 로그 수집·검색 |

---

## 2. 시스템 디자인 (기술적 설명)

### 2.1 아키텍처

- **MSA 위주**: 전시는 **처음부터** Spring Boot 기반 MSA로 구성된다. 코너·플랜샵·서비스샵·브랜드스토어·모듈·프론트 API·컨슈머를 **서비스 단위**로 나눠 배포·스케일링을 독립적으로 한다.
- **전시 API 진입점**: **FRONT/ssg-dp.front.api-webapp**이 context-path `/dp/api` 등으로 전시 전용 API를 제공한다. 채널(ssgmall·grocery·department 등)은 이 API나 Cloud API의 /dp/* 를 호출해 코너·플랜샵·카테고리 데이터를 가져온다. 회원(member/*)과 **도메인별 진입점**이 분리되어 있다.
- **트래픽 경로**: `FO/클라이언트 → Cloud API 또는 /dp/api → 전시 MSA(ssg-dp.*)`. 설정·이벤트는 RabbitMQ·Spring Cloud Config로 전파한다.
- **이벤트·로그 파이프라인**: Kafka/메시지로 들어온 접근 로그·트리립 로그를 **ssg-dp.consumer**가 소비해 Elasticsearch에 적재한다. 로그는 **비동기·대량**이므로 메시지 큐 → ES 구조가 적합하다.

### 2.2 데이터·인프라

- **Oracle**: 전시·코너·플랜샵 등 **트랜잭션·마스터 데이터**. MyBatis, HikariCP로 접근.
- **MongoDB**: **설정·비정형·캐시성** 데이터(devdispdb 등). RabbitMQ·Config와 함께 이벤트/설정 저장에 사용. RDB와 **역할 분리**(폴리글랏 퍼시스턴스).
- **Elasticsearch**: **검색·로그** 전용. 카테고리 인덱싱(CategoryStaticIndexingConfiguration), API 접근 로그(AccessLogConsumer), 트리립 로그(TriipLogConsumer). “쓰기는 배치/컨슈머, 읽기는 검색/분석”으로 흐름을 나눈다.
- **Redis**: cache-config로 조회 캐시. standalone/cluster 환경별 구성.
- **Kafka·RabbitMQ**: Kafka는 이벤트·로그 스트림, RabbitMQ는 Config·작업 큐. 전시 컨슈머가 둘 다 사용한다.

### 2.3 검색·로그 설계

- **카테고리 인덱싱**: 배치(ssg-dp.batch)나 컨슈머가 카테고리 데이터를 ES에 인덱싱해 두고, 온라인은 **ES만 조회**하는 패턴. RDB 부하를 줄이고 검색 성능을 확보한다.
- **로그 수집**: AccessLogConsumer, TriipLogConsumer가 Kafka 등에서 로그를 받아 ES에 적재. **중앙 로그 검색·분석**이 가능하다. logback-elasticsearch-appender로 앱 로그도 ES에 넣을 수 있다.

---

## 3. 잘한 점

1. **도메인 경계 명확**: 전시는 **MSA·FRONT**로만 구성되고, 회원(MMBR)과 프로젝트·경로가 분리되어 있다. `/dp/api` vs `/member/*` 로 진입점이 나뉘어 **책임 경계**가 분명하다.
2. **검색·로그 레이어 분리**: RDB는 트랜잭션, **Elasticsearch**는 검색·로그로 쓰여 **역할 분리**가 잘 되어 있다. 카테고리 인덱싱·접근 로그·트리립 로그를 ES로 모으면 조회 성능과 분석이 동시에 가능하다.
3. **이벤트 드리븐 로그**: 로그를 Kafka 등으로 받아 컨슈머가 ES에 적재하는 **비동기 파이프라인**으로, 온라인 API 부하와 분리되어 있다.
4. **폴리글랏 퍼시스턴스**: Oracle(트랜잭션) + MongoDB(설정·비정형) + Elasticsearch(검색·로그)를 용도별로 선택해 **데이터 특성**에 맞게 저장소를 쓴다.
5. **전시 전용 API 서버**: ssg-dp.front.api-webapp으로 채널이 “전시만” 쓰는 API를 한 진입점으로 사용할 수 있어, BFF(Cloud API)와의 역할 나누기나 채널별 최적화가 용이하다.
6. **레거시 부담 적음**: 회원처럼 LGC(WAR)가 없어 **Spring Boot·Gradle·MSA** 일관성이 있고, 신규 인력이 이해하기 상대적으로 수월하다.

---

## 4. 못한 점 / 한계

1. **전시 MSA와 Istio 관계**: 회원 MMBR-MSA는 Istio Gateway로 K8s 노출·회로 차단이 문서에 나오는데, **전시 MSA**가 Istio를 쓰는지·라우팅·회로 차단 정책이 코드베이스만으로는 명확하지 않을 수 있다. “전시도 K8s·Istio를 쓴다”면 문서화해 두는 것이 좋다.
2. **서비스 수 다수**: ssg-dp.module, corner, planshop, service-shop, brandstore-mvc, internal-verifier, front, consumer 등 **서비스가 많아** 운영·배포·의존성 관리 비용이 커질 수 있다. 도메인 내부에서도 **서비스 통합** 후보(예: 코너+플랜샵)를 검토할 여지가 있다.
3. **MongoDB·Oracle 역할 중복 가능성**: “설정·비정형은 MongoDB, 트랜잭션은 Oracle”로 나눴지만, 어떤 기능이 어디에 저장되는지 경계가 모호하면 **데이터 이중화·일관성** 이슈가 생길 수 있다. 저장소별 사용 범위를 문서로 정리해 두면 좋다.
4. **캐시·Redis 사용 범위**: 회원처럼 “세션·공통코드”가 아니라 **전시 조회 캐시** 위주일 수 있는데, 캐시 키 설계·무효화 전략이 명확하지 않으면 **스테일 데이터** 위험이 있다. TTL·이벤트 기반 무효화 등이 정의되어 있는지 확인할 만하다.
5. **모니터링 통합**: Sentry·Logback·ES Appender는 있으나, **Prometheus/Pushgateway**가 전시 MSA에서 어떻게 쓰이는지(회원과 동일 스택인지)가 문서에 드러나 있으면 “관찰 가능성”을 면접에서 말하기 쉽다.

---

## 5. 다른 구조로 했으면 좋았을 것

1. **전시 게이트웨이·라우팅 문서화**: Cloud API의 /dp/* 가 어떤 ssg-dp.* 서비스로 매핑되는지, **라우팅 표·다이어그램**을 두면 온보딩·장애 대응이 빨라진다. Istio를 쓴다면 VirtualService·DestinationRule을 전시용으로도 명시해 두는 것이 좋다.
2. **서비스 그룹화 검토**: 코너·플랜샵·서비스샵 등이 **실제로 독립 배포·스케일이 필요한지**를 주기적으로 검토하고, 트래픽이 작은 서비스는 **하나의 “전시 코어” 서비스**로 묶어 서비스 수를 줄이는 방안을 고려할 수 있다. 마이크로서비스는 “작을수록 좋다”가 아니라 **경계가 맞을 때** 유리하다.
3. **캐시 무효화 전략**: 전시 데이터(코너·카테고리) 변경 시 Redis 캐시를 **이벤트 기반**으로 무효화하거나, TTL을 짧게 두는 정책을 설계해 두면 스테일 리스크를 줄일 수 있다. “캐시 쓴다” 수준을 넘어 **무효화 전략**을 말할 수 있으면 설계 깊이를 보여준다.
4. **로그·메트릭 표준화**: ES에 로그를 넣는 포맷·인덱스 전략을 팀 표준으로 두고, Prometheus 메트릭도 전시 MSA에서 **회원과 동일한 Pushgateway**를 쓰는지 확인해 “관찰 가능성”을 도메인 공통으로 맞추면 운영이 편해진다.
5. **BFF와 전시 API 역할 정리**: Cloud API(BFF)가 /dp/* 를 그대로 전시 MSA로 프록시하는지, ssg-dp.front.api-webapp이 **BFF에 가까운 집계**를 하는지 역할을 정리해 두면, “전시 클라이언트는 어디를 호출하는가”를 명확히 설명할 수 있다.

---

## 6. 면접에서 강조할 포인트

- **“전시 도메인은 MSA·FRONT로만 구성되고, 회원(MMBR)과 프로젝트·진입점이 분리되어 있다”**라고 한 줄로 정리할 수 있으면 좋다.
- **검색·로그를 Elasticsearch로 분리**한 이유(성능·분석)와 **이벤트 드리븐 로그 파이프라인**(Kafka → 컨슈머 → ES)을 구체적으로 말할 수 있으면 좋다.
- **폴리글랏 퍼시스턴스**(Oracle + MongoDB + ES)를 “데이터 특성에 맞게 저장소를 선택했다”는 식으로 설명할 수 있으면 좋다.
- **한계**는 “서비스 수 많음”, “캐시 무효화·라우팅 문서화 보강” 정도를 말하고, **개선 방향**(서비스 그룹화 검토, 캐시 무효화 전략, 라우팅·모니터링 문서화)을 같이 말하면 설계 판단력을 보여줄 수 있다.

---

작성일: 2025-02-27  
기준: MSA/ssg-dp.*, FRONT/ssg-dp.front.api-webapp (전시 도메인만)
