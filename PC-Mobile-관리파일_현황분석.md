# AddCon PC/MO 화면 통합 분석 보고서

> 조사 대상: `addcon-svc` (사용자 사이트), `addcon-admin` (관리자 사이트)
> 조사 방법: 소스코드 정적 분석 (JSP/JS 디렉터리 비교, Controller 조건분기 grep)
> 결론 요약: **addcon-admin은 PC/MO 분리가 없음(관리자 전용, PC 단일)**. 분리는 **addcon-svc에만 존재**하며, "동일 컨트롤러 + ViewResolver가 자동으로 `/mobile/` 하위 JSP로 위임" 하는 구조 + 컨트롤러 내부의 `accessDevice.isPC()` 조건분기로 로직 일부가 분기됨.

---

## 1단계 - 현황 파악

### 1-1. 분기 메커니즘 (요약)

- `UserAgentInterceptor` → `AccessDeviceHandler`가 **User-Agent 문자열을 파싱**하여 `AccessDevice`(PC/MOBILE/TABLET) 판별 후 `request`에 저장
- `AccessDeviceDelegatingViewResolver`가 View 이름 반환 시, 디바이스가 MOBILE/TABLET이면 **동일한 viewName 앞에 `/mobile/`을 자동으로 붙여** `WEB-INF/jsp/mobile/...` 로 위임
  → 즉 **컨트롤러 1개, JSP 2세트**(PC용 원본 + `mobile/` 미러 디렉터리)가 기본 패턴이며, `XxxMoController` 같은 별도 컨트롤러 클래스는 존재하지 않음(grep 결과 0건)
- 일부 컨트롤러는 `@ModelAttribute`처럼 주입되는 `AccessDevice accessDevice` 파라미터(`AccessDeviceArgumentResolver`)를 받아 **`accessDevice.isPC()` 조건분기로 로직 자체를 다르게 실행**(페이지당 건수, 리다이렉트 대상 뷰, channel 코드 등)

### 1-2. 전체 건수

| 구분 | PC | Mobile | 동일 경로(양쪽 존재) | Mobile 전용 | PC 전용(모바일 미대응) |
|---|---|---|---|---|---|
| JSP (addcon-svc) | 239 | 126 | **93** | 33 | 146 |
| JS (addcon-svc, static/js) | 133 | 30 | **10** | 20 | 123 |
| Controller 내 `accessDevice.isPC()` 분기 | - | - | **11개 파일 / 약 30곳** | - | - |

> "PC 전용(146/123)"은 모바일 미러가 없는 파일로, 관리 팝업·부분(include) 조각·PC 전용 정책 화면 등이 섞여 있어 **전부가 통합 대상은 아님**(모바일에서 애초에 접근하지 않는 화면 다수 포함).

### 1-3. 분리 화면 목록 (카테고리별 대표 샘플, 전체 93건 중 발췌)

| 화면 분류 | 건수 | PC 경로(예) | MO 경로(예) | 공통 컨트롤러 | 차이점 |
|---|---|---|---|---|---|
| 팝업(popup) | 22 | `jsp/popup/orderCertKakaoPop.jsp` | `jsp/mobile/popup/orderCertKakaoPop.jsp` | O (단일) | UI/레이아웃만 다름(팝업 vs 풀페이지 형태) |
| 마이페이지(mypage) | 13 | `jsp/mypage/myCoupon.jsp` (8.2KB) | `jsp/mobile/mypage/myCoupon.jsp` (7.0KB) | `MypageController` | UI 위주, 일부 리스트 rows/paging 차이 |
| 회원(user) | 11 | `jsp/user/joinStep1.jsp` (4.2KB) | `jsp/mobile/user/joinStep1.jsp` (2.9KB) | `UserController` | 입력 폼 레이아웃 단순화, 로직은 Controller 공용 |
| 공통(common) | 10 | `jsp/common/login.jsp` | `jsp/mobile/common/login.jsp` | `LoginController` | 로그인 로직 동일, Channel 값만 WEB/MOBILE_WEB로 분기 |
| 상품(goods) | 10 | `jsp/goods/goodsDetail.jsp` (14.8KB) | `jsp/mobile/goods/goodsDetail.jsp` (10.2KB) | `GoodsController` | **로직도 분기**: PC/MO 목록 건수 상수 다름(`GOODS_RECORD_CNT` vs `MOBILE_GOODS_RECORD_CNT`) |
| 콘텐츠(contents) | 9 | `jsp/contents/eventGeneral.jsp` | `jsp/mobile/contents/eventGeneral.jsp` | `ContentsController` | Channel 분기 + UI 차이 |
| 주문(order) | 6 | `jsp/order/orderResult.jsp` (9.1KB) | `jsp/mobile/order/orderResult.jsp` (7.4KB) | `OrderController` | **로직 분기 큼**: 결제수단/인증 방식별 뷰 분기, private account info 뷰 별도(`MOBILE_MYPAGE_ORDER_PRIVATE_ACCOUNT_INFO`) |
| include(공통조각) | 5 | `jsp/include/commonHeader.jsp` | `jsp/mobile/include/commonHeader.jsp` | - | 헤더/푸터/배너 UI 골격만 다름 |
| 혜택(benefit) | 4 | `jsp/benefit/gradeUp.jsp` | `jsp/mobile/benefit/gradeUp.jsp` | - | UI 위주 |
| 기타(intro, main, error) | 3 | `main.jsp` 등 | `mobile/main.jsp` 등 | - | UI 위주 |

