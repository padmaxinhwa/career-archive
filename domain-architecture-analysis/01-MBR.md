# 회원 도메인 (MMBR) – 면접 준비용 정리

회원 도메인(MMBR-LGC, MMBR-MSA)의 **사용 기술**, **시스템 디자인**, **잘한 점**, **못한 점**, **다른 구조로 했으면 좋았을 것**을 기술 관점에서 정리했습니다.

---

## 1. 사용 기술 요약

| 영역 | 기술 | 용도 |
|------|------|------|
| **레거시(LGC)** | Spring MVC, WAR, Maven, JSP/Tiles | 회원·멤버십 FO/API (member-webapp, mmember-webapp, api-member-webapp, fido-api-webapp) |
| **MSA** | Spring Boot, Gradle, K8s, Istio | 회원 query/command, 멤버십 query/command, alln(제휴), consumer, 독립 배포·스케일링 |
| **배치** | Azkaban, Shell, Spring Batch | 등급 산정, 쿠폰 적재, 어뷰징 알림, 멤버십 동기화 (batchExcutor.sh) |
| **DB** | Oracle, MyBatis, HikariCP | 트랜잭션 데이터, XML 매퍼, 연결 풀 |
| **캐시** | Redis (Read/Write 분리), Spring Data Redis | 세션·공통코드·조회 캐시, prod는 cluster |
| **메시지** | Kafka (배치 적재·코드 동기화), RabbitMQ (Config·이벤트) | 비동기 전파, 배치–온라인 경계 |
| **인증** | Cookie 세션, SSO 토큰(v2), Blossom SSO, FIDO(WebAuthn), JWT(옵션) | FO 로그인, 사내/외부 SSO, 생체인증, API 간 인증 |
| **게이트웨이** | Istio Gateway + VirtualService, Cloud API(BFF), HttpInvoker(PG) | MSA 라우팅·회로 차단, 단일 진입점, PG 연동 |
| **모니터링** | Prometheus, Pushgateway, Sentry, Logback | 메트릭·에러·로그 통합 |

---

## 2. 시스템 디자인 (기술적 설명)

### 2.1 아키텍처

- **하이브리드**: 레거시(WAR)와 MSA(Spring Boot)가 **동시에** 서비스한다. Cloud API(BFF)가 FO/BO 요청을 받아 회원·멤버십은 MMBR-MSA 또는 MMBR-LGC로 라우팅한다.
- **CQRS**: 회원·멤버십을 **query**(조회)와 **command**(변경) 서비스로 분리해 읽기/쓰기 부하와 모델을 나눴다. 조회 트래픽이 많을 때 query만 스케일 아웃할 수 있다.
- **배치 분리**: 대량 작업(등급 산정, 쿠폰 적재, 어뷰징, 동기화)은 Azkaban 플로우로 스케줄링하고, 전용 배치 앱(MMBR-LGC batch / MMBR-MSA batch)에서만 실행한다. 온라인 트랜잭션 서버와 **리소스·장애**를 분리했다.
- **트래픽 경로**: `FO/클라이언트 → Cloud API → (member/*) → MMBR-MSA 또는 MMBR-LGC`. MSA는 K8s + Istio로 노출되고, DestinationRule으로 연속 실패 시 해당 파드를 트래픽에서 제외하는 **회로 차단**을 둔다.

### 2.2 데이터·인프라

- **Oracle**: 회원·멤버십의 단일 소스. LGC는 resources에 db.url, MSA는 datasource로 접속. 서비스를 나눠도 **데이터는 단계적으로 분리**하는 전략(같은 DB 공유 → 필요 시 스키마/DB 분리).
- **Redis**: 세션·공통코드·조회 캐시. LGC는 write/read 포트 분리로 쓰기 1대·읽기 다대 구성. Prod는 redis.member.ssgadm.com 클러스터.
- **Kafka**: 배치가 “쿠폰 대상 적재” 등 결과를 토픽에 넣고, 컨슈머가 구독해 처리. 동기 호출 없이 **이벤트 드리븐**으로 서비스 간 결합을 낮춤.
- **설정**: Spring Cloud Config로 환경별 설정(DB, Redis, API URL)을 코드와 분리. mmember, member 등 프로파일로 회원 도메인 설정을 가져옴.

### 2.3 인증·세션

- **FO**: 쿠키·서버 세션(MbrSessionService, SsgCookieHandler). 로그인 후 FrontUserInfo로 사용자 유지.
- **SSO**: 88자 v2 토큰을 쿼리로 받아 SsoApiService로 검증 후 로그인 처리. Blossom SSO는 사내 임직원용 별도 플로우.
- **FIDO**: ssg-fido-api-webapp으로 WebAuthn RP 담당. 인증 정책 변경 시 회원/결제 앱과 **분리 배포** 가능.
- **API 간**: JWT는 dp-framework 옵션으로, 컨슈머·파트너 API에서 필요 시 사용.

---

## 3. 잘한 점

