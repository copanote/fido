# 2. FIDO 스펙

FIDO Alliance는 크게 세 가지 프로토콜 스펙을 정의해왔다.

---

## 2-1. UAF (Universal Authentication Framework)

### 목적

모바일 기기에서 **비밀번호를 완전히 없애고** 생체인식(지문, 얼굴) 또는 PIN으로 인증.

### 주요 사용처

- 삼성페이, 카카오페이 등 모바일 결제 앱
- 국내 은행·증권 앱의 생체 로그인
- 삼성 Pass 등 기기 인증 서비스

### 동작 흐름

```
[등록 흐름]

사용자 기기                           FIDO 서버
    |                                     |
    |<------- 1. 등록 요청 --------------|
    |                                     |
    |  2. 사용자 생체인식 수행 (지문 등) |
    |  3. 기기 내 키 쌍 생성             |
    |     - 개인키: TEE(보안 영역)에 저장 |
    |     - 공개키: 서버로 전송           |
    |------  공개키 + Authenticator 정보 ->|
    |                                     |
    |<------- 4. 등록 완료 응답 ---------|

[인증 흐름]

사용자 기기                           FIDO 서버
    |                                     |
    |<------- 1. 챌린지 전송 ------------|
    |                                     |
    |  2. 생체인식 수행 (지문 등)        |
    |  3. TEE에서 개인키로 챌린지 서명  |
    |------  서명값 전송 ---------------->|
    |                                     |
    |  4. 서버가 공개키로 서명 검증      |
    |<------- 5. 인증 성공/실패 ---------|
```

### 주요 구성요소

| 구성요소 | 역할 |
|---------|------|
| **FIDO Client** | 기기 내 FIDO 처리 모듈. 앱과 ASM 사이를 중계 |
| **ASM (Authenticator Specific Module)** | FIDO Client와 Authenticator 사이의 인터페이스 레이어 |
| **Authenticator** | 생체인식 등 실제 인증을 수행하는 하드웨어/소프트웨어 (지문 센서, TEE 등) |
| **FIDO Server** | 공개키 저장, 챌린지 생성, 서명 검증 수행 |

### TEE (Trusted Execution Environment)

- 스마트폰 내 일반 앱과 격리된 **보안 실행 환경**
- 개인키는 TEE 안에 저장되어 일반 앱은 물론 루팅된 기기에서도 직접 접근 불가
- ARM TrustZone 기술 기반 (안드로이드), Secure Enclave (iOS)

### 스펙 버전

- UAF 1.0 (2014년)
- UAF 1.1 (2017년): 트랜잭션 확인(결제 금액 표시 후 인증) 기능 강화

---

## 2-2. U2F (Universal 2nd Factor)

### 목적

기존 ID/비밀번호에 **2차 인증을 추가**하는 방식. 비밀번호를 대체하지 않고 보완.

### 주요 사용처

- 구글 계정 2단계 인증
- GitHub, Dropbox, Facebook 등 웹 서비스 2FA

### 동작 방식

- USB 보안키(YubiKey 등) 또는 NFC 키를 물리적 인증기로 사용
- 로그인 순서: ID/PW 입력 → 보안키 버튼 클릭(또는 NFC 태그) → 인증 완료
- UAF와 동일하게 공개키 암호화 기반, 도메인 바인딩 피싱 방지

### UAF vs U2F 차이

| 항목 | UAF | U2F |
|------|-----|-----|
| 목적 | 비밀번호 완전 대체 | 비밀번호 보완 (2FA) |
| 인증기 | 스마트폰 내장 생체인식 | 외장 USB/NFC 보안키 |
| 주 사용 환경 | 모바일 앱 | 웹 브라우저 |

---

## 2-3. FIDO2

### 목적

UAF + U2F를 통합하고 **웹 브라우저 표준**으로 확장하여, 비밀번호 없는 웹 인증을 실현.

### 구성

FIDO2는 두 개의 하위 스펙으로 구성된다.

