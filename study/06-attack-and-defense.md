# 06. 보안 위협과 방어 전략 — FIDO가 막는 공격과 남아있는 위협

FIDO가 모든 보안 문제를 해결하지는 않습니다.
어떤 공격을 막고, 어떤 위협이 여전히 남아있는지 명확히 아는 것이 중요합니다.

---

## 1. FIDO가 막는 공격

### 1-1. 피싱 (Phishing) — 완전 차단

FIDO 최대 강점. 도메인 바인딩 덕분에 원천 차단.

```
공격 시나리오:
  1. 공격자가 myb4nk.com (가짜 사이트) 생성
  2. 피해자에게 링크 전송 → 피해자가 접속
  3. 피해자가 Passkey/FIDO로 로그인 시도

FIDO 동작:
  4. 브라우저가 origin = "https://myb4nk.com" 으로 clientDataJSON 생성
  5. 진짜 mybank.com에 등록된 키는 myb4nk.com origin을 서명하지 않음
     (UAF: FacetID 불일치 / FIDO2: rpId 불일치)
  6. 인증 자체가 실패 → 자격증명 전달 불가

결과: 피해자가 피싱 사이트에 접속해도 인증 정보 유출 없음 ✓
```

### 1-2. 비밀번호 재사용 공격 — 구조적 차단

```
비밀번호 방식:
  A 사이트 DB 해킹 → 비밀번호 목록 확보 → B 사이트에 대입 (크리덴셜 스터핑)

FIDO:
  각 서비스마다 별도 키 쌍 생성 (rp.id 단위)
  → A 사이트의 개인키로 B 사이트 인증 불가
  → 크리덴셜 스터핑 원천 불가
```

### 1-3. 서버 DB 유출 — 피해 최소화

```
비밀번호 방식:
  DB 유출 → 비밀번호 해시 크래킹 → 계정 탈취

FIDO:
  DB 유출 → 공개키만 유출
  공개키로는 개인키 역산 불가 (수학적 보장)
  → 기존에 등록된 자격증명은 무효화 필요하지만
     공개키 자체는 공격자에게 쓸모없음
```

### 1-4. 중간자 공격 (MITM) — HTTPS + 바인딩으로 차단

```
TLS 레이어: HTTPS가 전송 중 데이터 보호
도메인 바인딩: 서버 인증서와 연결된 origin 검증
→ TLS 종단점과 FIDO origin이 일치해야 인증 성공
→ MITM 프록시가 있어도 origin 위조 불가
```

### 1-5. 리플레이 공격 — 챌린지로 차단

```
공격: 인증 패킷 캡처 → 재전송
방어: 챌린지는 1회용 + TTL 5분
      SignCounter로 이전 서명 재사용 탐지
```

---

## 2. FIDO가 막지 못하는 공격

### 2-1. 기기 탈취 (Device Theft)

```
시나리오:
  스마트폰을 물리적으로 훔침
  → 화면 잠금 해제 방법을 안다면 (어깨 너머로 PIN 훔쳐봄 등)
  → Passkey 사용 가능

FIDO 한계:
  FIDO는 "기기 소유자 = 정당 사용자"로 가정
  기기 자체를 빼앗기면 방어 불가

대응책:
  - 원격 기기 잠금/초기화 (Find My iPhone, Google Find My Device)
  - 서버 측에서 비정상 위치/패턴 탐지 (이상 행동 탐지)
  - 고위험 작업(이체)에는 추가 인증 요구
```

### 2-2. 악성 앱 / 악성코드 (Malware)

```
시나리오:
  기기에 악성 앱이 설치됨
  → 사용자가 합법적으로 생체인증을 수행하는 순간
  → 악성 앱이 그 인증 결과를 가로채 다른 서버에 전송

FIDO 한계:
  서명이 완료된 assertion을 전송하는 경로를 악성 앱이 가로채면
  해당 챌린지에 대한 인증은 공격자가 사용 가능

완화책:
  - 도메인 바인딩: 특정 서비스의 챌린지로만 서명 가능
  - Transaction Confirmation (UAF 1.1): 서명 대상 내용을 화면에 표시
    예) "100만원을 A 계좌로 이체합니다" → 사용자가 확인 후 서명
  - TEE Transaction Binding: 서명 전 내용을 TEE에서 사용자에게 표시
```

### 2-3. 계정 복구 경로 (Account Recovery)