1. **점진적 MSA 전환**: 한 번에 모놀리스를 없애지 않고 LGC와 MSA를 **공존**시켜, 리스크를 나누고 단계적으로 이전할 수 있게 했다. BFF(Cloud API)가 두 백엔드를 동시에 라우팅하는 구조가 전환기에 적합하다.
2. **CQRS·서비스 분리**: 회원/멤버십을 query·command로 나눠 **읽기 부하 분리**와 **도메인 경계**를 명확히 했다. 조회만 많은 서비스는 독립 스케일링이 가능하다.
3. **배치와 온라인 분리**: Azkaban + 전용 배치 앱으로 대량 작업을 트랜잭션 서버와 분리해 **리소스 경합**과 **장애 전파**를 줄였다. Kafka로 배치 결과를 전달해 이벤트 드리븐 구조를 쓴 점도 좋다.
4. **인증 채널별 분리**: FO(세션·SSO), BO(DB·AD), API(JWT)를 나눠 **적합한 방식**을 쓰고, FIDO를 별도 앱으로 둬 보안·정책 변경 시 영향 범위를 줄였다.
5. **관찰 가능성**: Prometheus(Pushgateway), Sentry, Logback을 LGC·MSA 공통으로 써 **메트릭·에러·로그**를 한곳에서 보는 구조. Istio 회로 차단과 함께 **복원력**과 **원인 추적**을 동시에 만족한다.
6. **설정 외부화**: Spring Cloud Config로 환경별 설정을 중앙 관리해, **12 Factor**에 맞고 빌드 산출물 하나로 다환경 배포가 가능하다.

---

## 4. 못한 점 / 한계

1. **LGC와 MSA 이원화**: 회원 기능이 LGC와 MSA에 **동시에** 있어, 어떤 API가 어디로 가는지·트래픽 분배 정책을 새로 합류한 사람이 파악하기 어렵다. 문서화·라우팅 표가 없으면 운영 부담이 커진다.
2. **세션 저장 위치**: FO 세션이 **서버 메모리**에만 있으면 인스턴스 수평 확장 시 세션 불일치가 생길 수 있다. Redis에 세션을 두었다면 “세션 공유로 스케일 아웃”을 장점으로 말할 수 있으나, 문서만으로는 저장 위치가 명확하지 않다.
3. **배치 이원화**: MMBR-LGC(ssg-batch-member-app)와 MMBR-MSA(ssg-member-batch)에 배치가 나뉘어 있어, **어떤 Job을 어디서 돌리는지** 규칙이 복잡할 수 있다. Azkaban 플로우와 실제 실행 앱 매핑이 명확해야 한다.
4. **DB 단일 점**: 회원·멤버십이 **같은 Oracle**을 쓰면, DB 장애 시 회원 도메인 전체에 영향이 간다. MSA로 나눠도 저장소가 하나면 **장애 격리**에 한계가 있다.
5. **레거시 기술 부채**: JSP·WAR·HttpInvoker(PG) 등 레거시 스택이 남아 있어, 유지보수·인력 수급 측면에서 부담이 될 수 있다. FIDO·회원 앱을 최신 스택으로 통합하는 로드맵이 있으면 좋다.

---

## 5. 다른 구조로 했으면 좋았을 것

1. **세션 스토어 명시**: FO 세션을 **Redis 등 외부 스토어**에 두고, LGC 웹앱을 무상태로 만든다. 인스턴스 추가만으로 수평 확장이 가능하고, “세션 기반이지만 스케일 아웃 가능”을 설계 의도로 설명할 수 있다.
2. **배치 단일화**: 회원 배치를 **한 플랫폼**(예: MMBR-MSA Spring Batch만)으로 모으고, Azkaban은 “스케줄러·의존성”만 담당하게 한다. Job 코드·DB 접근이 한 곳에 모이면 유지보수와 모니터링이 쉬워진다.
3. **회원 DB 분리 단계 설계**: 단기에는 Oracle 공유, 중장기에는 **회원 전용 DB/스키마**를 두고, LGC·MSA가 각자 접근하거나 이벤트/CDC로 동기화하는 방안을 설계해 두면, 장애 격리와 도메인 경계가 분명해진다.
4. **API 버전·라우팅 문서화**: Cloud API에서 member/* 가 LGC vs MSA로 어떻게 라우팅되는지, 버전별로 **명시된 문서**나 설정이 있으면 온보딩·운영이 수월하다. “레거시와 MSA 공존”을 장점으로 쓰려면 **라우팅 규칙**이 잘 정의되어 있어야 한다.
5. **인증 경계 일원화**: FO 세션 검증을 BFF(Cloud API)에서만 하고, 하위 서비스는 “BFF가 넘긴 사용자 식별자”만 신뢰하는 구조로 정리하면, LGC·MSA 모두 **stateless**에 가깝게 가져갈 수 있다. (현재도 비슷할 수 있으나, 경계가 코드/문서에 드러나면 좋다.)

---

## 6. 면접에서 강조할 포인트

- **“회원 도메인은 레거시(LGC)와 MSA를 함께 쓰는 하이브리드”**라고 한 줄로 정리하고, **점진적 전환**과 **BFF가 두 백엔드를 라우팅**한다는 점을 말할 수 있으면 좋다.
- **CQRS**로 query/command를 나눈 **이유**(읽기 부하·도메인 분리)와 **효과**(조회 서비스만 스케일 등)를 구체적으로 설명할 수 있으면 좋다.
- **배치와 온라인 분리**로 “대량 작업이 트랜잭션 서버에 영향을 주지 않도록 했다”는 설계 의도를 말할 수 있으면 좋다.
- **한계**는 “LGC·MSA 이원화로 복잡도 증가”, “DB 단일 점”, “배치 이원화” 정도를 솔직히 말하고, **개선 방향**(세션 스토어, 배치 단일화, DB 분리 설계)을 같이 말하면 설계 판단력을 보여줄 수 있다.

---

작성일: 2025-02-27  
기준: MMBR-LGC, MMBR-MSA (회원 도메인만)
