# UAF FIDO 서비스 제공 - Step 1 실행 계획

## 1. 현재 보유 자산

| 자산 | 상태 | 비고 |
|------|------|------|
| UAF 1.1 FIDO 서버 | ✅ 운영 중 | Samsung Pay 연동으로 검증됨 |
| Registration API | ✅ 보유 | Challenge 생성, Attestation 검증, 공개키 저장 |
| Authentication API | ✅ 보유 | Challenge 검증, 서명 검증, Counter 관리 |
| Deregistration API | ✅ 보유 | 자격증명 삭제 |
| Public Key DB | ✅ 보유 | 사용자-기기-공개키 매핑 저장 |
| App SDK | ❌ 미보유 | Samsung Pay 전용 → 범용 SDK 필요 |
| 멀티테넌트 구조 | ❌ 미구축 | 단일 고객(Samsung Pay)용으로 구축됨 |

---

## 2. 전체 진행 방향

UAF 서버가 이미 존재하므로, **App SDK 개발이 최우선 과제**다.
SDK만 완성되면 즉시 고객사에 서비스를 제공할 수 있다.

```
Phase 1 (MVP) ──────────── Phase 2 (셀프서비스) ──────── Phase 3 (확장)
      │                           │                           │
 Android SDK                Developer Portal           FIDO2/WebAuthn
 iOS SDK                    Sandbox 환경               JS SDK
 멀티테넌트 서버 개선         문서 고도화                Passkey 지원
 기본 통합 문서
```

### Phase 1: MVP (B2B 서비스 최소 출시)
- Android SDK
- iOS SDK
- UAF 서버 멀티테넌트 개선
- 기본 통합 문서 (Getting Started, API 레퍼런스)

### Phase 2: 셀프서비스화
- Developer Portal (고객 셀프 온보딩)
- Sandbox 환경 (테스트용 별도 서버)
- 문서 고도화

### Phase 3: 서비스 확장
- FIDO2 / WebAuthn 서버 (브라우저 인증)
- JavaScript SDK
- Passkey 지원

---

## 3. 시작점: App SDK 개발

### 왜 SDK부터인가
- UAF 서버는 이미 동작한다. 고객사가 SDK 없이는 연동 불가능
- 고객사 앱에 SDK를 삽입하는 것이 FIDO 서비스 제공의 핵심 경로
- SDK 완성 후 서버 멀티테넌트 개선을 병행하면 즉시 서비스 제공 가능

### UAF SDK 레이어 구조 (Android/iOS 공통)

```
┌─────────────────────────────────┐
│         고객사 앱 (App)           │  ← 고객이 직접 개발
├─────────────────────────────────┤
│     FIDO Client SDK (공개 API)   │  ← 우리가 제공하는 SDK
├─────────────────────────────────┤
│     ASM (Authenticator Specific  │  ← SDK 내부 레이어
│          Module)                 │
├─────────────────────────────────┤
│     Authenticator                │  ← SDK 내부 (플랫폼 API 사용)
│  (TEE / Secure Enclave)          │
└─────────────────────────────────┘
         ↕ HTTPS
┌─────────────────────────────────┐
│       UAF 1.1 FIDO 서버          │  ← 우리 서버
└─────────────────────────────────┘
```

---

## 4. Android SDK 상세 설계

### 4-1. 모듈 구성

```
fido-android-sdk/
├── fido-client/          # 공개 API 모듈 (고객사가 직접 사용)
│   ├── FidoClient.kt
│   ├── model/
│   │   ├── RegistrationRequest.kt
│   │   ├── RegistrationResult.kt
│   │   ├── AuthenticationRequest.kt
│   │   ├── AuthenticationResult.kt
│   │   └── FidoError.kt
│   └── callback/
│       └── FidoCallback.kt
├── fido-asm/             # ASM 레이어 (내부 모듈)
│   ├── AsmHandler.kt
│   └── AsmRequest.kt
├── fido-authenticator/   # Authenticator 레이어 (내부 모듈)
│   ├── AndroidAuthenticator.kt
│   ├── KeystoreManager.kt
│   └── BiometricManager.kt
└── fido-core/            # 공통 유틸 (내부 모듈)
    ├── CryptoUtils.kt
    ├── UafMessageParser.kt
    └── HttpClient.kt
```

