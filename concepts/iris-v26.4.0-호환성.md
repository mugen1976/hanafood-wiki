# Iris KakaoTalk v26.4.0 대응 — 채팅방 이름 조회 수정

> 2026-05-18, 16864133(도 인호)
> 관련: KakaoDB.kt, ObserverHelper.kt, irispy-client models.py

## 문제

KakaoTalk v26.4.0 업데이트 이후 특정 채팅방(예: 용당요양병원, room_id=434459225541519)의 **방 이름이 대화상대 이름으로 잘못 조회**됨.

Iris APK가 전송하는 WebSocket JSON의 `"room"` 필드에 실제 방 이름 대신 발신자 이름(또는 모든 대화상대 이름)이 들어감.

## 원인

### DB 연결 방식 변경

Iris의 `KakaoDB.kt`가 v26.4.0 대응 커밋(`0b300ee`)에서 DB 연결 방식을 바꿈:

```kotlin
// BEFORE (v26.3.x)
connection = SQLiteDatabase.openDatabase(
    "$DB_PATH/KakaoTalk.db", null, SQLiteDatabase.OPEN_READWRITE
)

// AFTER (v26.4.0)
connection = SQLiteDatabase.openDatabase(":memory:", null, SQLiteDatabase.OPEN_READWRITE)
connection.execSQL("ATTACH DATABASE '$DB_PATH/KakaoTalk.db' AS db1")
connection.execSQL("ATTACH DATABASE '$DB_PATH/KakaoTalk2.db' AS db2")
```

`KakaoTalk.db`가 메인 DB에서 `db1`이라는 이름의 attached database로 변경됨.  
**문제**: `chat_rooms`, `chat_logs` 테이블을 참조하는 모든 쿼리가 `db1.` prefix 없이 작성되어, 실제로는 빈 `:memory:` DB에서 조회를 시도 → 실패.

### getChatInfo() 폴백

`KakaoDB.getChatInfo()`는 방 이름 조회에 실패하면 `return arrayOf(sender, sender)`로 폴백함.  
즉, 발신자 이름을 방 이름으로 사용 → 모든 메시지가 발신자명을 방 이름으로 표시.

### 추가 문제점

- `private_meta` JSON 구조가 v26.4.0에서 변경되었을 가능성 존재 (`name` 키 대신 `title`, `chatRoomName` 등)
- `NamesDB` 폴백 조건이 `senderName.isNullOrEmpty()`만 검사 → `roomName`이 null이어도 NamesDB에서 보충 안 함

## 수정 사항

### 1. Iris APK (Kotlin, `/root/Iris/`)

#### KakaoDB.kt

| 위치 | 수정 전 | 수정 후 |
|------|---------|---------|
| `botUserId` | `FROM chat_logs` | `FROM db1.chat_logs` |
| `getChatInfo()` private_meta | `FROM chat_rooms` | `FROM db1.chat_rooms` |
| `getChatInfo()` open_link | `link_id FROM chat_rooms` | `link_id FROM db1.chat_rooms` |
| `getChatInfo()` JSON 키 | `meta["name"]`만 시도 | `name, title, chatRoomName, roomName, chat_name` 순차 시도 |
| `logToDict` | `FROM chat_logs` | `FROM db1.chat_logs` |

#### ObserverHelper.kt

| 위치 | 수정 전 | 수정 후 |
|------|---------|---------|
| `checkChange()` | `FROM chat_logs` | `FROM db1.chat_logs` |
| `getNewLogCountFromDB()` | `FROM chat_logs` | `FROM db1.chat_logs` |
| NamesDB 폴백 조건 | `senderName.isNullOrEmpty()` | `roomName.isNullOrEmpty() \|\| senderName.isNullOrEmpty()` |

### 2. irispy-client (Python, `venv/lib/python3.12/site-packages/iris/bot/models.py`)

`Room.name`을 일반 인스턴스 속성에서 **`@property`**로 변경:

```python
@property
def name(self) -> str:
    # 최초 접근 시 Iris /query API로 DB 조회
    # 1. db1.chat_rooms.private_meta (JSON, 여러 키 시도)
    # 2. db2.chat_rooms.private_meta (v26.4.0 변형 대응)
    # 3. db2.open_link (오픈채팅방)
    # 전부 실패하면 Iris 서버가 보내준 초기값 사용
```

추가로 `Room.type`, `User.name`, `User.type`, `Avatar.url` 등의 `chat_rooms` 참조 쿼리에 `db1.` prefix 적용.

### 3. 패치 파일

`patches/irispy-client-room-name-fix.patch` — venv 재설치 시 아래 명령어로 재적용:

```bash
cd /root/hana_bot
patch -p0 < patches/irispy-client-room-name-fix.patch
```

## APK 빌드 및 배포

### 빌드 환경

| 항목 | 값 |
|------|-----|
| JDK | openjdk-17.0.18 |
| compileSdk | 35 |
| build-tools | 35.0.0 |
| AGP | 8.8.2 |
| Kotlin | 2.1.10 |
| ANDROID_HOME | /opt/android-sdk |
| 출력 APK | `output/Iris-debug.apk` (~7.9MB) |

### 빌드 명령

```bash
cd /root/Iris
export ANDROID_HOME=/opt/android-sdk
export ANDROID_SDK_ROOT=/opt/android-sdk
./gradlew assembleDebug
```

### 배포 (redroid)

```bash
# APK 푸시
adb -s emulator-5554 push output/Iris-debug.apk /data/local/tmp/Iris.apk

# MD5 검증
md5sum output/Iris-debug.apk
adb -s emulator-5554 shell md5sum /data/local/tmp/Iris.apk

# 기존 프로세스 종료
adb -s emulator-5554 shell ps -ef | grep app_process
adb -s emulator-5554 shell su root kill -9 <PID>

# 재시작 (startup script)
adb -s emulator-5554 shell /data/local/tmp/start_iris_root.sh

# 포트 확인
docker exec redroid ss -tlnp | grep 3000
```

## 관련 저장소

- **Iris APK** — `git@github.com:mugen1976/Iris.git` (`/root/Iris/`)
- **hana_bot** — `git@github.com:mugen1976/hana_bot.git` (`/root/hana_bot/`)
- **Iris upstream** — `https://github.com/dolidolih/Iris`

## 참고

- KakaoTalk v26.4.0 대응 PR: dolidolih/Iris#124, #125
- v26.4.0 커밋: `0b300ee` (ye-seola), `748b29d` (dolidolih merge)
- irispy-client 버전: 0.2.4
