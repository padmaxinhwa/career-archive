# 시스템 사용처·기술 정리

코드베이스 기준으로 정리한 **사용처**, **기술 스택**, **내용 설명**, **시스템 디자인과의 연관**입니다.

---

## 도메인 구분

문서에서는 **도메인별**로 사용처를 구분합니다.

| 도메인 | 구분 | 프로젝트/경로 | 설명 |
|--------|------|----------------|------|
| **회원 도메인** | MMBR | **MMBR-LGC**, **MMBR-MSA** | 회원·멤버십·제휴(alln)·FIDO·배치 등. `MMBR-` 접두사가 붙은 레포/프로젝트만 해당. |
| **전시 도메인** | DP | **MSA/ssg-dp.*** , **FRONT/ssg-dp.*** | 전시(Display)·코너·플랜샵·브랜드스토어·카테고리·컨슈머 등. MSA 폴더의 `ssg-dp.*` 와 FRONT의 전시용 API·화면. |

- **회원 도메인**: `MMBR-LGC`(레거시 웹/API/배치), `MMBR-MSA`(회원 query/command, 멤버십, alln, consumer, 배치). 회원·로그인·멤버십·쿠폰·어뷰징 등만 해당.
- **전시 도메인**: `MSA/` 아래 **ssg-dp.module**, **ssg-dp.batch**, **ssg-dp-corner**, **ssg-dp.planshop**, **ssg-dp.service-shop**, **ssg-dp.brandstore-mvc**, **ssg-dp.internal-verifier**, **ssg-dp.front.api-webapp**, **ssg-dp.consumer** 등 전시·카테고리·코너·플랜샵·로그 수집. **FRONT** 쪽에서는 전시 API를 소비하는 웹앱(예: ssg-dp.front.api-webapp) 또는 채널(ssgmall·grocery·department 등)이 전시 데이터를 사용.

아래 섹션에서 **회원** / **전시** 를 명시한 것은 해당 도메인에 한정된 사용처입니다. BACK(BO/PO/리소스)는 공통·채널 레이어로 두 도메인 모두 연동할 수 있습니다.

---

## 시스템 디자인 개요

### 회원 도메인 (MMBR)

- **MMBR-LGC**: 회원·멤버십 FO/API를 Spring MVC WAR로 서빙. JSP/Tiles, 세션·쿠키 기반 로그인. ssg-member-webapp, ssg-mmember-webapp, ssg-api-member-webapp, ssg-fido-api-webapp, ssg-batch-member-app.
- **MMBR-MSA**: 회원 query/command, 멤버십 query/command, alln, consumer, 배치를 Spring Boot 서비스로 분리해 K8s에서 운영. CQRS·도메인 분리.
- **트래픽**: FO/Cloud API → 회원·멤버십 MSA 또는 LGC. 배치는 Azkaban + Kafka·Oracle 연동.

### 전시 도메인 (MSA·FRONT)

- **MSA/ssg-dp.*** : 전시(DP) API·배치·컨슈머. 코너, 플랜샵, 서비스샵, 브랜드스토어, 카테고리 인덱싱, API 접근 로그·트리립 로그 수집 등. Spring Boot(Gradle), MongoDB·RabbitMQ·Elasticsearch·Kafka 연동.
- **FRONT**: 채널 웹앱(ssg-ssgmall, ssg-grocery, ssg-department, ssg-small 등)이 전시·회원 등 여러 도메인 API를 소비. 전시 전용 API 클라이언트는 **ssg-dp.front.api-webapp**(context-path 예: /dp/api)으로 제공.
- **트래픽**: FO/클라이언트 → Cloud API 또는 /dp/api → 전시 MSA(ssg-dp.*). 설정·이벤트는 RabbitMQ·Config로 전파.

### 공통·BFF

- **Cloud API**: 회원·전시·주문·결제 등 **여러 도메인**을 하나의 BFF로 라우팅. 회원은 MMBR-MSA/LGC, 전시는 MSA/ssg-dp.* 로 라우팅된다.
- 기술 선택은 “기존 LGC 호환·점진적 MSA 전환·운영 통일(모니터링·설정)”을 함께 만족하도록 되어 있습니다.

---

## 1. 클라이언트 / UI

### 사용처·기술 요약

| 구분 | 기술 | 사용처 | 도메인 |
|------|------|--------|--------|
| **프론트 웹** | JSP, Apache Tiles | FRONT: ssg-ssgmall-webapp, ssg-small-webapp, ssg-grocery-webapp, ssg-department-webapp, ssg-mgrocery-webapp, ssg-front.mnweb-webapp, ssg-resource.front-webapp, ssg-mapi-webapp 등 | 채널 공통(회원·전시 모두 소비) |
| **UI 영역** | React (react-area) | 메인/유닛 섹션(히어로배너, 스마일클럽, 카테고리 등) – JSP 내 react-area로 React 컴포넌트 로드 | 채널 공통 |
| **백오피스/리소스** | JSP, jQuery, JUI (Grid/Chart/Core) | BACK: ssg-bo-webapp, ssg-po-webapp, ssg-resource-webapp – CP/배치/관리 화면 | 공통 |
| **전시 API 진입** | Spring Boot, context-path | **FRONT/ssg-dp.front.api-webapp** – context-path 예: /dp/api. 전시(코너·플랜샵·카테고리 등) API 제공 | **전시** |
| **회원 API 진입** | 서블릿, context-path | MMBR-LGC 웹앱, MMBR-MSA 서비스 – /member/consumer 등 | **회원** |