### 4-2. 공개 API 인터페이스

```kotlin
class FidoClient private constructor(
    private val context: Context,
    private val serverUrl: String,
    private val appId: String
) {

    companion object {
        fun builder(context: Context): Builder = Builder(context)
    }

    class Builder(private val context: Context) {
        private var serverUrl: String = ""
        private var appId: String = ""
        private var connectTimeoutMs: Long = 10_000
        private var readTimeoutMs: Long = 30_000

        fun serverUrl(url: String) = apply { this.serverUrl = url }
        fun appId(appId: String) = apply { this.appId = appId }
        fun connectTimeout(ms: Long) = apply { this.connectTimeoutMs = ms }
        fun readTimeout(ms: Long) = apply { this.readTimeoutMs = ms }
        fun build(): FidoClient = FidoClient(context, serverUrl, appId)
    }

    /**
     * FIDO 등록 (최초 생체인증 등록)
     * @param userId 서비스의 사용자 식별자
     * @param activity BiometricPrompt를 표시할 Activity
     * @param callback 결과 콜백
     */
    fun register(
        userId: String,
        activity: FragmentActivity,
        callback: FidoCallback<RegistrationResult>
    )

    /**
     * FIDO 인증 (등록된 생체인증으로 로그인)
     * @param userId 서비스의 사용자 식별자
     * @param activity BiometricPrompt를 표시할 Activity
     * @param callback 결과 콜백
     */
    fun authenticate(
        userId: String,
        activity: FragmentActivity,
        callback: FidoCallback<AuthenticationResult>
    )

    /**
     * FIDO 등록 해제 (기기에서 생체인증 삭제)
     * @param userId 서비스의 사용자 식별자
     * @param callback 결과 콜백
     */
    fun deregister(
        userId: String,
        callback: FidoCallback<Unit>
    )

    /**
     * 현재 기기에 FIDO 등록 여부 확인
     */
    fun isRegistered(userId: String): Boolean
}

interface FidoCallback<T> {
    fun onSuccess(result: T)
    fun onError(error: FidoError)
}

data class FidoError(
    val code: Int,
    val message: String
) {
    companion object {
        const val ERR_UNKNOWN = 0x01
        const val ERR_BIOMETRIC_CANCELLED = 0x02
        const val ERR_BIOMETRIC_LOCKED = 0x03
        const val ERR_BIOMETRIC_NOT_ENROLLED = 0x04
        const val ERR_BIOMETRIC_NOT_SUPPORTED = 0x05
        const val ERR_NETWORK = 0x06
        const val ERR_SERVER = 0x07
        const val ERR_NOT_REGISTERED = 0x08
        const val ERR_INVALID_APPID = 0x09
        const val ERR_KEY_COMPROMISED = 0x0A  // 기기 복제 감지 (Counter 불일치)
    }
}
```

### 4-3. Registration 내부 처리 흐름

