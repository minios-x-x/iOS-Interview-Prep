# iOS 앱에서 로컬 푸시 알림(Local Push Notification)을 구현하는 방법은 무엇인가요?

# 1. 로컬 알림 구현 흐름

로컬 알림은 **앱이 직접 알림을 예약하면, 그 시점에 시스템이 알림을 띄워주는** 방식이다. 이 글에서는 "할 일 리마인더"를 예시로 쓴다. 구현은 다섯 단계다.

## 1-1. 권한 요청

사용자가 허용해야 알림을 띄울 수 있다.

```swift
let center = UNUserNotificationCenter.current()
let granted = try await center.requestAuthorization(options: [.alert, .sound, .badge])
```

현재 권한 상태는 `getNotificationSettings()`로 확인할 수 있다. 사용자가 거부했다면 알림이 표시되지 않으므로, 설정 화면으로 안내하는 흐름이 필요하다.

## 1-2. 내용(Content) 만들기

```swift
let content = UNMutableNotificationContent()
content.title = "리마인더"
content.body = "보고서 제출하기"
content.sound = .default
content.userInfo = ["taskID": "task-42"]   // 탭했을 때 어떤 할 일인지 알기 위한 정보
```

## 1-3. 트리거(Trigger) 정하기

```swift
let trigger = UNTimeIntervalNotificationTrigger(timeInterval: 60 * 30, repeats: false)   // 30분 뒤
```

## 1-4. 요청(Request)으로 묶어서 등록

```swift
let request = UNNotificationRequest(identifier: "task-42", content: content, trigger: trigger)
try await center.add(request)
```

## 1-5. 델리게이트 설정

앱이 켜져 있을 때 알림을 보여줄지, 알림을 탭했을 때 무엇을 할지는 `UNUserNotificationCenterDelegate`로 처리한다. 앱이 launch를 끝내기 전에 설정해야 한다. (4장에서 자세히 다룬다.)

```swift
func application(_ application: UIApplication,
                 didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    UNUserNotificationCenter.current().delegate = self
    return true
}
```

# 2. 로컬 푸시 알림과 원격 푸시 알림의 차이점

## 2-1. 한눈에 비교

| | 로컬 알림 | 원격 푸시 |
|---|---|---|
| 알림을 만드는 주체 | **앱이 기기 안에서** 예약 | 서버 → APNs(Apple 푸시 서버) → 기기 |
| 서버/네트워크 | 불필요 | 필요 |
| 필요한 설정 | 알림 권한 | 알림 권한 + Push Notifications capability + **디바이스 토큰 등록** + 서버 연동 |
| 발송 시점 | 앱이 **미리** 정함 (시간/캘린더/위치) | 서버가 **그때그때** 정함 |
| 앱이 꺼져 있을 때 | **미리 예약해둔 것**은 울림 | 서버가 보내면 울림 |
| 적합한 용도 | 리마인더, 타이머, 알람 | 새 메시지, 마케팅 등 서버에서 생기는 이벤트 |

핵심은 **발송 시점을 앱이 미리 알 수 있는가**다. 시간이나 장소처럼 앱이 미리 정할 수 있으면 로컬 알림이고, 서버에서 일어나는 일(상대방의 메시지)이면 원격 푸시다. 앱이 꺼져 있으면 임의의 순간에 알림을 만들 수 없기 때문에, 로컬 알림은 **미리 예약**해둬야 한다. 또 로컬 알림은 시스템이 **가장 빨리 울릴 64개만** 유지하는 제한이 있다.

## 2-2. 원격 푸시의 구조

원격 푸시는 페이로드(`aps`)로 내용이 전달된다. 서버가 APNs로 보내려면 앱이 `registerForRemoteNotifications()`로 받은 디바이스 토큰이 필요하다.

```json
{
  "aps": {
    "alert": { "title": "새 메시지", "body": "안녕하세요" },
    "sound": "default",
    "mutable-content": 1
  }
}
```

## 2-3. 원격 푸시 전용 — Notification Service Extension

`mutable-content: 1`을 주면, 앱과 **별개의 프로세스**로 동작하는 **Notification Service Extension**이 알림이 표시되기 직전에 내용을 가공할 수 있다. 앱이 종료돼 있어도 알림이 표시되는 원격 푸시라면 동작한다. 로컬 알림에는 없는 원격 푸시 전용 확장 지점이다.

대표적인 용도는 **이미지 첨부**다. APNs 페이로드는 최대 4KB라서 이미지를 직접 실을 수 없으므로, 서버는 이미지 URL만 보내고 확장이 내려받아서 알림에 붙인다. 그 외에 텍스트 가공, 암호화된 내용의 복호화, 서버 API로 추가 정보 조회 같은 작업도 한다.

