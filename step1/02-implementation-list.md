# 구현해야 할 것 목록

UAF 서버를 보유한 상태에서, 고객사에 서비스하기 위해 추가로 만들어야 하는 것들.

---

## 전체 구성도

```
┌─────────────────────────────────────────────────────────────────────┐
│                         고객사 (B2B)                                 │
│                                                                     │
│   고객사 Android 앱          고객사 iOS 앱        고객사 웹 서비스    │
│   └─ 우리 Android SDK        └─ 우리 iOS SDK      └─ 우리 JS SDK    │
└──────────────────────────────────┬──────────────────────────────────┘
                                   │
┌──────────────────────────────────▼──────────────────────────────────┐
│                         우리 인프라                                  │
│                                                                     │
│  ┌─────────────────┐  ┌───────────────┐  ┌───────────────────────┐  │
│  │  UAF API 서버   │  │  관리자 포털  │  │  고객사 연동 포털     │  │
│  │  (현재 보유)    │  │  (신규 개발)  │  │  (신규 개발)          │  │
│  └─────────────────┘  └───────────────┘  └───────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 구현 목록

### [A] App SDK — 가장 중요, 가장 큰 작업

고객사가 자기 앱에 FIDO 생체인증을 넣으려면 반드시 SDK가 필요합니다.
**"우리 UAF 서버를 쓰려면 우리 SDK를 앱에 탑재해야 한다"** 는 구조입니다.

#### A-1. Android SDK

```
역할:
  고객사 Android 앱에 탑재하는 라이브러리 (.aar 파일)
  앱 개발자가 3~5줄 코드로 생체인증을 연동할 수 있게 제공

구현 내용:
  ┌─────────────────────────────────────────────────────┐
  │ FIDO Client 레이어                                  │
  │  - UAF 메시지 생성/파싱                             │
  │  - AppID 검증 (FacetID 확인)                        │
  │  - 서버 통신 (HTTPS)                                │
  ├─────────────────────────────────────────────────────┤
  │ ASM 레이어                                          │
  │  - Android Keystore / StrongBox 연동                │
  │  - BiometricPrompt API 연동 (지문/Face)             │
  │  - TEE 키 생성 및 관리                              │
  ├─────────────────────────────────────────────────────┤
  │ 개발자용 API (고객사 앱 개발자가 호출하는 인터페이스) │
  │  - FidoClient.register(userId, callback)            │
  │  - FidoClient.authenticate(userId, callback)        │
  │  - FidoClient.deregister(userId, callback)          │
  └─────────────────────────────────────────────────────┘

지원 범위:
  Android 7.0+ (API 24+)
  BiometricPrompt API (Android 9+)
  StrongBox (하드웨어 보안 모듈, 최신 기기)

기술 스택:
  Kotlin / Java, Android Keystore, BiometricPrompt
  Retrofit (서버 통신), Gson/Moshi (JSON)

배포 방식:
  Maven Central or JitPack 배포
  또는 .aar 파일 직접 제공
```

#### A-2. iOS SDK

```
역할:
  고객사 iOS 앱에 탑재하는 라이브러리 (.xcframework)

구현 내용:
  ┌─────────────────────────────────────────────────────┐
  │ FIDO Client 레이어                                  │
  │  - UAF 메시지 생성/파싱                             │
  │  - AppID 검증                                       │
  │  - 서버 통신                                        │
  ├─────────────────────────────────────────────────────┤
  │ ASM 레이어                                          │
  │  - Secure Enclave 연동 (개인키 저장)                │
  │  - LocalAuthentication 프레임워크 (Face ID/Touch ID)│
  │  - SecKey API로 키 생성/서명                        │
  ├─────────────────────────────────────────────────────┤
  │ 개발자용 API                                        │
  │  - FidoClient.shared.register(userId:completion:)   │
  │  - FidoClient.shared.authenticate(userId:completion:)│
  │  - FidoClient.shared.deregister(userId:completion:) │
  └─────────────────────────────────────────────────────┘

지원 범위:
  iOS 13.0+
  Face ID / Touch ID
  Secure Enclave (iPhone 5S+)

기술 스택:
  Swift, Secure Enclave, LocalAuthentication
  URLSession (서버 통신), Codable (JSON)

배포 방식:
  CocoaPods / Swift Package Manager
  또는 .xcframework 직접 제공
```

---

### [B] 고객사 연동 포털 (Developer Portal)

```
역할:
  고객사가 셀프로 FIDO 서비스를 설정하고 관리하는 웹 포털

