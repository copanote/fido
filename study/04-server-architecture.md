# 04. FIDO 서버 아키텍처 — 설계와 구현

FIDO 서버의 내부 구조, DB 설계, API 설계, 핵심 검증 로직을 다룹니다.
우리가 구축한 UAF 서버와 FIDO2 서버를 모두 다룹니다.

---

## 1. 전체 아키텍처

```
                    ┌─────────────────────────────┐
                    │      클라이언트 (앱/브라우저) │
                    └──────────────┬──────────────┘
                                   │ HTTPS
                    ┌──────────────▼──────────────┐
                    │         API Gateway         │
                    │    (인증, Rate Limiting)     │
                    └──────────────┬──────────────┘
                                   │
          ┌────────────────────────┼────────────────────────┐
          │                        │                        │
┌─────────▼──────────┐  ┌─────────▼──────────┐  ┌─────────▼──────────┐
│   UAF API 서버     │  │  FIDO2 API 서버    │  │    관리 서버        │
│  /uaf/1.1/...      │  │  /webauthn/...     │  │  등록 관리,         │
│                    │  │                    │  │  통계, 설정         │
└─────────┬──────────┘  └─────────┬──────────┘  └────────────────────┘
          │                        │
          └────────────┬───────────┘
                       │
          ┌────────────▼────────────┐
          │       비즈니스 로직      │
          │  - 챌린지 생성/관리      │
          │  - 서명 검증 엔진        │
          │  - Attestation 검증      │
          │  - 정책 엔진             │
          └────────────┬────────────┘
                       │
          ┌────────────┼────────────────────┐
          │            │                    │
┌─────────▼──────┐  ┌──▼───────────┐  ┌────▼─────────────┐
│    DB (주)      │  │  캐시 (Redis) │  │  MDS 연동        │
│  - 자격증명    │  │  - 챌린지     │  │  (Metadata Svc)  │
│  - 공개키      │  │  - 세션       │  │                  │
│  - 사용자 정보 │  │               │  │                  │
└────────────────┘  └───────────────┘  └──────────────────┘
```

---

## 2. DB 스키마 설계

### 핵심 테이블

```sql
-- 사용자 자격증명 테이블 (UAF + FIDO2 공통)
CREATE TABLE fido_credentials (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id         VARCHAR(256) NOT NULL,        -- 서비스 내 사용자 식별자
    credential_id   VARCHAR(512) NOT NULL UNIQUE,  -- 인증기가 발급한 고유 ID
    public_key      BLOB NOT NULL,                 -- DER/CBOR 인코딩 공개키
    public_key_alg  INTEGER NOT NULL,              -- -7: ES256, -257: RS256
    sign_count      BIGINT DEFAULT 0,              -- 마지막 서명 카운터
    aaguid          VARCHAR(36),                   -- Authenticator 종류 (FIDO2)
    aaid            VARCHAR(9),                    -- Authenticator 종류 (UAF)
    transports      VARCHAR(128),                  -- usb,nfc,ble,internal
    uv_initialized  BOOLEAN DEFAULT FALSE,         -- 사용자 검증 설정 여부
    backup_eligible BOOLEAN DEFAULT FALSE,         -- Passkey 동기화 가능 여부
    backup_state    BOOLEAN DEFAULT FALSE,         -- 현재 백업(동기화) 상태
    attestation_fmt VARCHAR(64),                   -- none, packed, tpm, android-key
    attestation_data BLOB,                         -- Attestation 원문 (감사용)
    device_name     VARCHAR(256),                  -- 사용자 표시용 기기 이름
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    last_used_at    DATETIME,
    is_active       BOOLEAN DEFAULT TRUE,
    INDEX idx_user_id (user_id),
    INDEX idx_credential_id (credential_id)
);

-- 챌린지 테이블 (Redis로 대체 가능)
CREATE TABLE fido_challenges (
    challenge       VARCHAR(128) PRIMARY KEY,      -- Base64URL 챌린지값
    user_id         VARCHAR(256),                  -- 챌린지 발급 대상 (인증 시)
    operation       ENUM('reg', 'auth', 'dereg'),  -- 어떤 작업용 챌린지인가
    session_data    TEXT,                          -- 추가 세션 데이터 (JSON)
    created_at      DATETIME NOT NULL,
    expires_at      DATETIME NOT NULL,             -- 보통 5분 후
    used_at         DATETIME,                      -- 사용 시각 (리플레이 방지)
    INDEX idx_expires (expires_at)
);

-- Authenticator 메타데이터 캐시 (MDS 로컬 캐시)
CREATE TABLE authenticator_metadata (
    aaguid          VARCHAR(36) PRIMARY KEY,
    description     VARCHAR(512),
    status          VARCHAR(64),                   -- FIDO_CERTIFIED, REVOKED, 등
    metadata_json   LONGTEXT,                      -- MDS에서 받은 전체 메타데이터
    root_certs      LONGTEXT,                      -- Attestation 검증용 루트 인증서
    updated_at      DATETIME
);
```