```
시나리오:
  "기기를 잃어버렸어요, 계정 복구해주세요"
  → 공격자가 피해자인 척 고객센터에 연락
  → 고객센터가 신원 확인 후 새 기기 등록 허용
  → 공격자가 접근권 획득 (FIDO를 우회!)

핵심 문제:
  FIDO 인증 자체는 완벽하지만
  계정 복구 프로세스가 취약하면 전체 보안이 무너짐

대응책:
  - 강력한 신원 확인 절차 (신분증 + 영상통화 등)
  - 복구 코드 사전 발급 (등록 시 일회용 복구 코드 제공)
  - 복구 시 기존 세션/연결 기기 전부 로그아웃
  - 복구 후 일정 기간 고위험 거래 제한
```

### 2-4. 내부자 공격 (Insider Threat)

```
시나리오:
  DB 접근 권한 있는 내부자가 공개키 교체
  → 공격자 기기의 공개키로 치환
  → 공격자가 해당 계정으로 인증 성공

대응책:
  - DB 접근 감사 로그
  - 공개키 변경 시 사용자 알림 (이메일, SMS)
  - 공개키 변경 이력 보관 (Append-only 로그)
  - 중요 작업에 다중 승인 필요
```

### 2-5. SIM 스와핑 (전화번호 탈취)

```
해당 없음:
  FIDO 자체는 전화번호와 무관
  그러나 계정 복구를 SMS로 처리하면 SIM 스와핑으로 우회 가능

대응책:
  - 계정 복구 경로에서 SMS 사용 금지 또는 최소화
  - FIDO 등록이 완료된 계정은 SMS 복구 비활성화 옵션 제공
```

---

## 3. 위협 모델 정리

```
위협                    | FIDO 보호 수준 | 추가 대응 필요
------------------------|---------------|---------------
원격 피싱               | ★★★★★ 완전 차단 | 없음
크리덴셜 스터핑          | ★★★★★ 완전 차단 | 없음
서버 DB 유출             | ★★★★☆ 강력 보호 | 공개키 무효화 절차
리플레이 공격            | ★★★★★ 완전 차단 | 없음
중간자 공격 (MITM)       | ★★★★★ 완전 차단 | HTTPS 유지
기기 탈취               | ★★☆☆☆ 제한적   | 원격 초기화, 이상 탐지
악성코드                | ★★★☆☆ 부분 보호 | 트랜잭션 확인, TEE
계정 복구 우회           | ★☆☆☆☆ 보호 안 됨 | 강력한 복구 절차
내부자 공격              | ★★☆☆☆ 제한적   | 감사 로그, 알림
SIM 스와핑 (복구 경로)   | ★☆☆☆☆ 보호 안 됨 | SMS 복구 제거
```

---

## 4. 서버 구현의 보안 실수 BEST 5

실제 FIDO 서버 구현 시 자주 발생하는 보안 실수.

### 실수 1: 챌린지를 약한 난수로 생성

```python
# 잘못된 예 (절대 금지)
import random
challenge = str(random.random())  # 예측 가능!

# 올바른 예
import secrets
challenge = secrets.token_bytes(32)  # CSPRNG 사용
```

### 실수 2: origin 검증 생략

```python
# 잘못된 예
def verify(response):
    # origin 검증 없이 서명만 확인
    verify_signature(response.signature)

# 올바른 예
def verify(response):
    client_data = parse(response.clientDataJSON)
    # origin 검증 필수! 피싱 방지의 핵심
    assert client_data.origin in ALLOWED_ORIGINS
    verify_signature(response.signature)
```

### 실수 3: SignCounter 검증 생략

```python
# 잘못된 예
def verify(response, credential):
    verify_signature(...)
    # counter 업데이트 없음 → 리플레이/클론 공격 탐지 불가

# 올바른 예
def verify(response, credential):
    verify_signature(...)
    if response.counter > 0:
        if response.counter <= credential.sign_count:
            alert_security_team(credential)
            raise CloneDetected()
    credential.sign_count = response.counter
    db.save(credential)
```

### 실수 4: 챌린지 소멸 처리 경쟁 조건 (Race Condition)

```python
# 잘못된 예 (동시 요청 시 챌린지 2회 사용 가능)
def verify(challenge):
    data = cache.get(challenge)
    if data is None:
        raise Error("없는 챌린지")
    # ← 이 사이에 다른 요청이 동일 챌린지로 진입 가능!
    cache.delete(challenge)
    return data

# 올바른 예 (원자적 처리)
def verify(challenge):
    # Redis GETDEL: 조회와 삭제를 원자적으로 수행
    data = redis.getdel(challenge)
    if data is None:
        raise Error("없거나 이미 사용된 챌린지")
    return data
```

### 실수 5: Attestation 서명 검증 없이 메타데이터만 신뢰