```
FIDO2
├── WebAuthn  (W3C 표준)     — 브라우저와 웹 서버 간 인터페이스
└── CTAP      (FIDO Alliance) — 브라우저와 외부 인증기 간 통신 프로토콜
```

---

### WebAuthn (Web Authentication API)

W3C와 FIDO Alliance가 공동으로 만든 **브라우저 내장 JavaScript API**.

- 크롬, 파이어폭스, 사파리, 엣지에 기본 탑재
- 웹 서비스 개발자는 아래 두 함수만으로 FIDO 인증 구현 가능

```javascript
// 등록
const credential = await navigator.credentials.create({ publicKey: options });

// 인증
const assertion = await navigator.credentials.get({ publicKey: options });
```

#### 등록 옵션 예시

```json
{
  "rp": { "id": "mybank.com", "name": "My Bank" },
  "user": { "id": "userid", "name": "user@mybank.com" },
  "challenge": "<서버가 생성한 무작위값>",
  "pubKeyCredParams": [{ "type": "public-key", "alg": -7 }]
}
```

`rp` (Relying Party): 도메인 바인딩을 위한 서비스 식별자. 이 도메인에서만 인증 작동.

---

### CTAP (Client to Authenticator Protocol)

브라우저(클라이언트)와 **외부 인증기(USB 보안키, 스마트폰 등)** 간의 통신 프로토콜.

| 버전 | 설명 |
|------|------|
| **CTAP1** | 기존 U2F와 호환 |
| **CTAP2** | PIN 지원, 생체인식 지원, 상주 키(Resident Key) 지원 추가 |

#### 상주 키(Resident Key / Discoverable Credential)

- CTAP2에서 추가된 개념
- 인증기 내부에 사용자 ID를 포함한 자격증명을 저장
- 서버가 사용자 ID를 먼저 알려주지 않아도 인증기 스스로 자격증명을 찾을 수 있음
- Passkey 구현의 핵심 기반 기술

---

### Passkey (패스키)

FIDO2의 최신 진화 형태. 2022년 이후 빠르게 확산 중.

#### 기존 FIDO2와의 차이

| 항목 | 기존 FIDO2 | Passkey |
|------|-----------|---------|
| 자격증명 저장 위치 | 등록한 기기 1대 | 클라우드 동기화 (iCloud, Google) |
| 기기 분실 시 | 재등록 필요 | 새 기기에서 자동 복구 |
| 크로스 디바이스 인증 | 제한적 | PC에서 스마트폰으로 인증 가능 |

#### Passkey 크로스 디바이스 인증 흐름

```
PC 웹브라우저                   스마트폰
     |                               |
     | QR 코드 표시                  |
     |----QR 코드 스캔------------->|
     |                  생체인증(지문)|
     |<----인증 결과 전송 (BLE)------|
     |
  로그인 완료
```

---

## 스펙 전체 비교 요약

| 항목 | UAF | U2F | FIDO2 (WebAuthn+CTAP) |
|------|-----|-----|----------------------|
| 도입 목적 | 모바일 비밀번호 대체 | 웹 2FA 강화 | 웹+모바일 통합, 완전한 비밀번호 대체 |
| 주요 환경 | 모바일 앱 | 웹 브라우저 | 웹 브라우저 + 모바일 |
| 브라우저 내장 지원 | X | X | O (WebAuthn API) |
| 하드웨어 보안키 지원 | △ | O | O (CTAP) |
| 기기 간 동기화 | X | X | O (Passkey) |
| 피싱 방지 | O | O | O |
| 공개 표준 발표 | 2014 | 2014 | 2018 |

---

## 우리 서버가 구현한 스펙: UAF

현재 구축된 FIDO 서버는 **UAF 스펙** 기반이다.
삼성페이 연동을 위해 UAF를 구현했으며, 모바일 앱 생체인증 서비스 제공이 가능하다.

향후 고객사 확대 시 **FIDO2(WebAuthn)** 지원 추가 여부를 검토할 필요가 있다.
웹 기반 서비스나 Passkey를 원하는 고객사에게는 FIDO2 서버가 필요하기 때문이다.