주의할 점은 다음과 같다.

- **시간이 짧다**(약 30초). 시간 안에 끝나지 않으면 가공 전 원본이 표시된다.
- 앱과 **메모리를 공유하지 않는다.** 앱의 데이터가 필요하면 App Group 같은 공유 저장소를 써야 한다.
- **알림을 숨기는 것은 제한**된다. iOS 13.3부터는 확장에서 알림을 아예 표시하지 않게 하려면 별도 권한(Notification Filtering entitlement)이 필요한 것으로 알려져 있다.

## 2-4. 알림을 보낼지 말지는 누가 결정하나

알림 on/off 설정은 원격 푸시와 로컬 알림에서 **결정하는 쪽이 다르다.**

- **원격 푸시**: 확장에서 숨길 수 없으므로, 사용자의 알림 설정을 **서버에 저장**해두고 꺼져 있으면 **서버가 푸시를 보내지 않는 것**이 가장 확실하다. 앱이 결정하게 하려고 무음 푸시를 쓰는 방법도 있지만, 무음 푸시는 시스템이 제한하고 강제 종료된 앱에는 전달되지 않는 것으로 알려져 있어 신뢰성이 낮다.
- **로컬 알림**: 앱이 직접 만드는 것이므로 **만들지 않으면 그만**이다. 설정값을 확인해서 예약 자체를 하지 않거나, 앱이 켜져 있을 때의 표시를 `willPresent`에서 막는다.

```swift
func userNotificationCenter(_ center: UNUserNotificationCenter,
                            willPresent notification: UNNotification,
                            withCompletionHandler completionHandler: @escaping (UNNotificationPresentationOptions) -> Void) {
    guard settings.isNotificationEnabled else {
        completionHandler([])       // 앱이 켜져 있을 때 표시하지 않음
        return
    }
    completionHandler([.banner, .sound])
}
```

## 2-5. 델리게이트는 같은 경로를 쓴다

로컬 알림과 원격 푸시는 모두 **같은 `UNUserNotificationCenterDelegate`**(`willPresent`, `didReceive`)를 거친다. 둘을 구분해야 한다면 트리거 타입으로 확인할 수 있다.

```swift
let isRemote = response.notification.request.trigger is UNPushNotificationTrigger
```

`userInfo`에 직접 표식을 넣어서 구분하는 방법도 쓴다.

# 3. 푸시 알림의 콘텐츠와 트리거의 역할

**콘텐츠는 "무엇을", 트리거는 "언제 (또는 어떤 조건에서)"**를 정한다.

## 3-1. 콘텐츠 (`UNMutableNotificationContent`)

| 프로퍼티 | 역할 |
|---|---|
| `title`, `subtitle`, `body` | 보이는 텍스트 |
| `sound`, `badge` | 소리, 앱 아이콘 배지 숫자 |
| `userInfo` | **앱이 쓰는 데이터** (탭했을 때 어디로 갈지 등). 화면에는 보이지 않는다. |
| `attachments` | 이미지/오디오/영상 첨부 |
| `categoryIdentifier` | 알림에 붙일 액션 버튼 묶음(카테고리) 지정 |
| `threadIdentifier` | 알림 센터에서 **대화/주제별로 묶어서** 표시 |

## 3-2. 트리거

| 트리거 | 의미 | 예 |
|---|---|---|
| `UNPushNotificationTrigger` | **원격 푸시가 도착했을 때** | APNs로 온 알림. 앱이 만드는 트리거가 아니라 **시스템이 붙인다.** |
| `UNTimeIntervalNotificationTrigger` | **지금부터 N초 뒤** | 30분 뒤 알림. `repeats: true`면 간격이 **최소 60초** |
| `UNCalendarNotificationTrigger` | **특정 날짜/시각** (반복 가능) | 매일 오전 9시 |
| `UNLocationNotificationTrigger` | **특정 지역에 들어가거나 나올 때** | 회사 도착 시 (위치 권한 필요) |
| `nil` | **즉시** 발송 | |

```swift
var components = DateComponents()
components.hour = 9
let trigger = UNCalendarNotificationTrigger(dateMatching: components, repeats: true)   // 매일 9시
```

## 3-3. 알림을 구분하는 세 가지 identifier

알림에는 이름에 `Identifier`가 붙는 값이 세 개 있다. 이름은 비슷하지만 **구분하는 대상이 전혀 다르다.**