구현 내용:

  1. 가입 / API 키 발급
     - 고객사 계정 생성
     - 앱 등록 (AppID, Bundle ID, Package Name 등록)
     - API Key / Secret 발급

  2. SDK 다운로드 및 연동 가이드
     - Android / iOS SDK 다운로드
     - 연동 문서 (Getting Started)
     - 코드 샘플

  3. 대시보드
     - 등록 사용자 수
     - 일/월별 인증 건수
     - 인증 성공/실패율
     - 기기 유형 분포

  4. 설정
     - 허용 Authenticator 정책 설정
     - 챌린지 TTL 설정
     - 이상 감지 임계값 설정
     - Webhook 설정 (인증 이벤트 알림)

기술 스택 예시:
  React / Next.js (프론트)
  Node.js / Spring Boot (백엔드)
```

---

### [C] UAF 서버 멀티 테넌트 개선

현재 서버가 삼성페이 단독 구성이라면, 여러 고객사를 동시에 처리하도록 개선 필요.

```
구현 내용:

  1. 테넌트 격리
     - 고객사별 AppID 네임스페이스 분리
     - 고객사별 키 DB 격리 (논리적 or 물리적)
     - 고객사 간 데이터 접근 불가

  2. API Key 인증
     - 모든 API 요청에 고객사 API Key 포함
     - API Key로 고객사 식별 + 권한 확인

  3. 고객사별 정책 설정
     - 허용 Authenticator 목록 (고객사마다 다를 수 있음)
     - UV(User Verification) 필수 여부
     - SignCounter 검증 엄격도

  4. 고객사별 사용량 집계
     - 과금을 위한 API 호출 카운팅
     - 고객사별 통계 분리
```

---

### [D] 연동 문서 (Documentation)

SDK만 있으면 안 됨. 개발자가 쉽게 연동하도록 문서 필수.

```
구현 내용:

  1. Getting Started
     - 5분 안에 데모 동작시키기
     - 설치 → 초기화 → 등록 → 인증 순서로

  2. Android 연동 가이드
     - 설치 (Gradle 설정)
     - 초기화 (Application.onCreate)
     - 등록 화면 구현 예시
     - 인증 화면 구현 예시
     - 에러 처리 가이드
     - ProGuard 설정

  3. iOS 연동 가이드
     - 설치 (CocoaPods / SPM)
     - Info.plist 설정 (생체인식 권한)
     - 등록/인증 구현 예시
     - 에러 처리

  4. 서버 API 레퍼런스
     - 각 API 엔드포인트 설명
     - 요청/응답 JSON 예시
     - 에러 코드 전체 목록

  5. FAQ / 트러블슈팅
     - 자주 발생하는 연동 오류
     - OS 버전별 동작 차이
```

---

### [E] 테스트 환경 (Sandbox)

```
구현 내용:

  실제 서버와 완전히 분리된 테스트용 서버
  → 고객사 개발팀이 실 서비스 영향 없이 개발/테스트 가능

  포함 내용:
  - Sandbox UAF 서버 (test.fido.우리도메인.com)
  - 테스트용 API Key 자동 발급
  - 모든 Attestation 검증 무시 (테스트 편의)
  - 로그 상세 출력 (디버깅용)
  - Postman Collection 제공
```

---

## 우선순위 및 작업 순서

```
Phase 1 — 첫 고객사 영업을 위한 최소 필요 구성
  ┌─────────────────────────────────────────────┐
  │  A-1. Android SDK            (필수, 최우선)  │
  │  A-2. iOS SDK                (필수, 최우선)  │
  │  C.   서버 멀티 테넌트 개선  (필수)          │
  │  D.   기본 연동 문서         (필수)          │
  └─────────────────────────────────────────────┘

Phase 2 — 고객사 자가 운영을 위한 구성
  ┌─────────────────────────────────────────────┐
  │  B.   고객사 연동 포털                       │
  │  E.   Sandbox 환경                           │
  │  D.   문서 고도화                            │
  └─────────────────────────────────────────────┘

Phase 3 — 서비스 확장
  ┌─────────────────────────────────────────────┐
  │  FIDO2 / WebAuthn 서버 추가                  │
  │  JS SDK (웹 브라우저용)                      │
  │  Passkey 지원                                │
  └─────────────────────────────────────────────┘
```

---

## 작업 규모 예상

| 항목 | 예상 규모 | 비고 |
|------|----------|------|
| Android SDK | 크다 | BiometricPrompt + Keystore + UAF 프로토콜 |
| iOS SDK | 크다 | Secure Enclave + LocalAuthentication + UAF 프로토콜 |
| 서버 멀티 테넌트 | 중간 | 기존 서버 구조에 따라 변동 |
| 연동 포털 | 중간 | 기능 범위에 따라 변동 |
| 연동 문서 | 중간 | 잘 만들수록 고객사 CS 비용 절감 |
| Sandbox | 작다 | 기존 서버 fork + 설정 변경 |
