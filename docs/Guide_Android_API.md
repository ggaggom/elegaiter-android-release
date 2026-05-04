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

## 2. BleManager
JAWS 센서와의 BLE 연결 및 통신을 관리합니다. 스캔, 연결, Threshold 설정, 에러 코드 전송을 담당합니다.

### 2.1 프로퍼티 (Properties)
| 프로퍼티 | 타입 | 설명 |
|----------|------|------|
| `connectionState` | `StateFlow<BleConnectionState>` | 현재 BLE 연결 상태 |
| `thresholdResponse` | `SharedFlow<Int>` | Threshold 응답 스트림 |

### 2.2 Methods
**`scan()`**
주변의 Elegaiter 장비를 스캔합니다.
* **Signature:** `suspend fun scan(): Result<List<ScannedDevice>>`

**`connect(device: ScannedDevice)`**
특정 장비에 연결을 시도합니다.
* **Signature:** `suspend fun connect(device: ScannedDevice): Result<Unit>`

**`disconnect()`**
현재 연결을 해제합니다.
* **Signature:** `fun disconnect()`

**`checkThreshold()`**
현재 장치에 설정된 Threshold 값을 확인하는 명령을 전송합니다.
* **Signature:** `suspend fun checkThreshold(): Result<Unit>`
* **Note:** 응답 값은 `thresholdResponse` (SharedFlow)를 통해 전달됩니다.

**`setThreshold(level: Int)`**
Threshold 값을 수동으로 설정합니다.
* **Signature:** `suspend fun setThreshold(level: Int): Result<Unit>`
* **Parameter:** `level` - 설정하고자 하는 Threshold 레벨 수치

**`setAutoThreshold()`**
Threshold 값을 자동으로 설정하는 명령(T0 프로토콜)을 전송합니다.
* **Signature:** `suspend fun setAutoThreshold(): Result<Unit>`
* **Description:** 장치가 스스로 최적의 임계값을 찾도록 유도합니다.

**`sendError(errorCode: Int)`**
장치 측으로 에러 코드를 전송합니다.
* **Signature:** `suspend fun sendError(errorCode: Int): Result<Unit>`
* **Parameter:** `errorCode` - 전송할 코드 (0 또는 1)
* **Signature:** `suspend fun setThreshold(level: Int): Result<Unit>`

---

## 3. AuthManager
사용자 계정과 관련된 모든 기능을 담당합니다. 단순 로그인뿐만 아니라 세션 관리, 비밀번호 찾기, 프로필 관리 등을 포함합니다.

### 3.1 프로퍼티 (Properties)
| 프로퍼티 | 타입 | 설명 |
| :--- | :--- | :--- |
| `currentUserId` | `StateFlow<String?>` | 현재 로그인된 사용자 ID (비로그인 시 null) |
| `savedExerciseInfo` | `Flow<ExerciseInfo?>` | 저장된 운동 설정 정보의 변경 사항 감지 Flow |
| `sessionExpiredEvent` | `SharedFlow<Unit>` | 세션 만료 이벤트 (로그인 화면 이동 필요) |

### 3.2 Methods

**`checkAuthStatus()`**
현재 세션(토큰)이 유효한지 확인하고 자동 로그인을 결정합니다.
* **Signature:** `suspend fun checkAuthStatus(): Result<Boolean>`
* **Note:** `true` 반환 시 메인 화면으로, `false` 반환 시 로그인 화면으로 이동하십시오.

**`checkIdAvailability(id: String)`**
아이디 사용 가능 여부를 확인합니다.
* **Signature:** `suspend fun checkIdAvailability(id: String): Result<Boolean>`

**`register(info: NewUserInfo)`**
새로운 사용자를 등록(회원가입)합니다.
* **Signature:** `suspend fun register(info: NewUserInfo): Result<Unit>`

**`login(id: String, password: String, isAutoLogin: Boolean)`**
사용자를 로그인시킵니다.
* **Signature:** `suspend fun login(id: String, password: String, isAutoLogin: Boolean): Result<Unit>`

**`logout()`**
현재 사용자를 로그아웃시킵니다.
* **Signature:** `suspend fun logout()`

**`findId(name: String, phone: String)`**
이름과 전화번호로 아이디를 찾습니다.
* **Signature:** `suspend fun findId(name: String, phone: String): Result<String?>`

**`verifyPasswordHint(id: String, hint: String, answer: String)`**
비밀번호 찾기 1단계: 등록한 힌트와 답변을 확인합니다.
* **Signature:** `suspend fun verifyPasswordHint(id: String, hint: String, answer: String): Result<String>`
* **Returns:** 성공 시, 5분간 유효한 비밀번호 재설정용 토큰(String) 반환