| | `request.identifier` | `content.threadIdentifier` | `content.categoryIdentifier` |
|---|---|---|---|
| 속한 곳 | 요청(`UNNotificationRequest`) | 콘텐츠 | 콘텐츠 |
| 구분하는 대상 | **알림 하나** | 알림의 **그룹**(대화/주제) | 알림에 붙는 **액션 버튼 세트** |
| 같은 값이면 | 이전 알림을 **교체** | 알림 센터에서 **한 묶음**으로 표시 (알림은 각각 유지) | 같은 액션 버튼을 사용 |
| 쓰임 | 수정, 취소, 조회 | 대화/주제별 묶음 | "완료", "미루기" 같은 버튼 |
| 원격 푸시의 대응 값 | `apns-collapse-id` 헤더 | `aps`의 `thread-id` | `aps`의 `category` |
| 예 | `"task-42"` | `"project-A"` | `"TASK"` |

**`request.identifier` — "이 알림이 어떤 알림인가"**

시스템이 아직 울리지 않은 알림(pending)과 이미 울린 알림(delivered)을 추적하는 **알림 하나의 고유한 이름**이다.

- **같은 `identifier`로 다시 등록하면 이전 알림이 교체**된다. 아직 울리지 않았다면 예약이 새 내용으로 바뀌고, 이미 울려서 알림 센터에 남아 있다면 그 알림이 새 내용으로 대체된다. 리마인더 시간을 바꿀 때 이전 예약을 따로 지우지 않아도 된다.
- **다른 `identifier`면 별개의 알림으로 쌓인다.** 매번 `UUID`를 쓰면 알림이 계속 쌓이고, 나중에 특정 알림을 취소하려면 그 ID를 따로 저장해두어야 한다.
- 그래서 `"task-42"`처럼 **알림이 가리키는 대상의 ID**로 만들어두면, 수정과 취소가 자연스럽다.
- 알림을 탭했을 때 `response.notification.request.identifier`로 어느 알림인지 알 수 있다.

```swift
center.removePendingNotificationRequests(withIdentifiers: ["task-42"])    // 아직 울리지 않은 알림 취소
center.removeDeliveredNotifications(withIdentifiers: ["task-42"])         // 이미 울린 알림 삭제
```

**`threadIdentifier` — "이 알림들은 같은 주제다"**

알림 센터와 잠금 화면에서 **같은 값을 가진 알림을 한 묶음(스택)으로 보여준다.** `request.identifier`와 달리 알림을 교체하지 않고, 각 알림은 그대로 유지된 채 **표시만 묶인다.** 메신저에서 대화방별로 알림을 묶거나, 프로젝트별로 리마인더를 묶을 때 쓴다. (알림 그룹화 방식은 사용자의 시스템 설정에도 영향을 받는다.)

```swift
content.threadIdentifier = "project-A"
```

**`categoryIdentifier` — "이 알림에는 어떤 버튼이 붙는가"**

앱이 `setNotificationCategories`로 **미리 등록해둔 카테고리 중 어떤 것을 쓸지** 가리키는 키다. 카테고리에는 액션 버튼의 목록이 들어 있어서, 같은 값을 가진 알림에는 같은 버튼(예: "완료", "미루기")이 붙는다. 카테고리를 등록하지 않았거나 값이 맞지 않으면 버튼이 표시되지 않는다. 알림을 탭했을 때 `content.categoryIdentifier`로 어떤 종류의 알림인지 구분하는 데도 쓸 수 있다. 액션 버튼을 만들고 처리하는 방법은 4-2에서 다룬다.

```swift
content.categoryIdentifier = "TASK"
```

## 3-4. 트리거는 알림의 빈도를 조절하지 않는다

트리거는 **발송 시점**만 정할 뿐이다. 짧은 지연(`timeInterval: 0.5`)을 줘도 알림마다 각자 그 시간만큼 밀려서 울릴 뿐, 연속으로 오는 알림의 간격을 벌려주지 않는다. 연속 알림을 줄이려면 같은 `identifier`로 **교체**하거나, `threadIdentifier`로 **묶거나**, 앱에서 일정 시간 안의 이벤트를 모아서 한 번에 보내는 **디바운스**를 따로 설계해야 한다.

# 4. 사용자가 알림을 탭했을 때 앱의 동작 처리

## 4-1. 델리게이트에서 탭을 받는다

```swift
extension AppDelegate: UNUserNotificationCenterDelegate {
    func userNotificationCenter(
        _ center: UNUserNotificationCenter,
        didReceive response: UNNotificationResponse,
        withCompletionHandler completionHandler: @escaping () -> Void
    ) {
        let userInfo = response.notification.request.content.userInfo

        if response.actionIdentifier == UNNotificationDefaultActionIdentifier,   // 알림 본문을 탭함
           let taskID = userInfo["taskID"] as? String {
            router.showTask(id: taskID)        // userInfo로 어느 화면으로 갈지 결정
        }
        completionHandler()
    }
}
```