```
앱 호출: fido.register(userId, activity, callback)
    │
    ▼
[FIDO Client]
1. 서버에 Registration Request 요청
   POST /uaf/1.1/registration/request
   Body: { "userId": "...", "appId": "..." }

    │ ← 서버 응답: RegistrationRequest (challenge, policy 포함)
    ▼

2. AppID / FacetID 검증 (피싱 방지)
   - AppID URL에서 facets.json 다운로드
   - 현재 앱의 FacetID (APK 서명 해시) 가 facets 목록에 있는지 검증
   - 검증 실패 시 즉시 ERR_INVALID_APPID 반환

    │
    ▼

[ASM → Authenticator]
3. OS(Android Keystore)에 키 쌍 생성 요청
   KeyPairGenerator.getInstance("EC", "AndroidKeyStore")
   ※ SDK가 직접 키를 생성하는 것이 아님. OS API를 호출해 TEE(하드웨어 보안 영역)에
     키 생성을 위임한다. 개인키는 TEE 밖으로 절대 추출되지 않으며 SDK/앱 메모리에
     노출되지 않는다.
   - EC (P-256) 알고리즘 사용
   - StrongBox 지원 기기면 StrongBox 사용 (TEE 보다 강함)
   - setUserAuthenticationRequired(true) 설정

4. BiometricPrompt 표시 (사용자 생체인증)
   - 인증 성공 시 OS가 해당 keyAlias의 키 사용 허가

5. Attestation 생성
   - OS Keystore API에 서명 요청 → TEE 내부에서 서명 수행 후 서명값만 반환
   - 디바이스 정보, 공개키, challenge를 포함한 구조체에 서명

    │
    ▼

[FIDO Client]
6. 서버에 Registration Response 전송
   POST /uaf/1.1/registration/response
   Body: UAF RegistrationResponse (공개키 + Attestation 포함)

    │ ← 서버 응답: 성공/실패
    ▼

7. 로컬 등록 상태 저장 (SharedPreferences)
   - userId → keyAlias 매핑 저장 (키 자체는 Keystore에만 존재)

8. callback.onSuccess(RegistrationResult) 호출
```

### 4-4. Authentication 내부 처리 흐름

```
앱 호출: fido.authenticate(userId, activity, callback)
    │
    ▼
[FIDO Client]
1. 서버에 Authentication Request 요청
   POST /uaf/1.1/authentication/request
   Body: { "userId": "...", "appId": "..." }

    │ ← 서버 응답: AuthenticationRequest (challenge 포함)
    ▼

2. AppID / FacetID 검증 (매 인증마다 수행)

    │
    ▼

[ASM → Authenticator]
3. BiometricPrompt 표시 (사용자 생체인증)
   - 인증 성공 시 OS가 해당 keyAlias의 키 사용 허가

4. OS(Android Keystore)에 서명 요청
   Signature.getInstance("SHA256withECDSA")
   ※ SDK가 개인키를 꺼내 직접 서명하는 것이 아님. OS API에 서명 대상 데이터를
     전달하면 TEE 내부에서 서명을 수행하고 서명값만 앱으로 반환한다.
   - 서명 대상: challenge + clientData + counter
   - Counter 1 증가 후 함께 전송

    │
    ▼

[FIDO Client]
5. 서버에 Authentication Response 전송
   POST /uaf/1.1/authentication/response
   Body: UAF AuthenticationResponse (서명값, counter 포함)

    │ ← 서버: 서명 검증 + Counter 검증 (클론 기기 탐지)
    ▼

6. callback.onSuccess(AuthenticationResult) 호출
   - AuthenticationResult에 서버 발급 토큰/세션 포함 가능
```

### 4-5. 키 저장 및 보안

```kotlin
// Android Keystore에 EC 키 쌍 생성
private fun generateKeyPair(keyAlias: String): KeyPair {
    val kpg = KeyPairGenerator.getInstance(
        KeyProperties.KEY_ALGORITHM_EC,
        "AndroidKeyStore"
    )
    val paramSpec = KeyGenParameterSpec.Builder(
        keyAlias,
        KeyProperties.PURPOSE_SIGN
    ).apply {
        setAlgorithmParameterSpec(ECGenParameterSpec("secp256r1"))
        setDigests(KeyProperties.DIGEST_SHA256)
        setUserAuthenticationRequired(true)
        // Android 11+: 매 사용마다 생체인증 요구
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
            setUserAuthenticationParameters(0, KeyProperties.AUTH_BIOMETRIC_STRONG)
        }
        // StrongBox 지원 기기에서 사용 (Pixel 3+, Samsung Galaxy S20+)
        if (hasStrongBox()) {
            setIsStrongBoxBacked(true)
        }
    }.build()

    kpg.initialize(paramSpec)
    return kpg.generateKeyPair()
}
```

### 4-6. FacetID 계산 (피싱 방지 핵심)