---

## 3. API 설계

### UAF API

```
POST /uaf/1.1/registration/request
  Body:  { username, displayName }
  Response: RegistrationRequest (UAF 스펙 형식)

POST /uaf/1.1/registration/response
  Body:  RegistrationResponse (UAF 스펙 형식)
  Response: { status: "ok" | "error", errorCode? }

POST /uaf/1.1/authentication/request
  Body:  { username? }
  Response: AuthenticationRequest (UAF 스펙 형식)

POST /uaf/1.1/authentication/response
  Body:  AuthenticationResponse (UAF 스펙 형식)
  Response: { status: "ok", userData? } | { status: "error", errorCode }

POST /uaf/1.1/deregistration
  Body:  DeregistrationRequest (UAF 스펙 형식)
  Response: { status: "ok" }
```

### FIDO2 / WebAuthn API

```
POST /webauthn/register/begin
  Body:  { username, displayName }
  Response: PublicKeyCredentialCreationOptions

POST /webauthn/register/finish
  Body:  PublicKeyCredential (attestation response)
  Response: { credentialId, status }

POST /webauthn/login/begin
  Body:  { username? }    ← Passkey는 username 없이도 가능
  Response: PublicKeyCredentialRequestOptions

POST /webauthn/login/finish
  Body:  PublicKeyCredential (assertion response)
  Response: { userId, sessionToken, status }

DELETE /webauthn/credentials/:credentialId
  Header: Authorization: Bearer <token>
  Response: { status: "ok" }
```

---

## 4. 챌린지 생성 및 관리

챌린지는 **예측 불가**하고 **일회성**이어야 한다.

```python
# 의사코드
import secrets

def generate_challenge():
    # CSPRNG (암호학적으로 안전한 난수 생성기) 사용 필수
    # Python: secrets.token_bytes
    # Java:   SecureRandom
    # Go:     crypto/rand
    raw = secrets.token_bytes(32)  # 256비트 = 32바이트
    return base64url_encode(raw)

def create_challenge(user_id, operation):
    challenge = generate_challenge()

    # Redis 또는 DB에 저장 (TTL 5분)
    cache.set(
        key=f"fido:challenge:{challenge}",
        value={
            "userId": user_id,
            "operation": operation,
            "createdAt": now()
        },
        ttl=300  # 5분
    )
    return challenge

def consume_challenge(challenge, expected_operation):
    data = cache.get(f"fido:challenge:{challenge}")

    if data is None:
        raise Error("챌린지 없음 또는 만료")
    if data["operation"] != expected_operation:
        raise Error("챌린지 용도 불일치")
    if data.get("usedAt") is not None:
        raise Error("이미 사용된 챌린지 (리플레이 공격)")

    # 즉시 사용 처리 (원자적으로 수행해야 함)
    cache.mark_used(f"fido:challenge:{challenge}")

    return data
```