`actionIdentifier`로 **무엇을 눌렀는지** 구분한다.

- `UNNotificationDefaultActionIdentifier`: 알림 본문을 탭
- `UNNotificationDismissActionIdentifier`: 알림을 밀어서 닫음 (카테고리에서 허용한 경우)
- 직접 만든 액션의 ID: 알림의 **액션 버튼**("완료", "미루기")을 누름

## 4-2. 액션 버튼

액션 버튼은 카테고리로 만들어서 등록하고, 콘텐츠의 `categoryIdentifier`로 연결한다.

```swift
let done = UNNotificationAction(identifier: "DONE", title: "완료", options: [])
let category = UNNotificationCategory(identifier: "TASK", actions: [done], intentIdentifiers: [])
center.setNotificationCategories([category])

content.categoryIdentifier = "TASK"
```

## 4-3. 델리게이트는 launch를 끝내기 전에 설정해야 한다

`UNUserNotificationCenter.current().delegate = self`를 `application(_:didFinishLaunchingWithOptions:)` 안에서 설정하지 않으면, **앱이 꺼진 상태에서 알림을 탭해서 앱이 시작될 때** `didReceive`를 놓친다.

## 4-4. 앱 상태에 따라 타이밍이 달라진다 (콜드 스타트)

앱이 꺼져 있다가 탭으로 시작되면 `didReceive`가 불리는 순간 **화면이 아직 준비되지 않았을** 수 있다. 바로 화면을 열려고 하면 실패하므로, 처리할 동작을 저장해뒀다가 화면이 준비된 뒤에 실행해야 한다.

```swift
final class Router {
    private var isReady = false
    private var pendingTaskID: String?

    func showTask(id: String) {
        if isReady {
            open(taskID: id)
        } else {
            pendingTaskID = id          // 화면이 준비될 때까지 보관
        }
    }

    func didBecomeReady() {             // 메인 화면 준비가 끝난 시점에 호출
        isReady = true
        if let id = pendingTaskID {
            open(taskID: id)
            pendingTaskID = nil
        }
    }
}
```

## 4-5. 앱이 켜져 있을 때는 기본적으로 알림이 보이지 않는다

앱이 화면에 떠 있을 때 도착한 알림은 **로컬 알림과 원격 푸시 모두 기본적으로 표시되지 않는다.** 알림이 도착하면 시스템이 앱의 `willPresent`를 호출해서 지금 보여줄지를 묻고, 앱이 `completionHandler`에 넘긴 옵션에 따라 표시 여부가 정해진다.

| 상황 | 결과 |
|---|---|
| `willPresent`를 구현하지 않음 | 표시되지 않음 |
| `completionHandler([])` | 표시되지 않음 |
| `completionHandler([.banner, .sound])` | 배너와 소리로 표시 |

```swift
func userNotificationCenter(_ center: UNUserNotificationCenter,
                            willPresent notification: UNNotification,
                            withCompletionHandler completionHandler: @escaping (UNNotificationPresentationOptions) -> Void) {
    completionHandler([.banner, .sound])   // 앱이 켜져 있어도 배너와 소리로 표시
}
```

앱이 알림을 아예 받지 못하는 것은 아니다. 포어그라운드에서도 `willPresent`가 호출되므로, 알림을 사용자에게는 **보이지 않게 하면서** 앱이 내용만 활용할 수도 있다. 예를 들어 채팅 화면이 열려 있을 때는 알림을 숨기고(`[]`) 화면의 메시지 목록만 갱신하는 식이다.

# 정리

- 로컬 알림은 권한 요청 → 콘텐츠 → 트리거 → 요청 등록 → 델리게이트 설정의 다섯 단계로 구현한다.
- 로컬 알림은 앱이 미리 예약하고 서버가 필요 없는 반면, 원격 푸시는 서버가 APNs를 통해 보내며 디바이스 토큰과 서버 연동이 필요하다. 원격 푸시만 Notification Service Extension으로 표시 직전에 가공할 수 있고, 보낼지 말지는 원격은 서버가, 로컬은 앱이 결정한다.
- 콘텐츠는 알림의 내용(`userInfo`, 첨부 등)이고, 트리거는 발송 조건(시간/캘린더/위치/즉시)이다. 트리거는 빈도를 조절하지 않으며, 알림 교체와 취소는 `identifier`로 한다.
- 알림 탭은 `didReceive`에서 `actionIdentifier`와 `userInfo`로 처리하고, 델리게이트는 launch가 끝나기 전에 설정해야 하며, 콜드 스타트에서는 화면이 준비될 때까지 동작을 보관해야 한다.
