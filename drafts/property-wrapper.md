# Swift의 Property Wrapper에 대해 설명해주세요.

## 1. Property Wrapper의 개요

### 1-1. Property Wrapper란

프로퍼티의 get/set에 공통으로 붙는 로직을 **재사용 가능한 타입으로 분리해서, `@이름` 한 줄로 프로퍼티에 적용할 수 있게 해주는 기능**이다.

예를 들어 "값을 0~100 범위로 제한한다"는 로직이 여러 프로퍼티에 필요하다고 해보자. Property Wrapper가 없다면 프로퍼티마다 같은 get/set을 반복해서 써야 한다.

```swift
struct Settings {
    private var _volume = 50
    var volume: Int {
        get { _volume }
        set { _volume = min(max(newValue, 0), 100) }
    }

    private var _brightness = 80
    var brightness: Int {
        get { _brightness }
        set { _brightness = min(max(newValue, 0), 100) }
    }
    // contrast, ... 전부 똑같은 로직을 또 써야 한다
}
```

이 로직을 Property Wrapper로 한 번만 만들어두면, 사용하는 쪽은 이렇게 간단해진다.

```swift
struct Settings {
    @Clamped var volume = 50
    @Clamped var brightness = 80
}
```

### 1-2. 기본적인 구성

```swift
@propertyWrapper
struct Clamped {
    private var value: Int
    private(set) var projectedValue: Bool = false

    var wrappedValue: Int {
        get { value }
        set {
            let clamped = min(max(newValue, 0), 100)
            projectedValue = (clamped != newValue)
            value = clamped
        }
    }

    init(wrappedValue: Int) {
        self.value = min(max(wrappedValue, 0), 100)
    }
}
```

Property Wrapper를 만들 때 필요한 구성은 다음과 같다.

- **`@propertyWrapper`** — 이 타입이 Property Wrapper임을 선언하는 속성이다.
- **`wrappedValue` (필수)** — 프로퍼티를 읽고 쓸 때 실제로 거치는 값이다. 여기에 get/set 로직을 넣는다.
- **`projectedValue` (선택)** — 값 자체와는 별개로, 그 프로퍼티와 관련된 부가 접근 수단을 `$이름`으로 노출하는 통로다. 타입은 래퍼를 만드는 쪽이 자유롭게 정한다.
- **`init(wrappedValue:)`** — `@Clamped var volume = 50`처럼 선언부에서 `= 50`으로 초기값을 줄 수 있게 해주는 이니셜라이저다.

```swift
var settings = Settings()
settings.volume = 150
print(settings.volume)    // 100  (wrappedValue)
print(settings.$volume)   // true (projectedValue: 150이 100으로 보정됐는지)
```

### 1-3. `name`, `_name`, `$name` — 컴파일러가 해주는 일

`@Clamped var volume = 50`이라고 쓰면, 컴파일러는 `volume` 하나를 아래 세 개로 나눠서 만들어준다.

| 이름 | 정체 | 비고 |
|---|---|---|
| `_volume` | 래퍼 인스턴스를 저장하는 **저장 프로퍼티** (`Clamped`) | 기본적으로 `private` |
| `volume` | `wrappedValue`로 연결되는 **계산 프로퍼티** (`Int`) | 내가 선언한 이름 |
| `$volume` | `projectedValue`로 연결되는 **계산 프로퍼티** (`Bool`) | 래퍼가 `projectedValue`를 정의한 경우에만 생성 |

개념적으로 풀면 대략 이런 모양이다. (컴파일러가 실제로 생성하는 코드 그대로는 아니고, 연결 관계를 설명하기 위한 스케치다.)

```swift
struct Settings {
    private var _volume = Clamped(wrappedValue: 50)   // 래퍼 인스턴스 저장
    var volume: Int {                                  // wrappedValue로 연결
        get { _volume.wrappedValue }
        set { _volume.wrappedValue = newValue }
    }
    // $volume → _volume.projectedValue
}
```

`$volume`은 값을 변환해주는 연산자가 아니라, 컴파일러가 `projectedValue`를 돌려주도록 만들어준 **별개의 프로퍼티**다. 그래서 `volume`(`Int`)과 `$volume`(`Bool`)의 타입이 완전히 달라도 된다. 래퍼가 `projectedValue`를 정의하지 않았다면 `$이름`은 아예 생성되지 않는다.

`_이름`은 실제로 존재하기 때문에, 같은 타입 안에서는 직접 쓸 수도 있다. SwiftUI에서 `init`으로 `@State`의 초기값을 넣는 코드가 대표적이다.

```swift
struct CounterView: View {
    @State private var count: Int

    init(initial: Int) {
        _count = State(initialValue: initial)   // 타입 내부이므로 private인 _count에 접근 가능
    }

    var body: some View { Text("\(count)") }
}
```

외부에서는 `CounterView(initial: 3)`처럼 `init`을 호출할 뿐이고, `_count`를 직접 건드리는 건 `init`이 대신 해준다.

## 2. Property Wrapper의 대표적인 예시

### 2-1. 기본 라이브러리 예시