**`resetLostPassword(newPassword: String, token: String)`**
비밀번호 찾기 2단계: 새 비밀번호로 재설정합니다.
* **Signature:** `suspend fun resetLostPassword(newPassword: String, token: String): Result<Unit>`

**`getUserProfile()`**
현재 로그인된 사용자의 프로필 정보를 가져옵니다.
* **Signature:** `suspend fun getUserProfile(): Result<UserProfile>`

**`updateUserProfile(updatedProfile: UserProfile)`**
현재 로그인된 사용자의 프로필 정보를 업데이트합니다.
* **Signature:** `suspend fun updateUserProfile(updatedProfile: UserProfile): Result<Unit>`

**`changePassword(currentPassword: String, newPassword: String)`**
비밀번호를 변경합니다.
* **Signature:** `suspend fun changePassword(currentPassword: String, newPassword: String): Result<Unit>`

**`deleteAccount(password: String)`**
회원에서 탈퇴합니다.
* **Signature:** `suspend fun deleteAccount(password: String): Result<Unit>`

**`saveExerciseInfo(info: ExerciseInfo)`**
운동 설정 정보를 저장합니다.
* **Signature:** `suspend fun saveExerciseInfo(info: ExerciseInfo)`

**`verifyCurrentPassword(password: String)`**
현재 비밀번호가 일치하는지(재인증) 확인합니다.
* **Signature:** `suspend fun verifyCurrentPassword(password: String): Result<Unit>`

---

## 4. GaitAnalysisManager
보행 데이터의 통계 분석 결과를 실시간으로 제공합니다.
### 4.1 프로퍼티 (Properties)
| 프로퍼티 | 타입 | 설명 |
| :--- | :--- | :--- |
| `gaitMetrics` | `StateFlow<GaitMetrics>` | 실시간 보행 분석 통계 데이터 스트림 |
| `elapsedTime` | `StateFlow<Long>` | 운동 경과 시간 (초) |
| `remainingTime` | `StateFlow<Long>` | 남은 운동 시간 (초) |
| `rawGaitStream` | `Flow<List<Float>>` | 실시간 그래프용 원본 데이터 스트림 |
| `leftStatStream / rightStatStream` | `Flow<List<Stats>>` | Median, IQR 그래프용 데이터 스트림 |
| `indexingFailEvent` | `SharedFlow<Unit>` | 인덱스 워킹 실패 이벤트 |
| `indexingSuccessEvent` | `SharedFlow<Unit>` | 인덱스 워킹 성공 이벤트 |

### 4.2 Methods
**`start(exerciseInfo, onExerciseFinished, startIndexingImmediately)`**
보행 분석 세션을 시작합니다.
* **Signature:** `fun start(exerciseInfo: ExerciseInfo, onExerciseFinished: () -> Unit, startIndexingImmediately: Boolean = true)`

**`startIndexing(exerciseInfo, startExerciseTimerWithDelay)`**
인덱스 워킹을 시작하거나 재시도합니다.
* **Signature:** `fun startIndexing(exerciseInfo: ExerciseInfo, startExerciseTimerWithDelay: Boolean = false)`

**`stop()`**
분석 세션을 중지하고 최종 보행 메트릭을 반환합니다.
* **Signature:** `fun stop(): GaitMetrics`

**`reset()`**
Median, IQR 등 분석 통계 데이터만을 리셋합니다.
* **Signature:** `fun reset()`

---

## 5. GaitRecordManager
보행 데이터 기록 및 서버 동기화를 관리합니다.
### 5.1 Methods
**`recordAndSyncGait(...)`**
데이터를 로컬에 저장하고 서버 동기화를 시도합니다.
* **Signature:** `suspend fun recordAndSyncGait(metrics: GaitMetrics, exerciseInfo: ExerciseInfo, userId: String, date: String, sessionCount: Int, elapsedTime: Long, sessionSegments: List<SessionSegment>): Result<GaitRecordDto>`

**`syncPendingRecords()`**
로컬에 남아있는 미전송 데이터를 다시 전송합니다.
* **Signature:** `suspend fun syncPendingRecords(): Result<Int>`

**`deleteRecord(fileName: String)`**
파일명으로 특정 운동 세션을 삭제합니다.
* **Signature:** `suspend fun deleteRecord(fileName: String): Result<Unit>`

**`listRecords()`**
저장된 모든 운동 기록 목록을 불러옵니다.
* **Signature:** `suspend fun listRecords(): Result<List<SessionInfo>>`

**`loadRecord(fileName: String)`**
파일명으로 특정 운동 기록의 상세 데이터(Metrics)를 로드합니다.
* **Signature:** `suspend fun loadRecord(fileName: String): Result<GaitMetrics?>`