**JS 이중 세트(10건)**: `contents-rouletteNew.js`, `special-shop.js`, `my-myCouponDetail.js`, `my-myDeposit.js`, `my-orderDetail.js`, `order-destPhoneList.js`, `order-kakaopay.js`, `pick-orderDetail.js`, `addr-pop.js` 등 — 대부분 DOM 셀렉터/이벤트 바인딩 대상만 다르고 로직은 유사.

**Mobile 전용(로직 자체가 모바일에만 존재, 33건)**: `order/orderPay.jsp`, `order/orderSend.jsp`, `order/partial/*`(카카오 프리뷰, 옵션/발송정보 부분 조각), `contents/external/puradak/*`(외부 이벤트 연동), `contents/eventOrderPay.jsp` 등 — **PC에는 대응 화면이 없어 통합이 아니라 "모바일 온리 기능"으로 유지 필요**.

---

## 2단계 - 구조 분석

### 2-1. 분기 방식

| 분기 방식 | 사용 여부 | 설명 |
|---|---|---|
| User-Agent 기반 디바이스 판별 | ✅ | `UserAgentInterceptor` → `AccessDeviceHandler` |
| ViewResolver 자동 위임(`/mobile/` prefix) | ✅ (주력 방식) | `AccessDeviceDelegatingViewResolver` — 컨트롤러 코드 변경 없이 JSP만 이원화 |
| 별도 URL 경로(`/mo/xxx`) | ❌ | 미사용 |
| 별도 Controller 클래스(`XxxMoController`) | ❌ | 미사용 (grep 0건) |
| 컨트롤러 내 `if (accessDevice.isPC())` 로직 분기 | ✅ | 11개 파일·약 30개 분기점 — 페이지네이션 건수, redirect view, channel 코드, 인증 채널(WEB/MOBILE_WEB) 등 |
| 별도 JS(jQuery) 세트 | ✅ (부분) | `static/js/mobile/` 하위 20개 모바일 전용 JS + 10개 동명 이원화 |

### 2-2. 유형 분류

| 유형 | 해당 건수(추정) | 특징 |
|---|---|---|
| **A. 단순 UI 차이 (반응형 흡수 가능)** | JSP 93건 중 약 70~75건 | 컨트롤러 완전 공유, JSP 레이아웃/CSS 클래스만 다름 (popup, include, benefit, 대부분 mypage/user) |
| **B. UI + 경미한 로직 차이 (분기 제거 필요)** | 약 15~18건 | `GoodsController`(페이지당 건수), `LoginController`/`ContentsController`(channel 값), `UserController`(channel) 등 — 조건문 제거하고 반응형 기준으로 통일 필요 |
| **C. 로직/데이터 흐름 자체가 다름 (설계 변경 필요)** | 약 5~8건 | `OrderController`/`OrderPayController`(결제 인증 팝업 vs 리다이렉트 전체 페이지 흐름 자체가 다름 — 카카오/카드 인증이 PC는 팝업, MO는 페이지 이동), `MyOrderController`/`MyDepositController`(PC 전용 표시 항목) |
| **D. Mobile 전용 신규 기능 (통합 대상 아님, 유지)** | 33건(JSP) + 20건(JS) | 외부 이벤트 연동(puradak), 모바일 결제 플로우 부분 조각 — PC에서 애초에 서비스하지 않음 |

---

## 3단계 - 통합 공수 산정

> 전제: "PC/MO 각각 별도 JSP를 유지하는 구조"를 **반응형 웹(단일 템플릿 + CSS 미디어쿼리/컴포넌트 분기)**으로 전환. 순수 개발 기준(디자인 시안, QA 제외), 1인일 = 8시간.