Combine과 SwiftUI에서 이미 쓰고 있는 `@Published`, `@State`, `@Binding`이 모두 Property Wrapper다. `$`로 접근하는 값이 `projectedValue`이고, 래퍼마다 그 타입이 다르다.

| 선언 | `wrappedValue` (`name`) | `projectedValue` (`$name`) |
|---|---|---|
| `@Published var text: String` | `String` | `Published<String>.Publisher` |
| `@State var count: Int` | `Int` | `Binding<Int>` |
| `@Binding var isOn: Bool` | `Bool` | `Binding<Bool>` |

```swift
final class SearchViewModel: ObservableObject {
    @Published var query = ""
}

viewModel.$query              // Published<String>.Publisher — 값이 바뀔 때마다 방출
    .sink { print($0) }
    .store(in: &cancellables)
```

```swift
struct ParentView: View {
    @State private var isOn = false

    var body: some View {
        Toggle("알림", isOn: $isOn)   // $isOn은 Binding<Bool>
    }
}
```

`$text`가 Publisher였던 것도, `$isOn`이 Binding이었던 것도 우연이 아니라, 각 래퍼가 `projectedValue`로 그 타입을 내보내도록 만들어져 있기 때문이다.

### 2-2. UserDefaults를 사용한 간단한 예시

UserDefaults 접근은 키와 기본값만 다를 뿐, 읽고 쓰는 로직은 매번 똑같다. 그래서 Property Wrapper로 만들기 좋은 대표적인 경우다.

```swift
@propertyWrapper
struct UserDefault<Value> {
    private let key: String
    private let defaultValue: Value
    private let storage: UserDefaults

    init(key: String, defaultValue: Value, storage: UserDefaults = .standard) {
        self.key = key
        self.defaultValue = defaultValue
        self.storage = storage
    }

    var wrappedValue: Value {
        get { storage.object(forKey: key) as? Value ?? defaultValue }
        set { storage.set(newValue, forKey: key) }
    }
}

enum Preferences {
    @UserDefault(key: "isOnboarded", defaultValue: false)
    static var isOnboarded: Bool

    @UserDefault(key: "nickname", defaultValue: "")
    static var nickname: String
}
```

```swift
Preferences.isOnboarded = true
print(Preferences.nickname)   // 저장된 값이 없으면 기본값 ""
```

키와 기본값은 선언부에 한눈에 모이고, 읽기/쓰기 로직은 래퍼 한 곳에만 있다. 실제로 쓸 때는 `nil`을 대입하면 키를 삭제하거나, 값이 바뀔 때 알림을 보내는 것 같은 부가 로직도 이 한 곳에 넣을 수 있다. Codable 타입은 JSON 인코딩/디코딩을 거치는 별도의 래퍼로 분리해서 같은 방식으로 쓸 수 있다.

(참고로 SwiftUI에서는 `@AppStorage`가 UserDefaults를 위한 기본 제공 래퍼 역할을 한다.)

#### 더 나아가면 — `projectedValue`로 변경 관찰 노출하기

값이 바뀔 때 `NotificationCenter`로 알림을 보내는 래퍼라면, 사용하는 쪽이 알림 이름과 `userInfo` 키를 알고 직접 필터링해야 한다. 이 "바뀐 걸 관찰하는 통로"는 `projectedValue`로 노출하기 좋은 후보다.

```swift
import Combine

extension Notification.Name {
    static let userDefaultDidChange = Notification.Name("userDefaultDidChange")
}

// UserDefault 래퍼에 추가
var wrappedValue: Value {
    get { storage.object(forKey: key) as? Value ?? defaultValue }
    set {
        storage.set(newValue, forKey: key)
        NotificationCenter.default.post(
            name: .userDefaultDidChange, object: nil, userInfo: ["key": key]
        )
    }
}

var projectedValue: AnyPublisher<Value, Never> {
    NotificationCenter.default.publisher(for: .userDefaultDidChange)
        .compactMap { $0.userInfo?["key"] as? String }
        .filter { $0 == key }
        .map { _ in wrappedValue }
        .eraseToAnyPublisher()
}
```

```swift
Preferences.$nickname
    .sink { print("닉네임 변경:", $0) }
    .store(in: &cancellables)
```

사용하는 쪽은 알림 이름이나 `userInfo`를 몰라도, 해당 프로퍼티의 변경만 `$nickname`으로 바로 구독할 수 있다.

## 정리

- Property Wrapper는 프로퍼티의 get/set에 공통으로 붙는 로직을 재사용 가능한 타입으로 분리하는 기능이다.
- `@propertyWrapper` 타입은 `wrappedValue`를 반드시 정의하고, 선택적으로 `projectedValue`를 정의한다.
- 컴파일러는 `@Wrapper var name`을 `_name`(래퍼 인스턴스 저장), `name`(`wrappedValue` 연결), `$name`(`projectedValue` 연결, 정의한 경우에만)으로 나눠서 만들어준다.
- `@Published`, `@State`, `@Binding`이 대표적인 예이고, UserDefaults처럼 "키와 기본값만 다르고 로직은 같은" 경우에 직접 만들어 쓰기 좋다.