### 내용 설명

- **JSP + Tiles**: 서버에서 HTML 조각을 조합해 페이지를 만드는 전형적인 MVC 뷰 계층. Tiles로 공통 레이아웃(헤더/푸터), 메인/유닛별 JSP를 재사용해 채널(SSG몰·이마트·백화점·소형몰 등)별로 다른 화면을 유지보수하기 쉽게 구성한다. **FRONT** 채널 웹앱(ssg-ssgmall, ssg-grocery, ssg-department 등)이 **회원(MMBR)**·**전시(DP)** API를 모두 소비한다.
- **React(react-area)**: 새로 만든 메인/카테고리/배너 등 인터랙티브한 구간만 React 컴포넌트로 구현하고, 기존 JSP 페이지 안에 `react-area`로 삽입한다. 전체를 SPA로 갈아타지 않고 **점진적으로 프론트 현대화**하는 전략이다.
- **전시 API(FRONT/ssg-dp.front.api-webapp)**: **전시 도메인** 전용 API. context-path 예: `/dp/api`. 코너·플랜샵·카테고리 등 전시 데이터를 채널 웹앱에 제공한다. 회원 API(/member/*)와 분리된 **도메인별 진입점**이다.
- **BO/PO JUI**: 그리드·차트·폼이 많은 관리자 화면에는 JUI(Grid/Chart/Core)로 데이터 테이블·엑셀 연동·차트를 통일해 개발 효율과 UI 일관성을 맞춘다.
- **context-path**: 서블릿 컨텍스트별로 앱을 나누어 `/dp/api`(전시), `/member/consumer`(회원) 등으로 진입점을 분리한다. 같은 인프라 위에 여러 “앱”을 올리는 구조다.

### 시스템 디자인과의 연관

- **채널별 웹앱 분리**: SSG몰·이마트·백화점·소형몰·기프트 등 **채널 = 별도 웹앱(프로젝트)** 로 두어, 도메인은 공유하되 화면·URL·배포 단위를 분리한 **멀티 채널 설계**다.
- **BFF와의 역할 분리**: UI는 “어떤 API를 부를지”만 알고, **실제 서비스 조합·인증·라우팅은 Cloud API(BFF)** 에 맡긴다. 클라이언트는 Cloud API 한 진입점만 알면 되므로 클라이언트 복잡도와 서버 변경 영향이 줄어든다.
- **점진적 SPA 도입**: 전면 SPA 전환 대신 react-area로 **섹션 단위 컴포넌트화**를 선택해, 기존 세션·쿠키·JSP 라이프사이클과 공존하면서 점진적으로 UI를 개선하는 설계다.

---

## 2. API 게이트웨이

### 사용처·기술 요약

| 구분 | 기술 | 사용처 | 도메인 |
|------|------|--------|--------|
| **MSA 인그레스(회원)** | Istio Gateway + VirtualService | **MMBR-MSA**: ssg-member-query, ssg-member-command, ssg-member-membership-query/command, ssg-member-alln-api-webapp – K8s 네임스페이스별 `istio: ingressgateway` | **회원** |
| **전시 MSA** | Spring Boot, context-path / Cloud API | **MSA/ssg-dp.*** (front, corner, planshop 등) – /dp/api 등. Cloud API가 전시(dp) 라우팅 | **전시** |
| **통합 API (BFF)** | Cloud API (qa-cloudapi.ssgadm.com / dev-cloudapi.ssgadm.com) | 회원(member/*), 전시(dp/*), 주문(order), 결제(paymt), 프로모션, FDS, BO, 아이템 MSA 등 | 공통 |
| **외부 노출** | stg-api.ssg.com / api.ssg.com 계열 | 회원/멤버십 등 외부 공개 API (Cloud API를 통한 노출) | **회원** 등 |
| **LGC 연동** | HttpInvoker (PaymentGateway) | **MMBR-LGC**: PG 결제 연동 – paymentGateway, paymentGatewayExt | **회원** |

### 내용 설명

- **Istio Gateway(회원)**: **MMBR-MSA** 서비스만 K8s Istio로 노출. 호스트/URI prefix로 회원 query/command, 멤버십, alln 등을 라우팅하고, DestinationRule으로 circuit breaker 적용.
- **전시 MSA**: **MSA/ssg-dp.*** (front, corner, planshop 등)는 Cloud API의 /dp/* 라우팅 또는 자체 context-path(/dp/api 등)로 노출. **FRONT/ssg-dp.front.api-webapp**이 전시 API 진입점 역할을 할 수 있다.
- **Cloud API**: **회원**(member/*), **전시**(dp/*), 주문·결제·프로모션·아이템 등 여러 도메인을 하나의 BFF로 라우팅. 클라이언트는 Cloud API만 호출하고, 내부에서 MMBR-MSA·LGC 또는 전시 MSA로 분기한다.
- **외부 API(api.ssg.com)**: 회원/멤버십 등 외부 공개 API는 Cloud API를 stg-api/api 도메인으로 노출. **회원 도메인** 위주.
- **HttpInvoker**: **MMBR-LGC** 웹앱이 PG(결제 게이트웨이)와 통신할 때 사용. **회원** 결제 연동.

### 시스템 디자인과의 연관

- **단일 진입점(BFF)**: FO·BO·모바일·파트너가 **Cloud API 하나**만 알면 되므로, 서비스 추가·라우트 변경·인증 정책 변경을 BFF에서만 반영할 수 있다. **라우팅·인증·버전 관리**를 중앙에서 처리하는 게이트웨이 패턴이다.
- **MSA와 LGC 이원화**: MSA는 Istio로 K8s 내부 트래픽을 제어하고, LGC는 Cloud API가 HTTP로 호출하거나(회원/멤버십 MSA URL 설정) LGC 자체가 직접 호출되는 혼합 구조다. **점진적 MSA 전환** 시 기존 LGC와 새 MSA가 동시에 살아있도록 하는 설계다.
- **장애 전파 완화**: Istio DestinationRule의 outlierDetection으로 한 서비스의 실패가 다른 서비스로 과도하게 퍼지지 않도록 **회로 차단**을 둔다.

---

## 3. 애플리케이션(백엔드)

### 사용처·기술 요약

| 구분 | 기술 | 사용처 | 도메인 |
|------|------|--------|--------|
| **레거시 (LGC)** | Spring MVC, WAR, Maven | **MMBR-LGC**: ssg-member-webapp, ssg-mmember-webapp, ssg-api-member-webapp, ssg-fido-api-webapp – 서블릿 기반 웹앱 | **회원** |
| **MSA (Spring Boot)** | Spring Boot, Gradle | **MMBR-MSA**: ssg-member-query/command-api-webapp, ssg-member-membership-query/command-api-webapp, ssg-member-alln-api-webapp, ssg-member-consumer-api-webapp | **회원** |
| **배치(회원)** | Azkaban + Shell, Spring Batch | **MMBR-LGC** ssg-batch-member-app (batchExcutor.sh), **MMBR-MSA** ssg-member-batch – 등급 산정, 쿠폰 적재, 어뷰징, 멤버십 동기화 등 | **회원** |
| **전시 MSA** | Spring Boot, Gradle(Kotlin 일부) | **MSA/ssg-dp.*** : ssg-dp.module, ssg-dp.batch, ssg-dp-corner, ssg-dp.planshop, ssg-dp.service-shop, ssg-dp.brandstore-mvc, ssg-dp.internal-verifier, ssg-dp.front.api-webapp, ssg-dp.consumer.api-webapp – 전시·코너·플랜샵·카테고리·로그 수집 | **전시** |
| **전시 API(FRONT)** | Spring Boot | **FRONT/ssg-dp.front.api-webapp** – 전시 API 제공(context-path 예: /dp/api). 채널 웹앱이 전시 데이터 조회 시 사용 | **전시** |
| **설정** | Spring Cloud Config | cloud-config.xml: spring.cloud.config (mmember, member, ssg.front.all 등) – Config Server 연동 | 공통 |

### 내용 설명

**회원 도메인 (MMBR)**  
- **LGC(Spring MVC WAR)**: 회원·멤버십 FO/API를 하나의 웹앱으로 서빙. 요청 → 컨트롤러 → 서비스 → MyBatis → Oracle, 세션·쿠키 로그인. WAR로 WAS(Tomcat 등)에 배포.  
- **MMBR-MSA(Spring Boot)**: 회원 query/command, 멤버십 query/command, 제휴(alln), 컨슈머를 서비스별로 분리. CQRS·도메인 분리, K8s에서 독립 배포·스케일링.  
- **배치**: 등급 산정, 쿠폰 적재, 어뷰징 알림, 멤버십 동기화 등은 **MMBR-LGC** ssg-batch-member-app(batchExcutor.sh) 또는 **MMBR-MSA** ssg-member-batch 로 실행. Kafka·Oracle·Elasticsearch 연동.

**전시 도메인 (MSA·FRONT)**  
- **MSA/ssg-dp.*** : 전시(Display)·코너·플랜샵·브랜드스토어·서비스샵·카테고리 등 API·배치·컨슈머. Spring Boot(Gradle), MongoDB·RabbitMQ·Elasticsearch·Kafka 사용. 카테고리 인덱싱, API 접근 로그·트리립 로그 수집은 **ssg-dp.consumer**, **ssg-dp.batch** 등에서 수행.  
- **FRONT/ssg-dp.front.api-webapp**: 전시 데이터를 채널(ssgmall·grocery 등)에 제공하는 API 서버. context-path 예: /dp/api. 전시 도메인 전용 “프론트(진입) API” 역할.

**공통**  
- **Spring Cloud Config**: DB URL, Redis, 외부 API URL 등 설정을 Config Server에 두고, 앱 기동 시 가져옴. 회원(mmember, member)·전시(ssg.front.all 등) 도메인 모두 사용.

### 시스템 디자인과의 연관

- **CQRS·도메인 분리**: 회원 “조회”와 “변경”을 query/command 서비스로 나눠 **읽기 부하 분리**와 **변경 모델 단순화**를 꾀한다. 멤버십·제휴(alln)도 동일하게 서비스 단위로 나누어 **도메인 경계**를 명확히 한다.
- **배치와 온라인 분리**: 대량 작업은 Azkaban·배치 전용 앱에서만 수행하고, 온라인 트랜잭션 서버와 분리해 **리소스 경합**과 **장애 전파**를 줄인다. 배치 결과는 Kafka·DB를 통해 온라인에서 활용한다.
- **설정과 코드 분리**: Config Server로 환경별 설정을 관리하면 빌드 산출물 하나로 여러 환경에 배포할 수 있고, **12 Factor**의 설정 외부화에 부합한다.

---

## 4. 인증 / 세션

### 사용처·기술 요약

| 구분 | 기술 | 사용처 | 도메인 |
|------|------|--------|--------|
| **FO 로그인·세션** | Cookie 기반 세션, MbrSessionService, SsgCookieHandler | **MMBR-LGC**: 로그인 후 세션/쿠키로 사용자 유지 (FrontUserInfo, ssgLoginService.getUser) | **회원** |
| **SSO** | SSO 토큰(v2 88자), SsoApiService.checkToken, Blossom SSO | **MMBR-LGC**: SsoTokenLoginController, SsoInterceptorAdapter – 사내 Blossom SSO 및 SSG SSO 토큰 검증 | **회원** |
| **BO/PO/리소스 로그인** | DB 기반 로그인(MyBatis), cmSessionService, SessionKey | BACK: LoginController, Login.xml – sysDivCd별 로그인 뷰 분기 | 공통 |
| **FIDO** | FIDO RP (WebAuthn) | **MMBR-LGC**: ssg-fido-api-webapp – 생체/보조인증 | **회원** |
| **JWT** | dp-framework api-config (jwt-token-parse-enabled) | **MMBR-MSA** consumer, **전시** MSA(ssg-dp.*) – API 간 인증·파트너 API 시 옵션 사용 | **회원**·**전시** |
| **AD 연동** | cloudapi.ad.oracle.url | BO: AD(Active Directory) Oracle 연동 URL로 인증 연동 | 공통 |

### 내용 설명

- **FO 세션**: 고객이 로그인하면 서버 세션과 쿠키(예: pcId, fsId)에 사용자 정보를 두고, 이후 요청마다 `ssgLoginService.getUser(request, response)` 로 로그인 여부·회원 정보를 판단한다. **상태 유지(Stateful)** 방식으로, LGC 웹앱과 단일 도메인 내에서 동작한다.
- **SSO**: 다른 채널(신세계그룹 사이트·Blossom 등)에서 넘어올 때 **SSO 토큰(88자 v2)** 을 쿼리로 받아, SsoApiService로 검증 후 동일 사용자면 로그인 처리한다. Blossom SSO는 사내 임직원 인증용으로 별도 플로우를 탄다.
- **BO/PO 로그인**: 관리자·파트너는 DB에 저장된 계정으로 로그인한다. MyBatis로 사용자 조회·실패 횟수 등을 관리하고, sysDivCd(BO/PO/CSO/AD 등)에 따라 로그인 화면·권한을 나눈다. cmSessionService·SessionKey로 세션에 사용자/역할을 넣는다.
- **FIDO**: 비밀번호 대체·2차 인증으로 FIDO2/WebAuthn을 쓰기 위한 RP( Relying Party) 서버를 ssg-fido-api-webapp이 담당한다.
- **JWT**: MSA·컨슈머에서 API 간 인증이 필요할 때 dp-framework의 jwt-token-parse 옵션으로 토큰을 파싱해 사용한다. FO 세션과 별도로 **서비스 간 통신·파트너 API** 에 활용할 수 있는 구조다.
- **AD**: BO 일부는 Active Directory와 연동해 사내 계정으로 로그인할 수 있도록 Cloud API의 AD Oracle URL을 사용한다.

### 시스템 디자인과의 연관

- **진입 채널별 인증 분리**: FO는 **쿠키 세션 + SSO 토큰**, BO/PO는 **DB 계정 + AD**, API는 **JWT** 등 **채널·클라이언트 유형별로 적합한 인증 방식**을 선택해, 하나의 “통합 로그인”에 묶지 않고 도메인별로 단순하게 유지한다.
- **SSO와 토큰 검증**: 외부 유입 트래픽은 **토큰 한 번 검증**으로 신뢰하고, 내부에서는 세션으로만 상태를 유지한다. 토큰 검증을 한 서비스(또는 BFF)에 집중하면 **인증 경계**가 명확해진다.
- **FIDO 분리 배포**: 인증 수단을 별도 웹앱(ssg-fido-api-webapp)으로 두어 **보안·인증 정책 변경** 시 회원/결제 도메인과 분리해 배포·감사할 수 있게 한다.

---

## 5. DB

### 사용처·기술 요약

| 구분 | 기술 | 사용처 | 도메인 |
|------|------|--------|--------|
| **RDBMS** | Oracle | **MMBR-LGC** resources*.xml (db.url), **MMBR-MSA** application*.yml (datasource), BACK | **회원**·공통 |
| **접근 방식** | MyBatis (Mapper XML) | BACK, **MMBR-LGC**, **MMBR-MSA**, **MSA/ssg-dp.*** – *Mapper.xml, mapper-location | **회원**·**전시**·공통 |
| **연결 풀** | HikariCP (Spring Boot), 전통 DataSource (LGC) | MMBR-MSA·전시 MSA: HikariCP / MMBR-LGC: db.url 등 | **회원**·**전시** |
| **로깅** | log4jdbc | **MMBR-MSA** 배치/컨슈머: jdbc:log4jdbc:oracle:thin – SQL 로깅 | **회원** |
| **NoSQL** | MongoDB | **MMBR-MSA** consumer, **전시** MSA/ssg-dp.consumer·FRONT dp (devdispdb) – 설정·비정형 데이터, RabbitMQ와 연동 | **회원**·**전시** |

### 내용 설명

- **Oracle**: 회원·멤버십·주문·정산 등 **트랜잭션 데이터**의 주 저장소다. LGC는 resources에 db.url로, MSA는 datasource jdbc-url로 환경별 인스턴스에 접속한다. 기존 사내 표준 RDB를 그대로 활용하는 선택이다.
- **MyBatis**: SQL을 XML 매퍼에 두고 Java와 매핑한다. 복잡한 쿼리·조건 분기·대량 처리에 유리하고, LGC·BACK·MSA 전반에서 동일한 패턴을 써 **운영·이관** 시에도 익숙한 구조를 유지한다.
- **HikariCP**: Spring Boot 기본 연결 풀이다. maximumPoolSize·minimumIdle·maxLifetime으로 연결 수와 유휴 시간을 제한해 **리소스 사용**과 **DB 부하**를 조절한다.
- **log4jdbc**: JDBC URL을 log4jdbc로 감싸 실행 SQL·바인드 값을 로그로 남긴다. 배치·컨슈머에서 **디버깅·감사**용으로 사용한다.
- **MongoDB**: **MMBR-MSA** consumer, **전시** MSA(ssg-dp.consumer)·FRONT dp에서 **설정·비정형 데이터·캐시성 데이터** 저장. RabbitMQ·Config와 함께 이벤트/설정 저장소로 쓰인다. 전시 도메인에서 devdispdb 등으로 사용.

### 시스템 디자인과의 연관

- **단일 RDB 도메인 공유**: 회원·멤버십이 같은 Oracle 스키마(또는 DB)를 쓰면 **트랜잭션 일관성**과 **조인·배치**가 쉽다. 서비스를 MSA로 나눠도 **데이터는 단계적으로 분리**하는 전략과 맞는다.
- **쓰기/읽기 분리(Redis)**: DB는 “단일 소스 of truth”로 두고, **읽기 부하**는 Redis 캐시와 Read Replica(설정 시)로 줄이는 구조를 전제로 한다. DB 절은 **일관성·쓰기**에 집중한다.
- **NoSQL 역할 분리**: MongoDB는 **트랜잭션 핵심**이 아닌 설정·로그·전시 데이터에만 쓰여, Oracle과 **역할이 분리**된다. 도메인별로 저장소를 나누는 **폴리글랏 퍼시스턴스**에 가깝다.

---

## 6. 캐시

### 사용처·기술 요약

| 구분 | 기술 | 사용처 | 도메인 |
|------|------|--------|--------|
| **캐시 서버** | Redis | **MMBR-LGC**: redis.host, write/read port 분리. **MMBR-MSA**: redis (application.yml). **전시** MSA: cache-config(standalone/cluster) | **회원**·**전시** |
| **클라이언트** | Spring Data Redis, redis.clients (Jedis 등) | MMBR-LGC pom, MMBR-MSA application.yml, 전시 consumer cache-config | **회원**·**전시** |
| **운영** | Redis Standalone / Cluster | dev: standalone, prod: cluster (회원 redis.member.ssgadm.com 등) | **회원** 위주(전시는 설정에 따라) |

### 내용 설명

- **Redis**: 세션·공통 코드·조회 결과 등 **휘발성·고속 조회**용 데이터를 메모리에 둔다. LGC는 write port / read port 를 분리해 **쓰기 한 곳, 읽기 여러 곳**으로 부하를 나눈다.
- **Spring Data Redis**: Redis에 접근할 때 추상화 레이어를 두어 키/값 직렬화·연결 관리를 통일한다. LGC·MSA 모두 같은 클라이언트 스택을 써 **운영·모니터링**을 맞춘다.
- **Standalone vs Cluster**: 개발/QA는 단일 인스턴스(standalone), 프로덕션은 redis-topology-mode: cluster 로 **가용성·확장성**을 확보한다. 회원 도메인은 redis.member.ssgadm.com 으로 전용 클러스터를 쓰는 구성이다.

### 시스템 디자인과의 연관

- **DB 부하 감소**: 자주 조회되는 코드·회원 요약·세션을 Redis에 두어 **DB 쿼리 수**를 줄이고 **응답 시간**을 낮춘다. 읽기 비중이 큰 서비스에서 **캐시 레이어**로 쓰는 전형적인 패턴이다.
- **세션 저장소**: FO 세션을 서버 메모리가 아닌 Redis에 두면 **여러 인스턴스가 세션을 공유**할 수 있어, 수평 확장 시에도 로그인 상태가 유지된다. (실제 FO 세션 저장 위치는 구현에 따라 서버 메모리일 수 있으나, Redis는 그에 적합한 저장소다.)
- **환경별 토폴로지**: dev는 단일, prod는 cluster 로 **비용과 안정성**을 환경별로 다르게 가져가는 설계다.

---

## 7. 메시지 / 큐

### 사용처·기술 요약

| 구분 | 기술 | 사용처 | 도메인 |
|------|------|--------|--------|
| **메시지 브로커** | Kafka | **MMBR-MSA** ssg-member-batch(프로듀서), consumer code-consumer – 멤버십 쿠폰 적재, 코드/설정 동기화. BACK BO. **전시** MSA(ssg-dp.consumer 등) – 이벤트/로그 수집 | **회원**·**전시**·공통 |
| **Kafka 클라이언트** | kafka-clients, Spring Kafka | **MMBR-MSA** 배치·컨슈머, **전시** ssg-dp.consumer·batch – key/value String 직렬화 | **회원**·**전시** |
| **AMQP** | RabbitMQ | **MMBR-MSA** consumer, **전시** MSA/FRONT dp: spring.rabbitmq (virtual-host: config) – 설정/이벤트 전달, Spring Cloud Config 연동 | **회원**·**전시** |

### 내용 설명

- **Kafka**: 대량 이벤트·배치 결과를 **토픽 단위**로 쌓고, 여러 컨슈머가 구독한다. 회원 배치에서는 멤버십 쿠폰 대상 적재(MbrspCouponMonthLoadJob 등), 코드/설정 변경 전파에 사용한다. **로그성·이벤트 소싱**에 가깝게 쓰인다.
- **kafka-clients / Spring Kafka**: 프로듀서·컨슈머를 구현할 때 사용하며, key/value는 String 직렬화로 단순하게 둔다. 배치가 프로듀서, consumer-api가 컨슈머 역할을 한다.
- **RabbitMQ**: Spring Cloud Config와 연동해 **설정 변경 이벤트**를 푸시하거나, DP·컨슈머에서 **작업 큐·이벤트 전달**에 쓴다. virtual-host로 논리적 구분(config 등)을 한다. Kafka보다 **요청/응답·작업 큐**에 가깝게 쓰이는 경우가 많다.

### 시스템 디자인과의 연관

- **비동기·결합도 완화**: 배치가 “쿠폰 대상 적재” 결과를 Kafka에 넣고, 다른 서비스가 이를 구독해 처리하면 **동기 호출 없이** 데이터를 전달할 수 있다. **이벤트 드리븐** 구조로 서비스 간 결합을 낮춘다.
- **배치–온라인 경계**: 대량 적재·동기화는 배치가 Kafka에 적재하고, 온라인 서비스는 필요한 만큼 소비한다. **처리 지연**을 허용하는 흐름은 메시지 큐로 버퍼링하는 설계다.
- **Kafka vs RabbitMQ 역할**: Kafka는 **이벤트 로그·재생·대량 스트림**, RabbitMQ는 **설정 푸시·작업 큐**처럼 용도를 나누어, 각 브로커의 특성에 맞게 선택한 구조다.

---

## 8. 파일 / 스토리지

### 사용처·기술 요약

| 구분 | 기술 | 사용처 |
|------|------|--------|
| **로컬·NAS** | fileUpload.tempDir, fileUpload.upLoadDir (/resize/), fileUpload.upLoadDefaultUrl | BACK: resources – 파일 업로드 임시 디렉토리, NAS 경로, temp_up 등 |
| **이미지 도메인** | fileUpload.imageDomain, web.cdn.uccImgPath (succ.ssgcdn.com) | BO: 업로드 파일 URL, CDN 이미지 |
| **용도** | 업로드 파일(이미지/엑셀 샘플), 브랜드/업체 이미지 | web.jsp.brandImgPath, vendorImgPath, 샘플 엑셀 경로 등 |
| **비고** | S3/MinIO 등 오브젝트 스토리지 설정은 코드베이스 내에서 확인되지 않음 (NAS/로컬 위주) | - |

### 내용 설명

- **임시·업로드 경로**: 업로드 시 먼저 fileUpload.tempDir 에 올린 뒤, 검증·처리 후 fileUpload.upLoadDir (NAS /resize/) 와 upLoadDefaultUrl(temp_up/) 조합으로 최종 경로를 만든다. **임시 ↔ 영구** 구분으로 디스크 정리·권한 관리가 쉬워진다.
- **이미지·CDN**: 서빙 URL은 fileUpload.imageDomain 과 upLoadDefaultUrl 로 구성하고, UCC 이미지는 web.cdn.uccImgPath(succ.ssgcdn.com) 로 CDN을 탄다. **정적 자원**은 앱 서버가 아닌 NAS·CDN에서 제공해 **애플리케이션 부하**를 줄인다.
- **용도**: 브랜드/업체 이미지, 샘플 엑셀(일괄등록용) 등 **관리자 업로드 파일**이 주 사용처다. 코드베이스 상에는 S3/MinIO 같은 오브젝트 스토리지 설정은 보이지 않고, NAS·로컬 위주다.

### 시스템 디자인과의 연관

- **앱과 스토리지 분리**: 파일 내용은 NAS에 두고 앱은 **경로·URL만** 관리한다. 여러 WAS가 같은 NAS를 바라보면 **공유 스토리지**로 확장할 수 있고, 나중에 오브젝트 스토리지로 이전할 때도 “경로/URL 정책”만 맞추면 된다.
- **CDN으로 읽기 부하 이전**: 이미지 트래픽을 CDN으로 보내 **원본 서버·네트워크 부하**를 줄이는 전형적인 **정적 자산 오프로드** 설계다.
- **일괄 작업과 파일**: 엑셀 샘플·일괄 업로드 파일은 “업로드 → 배치/배치 유사 처리” 흐름과 맞닿아 있어, **파일 업로드 경로**가 배치·관리자 워크플로와 함께 설계된다.

---

## 9. 검색 (전문 검색, 자동완성)

### 사용처·기술 요약

| 구분 | 기술 | 사용처 | 도메인 |
|------|------|--------|--------|
| **엔진** | Elasticsearch | BACK (BO/PO 공지·호텔). **MMBR-LGC** 배치(Zipcd). **MMBR-MSA** ssg-member-batch. **전시** MSA ssg-dp.batch / ssg-dp.consumer – 카테고리 인덱싱, API 접근 로그, 트리립 로그 | **회원**·**전시**·공통 |
| **클라이언트** | Transport Client (구), RestHighLevelClient (7.x) | **MMBR-LGC** 배치: transport. **전시** MSA: elasticsearch-rest-high-level-client (CategoryStaticIndexingConfiguration, AccessLogConsumer, TriipLogConsumer) | **회원**·**전시** |
| **인덱스/용도** | ssgpt_search_core, ssgpt_search (공지 등), 카테고리 인덱싱, API 접근 로그, 트리립 로그 | BACK: NoticeEsService, RsvtService(호텔). **전시** MSA: CategoryStaticIndexingConfiguration, AccessLogConsumer, TriipLogConsumer | 공통·**전시** |
| **Spring Data ES** | spring-boot-starter-data-elasticsearch | **MMBR-MSA** ssg-member-batch (build.gradle) | **회원** |
| **자동완성** | 전용 검색 API 미확인. HTML input autocomplete 속성 수준 다수 | FO/BO 폼 (autocomplete="off" 등) | 공통 |

### 내용 설명

- **Elasticsearch**: **공통** BO/PO 공지 검색(ssgpt_search_core), 호텔 예약 검색. **회원** MMBR-LGC 배치 Zipcd 검색·이관, MMBR-MSA 배치. **전시** MSA(ssg-dp.batch, ssg-dp.consumer)에서 카테고리 인덱싱(CategoryStaticIndexingConfiguration), API 접근 로그(AccessLogConsumer), 트리립 로그(TriipLogConsumer) 적재에 사용한다.
- **클라이언트**: LGC 배치는 구 Transport Client, MSA는 RestHighLevelClient(7.x)를 쓴다. 버전 차이는 레거시 호환과 점진적 업그레이드 때문이다.
- **자동완성**: 상품/주소 등 **전문 검색 기반 자동완성 API**는 코드베이스에서 별도로 보이지 않고, 폼에서는 HTML autocomplete 속성(주로 off) 수준만 확인된다. 자동완성 기능이 있다면 Cloud API·다른 서비스에서 제공할 수 있다.

### 시스템 디자인과의 연관

- **RDB와 검색 엔진 분리**: “검색·필터·집계”는 Elasticsearch에 맡기고, “트랜잭션·정규 데이터”는 Oracle에 둔다. **역할 분리**로 RDB 부하를 줄이고 검색 성능을 확보하는 **검색 전용 레이어** 설계다.
- **로그·이벤트 저장소**: API 접근 로그·트리립 로그를 Kafka 등에서 받아 ES에 적재하면 **중앙 로그 검색·분석**이 가능하다. 로그는 **비동기·대량**이므로 메시지 큐 → ES 파이프라인과 잘 맞는다.
- **배치와 인덱스**: Zipcd·카테고리처럼 **마스터/참조 데이터**를 배치가 ES에 인덱싱해 두고, 온라인은 ES만 조회한다. **쓰기는 배치, 읽기는 검색 엔진**으로 흐름을 나누는 패턴이다.

---

## 10. 모니터링 / 로깅

### 사용처·기술 요약

| 구분 | 기술 | 사용처 | 도메인 |
|------|------|--------|--------|
| **메트릭** | Prometheus, Pushgateway | **MMBR-LGC** monitoring.pushGateway. **MMBR-MSA** management.endpoints, prometheus.metrics.export.pushgateway. **전시** MSA도 동일 스택 사용 가능 | **회원**·**전시** |
| **푸시** | simpleclient_pushgateway | **MMBR-LGC** pom (member-webapp, mmember-webapp), **MMBR-MSA** (membership-command/query, member-query 등) | **회원** |
| **에러 추적** | Sentry | **MMBR-MSA** sentry.dsn. **전시** FRONT/ssg-dp.front 등: Sentry DSN – 예외/에러 수집 | **회원**·**전시** |
| **로깅** | Logback, SLF4J | 전역 – logback-spring.xml (MSA), Spring Boot 기본 로깅 | 공통 |
| **ES 로그 수집** | logback-elasticsearch-appender | **전시** MSA ssg-dp.consumer.api-webapp – 로그 → Elasticsearch 적재 | **전시** |
| **트래픽 제어** | Istio DestinationRule (outlierDetection) | **MMBR-MSA** ingress – circuit breaker. (전시 MSA도 Istio 사용 시 동일) | **회원**·**전시** |

### 내용 설명

- **Prometheus + Pushgateway**: 앱이 메트릭을 **푸시**하는 방식으로, 단발성·배치·내부망 서비스도 Prometheus가 수집할 수 있게 한다. LGC·MSA 모두 동일한 Pushgateway 주소를 써 **메트릭 수집 방식**을 통일한다.
- **Sentry**: 예외가 발생하면 스택 트레이스·컨텍스트를 Sentry로 보내 **에러 추적·알림**을 한다. MSA·DP 프론트에서 사용해 **분산 환경에서도 에러를 한곳에서** 모은다.
- **Logback / SLF4J**: 애플리케이션 로그의 표준 출력이다. MSA는 logback-spring.xml 로 프로파일별로 로그 포맷·출력 대상을 나눌 수 있다.
- **logback-elasticsearch-appender**: 로그 이벤트를 Elasticsearch에 직접 넣어 **중앙 로그 검색** 파이프라인을 만든다. **전시** MSA ssg-dp.consumer.api-webapp에서 사용한다.
- **Istio outlierDetection**: 특정 파드가 연속 실패하면 일정 시간 트래픽에서 제외한다. **회로 차단**으로 장애가 있는 인스턴스로의 요청을 줄여 **가용성**을 지킨다.

### 시스템 디자인과의 연관

- **관찰 가능성(Observability)**: 메트릭(Prometheus)·로그(Logback/ES)·에러(Sentry)를 한 세트로 두어 **분산 MSA에서도 동작을 추적**할 수 있게 한다. LGC와 MSA가 같은 Pushgateway를 쓰면 **통합 대시보드·알람**을 만들기 쉽다.
- **푸시 모델**: Pull이 어려운 환경(방화벽·단발성 작업)에서는 Pushgateway로 **메트릭만 푸시**하는 선택이 맞다. 배치·레거시 웹앱까지 같은 메트릭 스택에 포함시키는 설계다.
- **장애 격리와 모니터링**: Istio로 **트래픽 제어**를 하고, Prometheus/Sentry로 **원인 분석**을 하는 구조다. “어디서 실패했는지”와 “실패한 인스턴스를 트래픽에서 뺐는지”를 함께 보면 **복원력** 설계와 **운영 대응**이 연결된다.

---

## 요약 매트릭스

| 영역 | 주요 기술 |
|------|-----------|
| 클라이언트/UI | JSP, Tiles, React(일부), jQuery, JUI |
| API 게이트웨이 | Istio Gateway, Cloud API (BFF), HttpInvoker(PG) |
| 백엔드 | Spring MVC(WAR), Spring Boot(MSA), Azkaban+Shell 배치, Spring Cloud Config |
| 인증/세션 | Cookie 세션, SSO 토큰, Blossom SSO, MyBatis 로그인, FIDO, JWT(선택) |
| DB | Oracle, MyBatis, HikariCP, MongoDB(일부) |
| 캐시 | Redis (Standalone/Cluster), Spring Data Redis |
| 메시지/큐 | Kafka, RabbitMQ |
| 파일/스토리지 | NAS(/resize/), 로컬 임시 디렉토리, CDN(이미지) |
| 검색 | Elasticsearch (전문 검색, 로그/인덱싱), 자동완성은 HTML/폼 수준 |
| 모니터링/로깅 | Prometheus, Pushgateway, Sentry, Logback, ES Appender |

---

## 시스템 디자인 관점 요약

- **도메인 분리**: **회원 도메인(MMBR)** 은 MMBR-LGC(레거시)·MMBR-MSA(회원/멤버십/alln/컨슈머/배치)만 해당한다. **전시 도메인**은 **MSA/ssg-dp.*** (전시·코너·플랜샵·컨슈머 등)와 **FRONT/ssg-dp.front.api-webapp**(전시 API)로 구분된다. Cloud API(BFF)가 두 도메인을 모두 라우팅한다.
- **하이브리드 아키텍처**: 회원은 LGC(모놀리식)와 MSA 공존, 전시는 MSA(Spring Boot) 위주. BFF가 단일 진입점으로 회원·전시·주문·결제 등을 라우팅하고, 백엔드는 서비스 단위로 배포·확장한다.
- **데이터 계층 분리**: **Oracle**은 회원·전시·공통 트랜잭션, **Redis**는 캐시·세션(회원 위주), **Elasticsearch**는 검색·로그(전시 카테고리/접근 로그·회원 배치 Zipcd 등), **MongoDB**는 전시·컨슈머 설정·비정형 데이터로 역할을 나눈다.
- **비동기·이벤트**: **Kafka**는 회원 배치·컨슈머와 전시 컨슈머에서 사용. **RabbitMQ**는 회원·전시 모두 Config·작업 큐 연동에 사용한다.
- **관찰 가능성·복원력**: **Prometheus/Sentry/Logback**은 회원·전시 공통. **ES 로그 수집**은 전시 ssg-dp.consumer에서 사용. **Istio**는 회원 MMBR-MSA(및 전시 MSA 사용 시)에서 회로 차단·라우팅을 담당한다.
- **점진적 전환**: JSP+React 혼용, 회원 LGC와 MSA 공존, 전시는 MSA·FRONT로 제공. Config/모니터링은 도메인 공통으로 두어 **기존 시스템을 유지하면서** 채널 확장·도메인별 배포를 단계적으로 진행하는 설계다.

---

작성일: 2025-02-27  
기준: IdeaProjects 코드베이스  
- **회원 도메인**: MMBR-LGC, MMBR-MSA (MMBR- 접두사 프로젝트만)  
- **전시 도메인**: MSA/ssg-dp.*, FRONT/ssg-dp.front.api-webapp  
- **공통**: BACK(BO/PO/리소스), Cloud API, 채널 FRONT(ssg-ssgmall·grocery·department 등)