---

## 5. 서명 검증 엔진

FIDO 서버의 핵심. ECDSA P-256 서명 검증.

### FIDO2 서명 검증

```python
# 의사코드
from cryptography.hazmat.primitives.asymmetric.ec import (
    ECDSA, EllipticCurvePublicKey
)
from cryptography.hazmat.primitives import hashes

def verify_assertion_signature(credential, response):
    """
    WebAuthn 인증 응답의 서명을 검증한다.
    """
    # 1. clientDataJSON 파싱
    client_data = json.parse(response.clientDataJSON)

    # 2. clientDataHash 계산
    client_data_hash = SHA256(response.clientDataJSON_raw_bytes)

    # 3. 서명 대상 데이터 조합
    # 서명 = Sign(privateKey, authenticatorData || clientDataHash)
    verification_data = response.authenticatorData + client_data_hash

    # 4. 저장된 공개키로 검증
    public_key = load_public_key(credential.public_key)  # EC 공개키 로드

    try:
        public_key.verify(
            signature=response.signature,
            data=verification_data,
            signature_algorithm=ECDSA(SHA256())
        )
        return True
    except InvalidSignature:
        return False
```

### Attestation 검증 (등록 시)

```python
# 의사코드 - packed format 예시
def verify_attestation(attestation_object, client_data_hash):
    """
    등록 시 Attestation Statement를 검증한다.
    """
    fmt = attestation_object["fmt"]

    if fmt == "none":
        # Attestation 없음 - 기기 신뢰성 검증 포기
        return AttestationType.NONE

    elif fmt == "packed":
        att_stmt = attestation_object["attStmt"]
        auth_data = attestation_object["authData"]

        # 서명 대상: authData || clientDataHash
        verification_data = auth_data + client_data_hash

        if "x5c" in att_stmt:
            # Full Attestation: 인증서 체인 포함
            cert_chain = att_stmt["x5c"]
            leaf_cert = parse_cert(cert_chain[0])

            # 인증서 체인 검증
            verify_cert_chain(cert_chain)

            # MDS에서 루트 인증서 가져와서 검증
            root_cert = mds.get_root_cert(auth_data.aaguid)
            verify_cert_chain_root(cert_chain, root_cert)

            # 서명 검증
            leaf_cert.public_key().verify(att_stmt["sig"], verification_data)

            return AttestationType.BASIC

        else:
            # Self Attestation: 자신의 FIDO 키로 서명
            credential_public_key = auth_data.credentialPublicKey
            credential_public_key.verify(att_stmt["sig"], verification_data)

            return AttestationType.SELF
```

---

## 6. 정책 엔진 (Policy Engine)

고객사마다 다른 보안 요구사항을 처리.

```python
# 의사코드
class FidoPolicy:
    """
    고객사별 FIDO 인증 정책
    """

    def check_registration(self, authenticator_info, policy_config):
        # 허용된 Authenticator 종류 확인
        if policy_config.allowed_aaguids:
            if authenticator_info.aaguid not in policy_config.allowed_aaguids:
                raise PolicyViolation("허용되지 않은 인증기")

        # Attestation 요구 수준 확인
        if policy_config.require_attestation:
            if authenticator_info.attestation_type == AttestationType.NONE:
                raise PolicyViolation("Attestation 필수")

        # 사용자 검증(UV) 요구 확인
        if policy_config.require_user_verification:
            if not authenticator_info.flags.UV:
                raise PolicyViolation("사용자 검증(지문/PIN) 필수")

        # 키 보안 수준 확인
        if policy_config.require_hardware_key:
            if authenticator_info.key_protection != KEY_PROTECTION_HARDWARE:
                raise PolicyViolation("하드웨어 보안 키 필수")

    def check_authentication(self, assertion, credential, policy_config):
        # 사용자 검증 확인
        if policy_config.require_user_verification:
            if not assertion.flags.UV:
                raise PolicyViolation("사용자 검증 필요")

        # 카운터 검증 (클론 기기 탐지)
        if assertion.counter > 0:  # 0은 카운터 미지원 기기
            if assertion.counter <= credential.sign_count:
                self.handle_counter_anomaly(credential)
                raise PolicyViolation("카운터 이상 - 기기 복제 의심")
```