**`loadRecordMetaData(fileName: String)`**
파일명으로 특정 운동 기록의 메타 데이터(Dto)를 로드합니다.
* **Signature:** `suspend fun loadRecordMetaData(fileName: String): Result<GaitRecordDto?>`

---

## 6. 데이터 타입 정의
안드로이드 SDK에서 사용하는 주요 데이터 클래스 및 Enum 정의입니다. 대부분의 보행 데이터 모델은 직렬화(Serialization)를 지원합니다.

### 6.1 BLE 관련
**ScannedDevice**
```kotlin
data class ScannedDevice(val name: String, val address: String)
```
**BleConnectionState**
```kotlin
enum class BleConnectionState { CONNECTED, DISCONNECTED, CONNECTING, ERROR }
```

### 6.2 운동 및 보행 관련
**GaitMetrics**
```kotlin
@Serializable
data class GaitMetrics(
    val totalSteps: Int,
    val cadence: Int,
    val overallMedian: Double,
    val overallIqr: Double,
    val stepTypeStats: StepTypeStatistics,
    val footIntensity: FootIntensityStats,
    val stepDuration: StepDurationStats,
    val strideDuration: StrideDurationStats,
    val leftMedianData: List<Float> = emptyList(),
    val rightMedianData: List<Float> = emptyList()
)
```

**ExerciseInfo**
```kotlin
@Serializable
data class ExerciseInfo(
    val speed: Float = 0f,
    val incline: Float = 0f,
    val duration: Int = 0,
    val indexFoot: String = "left",
    val autoSave: Boolean = false
)
```

### 6.3 사용자 관련
**NewUserInfo**
```kotlin
data class NewUserInfo(
    val id: String,
    val pass: String,
    val pwhint: String,
    val pwhintAns: String,
    val name: String,
    val gender: String,
    val birthday: String,
    val phone: String,
    val height: Float,
    val weight: Float
)
```

### 6.4 공통 타입
**Result**
```kotlin
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val exception: Throwable) : Result<Nothing>()
}
```

---

## 7. 전체 사용 흐름 예시
안드로이드 환경에서는 **Kotlin Coroutines**를 사용하여 비동기 흐름을 관리합니다. 다음은 기본적인 초기화부터 운동 저장까지의 워크플로우 예시입니다.
```kotlin
import androidx.lifecycle.lifecycleScope
import kotlinx.coroutines.launch

// 1. SDK 초기화
Elegaiter.initialize(context, apiKey = "YOUR_API_KEY") { result ->
    if (result != ElegaiterInitResult.SUCCESS) return@initialize
    
    val sdk = Elegaiter.getInstance()

    // 2. 비동기 작업 시작 (Activity/ViewModel Scope)
    lifecycleScope.launch {
        
        // 3. BLE 스캔 및 연결
        val scanResult = sdk.bleManager.scan()
        if (scanResult is Result.Success) {
            val device = scanResult.data.firstOrNull() ?: return@launch
            
            val connectResult = sdk.bleManager.connect(device)
            if (connectResult is Result.Error) return@launch
        }

        // 4. 로그인
        val loginResult = sdk.authManager.login(id = "user", password = "pass", isAutoLogin = true)
        if (loginResult is Result.Error) return@launch

        // 5. 보행 분석 시작 (측정 설정)
        val exerciseInfo = ExerciseInfo(
            speed = 3.0f, 
            incline = 0f, 
            duration = 600, 
            indexFoot = "left", 
            autoSave = true
        )
        
        sdk.gaitAnalysisManager.start(
            exerciseInfo = exerciseInfo,
            onExerciseFinished = {
                // 운동 타이머 종료 시 처리 로직
            },
            startIndexingImmediately = true
        )

        // (실제 운동 진행... 사용자가 중지 버튼을 누른 경우)

        // 6. 운동 종료 및 최종 메트릭 획득
        val finalMetrics = sdk.gaitAnalysisManager.stop()

        // 7. 데이터 기록 및 서버 동기화
        val recordResult = sdk.gaitRecordManager.recordAndSyncGait(
            metrics = finalMetrics,
            exerciseInfo = exerciseInfo,
            userId = "user",
            date = "2026-05-04",
            sessionCount = 1,
            elapsedTime = 600,
            sessionSegments = emptyList() // 세션 세그먼트가 있을 경우 전달
        )
        
        if (recordResult is Result.Success) {
            // 저장 성공 처리
        }

        // 8. BLE 연결 해제
        sdk.bleManager.disconnect()
    }
}
```

---
## 8. 문서 및 지원

- **설치 및 기본 사용**: [Elegaiter SDK 연동 가이드 (README).md](../README.md)
- **문의**: SDK 사용 문의, API Key 발급, 기술 지원 — **주식회사 사이클룩스 기술연구소**