### 3-1. 유형별 공수

| 유형 | 대상 화면 수 | 화면당 공수(인일) | 소계(인일) | 근거 |
|---|---|---|---|---|
| A. 단순 UI 통합 | 72 | 0.5 ~ 1.0 | 36 ~ 72 | JSP 2개 → 1개 병합, CSS 미디어쿼리 적용, 퍼블리싱 재조립. 로직 변경 없음 |
| B. UI+경미 로직 통합 | 16 | 1.0 ~ 2.0 | 16 ~ 32 | 컨트롤러 `isPC()` 분기 제거/통합, 상수 통일(`GOODS_RECORD_CNT` 등), Channel 값 정책 재정의 |
| C. 로직 재설계 | 6 | 3.0 ~ 6.0 | 18 ~ 36 | 결제/인증 플로우(팝업↔페이지 이동) 전면 재설계, PG/카카오 인증 콜백 URL 영향, 회귀 테스트 큼 |
| D. 모바일 전용 유지(통합 제외) | - | - | 0 | 통합 대상 아님, 별도 트랙 |
| 공통 include/헤더/푸터/네비게이션 재구성 | - | 3 ~ 5 | 3 ~ 5 | 반응형 공통 레이아웃(header/footer/gnb) 1회성 구축 |
| JS 통합(이벤트 바인딩/반응형 스크립트 정리) | 30(10 이원화+20 모바일 전용 검토) | - | 10 ~ 15 | jQuery 기반 DOM 로직 반응형 통합, 이벤트 핸들러 중복 제거 |
| 통합 테스트/회귀 테스트(QA 제외, 개발자 자체 검증) | - | - | 15 ~ 25 | 93개 화면 x PC/모바일 브라우저 교차 검증, 결제 시나리오 회귀 |

### 3-2. 전체 합산 (Best / Avg / Worst)

| 시나리오 | 합산 공수(인일) | 순수 개발 기간(1인 기준) | 순수 개발 기간(3인 병렬 기준) |
|---|---|---|---|
| **Best Case** | 98 인일 | 약 4.5개월 | 약 6.5주 |
| **Average Case** | 133 인일 | 약 6개월 | 약 9주 |
| **Worst Case** | 190 인일 | 약 8.5개월 | 약 12.5주 |

- 위 기간은 **디자인 시안/QA 전담 인력 제외**, 백엔드+퍼블리싱 겸용 개발자 기준
- Worst Case는 결제(OrderController/OrderPayController) 플로우 재설계 중 PG사(KCP/KICC/키움페이/네이버페이) 및 카카오 인증 콜백 규격 변경이 필요한 경우를 반영

---

## 4단계 - 영향도 분석

### 4-1. 수정 필요 파일/모듈 목록

| 구분 | 파일/모듈 | 비고 |
|---|---|---|
| View 리졸버 제거/변경 | `AccessDeviceDelegatingViewResolver`, `AccessDeviceArgumentResolver`, `UserAgentInterceptor`, `AccessDeviceHandler` | 반응형 전환 시 자동 `/mobile/` 위임 로직 제거 또는 단순화 |
| Controller (11개, isPC 분기 제거) | `GoodsController`, `OrderController`, `OrderPayController`, `PickOrderController`, `UserController`, `MyOrderController`, `MyDepositController`, `ContentsController`, `CommonController`, `LoginController`, `ReturnGiftController` | 약 30개 분기 코드 정리 |
| JSP 병합 | `WEB-INF/jsp/**` (93개 PC 파일 + 대응 mobile 93개 = 186개 파일 → 93개로 병합) | 레이아웃 구조 재설계 필요 |
| JS 병합/정리 | `static/js/**`, `static/js/mobile/**` (동일 10개 병합 + 모바일 전용 20개 검토) | 이벤트 바인딩 반응형 대응 |
| Mapper/Service | 현재 grep 기준 **PC/MO 전용 쿼리 분기는 발견되지 않음** | 데이터 계층은 대체로 공유 — 리스크 낮음 |
| 설정(yml) | 특이 설정 분기 없음 | 영향 없음 |

### 4-2. 외부 연동 영향