```kotlin
// Android FacetID = "android:apk-key-hash:<APK 서명의 SHA-1 Base64>"
private fun getFacetId(context: Context): String {
    val packageInfo = context.packageManager.getPackageInfo(
        context.packageName,
        PackageManager.GET_SIGNATURES
    )
    val signature = packageInfo.signatures[0]
    val md = MessageDigest.getInstance("SHA-1")
    val hash = md.digest(signature.toByteArray())
    return "android:apk-key-hash:" + Base64.encodeToString(
        hash, Base64.URL_SAFE or Base64.NO_PADDING or Base64.NO_WRAP
    )
}
```

### 4-7. 보안 고려사항

| 항목 | 구현 방법 |
|------|-----------|
| 루팅 감지 | SafetyNet Attestation API / Play Integrity API 사용 |
| 화면 캡처 방지 | `FLAG_SECURE` 설정 (BiometricPrompt 표시 중) |
| 인증서 피닝 | OkHttp `CertificatePinner` 로 서버 인증서 고정 |
| 키 백업 방지 | `setIsStrongBoxBacked` + `allowedAuthenticators` 설정 |
| 난독화 | ProGuard / R8 규칙으로 SDK 내부 클래스 보호 |

### 4-8. 배포 형태

- **AAR 파일** 직접 배포 (초기 Phase 1)
- **Maven/JCenter** 배포 (Phase 2 이후): `implementation 'com.yourcompany:fido-sdk:1.0.0'`
- **최소 지원 버전**: Android API 23 (Android 6.0, BiometricPrompt는 API 28부터 네이티브)

---

## 5. iOS SDK 상세 설계

### 5-1. 모듈 구성

```
fido-ios-sdk/
├── FidoClient/           # 공개 API (고객사가 직접 사용)
│   ├── FidoClient.swift
│   ├── FidoError.swift
│   └── Models/
│       ├── RegistrationResult.swift
│       └── AuthenticationResult.swift
├── ASM/                  # ASM 레이어 (내부)
│   └── AsmHandler.swift
├── Authenticator/        # Authenticator 레이어 (내부)
│   ├── iOSAuthenticator.swift
│   ├── SecureEnclaveManager.swift
│   └── BiometricManager.swift
└── Core/                 # 공통 유틸 (내부)
    ├── CryptoUtils.swift
    ├── UafMessageParser.swift
    └── HttpClient.swift
```

### 5-2. 공개 API 인터페이스

```swift
public class FidoClient {

    public struct Configuration {
        public let serverUrl: String
        public let appId: String
        public var connectTimeout: TimeInterval = 10.0
        public var readTimeout: TimeInterval = 30.0

        public init(serverUrl: String, appId: String) {
            self.serverUrl = serverUrl
            self.appId = appId
        }
    }

    public init(configuration: Configuration)

    /**
     * FIDO 등록 (최초 생체인증 등록)
     */
    public func register(
        userId: String,
        presenter: UIViewController,
        completion: @escaping (Result<RegistrationResult, FidoError>) -> Void
    )

    /**
     * FIDO 인증
     */
    public func authenticate(
        userId: String,
        presenter: UIViewController,
        completion: @escaping (Result<AuthenticationResult, FidoError>) -> Void
    )

    /**
     * FIDO 등록 해제
     */
    public func deregister(
        userId: String,
        completion: @escaping (Result<Void, FidoError>) -> Void
    )

    /**
     * 등록 여부 확인
     */
    public func isRegistered(userId: String) -> Bool
}

public enum FidoError: Error {
    case biometricCancelled
    case biometricLockout
    case biometricNotEnrolled
    case biometricNotAvailable
    case networkError(underlying: Error)
    case serverError(code: Int, message: String)
    case notRegistered
    case invalidAppId
    case keyCompromised      // Counter 불일치 (기기 복제 감지)
    case unknown
}
```

### 5-3. 키 저장: Secure Enclave