```python
# 잘못된 예
def register(response):
    aaguid = parse_aaguid(response.attestationObject)
    metadata = mds.get(aaguid)
    # Attestation 서명 검증 없이 aaguid만 보고 신뢰
    save_credential(response)

# 올바른 예
def register(response):
    aaguid = parse_aaguid(response.attestationObject)
    metadata = mds.get(aaguid)
    root_cert = metadata.attestation_root_cert
    # 실제 서명 검증 필수
    verify_attestation_signature(response.attestationObject, root_cert)
    save_credential(response)
```

---

## 5. 이상 감지 (Anomaly Detection)

FIDO 인증 이후 계층에서 추가로 수행해야 할 보안 감지.

```python
# 의사코드 - 이상 감지 예시
class FidoAnomalyDetector:

    def check(self, user_id, auth_result, request_context):

        # 1. 새 기기 첫 사용 감지
        credential = auth_result.credential
        if credential.last_used_at is None:
            self.notify_user("새 기기에서 로그인되었습니다", request_context)

        # 2. 위치 급변 감지
        last_location = self.get_last_location(user_id)
        current_location = request_context.ip_location
        if is_impossible_travel(last_location, current_location):
            self.flag_for_review(user_id, "불가능한 이동 속도")

        # 3. 비정상 시간대 접근
        if is_unusual_time(user_id, request_context.timestamp):
            self.log_warning(user_id, "비정상 시간대 접근")

        # 4. 짧은 시간 내 여러 기기 동시 인증
        recent_auths = self.get_recent_auths(user_id, minutes=5)
        if len(set(a.credential_id for a in recent_auths)) > 3:
            self.flag_for_review(user_id, "다수 기기 동시 인증")

        # 5. 등록 직후 고위험 거래
        if credential.age_minutes() < 10:
            return AuthDecision.REQUIRE_ADDITIONAL_VERIFY
```

---

## 6. 고객사 영업 시 보안 질문 대응

고객사 보안 팀이 자주 하는 질문과 답변.

**Q: FIDO 서버 자체가 해킹되면 어떻게 되나요?**
> 서버에는 공개키만 있습니다. 공개키만으로는 어떤 서비스에도 로그인할 수 없습니다. 개인키는 사용자 기기 내 TEE에만 있으므로 서버 해킹으로 계정이 탈취되지 않습니다. 단, 공격자가 공개키를 교체하는 권한을 얻으면 새 인증기를 등록할 수 있으므로, DB 접근 제어와 감사 로그가 중요합니다.

**Q: 사용자가 기기를 잃어버리면 계정 접근이 불가능한가요?**
> Passkey(동기화 자격증명)를 사용하면 클라우드 복구가 가능합니다. Device-bound 자격증명만 사용하는 경우 계정 복구 절차를 별도로 설계해야 합니다. 복구 코드 사전 발급, 신원 확인 후 재등록 등의 방법이 있습니다.

**Q: 생체인식 데이터가 서버에 저장되나요?**
> 절대 아닙니다. 생체인식은 기기 내 Authenticator에서만 처리되며, 서버에는 공개키만 전달됩니다. 지문/얼굴 이미지는 기기 밖으로 나가지 않습니다.

**Q: 여러 기기를 등록할 수 있나요?**
> 네. 사용자는 여러 기기를 각각 등록할 수 있습니다. 각 기기마다 별도의 키 쌍이 생성되어 서버에 저장됩니다. 기기별로 독립적으로 인증하며, 특정 기기를 분실하면 그 기기의 자격증명만 폐기하면 됩니다.

---

## 전체 학습 완료 체크리스트

```
01 암호화 기초:
  ☐ 공개키/개인키 개념
  ☐ 디지털 서명 원리
  ☐ 챌린지-응답 방식

02 UAF:
  ☐ 4개 컴포넌트 역할 (Client, ASM, Authenticator, Server)
  ☐ 등록/인증 시퀀스
  ☐ SignCounter 역할
  ☐ AppID/FacetID 피싱 방지

03 FIDO2/WebAuthn:
  ☐ WebAuthn API 두 함수
  ☐ clientDataJSON의 origin 역할
  ☐ authenticatorData 구조
  ☐ Resident Key 개념

04 서버 아키텍처:
  ☐ DB 스키마 핵심 필드
  ☐ 챌린지 생성/소멸 처리
  ☐ 서명 검증 로직
  ☐ Attestation 검증

05 Passkey:
  ☐ Synced vs Device-bound
  ☐ 클라우드 동기화 보안 원리
  ☐ 크로스 디바이스 인증 (BLE)
  ☐ backup_eligible / backup_state

06 보안:
  ☐ FIDO가 막는 공격 5가지
  ☐ FIDO가 막지 못하는 공격 5가지
  ☐ 서버 구현 보안 실수 5가지
  ☐ 고객사 Q&A
```