| 연동 | 영향 여부 | 설명 |
|---|---|---|
| PG 결제(KCP, KICC, KiwoomPay, Naverpay) | **있음** | `order/kcp/*`, `order/daoupay/*`, `order/naverpay/*` 가 PC/MO 별도 JSP·리다이렉트 URL 사용 → 통합 시 콜백 URL, iframe/팝업 방식 변경에 따른 PG사 사전 협의 필요 |
| 카카오 인증(orderCertKakaoPop 등) | **있음** | PC는 팝업(`orderCertKakaoPop.jsp`), MO는 페이지 이동 방식 — 카카오 인증 콜백 URL 및 Channel 값(`WEB`/`MOBILE_WEB`) 정책과 연결되어 있어 통합 시 인증 플로우 자체 재설계 필요 |
| 알림톡/SMS 내 링크 | **가능성 있음** | 발송된 알림톡·문자 내 링크가 `AccessDevice` 판별에 의존하는 URL(고정 경로)을 사용 중이면, 기존에 발송된 메시지의 링크가 통합 후에도 정상 동작하는지(구형 URL 하위호환) 확인 필요 |
| 외부 이벤트 연동(puradak 등) | **낮음** | 모바일 전용 기능(D유형)으로 통합 범위에서 제외, 그대로 유지 |

### 4-3. 리스크 요소

- **[High] 결제/인증 플로우 유실 위험**: PC=팝업, MO=페이지 이동 방식이 다른 `OrderPayController`/카카오·카드 인증 플로우를 반응형 단일 흐름으로 바꾸면 회귀 테스트 실패 시 결제 장애로 직결(매출 영향)
- **[Medium] 채널 코드(Channel.WEB/MOBILE_WEB) 데이터 정합성**: 통합 후에도 통계/정산 목적으로 channel 값이 필요하면, User-Agent 판별 로직 자체는 유지하되 화면 렌더링만 통합해야 함(로직·통계는 유지, 뷰만 병합하는 절충안 필요)
- **[Medium] 페이지네이션/목록 건수 정책 상이**(`GOODS_RECORD_CNT` vs `MOBILE_GOODS_RECORD_CNT`): 통합 시 정책 재정의 필요, 상품 노출수 변경에 따른 UX/성능(쿼리 LIMIT) 영향 검토 필요
- **[Medium] 기존 URL 하위 호환성**: 알림톡·마케팅 메일 등에 이미 발송된 링크가 있다면, ViewResolver 제거 후에도 기존 요청 경로가 동일하게 동작하는지 확인(Controller `@RequestMapping` 경로는 유지되므로 URL 자체는 안전하나, 뷰 렌더링 결과가 달라짐에 유의)
- **[Low] Mobile 전용 33개 JSP 유실 가능성**: 통합 작업 중 실수로 mobile-only 파일을 삭제/누락하면 외부 이벤트(puradak) 등 기능 손실 → 통합 대상에서 명확히 제외하고 별도 트래킹 필요

---

## 요약 Bullet

- PC/MO 분리는 **addcon-svc에만 존재**(admin은 PC 단일, 통합 불필요)
- 분기 메커니즘은 **User-Agent 판별 + ViewResolver 자동 `/mobile/` 위임**(컨트롤러는 대부분 공유) + **11개 컨트롤러의 `isPC()` 조건분기**(~30곳)
- JSP 이중화 **93건**(PC 239 / MO 126), JS 이중화 **10건**(모바일 전용 20건 별도), Mobile 전용(통합 제외) **33건**
- 전체 93건 중 **약 75%(72건)는 단순 UI 차이**로 반응형 흡수 가능, **약 25%(24건)는 로직 분기 제거/재설계 필요**, 그중 **결제/인증 6건은 플로우 자체를 재설계**해야 하는 고위험 영역
- 예상 공수: **Best 98인일 / Average 133인일 / Worst 190인일**, 순수 개발 기간 **1인 기준 4.5~8.5개월, 3인 병렬 기준 6.5~12.5주** (디자인/QA 제외)
- 최대 리스크는 **결제·인증 플로우(PC 팝업 vs MO 페이지이동)의 통합**과 **PG/카카오 콜백 URL 하위 호환성**

## 합쳐서 사용 시 권장 접근 방식

**전면 빅뱅 통합보다는 단계적 접근을 권장**합니다. ① 1단계로 UI 전용 72건(팝업/마이페이지/회원가입 등)을 반응형으로 우선 통합해 리스크 없이 공수를 검증하고, ② 결제/인증(Order 계열) 6건은 별도 트랙으로 분리해 PG사·카카오 협의를 선행한 뒤 마지막에 통합하며, ③ `isPC()` 분기는 뷰 렌더링과 분리해 통계용 Channel 값 로직은 유지한 채 화면만 병합하는 절충안을 적용하는 것이 안전합니다.