```swift
// Secure Enclave에서 EC 키 쌍 생성
private func generateKeyPair(keyAlias: String) throws -> SecKey {
    let access = SecAccessControlCreateWithFlags(
        kCFAllocatorDefault,
        kSecAttrAccessibleWhenUnlockedThisDeviceOnly,  // 기기 외부 추출 불가
        [.privateKeyUsage, .biometryCurrentSet],        // 생체인증 필수
        nil
    )!

    let attributes: [String: Any] = [
        kSecAttrKeyType as String:            kSecAttrKeyTypeECSECPrimeRandom,
        kSecAttrKeySizeInBits as String:      256,
        kSecAttrTokenID as String:            kSecAttrTokenIDSecureEnclave,
        kSecPrivateKeyAttrs as String: [
            kSecAttrIsPermanent as String:    true,
            kSecAttrApplicationTag as String: keyAlias.data(using: .utf8)!,
            kSecAttrAccessControl as String:  access
        ]
    ]

    var error: Unmanaged<CFError>?
    guard let privateKey = SecKeyCreateRandomKey(attributes as CFDictionary, &error) else {
        throw FidoError.unknown
    }
    return privateKey
}
```

### 5-4. 생체인증: LocalAuthentication

```swift
import LocalAuthentication

private func evaluateBiometric(
    reason: String,
    completion: @escaping (Bool, Error?) -> Void
) {
    let context = LAContext()
    context.localizedCancelTitle = "취소"

    // Face ID 또는 Touch ID 자동 선택
    var error: NSError?
    guard context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &error) else {
        completion(false, error)
        return
    }

    context.evaluatePolicy(
        .deviceOwnerAuthenticationWithBiometrics,
        localizedReason: reason
    ) { success, authError in
        DispatchQueue.main.async {
            completion(success, authError)
        }
    }
}
```

### 5-5. FacetID 계산 (iOS)

```swift
// iOS FacetID = "ios:bundle-id:<Bundle Identifier>"
// 참고: UAF 스펙에서 iOS는 bundle ID 기반으로 검증
private func getFacetId() -> String {
    let bundleId = Bundle.main.bundleIdentifier ?? ""
    return "ios:bundle-id:\(bundleId)"
}
```

### 5-6. 보안 고려사항

| 항목 | 구현 방법 |
|------|-----------|
| 탈옥 감지 | `/Applications/Cydia.app`, `cydia://` scheme 존재 여부 확인 |
| 인증서 피닝 | `URLSession` delegate에서 서버 인증서 공개키 해시 검증 |
| 키 추출 방지 | `kSecAttrTokenIDSecureEnclave` 사용 (하드웨어 내부에서만 서명 수행) |
| 생체 변경 감지 | `kSecAccessControlBiometryCurrentSet` 으로 생체 변경 시 키 무효화 |

### 5-7. 배포 형태

- **XCFramework** 직접 배포 (초기 Phase 1)
- **Swift Package Manager** 배포 (Phase 2 이후)
- **CocoaPods** 지원: `pod 'FidoSDK', '~> 1.0'`
- **최소 지원 버전**: iOS 13 (Secure Enclave + LocalAuthentication 안정화)

---

## 6. SDK 공통 고려사항

### 6-1. 에러 코드 표준화

서버 UAF 에러 코드와 SDK 에러 코드를 통일한다.

| 코드 | 의미 | 대응 |
|------|------|------|
| 0x01 | Unknown | 재시도 또는 고객센터 안내 |
| 0x02 | 생체인증 취소 | 사용자 취소, 재시도 안내 |
| 0x03 | 생체인증 잠금 | 기기 잠금 해제 후 재시도 안내 |
| 0x04 | 생체인증 미등록 | 기기 설정으로 이동 안내 |
| 0x05 | 생체인증 미지원 | 대체 인증 수단 제공 |
| 0x06 | 네트워크 오류 | 네트워크 확인 후 재시도 |
| 0x07 | 서버 오류 | 서버 점검 안내 |
| 0x08 | 미등록 기기 | 등록 유도 |
| 0x09 | AppID 검증 실패 | 앱 버전 업데이트 안내 (서명 불일치) |
| 0x0A | 키 위변조 감지 | 재등록 요청 (기기 복제 의심) |