---

## 7. 에러 코드 설계

UAF 스펙 표준 에러 코드 + 우리 서버 확장 코드.

```python
class UAFError:
    # UAF 1.1 표준 에러 코드
    NO_ERROR            = 0x0     # 성공
    WAIT_USER_ACTION    = 0x1     # 사용자 행위 대기 중
    INSECURE_TRANSPORT  = 0x2     # 비보안 전송 (HTTPS 아님)
    USER_CANCELLED      = 0x3     # 사용자 취소
    UNSUPPORTED_VERSION = 0x4     # 지원하지 않는 UAF 버전
    NO_SUITABLE_AUTH    = 0x5     # 적합한 Authenticator 없음
    PROTOCOL_ERROR      = 0x6     # 프로토콜 형식 오류
    UNTRUSTED_FACET     = 0x7     # 신뢰되지 않은 FacetID (피싱 감지!)
    KEY_DISAPPEARED     = 0x9     # 기기에서 키 삭제됨 (기기 초기화 등)
    INVALID_TRANSACTION = 0xA     # 트랜잭션 확인 실패
    AUTH_BLOCKED        = 0xB     # 잠금 (너무 많은 실패 시도)
    EXPIRED_CHALLENGE   = 0xC     # 챌린지 만료
    REPLAY_ATTACK       = 0xD     # 리플레이 공격 감지

    # 우리 서버 확장 에러 코드
    COUNTER_ANOMALY     = 0x1001  # 카운터 이상 감지
    USER_NOT_FOUND      = 0x1002  # 사용자 없음
    CREDENTIAL_REVOKED  = 0x1003  # 자격증명 취소됨
```

---

## 8. 보안 고려사항 체크리스트

서버 구현 시 반드시 확인해야 할 항목들.

```
챌린지 관련:
  ☑ 챌린지는 CSPRNG로 생성 (Math.random() 절대 금지)
  ☑ 챌린지 길이 최소 16바이트 (권장 32바이트)
  ☑ 챌린지 TTL 설정 (5~10분)
  ☑ 챌린지 1회 사용 후 즉시 소멸
  ☑ 챌린지 소멸 처리는 원자적(Atomic)으로 (Redis GETDEL 등)

서명 검증:
  ☑ origin 검증 (서버 도메인과 일치 여부)
  ☑ rpId 검증
  ☑ type 필드 검증 ("webauthn.create" or "webauthn.get")
  ☑ UP(User Presence) 플래그 확인
  ☑ UV(User Verification) 정책에 따라 확인
  ☑ SignCounter 검증 및 업데이트

Attestation:
  ☑ Attestation 정책 수준 결정 (none/indirect/direct)
  ☑ MDS 최신 상태 유지 (주기적 업데이트)
  ☑ Revoke된 Authenticator 차단

저장:
  ☑ 공개키 DB 저장 시 암호화 (at-rest encryption)
  ☑ credential_id 중복 검사
  ☑ 사용자당 등록 가능한 기기 수 제한 정책
```

---

## 체크리스트

- [ ] 챌린지를 Redis에 저장하는 이유는 무엇인가?
- [ ] SignCounter를 업데이트할 때 왜 원자적 처리가 필요한가?
- [ ] Attestation 검증을 생략하면 어떤 위험이 있는가?
- [ ] 고객사마다 다른 정책을 어떻게 처리해야 하는가?
- [ ] 기기 초기화 후 사용자가 재등록을 요청하면 어떻게 처리해야 하는가?
