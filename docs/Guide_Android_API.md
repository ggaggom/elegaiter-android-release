# Elegaiter SDK API 레퍼런스 (Android 파트너 개발 가이드)

| 항목 | 내용 |
|------|------|
| 대상 | 파트너사 안드로이드(Android) 앱 개발자 |
| 목적 | SDK 4개 매니저의 메소드 시그니처, 설명, 사용법 안내 |
| 참고 | SDK 연동 가이드(README)에서 필수 권한, 설치 및 기본 사용법 확인 요망 |

## 1. 개요
Elegaiter SDK는 `Elegaiter.getInstance()`를 통해 인스턴스를 획득한 후, 다음 4개의 Manager에 접근하여 사용합니다. 안드로이드 SDK의 비동기 작업은 **Kotlin Coroutines** 및 **Flow** 기반으로 설계되었습니다.

| Manager | 역할 |
|---------|------|
| `BleManager` | BLE 기기 스캔·연결·해제, Threshold 설정, 에러 코드 전송 |
| `AuthManager` | 세션 상태 확인, 로그인·회원가입·프로필·비밀번호 관리 |
| `GaitAnalysisManager` | 실시간 보행 분석 시작·중지, 실시간 메트릭/이벤트 스트림 제공 |
| `GaitRecordManager` | 보행 기록 로컬 저장, 서버 동기화, 과거 기록 조회·삭제 |
> **비동기 함수 반환 타입 안내:** `suspend` 키워드가 붙은 함수는 `viewModelScope.launch { }` 또는 `lifecycleScope.launch { }` 블록 내에서 호출해야 합니다. 대부분의 비동기 함수는 `Result<T>` 타입을 반환하며, `Result.Success`와 `Result.Error`로 분기 처리합니다. 자세한 타입 정의는 [6.4 공통 타입](#64-공통-타입)을 참고하세요.


## 2. BleManager
JAWS 센서와의 BLE 연결 및 통신을 관리합니다. 스캔, 연결, Threshold 설정, 에러 코드 전송을 담당합니다.
> **⚠️ 권한 필수:** `scan()` 및 `connect()` 호출 전, 반드시 런타임 BLE 권한이 허용된 상태여야 합니다. 권한 설정 방법은 연동 가이드 4단계를 참고하세요.

### 2.1 프로퍼티 (Properties)

| 프로퍼티 | 타입 | 설명 |
|----------|------|------|
| `connectionState` | `StateFlow<BleConnectionState>` | 현재 BLE 연결 상태. UI에서 `collectAsState()`로 구독하여 연결 상태 변화를 실시간으로 반영할 수 있습니다. |
| `thresholdResponse` | `SharedFlow<Int>` | `checkThreshold()`, `setThreshold()`, `setAutoThreshold()` 호출 후 기기로부터 수신된 Threshold 값 스트림. 각 메서드 호출 후 직접 반환값이 없으므로 이 Flow를 구독하여 응답을 수신해야 합니다. |

### 2.2 Methods

---

#### `scan()`

주변의 Elegaiter 장비를 스캔하여 발견된 기기 목록을 반환합니다. 스캔은 일정 시간 동안 진행되며, 완료되면 발견된 모든 기기를 한 번에 반환합니다.

```kotlin
suspend fun scan(): Result<List<ScannedDevice>>
```

| 반환 | 설명 |
|------|------|
| `Result.Success<List<ScannedDevice>>` | 발견된 기기 목록. 목록이 비어있을 수 있음 |
| `Result.Error` | 권한 부족 또는 블루투스 비활성화 시 |

```kotlin
// 사용 예시
viewModelScope.launch {
    when (val result = sdk.bleManager.scan()) {
        is Result.Success -> {
            // "EL"로 시작하는 Elegaiter 기기만 필터링
            val devices = result.data.filter { it.name.startsWith("EL") }
        }
        is Result.Error -> {
            // 권한 확인 또는 블루투스 활성화 안내
        }
    }
}
```

---

#### `connect(device: ScannedDevice)`

`scan()`으로 발견한 특정 기기에 BLE 연결을 시도합니다. 연결 성공 이후 `connectionState`가 `CONNECTED`로 변경됩니다.

```kotlin
suspend fun connect(device: ScannedDevice): Result<Unit>
```

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `device` | `ScannedDevice` | `scan()` 결과로 얻은 연결 대상 기기 |

| 반환 | 설명 |
|------|------|
| `Result.Success<Unit>` | 연결 성공 |
| `Result.Error` | 연결 실패 (기기 범위 초과, 기기 거부 등) |

```kotlin
// 사용 예시
val targetDevice = scannedDevices.first()
when (sdk.bleManager.connect(targetDevice)) {
    is Result.Success -> { /* 연결 성공 → 다음 단계 진행 */ }
    is Result.Error -> { /* 재시도 또는 에러 안내 */ }
}
```

---

#### `disconnect()`

현재 연결된 BLE 기기와의 연결을 해제합니다. 반환값이 없으며, 호출 즉시 연결이 끊깁니다. `connectionState`는 `DISCONNECTED`로 변경됩니다.

```kotlin
fun disconnect()
```

```kotlin
// 사용 예시 (운동 종료 후 연결 해제)
sdk.bleManager.disconnect()
```

---

#### `checkThreshold()`

현재 연결된 기기에 설정된 Threshold 값을 조회하는 명령을 전송합니다.

> **⚠️ 주의:** 이 함수는 응답값을 직접 반환하지 않습니다. 기기로부터의 응답은 `thresholdResponse` (SharedFlow)를 통해 비동기로 수신됩니다. 반드시 `thresholdResponse`를 구독한 상태에서 호출하세요.

```kotlin
suspend fun checkThreshold(): Result<Unit>
```

| 반환 | 설명 |
|------|------|
| `Result.Success<Unit>` | 명령 전송 성공 (응답은 `thresholdResponse`로 수신) |
| `Result.Error` | 명령 전송 실패 |

```kotlin
// 사용 예시
// 1. 먼저 응답 스트림 구독
viewModelScope.launch {
    sdk.bleManager.thresholdResponse.collect { threshold ->
        // 수신된 Threshold 값 처리
    }
}

// 2. 조회 명령 전송
viewModelScope.launch {
    sdk.bleManager.checkThreshold()
}
```

---

#### `setThreshold(level: Int)`

현재 연결된 기기에 Threshold 값을 수동으로 설정합니다. 명령 전송에 성공하면, 기기는 설정된 최종 값을 `thresholdResponse`를 통해 응답합니다.

```kotlin
suspend fun setThreshold(level: Int): Result<Unit>
```

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `level` | `Int` | 설정할 Threshold 레벨 수치 |

| 반환 | 설명 |
|------|------|
| `Result.Success<Unit>` | 설정 명령 전송 성공 |
| `Result.Error` | 설정 명령 전송 실패 |

```kotlin
// 사용 예시
viewModelScope.launch {
    sdk.bleManager.setThreshold(level = 40)
}
```

---

#### `setAutoThreshold()`

기기가 스스로 최적의 Threshold 값을 찾도록 자동 설정 명령(T0 프로토콜)을 전송합니다. 기기가 계산을 완료하면 갱신된 값을 thresholdResponse로 반환합니다.

```kotlin
suspend fun setAutoThreshold(): Result<Unit>
```

| 반환 | 설명 |
|------|------|
| `Result.Success<Unit>` | 자동 설정 명령 전송 성공 |
| `Result.Error` | 명령 전송 실패 |

```kotlin
// 사용 예시
viewModelScope.launch {
    sdk.bleManager.setAutoThreshold()
}
```

---

#### `sendError(errorCode: Int)`

연결된 기기 측으로 에러 코드를 전송합니다.

```kotlin
suspend fun sendError(errorCode: Int): Result<Unit>
```

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `errorCode` | `Int` | 전송할 에러 코드 (`0` 또는 `1`) |

| 에러 코드 (`errorCode`) | 의미 | 기기 동작 (JAWS) | LED 상태 |
|-------------------------|------|------------------|----------|
| 1 | 에러 발생 | 데이터 전송 중지 | 빨간색 (Red) |
| 0 | 에러 해소 | 데이터 전송 재개 | 파란색 (Blue) |

```kotlin
// 사용 예시
viewModelScope.launch {
    sdk.bleManager.sendError(errorCode = 1)
}
```

---

## 3. AuthManager
사용자 계정과 관련된 모든 기능을 담당합니다. 로그인/로그아웃, 세션 관리, 회원가입, 비밀번호 찾기/변경, 프로필 관리 등을 포함합니다.

### 3.1 프로퍼티 (Properties)

| 프로퍼티 | 타입 | 설명 |
|----------|------|------|
| `currentUserId` | `StateFlow<String?>` | 현재 로그인된 사용자 ID. 비로그인 상태일 경우 `null`을 방출합니다. |
| `savedExerciseInfo` | `Flow<ExerciseInfo?>` | 저장된 운동 설정 정보의 변경 사항을 감지하는 Flow. `saveExerciseInfo()` 호출 시 새 값이 방출됩니다. |
| `sessionExpiredEvent` | `SharedFlow<Unit>` | 서버 토큰 만료 이벤트. 이 이벤트 수신 시 로그인 화면으로 이동하는 처리가 필요합니다. |

### 3.2 Methods

---

#### `checkAuthStatus()`

앱 시작 시 저장된 토큰의 유효성을 검사하여 자동 로그인 여부를 결정합니다. Splash 또는 MainActivity에서 가장 먼저 호출하는 것을 권장합니다.

> **💡 자동 로그인 구현 가이드**
> * 자동 로그인을 사용하는 경우: Splash 화면에서 호출하여 메인 화면으로 즉시 진입할지 결정하는 용도로 사용합니다.
> * 자동 로그인을 사용하지 않는 경우: 이 함수를 호출할 필요가 없으며, 앱 시작 시 항상 로그인 화면을 노출하도록 설계합니다. 다만, 로그아웃 후 재진입 시의 상태 정합성을 위해 호출하는 것을 권장합니다.

```kotlin
suspend fun checkAuthStatus(): Result<Boolean>
```

| 반환 | 설명 |
|------|------|
| `Result.Success(true)` | 토큰 유효 → 메인 화면으로 이동 |
| `Result.Success(false)` | 토큰 없음 또는 자동 로그인 미설정 → 로그인 화면으로 이동 |
| `Result.Error` | 네트워크 오류 등 검증 실패 |

```kotlin
// 사용 예시 (SplashViewModel)
viewModelScope.launch {
    // 자동 로그인을 지원하지 않는 앱이라면 이 과정을 생략하고 바로 navigateToLogin()을 호출하세요.
    when (val result = sdk.authManager.checkAuthStatus()) {
        is Result.Success -> {
            if (result.data) navigateToMain() else navigateToLogin()
        }
        is Result.Error -> { /* 네트워크 에러 안내 */ }
    }
}
```

---

#### `checkIdAvailability(id: String)`

회원가입 전 아이디 중복 여부를 확인합니다.

```kotlin
suspend fun checkIdAvailability(id: String): Result<Boolean>
```

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `id` | `String` | 중복 확인할 아이디 |

| 반환 | 설명 |
|------|------|
| `Result.Success(true)` | 사용 가능한 아이디 |
| `Result.Success(false)` | 이미 사용 중인 아이디 |
| `Result.Error` | 네트워크 오류 등 |

```kotlin
// 사용 예시
viewModelScope.launch {
    when (val result = sdk.authManager.checkIdAvailability("newUser123")) {
        is Result.Success -> {
            if (result.data) { /* 사용 가능 안내 */ }
            else { /* 중복 안내 */ }
        }
        is Result.Error -> { /* 에러 처리 */ }
    }
}
```

---

#### `register(info: NewUserInfo)`

신규 사용자를 등록(회원가입)합니다. `NewUserInfo` 데이터 클래스의 모든 필드가 필수값입니다.

```kotlin
suspend fun register(info: NewUserInfo): Result<Unit>
```

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `info` | `NewUserInfo` | 신규 사용자 정보. 상세 필드는 [6.3 사용자 관련](#63-사용자-관련) 참고 |

> **📌 비밀번호 힌트(pwhint) 매핑 테이블**
> 
> 서버 및 SDK와의 통신 정합성을 위해, 사용자에게 보여지는 텍스트 대신 아래의 고정된 상수값을 전송해야 합니다.

| UI 표시 문구 (예시) | 전송해야 할 값 (String) |
| :--- | :--- |
| 좋아하는 색깔은? | `HINT_FAVORITE_COLOR` |
| 어린 시절 별명은? | `HINT_CHILDHOOD_NICKNAME` |
| 가장 좋아하는 음식은? | `HINT_FAVORITE_FOOD` |
| 졸업한 초등학교 이름은? | `HINT_ELEMENTARY_SCHOOL` |
| 내가 태어난 도시(고향)는? | `HINT_BIRTHPLACE` |
| 가장 기억에 남는 여행지는? | `HINT_MEMORABLE_TRIP` |
| 아버지의 성함은? | `HINT_FATHER_NAME` |

```kotlin
// 사용 예시
val newUser = NewUserInfo(
    id = "newUser123",
    pass = "password!",
    // 주의: 실제 텍스트가 아닌 매핑 테이블의 상수를 전달합니다.
    pwhint = "HINT_BIRTHPLACE",
    pwhintAns = "서울",
    name = "홍길동",
    gender = "M",  // "M" 또는 "F"
    birthday = "19900101",
    phone = "01012345678",
    height = 175.0f,
    weight = 70.0f
)
viewModelScope.launch {
    when (sdk.authManager.register(newUser)) {
        is Result.Success -> { /* 가입 성공 → 로그인 화면 이동 */ }
        is Result.Error -> { /* 가입 실패 안내 */ }
    }
}
```

---

#### `login(id: String, password: String, isAutoLogin: Boolean)`

사용자를 로그인시킵니다. `isAutoLogin`을 `true`로 설정하면 토큰이 로컬에 저장되어 다음 앱 실행 시 `checkAuthStatus()`를 통해 자동 로그인됩니다.

```kotlin
suspend fun login(id: String, password: String, isAutoLogin: Boolean): Result<Unit>
```

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `id` | `String` | 사용자 아이디 |
| `password` | `String` | 사용자 비밀번호 |
| `isAutoLogin` | `Boolean` | `true` 설정 시 토큰 로컬 저장 → 자동 로그인 활성화 |

```kotlin
// 사용 예시
viewModelScope.launch {
    when (sdk.authManager.login(id = "user123", password = "password!", isAutoLogin = true)) {
        is Result.Success -> { /* 메인 화면으로 이동 */ }
        is Result.Error -> { /* 아이디/비밀번호 오류 안내 */ }
    }
}
```

---

#### `logout()`

현재 사용자를 로그아웃시킵니다. 로컬에 저장된 토큰을 삭제하며, `currentUserId`가 `null`로 변경됩니다.

```kotlin
suspend fun logout()
```

```kotlin
// 사용 예시
viewModelScope.launch {
    sdk.authManager.logout()
    navigateToLogin()
}
```

---

#### `findId(name: String, phone: String)`

이름과 전화번호로 등록된 아이디를 찾습니다.

```kotlin
suspend fun findId(name: String, phone: String): Result<String?>
```

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `name` | `String` | 사용자 이름 |
| `phone` | `String` | 등록된 전화번호 |

| 반환 | 설명 |
|------|------|
| `Result.Success(String)` | 찾은 아이디 문자열 |
| `Result.Success(null)` | 일치하는 사용자 없음 |
| `Result.Error` | 네트워크 오류 등 |

```kotlin
// 사용 예시
viewModelScope.launch {
    when (val result = sdk.authManager.findId(name = "홍길동", phone = "01012345678")) {
        is Result.Success -> {
            val foundId = result.data ?: "일치하는 계정 없음"
        }
        is Result.Error -> { /* 에러 처리 */ }
    }
}
```

---

#### `verifyPasswordHint(id: String, hint: String, answer: String)`

비밀번호 찾기 1단계입니다. 아이디, 비밀번호 힌트, 힌트 답변이 일치하는지 확인합니다. 성공 시 비밀번호 재설정에 사용할 임시 토큰을 반환합니다.

> **⚠️ 주의사항**
> 
> hint 파라미터에는 사용자가 보는 텍스트(예: "내가 태어난 도시")가 아닌, [6.3 사용자 관련](#63-사용자-관련) 섹션의 매핑 테이블에 정의된 고정 상수값을 전달해야 합니다.

```kotlin
suspend fun verifyPasswordHint(id: String, hint: String, answer: String): Result<String>
```

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `id` | `String` | 사용자 아이디 |
| `hint` | `String` | 등록한 비밀번호 힌트 질문 |
| `answer` | `String` | 힌트 답변 |

| 반환 | 설명 |
|------|------|
| `Result.Success(String)` | 비밀번호 재설정용 임시 토큰 (5분간 유효) |
| `Result.Error` | 인증 실패. 힌트/답변 불일치 시 `AuthException.InvalidHintException` 반환 |

```kotlin
// 사용 예시
viewModelScope.launch {
    when (val result = sdk.authManager.verifyPasswordHint(
        id = "user123",
        hint = "HINT_BIRTHPLACE",
        answer = "서울"
    )) {
        is Result.Success -> {
            val resetToken = result.data
            // 2단계: resetLostPassword(newPassword, resetToken) 호출
        }
        is Result.Error -> {
            when (result.exception) {
                is AuthException.InvalidHintException -> {
                    // 힌트 또는 답변이 일치하지 않는 경우 처리
                }
                else -> { /* 기타 네트워크 오류 처리 */ }
            }
        }
    }
}
```

---

#### `resetLostPassword(newPassword: String, token: String)`

비밀번호 찾기 2단계입니다. `verifyPasswordHint()`에서 받은 토큰을 사용하여 새 비밀번호로 재설정합니다. 토큰은 발급 후 5분간만 유효합니다.

```kotlin
suspend fun resetLostPassword(newPassword: String, token: String): Result<Unit>
```

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `newPassword` | `String` | 새 비밀번호 |
| `token` | `String` | `verifyPasswordHint()`에서 반환된 임시 토큰 |

| 반환 | 설명 |
|------|------|
| `Result.Success<Unit>` | 비밀번호 재설정 성공 |
| `Result.Error` | 재설정 실패. 토큰 만료 시 `AuthException.TokenExpiredException` 반환 |

```kotlin
// 사용 예시
viewModelScope.launch {
    when (val result = sdk.authManager.resetLostPassword(newPassword = "newPass!", token = resetToken)) {
        is Result.Success -> { /* 재설정 완료 → 로그인 화면 이동 */ }
        is Result.Error -> {
            when (result.exception) {
                is AuthException.TokenExpiredException -> {
                    // 토큰 만료 (5분 경과) 안내 및 1단계로 이동
                    showToast("인증 시간이 만료되었습니다. 다시 시도해주세요.")
                }
                else -> { /* 기타 네트워크 오류 처리 */ }
            }
        }
    }
}
```

---

#### `getUserProfile()`

현재 로그인된 사용자의 프로필 정보를 서버에서 불러옵니다.

```kotlin
suspend fun getUserProfile(): Result<UserProfile>
```

| 반환 | 설명 |
|------|------|
| `Result.Success<UserProfile>` | 사용자 프로필 데이터 |
| `Result.Error` | 비로그인 상태 또는 네트워크 오류 |

```kotlin
// 사용 예시
viewModelScope.launch {
    when (val result = sdk.authManager.getUserProfile()) {
        is Result.Success -> {
            val profile = result.data
            // profile.name, profile.height 등 활용
        }
        is Result.Error -> { /* 에러 처리 */ }
    }
}
```

---

#### `updateUserProfile(updatedProfile: UserProfile)`

현재 로그인된 사용자의 프로필 정보를 수정합니다.

```kotlin
suspend fun updateUserProfile(updatedProfile: UserProfile): Result<Unit>
```

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `updatedProfile` | `UserProfile` | 수정할 프로필 정보 |

```kotlin
// 사용 예시
val updatedProfile = currentProfile.copy(weight = 68.0f)
viewModelScope.launch {
    when (sdk.authManager.updateUserProfile(updatedProfile)) {
        is Result.Success -> { /* 수정 완료 안내 */ }
        is Result.Error -> { /* 에러 처리 */ }
    }
}
```

---

#### `verifyCurrentPassword(password: String)`

회원 정보 수정 등 민감한 페이지 진입 전 현재 사용자의 비밀번호가 일치하는지 재인증(Password Re-authentication)합니다. 보안을 위해 본인 확인이 필요한 시점에 호출합니다.

```kotlin
suspend fun verifyCurrentPassword(password: String): Result<Unit>
```

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `password` | `String` | 현재 비밀번호 |

| 반환 | 설명 |
|------|------|
| `Result.Success<Unit>` | 비밀번호 일치 → 다음 단계 진행 가능 |
| `Result.Error` | 비밀번호 불일치, 비로그인 상태 또는 네트워크 오류 |

```kotlin
// 사용 예시 (비밀번호 변경 전 재인증)
viewModelScope.launch {
    when (sdk.authManager.verifyCurrentPassword("currentPass!")) {
        is Result.Success -> { /* 인증 성공 → navigateToEditProfile() 진입 */ }
        is Result.Error -> { /* 비밀번호 불일치 안내 */ }
    }
}
```

---

#### `changePassword(currentPassword: String, newPassword: String)`

현재 비밀번호를 새 비밀번호로 변경합니다. 내부적으로 현재 비밀번호 검증 후 변경을 진행합니다.

```kotlin
suspend fun changePassword(currentPassword: String, newPassword: String): Result<Unit>
```

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `currentPassword` | `String` | 현재 비밀번호 |
| `newPassword` | `String` | 변경할 새 비밀번호 |

```kotlin
// 사용 예시
viewModelScope.launch {
    when (sdk.authManager.changePassword("currentPass!", "newPass!")) {
        is Result.Success -> { /* 변경 완료 안내 */ }
        is Result.Error -> { /* 현재 비밀번호 불일치 또는 에러 안내 */ }
    }
}
```

---

#### `deleteAccount(password: String)`

현재 계정을 탈퇴 처리합니다. 탈퇴 후 서버 데이터가 삭제될 수 있으므로 사용자에게 충분한 안내 후 호출하세요.

```kotlin
suspend fun deleteAccount(password: String): Result<Unit>
```

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `password` | `String` | 탈퇴 확인용 현재 비밀번호 |

```kotlin
// 사용 예시
viewModelScope.launch {
    when (sdk.authManager.deleteAccount("currentPass!")) {
        is Result.Success -> { /* 탈퇴 완료 → 로그인 화면 이동 */ }
        is Result.Error -> { /* 비밀번호 불일치 또는 에러 안내 */ }
    }
}
```

---

#### `saveExerciseInfo(info: ExerciseInfo)`

운동 속도, 경사, 시간 등 사용자 운동 설정을 로컬에 저장합니다. 저장 후 `savedExerciseInfo` Flow에 새 값이 방출됩니다.

```kotlin
suspend fun saveExerciseInfo(info: ExerciseInfo)
```

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `info` | `ExerciseInfo` | 저장할 운동 설정. 상세 필드는 [6.2 운동 및 보행 관련](#62-운동-및-보행-관련) 참고 |

```kotlin
// 사용 예시
val exerciseInfo = ExerciseInfo(speed = 3.5f, incline = 0f, duration = 10, indexFoot = "left", autoSave = true)
viewModelScope.launch {
    sdk.authManager.saveExerciseInfo(exerciseInfo)
}
```

---

## 4. GaitAnalysisManager

보행 분석 세션을 제어하고, 실시간 보행 데이터 스트림을 제공합니다. BLE 기기가 연결된 상태에서 사용해야 합니다.

### 4.1 프로퍼티 (Properties)

| 프로퍼티 | 타입 | 설명 |
|----------|------|------|
| `gaitMetrics` | `StateFlow<GaitMetrics>` | 실시간 보행 분석 통계 데이터 스트림. 걸음 수, 케이던스, Median/IQR 등 주요 지표가 실시간으로 업데이트됩니다. |
| `elapsedTime` | `StateFlow<Long>` | 운동 시작 후 경과 시간 (단위: 초) |
| `remainingTime` | `StateFlow<Long>` | `ExerciseInfo.duration` 기준 남은 운동 시간 (단위: 초) |
| `rawGaitStream` | `Flow<List<Float>>` | 실시간 그래프 렌더링용 원본 센서 데이터 스트림 |
| `leftStatStream` | `Flow<List<Stats>>` | 왼발 Median·IQR 그래프용 통계 데이터 스트림 |
| `rightStatStream` | `Flow<List<Stats>>` | 오른발 Median·IQR 그래프용 통계 데이터 스트림 |
| `indexingFailEvent` | `SharedFlow<Unit>` | 인덱스 워킹 실패 이벤트. 수신 시 사용자에게 재시도 안내가 필요합니다. |
| `indexingSuccessEvent` | `SharedFlow<Unit>` | 인덱스 워킹 성공 이벤트. 수신 시 운동 본 세션으로 진입 가능합니다. |

### 4.2 Methods

---

#### `start(exerciseInfo, onExerciseFinished, startIndexingImmediately)`

보행 분석 세션을 시작합니다. 인덱스 워킹(기준 보행 측정)을 선행한 뒤 본 운동 타이머를 시작합니다.

```kotlin
fun start(
    exerciseInfo: ExerciseInfo,
    onExerciseFinished: () -> Unit,
    startIndexingImmediately: Boolean = true
)
```

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `exerciseInfo` | `ExerciseInfo` | 운동 속도, 경사, 시간 등 운동 설정 정보 |
| `onExerciseFinished` | `() -> Unit` | `duration` 시간이 모두 경과했을 때 호출되는 콜백 |
| `startIndexingImmediately` | `Boolean` | `true`: 즉시 인덱스 워킹 시작 / `false`: `startIndexing()` 수동 호출 필요. 기본값 `true` |

```kotlin
// 사용 예시
sdk.gaitAnalysisManager.start(
    exerciseInfo = ExerciseInfo(speed = 3.5f, incline = 0f, duration = 10, indexFoot = "left", autoSave = true),
    onExerciseFinished = {
        // 운동 시간 종료 → 결과 화면으로 이동
        viewModelScope.launch { navigateToResult() }
    },
    startIndexingImmediately = true
)
```

---

#### `startIndexing(exerciseInfo, startExerciseTimerWithDelay)`

인덱스 워킹을 시작하거나 재시도합니다. `start()`에서 `startIndexingImmediately = false`로 설정한 경우, 이 함수를 직접 호출하여 인덱스 워킹을 시작합니다. `indexingFailEvent` 수신 후 재시도할 때도 사용합니다.

```kotlin
fun startIndexing(
    exerciseInfo: ExerciseInfo,
    startExerciseTimerWithDelay: Boolean = false
)
```

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `exerciseInfo` | `ExerciseInfo` | 운동 설정 정보 |
| `startExerciseTimerWithDelay` | `Boolean` | `true`: 인덱스 워킹 완료 후 2초 뒤에 타이머가 시작됩니다. 기본값 `false` |

> **상세 사용 시점**
> * 지연 시작 필요 시: `start()` 호출 시 `startIndexingImmediately = false`로 설정하고, 특정 워밍업 세션이 끝난 뒤 수동으로 시작할 때.
> * 인덱스 워킹 재시도: `indexingFailEvent`를 수신하여 분석기를 초기화하고 다시 측정을 시작해야 할 때.
> * UX 가이드 (Delay 파라미터): 인덱스 워킹 성공 직후 "이제 본 운동을 시작합니다"와 같은 안내 오버레이를 UI에 노출할 시간이 필요한 경우 true로 설정해주세요.

```kotlin
// 사용 예시 (인덱스 워킹 실패 후 재시도)
viewModelScope.launch {
    sdk.gaitAnalysisManager.indexingFailEvent.collect {
        // 재시도 버튼 노출 → 사용자가 재시도 클릭 시
        sdk.gaitAnalysisManager.startIndexing(exerciseInfo)
    }
}
```

---

#### `stop()`

보행 분석 세션을 중지하고 최종 집계된 `GaitMetrics`를 반환합니다. 반환된 값을 `GaitRecordManager.recordAndSyncGait()`에 전달하여 기록을 저장합니다.

```kotlin
fun stop(): GaitMetrics
```

| 반환 | 설명 |
|------|------|
| `GaitMetrics` | 세션 전체의 최종 보행 분석 결과 |

```kotlin
// 사용 예시 (사용자가 종료 버튼 클릭 시)
val finalMetrics = sdk.gaitAnalysisManager.stop()
// 이후 recordAndSyncGait()에 전달
```

---

#### `reset()`

Median, IQR 등 보행 분석 통계 데이터만 초기화합니다. 연결 상태나 세션 자체는 유지됩니다. 동일 세션 내에서 Median, IQR 측정 데이터만 리셋하고 싶을 때 사용합니다.

```kotlin
fun reset()
```

```kotlin
// 사용 예시
sdk.gaitAnalysisManager.reset()
```

---

## 5. GaitRecordManager

보행 데이터의 로컬 저장, 서버 동기화, 기록 조회 및 삭제를 관리합니다.

### 5.1 Methods

---

#### `recordAndSyncGait(...)`

`GaitAnalysisManager.stop()`으로 얻은 최종 메트릭을 로컬에 저장하고, 서버 동기화를 시도합니다. 네트워크 오류로 동기화 실패 시 로컬에만 저장되며, 이후 `syncPendingRecords()`로 재시도할 수 있습니다.

```kotlin
suspend fun recordAndSyncGait(
    metrics: GaitMetrics,
    exerciseInfo: ExerciseInfo,
    userId: String,
    date: String,
    sessionCount: Int,
    elapsedTime: Long,
    sessionSegments: List<SessionSegment>?,
    description: String?,
    extraMetadata: String?,
): Result<GaitRecordDto>
```

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `metrics` | `GaitMetrics` | `stop()`에서 반환된 최종 보행 분석 결과 |
| `exerciseInfo` | `ExerciseInfo` | 해당 세션의 운동 설정 정보 |
| `userId` | `String` | 현재 로그인된 사용자 ID (`currentUserId.value` 활용 권장) |
| `date` | `String` | 운동 날짜 (`"yyyy-MM-dd"` 형식, 예: `"2026-05-04"`) |
| `sessionCount` | `Int` | 해당 날짜의 운동 세션 횟수 |
| `elapsedTime` | `Long` | 실제 운동 경과 시간 (단위: 초) |
| `sessionSegments` | `List<SessionSegment>?` | 세션 구간 정보. 없으면 `null` 또는 `emptyList()` 전달 |
| `description` | `String?` | 해당 보행 기록에 남길 메모나 설명 (선택 사항) |
| `extraMetadata` | `String?` | 커스텀 JSON 문자 (선택 사항) |

```kotlin
// 사용 예시
val finalMetrics = sdk.gaitAnalysisManager.stop()
viewModelScope.launch {
    when (val result = sdk.gaitRecordManager.recordAndSyncGait(
        metrics = finalMetrics,
        exerciseInfo = exerciseInfo,
        userId = sdk.authManager.currentUserId.value ?: return@launch,
        date = "2026-05-04",
        sessionCount = 1,
        elapsedTime = 600L,
        sessionSegments = null,
        description = "오전 재활 훈련 세션",
        extraMetadata = """{"partner_id": "ABC-1234", "score": 95}"""
    )) {
        is Result.Success -> { /* 저장 및 동기화 성공 */ }
        is Result.Error -> { /* 로컬 저장은 완료, 서버 동기화 실패 안내 */ }
    }
}
```

---

#### `syncPendingRecords()`

네트워크 오류 등으로 서버에 전송되지 못하고 로컬에 남아있는 기록들을 서버로 재전송합니다. 앱 실행 시 또는 네트워크 연결 복구 시점에 호출하는 것을 권장합니다.

```kotlin
suspend fun syncPendingRecords(): Result<Int>
```

| 반환 | 설명 |
|------|------|
| `Result.Success(Int)` | 동기화 성공한 기록 건수 |
| `Result.Error` | 동기화 실패 |

```kotlin
// 사용 예시 (앱 시작 시 미전송 기록 재전송)
viewModelScope.launch {
    when (val result = sdk.gaitRecordManager.syncPendingRecords()) {
        is Result.Success -> { /* ${result.data}건 동기화 완료 */ }
        is Result.Error -> { /* 동기화 실패, 다음 기회에 재시도 */ }
    }
}
```

---

#### `listRecords()`

로컬에 저장된 모든 운동 기록의 목록을 불러옵니다. 상세 데이터(Metrics)가 아닌 목록 표시용 메타 정보(`SessionInfo`)를 반환합니다.

```kotlin
suspend fun listRecords(): Result<List<SessionInfo>>
```

| 반환 | 설명 |
|------|------|
| `Result.Success<List<SessionInfo>>` | 저장된 운동 기록 목록. 비어있을 수 있음 |
| `Result.Error` | 조회 실패 |

```kotlin
// 사용 예시
viewModelScope.launch {
    when (val result = sdk.gaitRecordManager.listRecords()) {
        is Result.Success -> {
            val records = result.data // List<SessionInfo>
        }
        is Result.Error -> { /* 에러 처리 */ }
    }
}
```

---

#### `loadRecord(fileName: String)`

파일명으로 특정 운동 세션의 상세 보행 분석 결과(`GaitMetrics`)를 불러옵니다.

```kotlin
suspend fun loadRecord(fileName: String): Result<GaitMetrics?>
```

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `fileName` | `String` | `listRecords()`로 얻은 `SessionInfo`의 `fileName` |

| 반환 | 설명 |
|------|------|
| `Result.Success<GaitMetrics>` | 해당 세션의 보행 분석 상세 데이터 |
| `Result.Success(null)` | 파일을 찾을 수 없음 |
| `Result.Error` | 로드 실패 |

---

#### `loadRecordMetaData(fileName: String)`

파일명으로 특정 운동 세션의 메타 데이터(`GaitRecordDto`)를 불러옵니다. 날짜, 운동 설정 등 기록 요약 정보가 필요할 때 사용합니다.

```kotlin
suspend fun loadRecordMetaData(fileName: String): Result<GaitRecordDto?>
```

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `fileName` | `String` | `listRecords()`로 얻은 `SessionInfo`의 파일명 |

---

#### `deleteRecord(fileName: String)`

파일명으로 특정 운동 세션 기록을 로컬에서 삭제합니다.

```kotlin
suspend fun deleteRecord(fileName: String): Result<Unit>
```

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `fileName` | `String` | 삭제할 세션의 파일명 |

```kotlin
// 사용 예시
viewModelScope.launch {
    when (sdk.gaitRecordManager.deleteRecord("session_20260504_001.json")) {
        is Result.Success -> { /* 삭제 완료 → 목록 갱신 */ }
        is Result.Error -> { /* 삭제 실패 안내 */ }
    }
}
```

---

## 6. 데이터 타입 정의

SDK에서 사용하는 주요 데이터 클래스 및 열거형(Enum) 정의입니다. `@Serializable`이 붙은 클래스는 JSON 직렬화를 지원하여 로컬 저장 및 서버 통신에 활용됩니다.

### 6.1 BLE 관련

---

#### `ScannedDevice`

`scan()`으로 발견된 BLE 기기를 나타냅니다. `connect()`에 그대로 전달하여 연결에 사용합니다.

```kotlin
data class ScannedDevice(
    val name: String,    // 기기 이름 (예: "EL-001")
    val address: String  // BLE MAC 주소 (예: "AA:BB:CC:DD:EE:FF")
)
```

---

#### `BleConnectionState`

BLE 연결 상태를 나타내는 열거형입니다. `connectionState` StateFlow를 구독하여 UI를 업데이트할 때 사용합니다.

```kotlin
enum class BleConnectionState {
    CONNECTED,     // 연결 완료
    DISCONNECTED,  // 연결 해제 상태
    CONNECTING,    // 연결 시도 중
    ERROR          // 연결 오류 발생
}
```

```kotlin
// 사용 예시 (Compose에서 연결 상태 UI 반영)
val connectionState by sdk.bleManager.connectionState.collectAsState()

when (connectionState) {
    BleConnectionState.CONNECTED -> Text("연결됨")
    BleConnectionState.CONNECTING -> CircularProgressIndicator()
    BleConnectionState.DISCONNECTED -> Text("연결 안됨")
    BleConnectionState.ERROR -> Text("연결 오류")
}
```

---

### 6.2 운동 및 보행 관련

---

#### `ExerciseInfo`

보행 분석 세션의 운동 설정 정보를 담는 클래스입니다. `start()`, `recordAndSyncGait()`, `saveExerciseInfo()` 등 여러 함수에서 사용됩니다.

```kotlin
@Serializable
data class ExerciseInfo(
    val speed: Float = 0f,           // 트레드밀 속도 (km/h)
    val incline: Float = 0f,         // 트레드밀 경사도 (%)
    val duration: Int = 0,           // 운동 목표 시간 (분)
    val indexFoot: String = "left",  // 인덱스 워킹 기준 발 ("left" 또는 "right")
    val autoSave: Boolean = false    // 운동 종료 시 자동 저장 여부
    val mood: String = "",           // 사용자 무드
    val exerciseMode: ExerciseMode? = null  // 운동 모
)
```

---

#### `GaitMetrics`

보행 분석 세션의 통계 결과를 담는 클래스입니다. `stop()`의 반환값이자 `recordAndSyncGait()`의 입력값입니다. 실시간 업데이트는 `gaitMetrics` StateFlow를 통해 수신합니다.

```kotlin
@Serializable
data class GaitMetrics(
    val totalSteps: Int,                       // 총 걸음 수
    val cadence: Int,                          // 분당 걸음 수 (steps/min)
    val overallMedian: Double,                 // 전체 보행 Median 값
    val overallIqr: Double,                    // 전체 보행 IQR 값
    val stepTypeStats: StepTypeStatistics,     // 보행 유형별 통계
    val footIntensity: FootIntensityStats,     // 발 강도 통계
    val stepDuration: StepDurationStats,       // 보폭 시간(한 발→다음 발)의 평균과 안정성을 나타내는 통계
    val strideDuration: StrideDurationStats,   // 활보 시간(한 발→같은 발)의 평균과 안정성을 나타내는 통계
    val lastStepType: StepType = StepType.NONE,      // 가장 최근에 감지된 걸음의 종류 (걷기, 뛰기, 미분류 등)
    val leftMedianData: List<Float> = emptyList(),   // 왼발 Median 시계열 데이터 (그래프용)
    val leftIqrData: List<Float> = emptyList(),      // 왼발 IQR 시계열 데이터 (그래프용)
    val rightMedianData: List<Float> = emptyList(),  // 오른발 Median 시계열 데이터 (그래프용)
    val rightIqrData: List<Float> = emptyList(),      // 오른발 IQR 시계열 데이터 (그래프용)
    val fullEnvelopeHistory: List<Int> = emptyList(), // 전체 운동 envelope 데이터 (서버 전송용)
    val stepLength: Double = 0.0,             // 보폭 길이 (cm)
)
```
---

#### `SessionInfo`

로컬 파일로 저장된 개별 운동 세션의 요약 정보입니다. 저장된 파일 목록을 UI에 리스트로 표시할 때 사용합니다.

```kotlin
data class SessionInfo(
    val fileName: String,      // 저장된 파일명 (예: "step_stats_20260506_1.json")
    val date: String,          // 운동 날짜 (yyyyMMdd)
    val session: Int,          // 해당 날짜의 세션 번호
    val elapsedTime: Long      // 실제 운동 수행 시간 (초)
)
```
---

#### `GaitRecordDto`

서버와의 데이터 동기화 또는 상세 기록 조회를 위해 운동 설정과 기록 요약을 결합한 데이터 전송 객체(DTO)입니다.

```kotlin
@Serializable
data class GaitRecordDto(
    val fileName: String,            // 파일명
    val userId: String,              // 사용자 아이디
    val date: String,                // 운동 날짜
    val sessionCount: Int,           // 세션 순번
    val exerciseInfo: ExerciseInfo,  // 운동 당시 설정 값
    val elapsedTime: Long,           // 실제 수행 시간 (초)
    val extraMetadata: String? = null  // 커스텀 JSON 문자열
)
```
---

#### `ExerciseMode` (Enum)
현재 수행 중인 운동의 모드를 정의합니다.

| 상수 | 의미 |
|----------|------|
| `INDEX` | 인덱스 워킹 |
| `STATIC` | 표준 보행 |
| `GAME` | 게임 모드 |

```kotlin
@Serializable
enum class ExerciseMode {
    INDEX, STATIC, GAME
}
```

#### `SessionSegment`
하나의 운동 세션 내에서 각 모드별로 소요된 시간 구간을 기록하는 클래스입니다.

```kotlin
@Serializable
data class SessionSegment(
    val startTime: Long,      // 구간 시작 타임스탬프
    var endTime: Long? = null, // 구간 종료 타임스탬프
    val mode: ExerciseMode    // 해당 구간의 운동 모드
)
```

---

### 6.3 사용자 관련

---

#### `NewUserInfo`

회원가입 시 필요한 신규 사용자 정보입니다. 모든 필드가 필수값입니다.

```kotlin
data class NewUserInfo(
    val id: String,         // 사용자 아이디
    val pass: String,       // 비밀번호
    val pwhint: String,     // 비밀번호 찾기 힌트 질문
    val pwhintAns: String,  // 힌트 답변
    val name: String,       // 이름
    val gender: String,     // 성별 ("M" 또는 "F")
    val birthday: String,   // 생년월일 ("yyyyMMdd" 형식, 예: "19900101")
    val phone: String,      // 전화번호 ("-" 없이, 예: "01012345678")
    val height: Float,      // 키 (cm)
    val weight: Float       // 체중 (kg)
)
```

> **📌 비밀번호 힌트(pwhint) 매핑 테이블**
> 
> 서버 및 SDK와의 통신 정합성을 위해, 사용자에게 보여지는 텍스트 대신 아래의 고정된 상수값을 전송해야 합니다.

| UI 표시 문구 (예시) | 전송해야 할 값 (String) |
| :--- | :--- |
| 좋아하는 색깔은? | `HINT_FAVORITE_COLOR` |
| 어린 시절 별명은? | `HINT_CHILDHOOD_NICKNAME` |
| 가장 좋아하는 음식은? | `HINT_FAVORITE_FOOD` |
| 졸업한 초등학교 이름은? | `HINT_ELEMENTARY_SCHOOL` |
| 내가 태어난 도시(고향)는? | `HINT_BIRTHPLACE` |
| 가장 기억에 남는 여행지는? | `HINT_MEMORABLE_TRIP` |
| 아버지의 성함은? | `HINT_FATHER_NAME` |


---

### 6.4 공통 타입

---

#### `Result<T>`

SDK의 모든 비동기 함수가 반환하는 공통 래퍼 타입입니다. `Result.Success`와 `Result.Error`로 성공/실패를 분기 처리합니다.

```kotlin
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val exception: Throwable) : Result<Nothing>()
}
```

```kotlin
// 일반적인 분기 처리 패턴
when (val result = sdk.someManager.someFunction()) {
    is Result.Success -> {
        val data = result.data  // 성공 데이터 접근
    }
    is Result.Error -> {
        val message = result.exception.message  // 에러 메시지 접근
        Log.e("SDK", "에러: $message")
    }
}
```

---

## 7. 전체 사용 흐름 예시

초기화부터 운동 기록 저장까지의 일반적인 워크플로우 예시입니다.

```kotlin
// 1. SDK 초기화 (Splash 또는 MainActivity)
Elegaiter.init(context, apiKey = "YOUR_API_KEY") { success, resultType, message ->
    // 이 콜백은 메인(UI) 스레드에서 호출됩니다.
    if (!success) return@init

    val sdk = Elegaiter.getInstance()

    lifecycleScope.launch {

        // 2. 자동 로그인 확인
        when (val authResult = sdk.authManager.checkAuthStatus()) {
            is Result.Success -> {
                if (!authResult.data) {
                    navigateToLogin()
                    return@launch
                }
            }
            is Result.Error -> return@launch
        }

        // 3. BLE 스캔 및 연결 (권한 허용 후 호출)
        val scanResult = sdk.bleManager.scan()
        if (scanResult is Result.Error) return@launch

        val device = (scanResult as Result.Success).data
            .firstOrNull { it.name.startsWith("EL") } ?: return@launch

        if (sdk.bleManager.connect(device) is Result.Error) return@launch

        // 4. 보행 분석 시작
        val exerciseInfo = ExerciseInfo(
            speed = 3.5f,
            incline = 0f,
            duration = 10,
            indexFoot = "left",
            autoSave = true,
            exerciseMode = ExerciseMode.STATIC
        )

        sdk.gaitAnalysisManager.start(
            exerciseInfo = exerciseInfo,
            onExerciseFinished = {
                viewModelScope.launch { navigateToResult() }
            },
            startIndexingImmediately = true
        )

        // 5. 실시간 보행 데이터 구독 (별도 코루틴)
        launch {
            sdk.gaitAnalysisManager.gaitMetrics.collect { metrics ->
                // UI 업데이트 (걸음 수, 케이던스 등)
            }
        }

        // (사용자가 중지 버튼 클릭 시)

        // 6. 운동 종료 및 기록 저장
        val finalMetrics = sdk.gaitAnalysisManager.stop()

        sdk.gaitRecordManager.recordAndSyncGait(
            metrics = finalMetrics,
            exerciseInfo = exerciseInfo,
            userId = sdk.authManager.currentUserId.value ?: return@launch,
            date = "2026-05-04",
            sessionCount = 1,
            elapsedTime = 600L,
            sessionSegments = null,
            description = null,
            extraMetadata = null
        )

        // 7. BLE 연결 해제
        sdk.bleManager.disconnect()
    }
}
```

---

## 8. 문서 및 지원

- **설치 및 기본 사용**: [Elegaiter SDK 연동 가이드 (README)](../README.md)
- **문의**: SDK 사용 문의, API Key 발급, 기술 지원 — **주식회사 사이클룩스 기술연구소**