### 6-2. 타임아웃 및 재시도 정책

- 서버 연결 타임아웃: 10초
- 응답 수신 타임아웃: 30초
- 자동 재시도: **하지 않음** (생체인증은 사용자 의도 기반이므로)
- 네트워크 오류 시 즉시 ERR_NETWORK 반환

### 6-3. 버전 관리 전략

```
v1.0.0 - UAF 1.1 기본 (등록/인증/해제)
v1.1.0 - Transaction Confirmation 지원 (간편결제)
v1.2.0 - Silent Authentication 지원 (백그라운드 세션 갱신)
v2.0.0 - FIDO2 / WebAuthn 지원 (Phase 3)
```

- Semantic Versioning 준수
- 하위 호환성: Minor 버전 업에서 API 제거 금지, Deprecated 처리 후 다음 Major에서 제거

---

## 7. 멀티테넌트 서버 개선 (Phase 1 병행)

App SDK 개발과 동시에 서버를 멀티테넌트로 개선해야 복수 고객사 대응이 가능하다.

| 항목 | 현재 | 개선 목표 |
|------|------|-----------|
| AppID 네임스페이스 | Samsung Pay 단일 | 테넌트별 분리 (`tenant_id` 기반) |
| API 인증 | 없거나 내부 전용 | API Key 발급 및 검증 |
| 키 DB 격리 | 단일 테이블 | `tenant_id` 컬럼 추가 + 인덱스 |
| 정책 설정 | 고정 | 테넌트별 Authenticator 정책, Challenge TTL 설정 |
| 사용량 집계 | 없음 | 테넌트별 일/월 통계 (청구 기반) |

---

## 8. 기본 통합 문서 (Phase 1)

| 문서 | 내용 |
|------|------|
| Getting Started | SDK 설치 → 초기화 → 등록 → 인증 (5분 완성 가이드) |
| Android 통합 가이드 | SDK 상세 API, 코드 샘플, ProGuard 규칙 |
| iOS 통합 가이드 | SDK 상세 API, 코드 샘플, App Store 심사 대응 |
| 서버 API 레퍼런스 | 엔드포인트, 요청/응답 스펙, 에러 코드 |
| FAQ | 자주 묻는 질문 (루팅 기기 대응, 기기 변경 시 처리 등) |

---

## 9. 즉시 시작 체크리스트

### Android SDK (Week 1~4)
- [ ] 프로젝트 구조 생성 (멀티모듈 Gradle 설정)
- [ ] `FidoClient` 공개 API 인터페이스 확정
- [ ] Android Keystore EC 키 생성/서명 구현
- [ ] BiometricPrompt 연동
- [ ] UAF Registration 서버 통신 구현
- [ ] UAF Authentication 서버 통신 구현
- [ ] AppID / FacetID 검증 구현
- [ ] 에러 처리 및 에러 코드 정의
- [ ] AAR 빌드 및 샘플 앱 연동 테스트

### iOS SDK (Week 1~4, Android와 병행)
- [ ] Xcode 프로젝트 구조 생성
- [ ] `FidoClient` Swift API 인터페이스 확정
- [ ] Secure Enclave EC 키 생성/서명 구현
- [ ] LocalAuthentication 연동 (Face ID / Touch ID)
- [ ] UAF Registration 서버 통신 구현
- [ ] UAF Authentication 서버 통신 구현
- [ ] AppID / FacetID 검증 구현
- [ ] XCFramework 빌드 및 샘플 앱 연동 테스트

### 서버 멀티테넌트 (Week 3~6)
- [ ] DB 스키마에 `tenant_id` 추가
- [ ] API Key 발급/검증 미들웨어 구현
- [ ] AppID 테넌트별 네임스페이스 분리
- [ ] 테넌트별 정책 설정 API

### 문서 (Week 5~6)
- [ ] Android Getting Started 가이드
- [ ] iOS Getting Started 가이드
- [ ] 서버 API 레퍼런스 초안
