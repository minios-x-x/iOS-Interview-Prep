# 델리게이션 패턴(Delegation Pattern)과 클로저의 차이점은 무엇인가요?

## 1. 델리게이션과 클로저의 차이점

같은 상황(파일 다운로드의 진행률, 완료, 실패를 알려주는 `Downloader`)을 두 방식으로 만들어서 비교한다.

### 1-1. 델리게이션 — 관련된 이벤트들을 프로토콜로 묶는다

```swift
protocol DownloaderDelegate: AnyObject {
    func downloader(_ downloader: Downloader, didUpdateProgress progress: Double)
    func downloader(_ downloader: Downloader, didFinishWith data: Data)
    func downloader(_ downloader: Downloader, didFailWith error: Error)
}

final class Downloader {
    weak var delegate: DownloaderDelegate?
    func download(url: URL) { /* ... */ }
}

final class DownloadViewController: UIViewController, DownloaderDelegate {
    private let downloader = Downloader()

    override func viewDidLoad() {
        super.viewDidLoad()
        downloader.delegate = self
        downloader.download(url: url)
    }

    func downloader(_ downloader: Downloader, didUpdateProgress progress: Double) { /* ... */ }
    func downloader(_ downloader: Downloader, didFinishWith data: Data) { /* ... */ }
    func downloader(_ downloader: Downloader, didFailWith error: Error) { /* ... */ }
}
```

### 1-2. 클로저 — 호출하는 자리에서 동작을 직접 넘긴다

```swift
final class Downloader {
    func download(
        url: URL,
        progress: @escaping (Double) -> Void,
        completion: @escaping (Result<Data, Error>) -> Void
    ) { /* ... */ }
}

downloader.download(
    url: url,
    progress: { [weak self] value in self?.progressView.progress = Float(value) },
    completion: { [weak self] result in /* ... */ }
)
```

### 1-3. 한눈에 비교

| | 델리게이션 | 클로저 |
|---|---|---|
| 형태 | 프로토콜 + 그걸 구현한 **객체** | 함수 값 |
| 이벤트 구조 | **관련된 여러 이벤트**를 하나의 계약으로 묶음 | 보통 단일 동작 (또는 이벤트 하나당 클로저 하나) |
| 코드 위치 | 델리게이트 객체의 **별도 메서드** (호출 지점과 떨어져 있음) | **호출하는 자리에 인라인**으로 작성 |
| 관계 | 보통 1:1 장기 관계 (`UITableView`와 뷰 컨트롤러처럼 오래 유지) | 호출 단위나 일회성 (완료 처리 등) |
| 상태/컨텍스트 | 델리게이트 객체의 프로퍼티를 직접 사용 | 클로저가 주변 값을 **캡처**해서 사용 |
| 메모리 관리 | `weak var delegate`로 순환 참조 방지 | **캡처 리스트**(`[weak self]` 등)로 직접 관리 |
| 선택적 이벤트 | `@objc optional`이나 `extension`의 기본 구현으로 | 클로저 프로퍼티를 옵셔널로 두고, `nil`이면 호출하지 않음 |

### 1-4. 이벤트가 여러 개일 때 — 클로저로도 가능하지만

"델리게이션은 여러 이벤트, 클로저는 단일 이벤트"는 규칙이 아니다. 클로저도 이벤트마다 하나씩 만들면 똑같이 쓸 수 있고, 호출 횟수도 기준이 아니다(클로저도 여러 번 호출될 수 있고, `UIImagePickerControllerDelegate`처럼 사실상 한 번만 불리는 델리게이트도 있다). 이벤트가 여러 개일 때 실제로 어떤 차이가 생기는지 비교해보면 이렇다.

```swift
// 델리게이션: 계약 하나, 설정 한 줄
scanner.delegate = self

// 클로저: 이벤트마다 하나씩
scanner.onBack = { [weak self] in self?.pop() }
scanner.onClose = { [weak self] in self?.dismiss() }
scanner.onSkip = { [weak self] in self?.skip() }
scanner.canUse = { [weak self] value in self?.validate(value) ?? false }
```

| | 델리게이션 | 이벤트별 클로저 |
|---|---|---|
| 구현 누락 | 필수 메서드를 빠뜨리면 **컴파일 에러** | 클로저를 안 넣어도 컴파일되고 **조용히 아무 일도 안 일어남** |
| 메모리 관리 | `weak var delegate` **한 곳** | 클로저마다 `[weak self]`를 **각각** 챙겨야 함 |
| 설정 | `delegate = self` 한 줄 | 이벤트 개수만큼 대입 |
| 보낸 쪽 구분 | 보낸 객체를 첫 인자로 넘겨서, 하나의 델리게이트가 여러 인스턴스를 구분 가능 | 인스턴스마다 클로저를 따로 설정하니 구분이 필요 없음 |
| 이벤트 추가 | 프로토콜에 메서드를 추가하면 기존 구현체가 깨짐 (기본 구현이 없으면) | 클로저 프로퍼티를 추가해도 기존 코드가 안 깨짐 |

이벤트가 많아질수록 델리게이션이 구조적으로 안전하고 정돈된 경우가 많지만, 델리게이션만 가능한 것은 아니다. 그리고 "구현하지 않으면 컴파일러가 잡아준다"는 필수 메서드에만 해당하고, `@objc optional`이나 기본 구현이 있는 메서드는 구현하지 않아도 컴파일된다.

### 1-5. 언제 어느 쪽을 쓰나

- **델리게이션이 어울리는 경우**: 이벤트가 여러 개이고 서로 관련 있을 때, 오래 유지되는 관계일 때, 그리고 **데이터를 요청하는 경우**(`UITableViewDataSource`처럼 "몇 개야?", "어떤 셀이야?"를 계속 물어보는 구조)
- **클로저가 어울리는 경우**: 한 번 실행하고 끝나는 결과 전달(완료 핸들러), 이벤트가 하나뿐일 때, 호출 맥락 안에서 읽히는 게 자연스러울 때

한 번 결과를 받는 완료 핸들러 용도의 클로저는 요즘 `async/await`가 대체하는 추세라서, 클로저의 남은 주 용도는 이벤트를 여러 번 받는 콜백과 짧은 동작 전달에 가까워졌다.

## 2. 델리게이션 패턴에서 메모리 누수가 발생할 수 있는 경우와 해결 방법

### 2-1. 왜 누수가 생기나 — 소유 관계가 원이 된다

`DownloadViewController`가 `Downloader`를 들고 있고, `Downloader`가 델리게이트인 뷰 컨트롤러를 다시 들고 있으면 서로가 서로를 놓지 못한다.

```swift
final class Downloader {
    var delegate: DownloaderDelegate?   // 강한 참조
}

final class DownloadViewController: UIViewController, DownloaderDelegate {
    private let downloader = Downloader()      // VC → downloader (강한 참조)

    override func viewDidLoad() {
        super.viewDidLoad()
        downloader.delegate = self             // downloader → VC (강한 참조)
    }

    deinit { print("VC deinit") }              // 화면을 닫아도 호출되지 않는다
}
```

```
VC ──강하게──▶ Downloader
 ▲                  │
 └──────강하게──────┘   ← 순환 참조: 레퍼런스 카운트가 0이 되지 않음
```

화면을 닫아도 둘 다 해제되지 않고 메모리에 남는다.

### 2-2. 해결 — delegate를 `weak`로 선언

```swift
final class Downloader {
    weak var delegate: DownloaderDelegate?   // 약한 참조
}
```

`weak`는 레퍼런스 카운트를 올리지 않아서 순환이 끊긴다. 소유 방향은 이렇게 정리된다.

- VC → Downloader: **소유** (강한 참조)
- Downloader → VC: **소유하지 않음** (약한 참조)

일반 규칙은 **소유하는 쪽에서 소유되는 쪽으로는 강하게, 반대 방향(자식이 부모를 가리키는 쪽)은 약하게**다. 델리게이트는 보통 부모를 가리키는 방향이라서 `weak`가 기본이고, UIKit의 `UITableView.delegate`, `dataSource`도 약한 참조다.

프로토콜에 `AnyObject`가 붙는 이유도 여기에 있다. `weak`는 참조 카운팅을 하는 **클래스 인스턴스에만** 쓸 수 있는데, 프로토콜이 `AnyObject`로 제한돼 있지 않으면 `struct`나 `enum`도 채택할 수 있어서 `weak var delegate: DownloaderDelegate?`가 컴파일 에러가 난다.

```swift
protocol DownloaderDelegate: AnyObject { /* ... */ }
```

### 2-3. `weak`로 바꿔도 조심할 점

`weak` 참조는 대상이 해제되면 **자동으로 `nil`**이 된다. 그래서 `delegate?.method()`로 호출하는데, 이게 함정을 만든다.

**델리게이트를 아무도 소유하지 않으면 바로 해제된다.**

```swift
downloader.delegate = Helper()   // Helper를 다른 곳에서 들고 있지 않으면 즉시 해제 → delegate가 nil
```

이벤트가 가도 아무 에러 없이 **조용히 사라진다.** "델리게이트 메서드가 호출이 안 돼요" 하는 버그의 흔한 원인이다.

**`unowned`로 선언하면 해제 후 접근할 때 크래시**가 난다. `weak`와 달리 `nil`이 되지 않고 이미 해제된 메모리를 가리키기 때문이다. (`weak`와 `unowned`의 차이는 3장에서 자세히 다룬다.)

### 2-4. `weak`를 써도 누수가 생기는 경우

**1) 델리게이트가 여러 개인 구조에서 배열에 강하게 저장한 경우**

하나의 이벤트를 여러 객체에게 동시에 알려야 할 때(멀티캐스트 델리게이트), 델리게이트를 배열에 모아두고 순회하며 호출한다. 이때 배열을 `[DownloaderDelegate]`로 만들면 **모든 델리게이트를 강하게 붙잡게** 된다.

```swift
final class Downloader {
    private var delegates: [DownloaderDelegate] = []   // 강한 참조 배열

    func add(_ delegate: DownloaderDelegate) { delegates.append(delegate) }

    func notifyFinished(_ data: Data) {
        delegates.forEach { $0.downloader(self, didFinishWith: data) }
    }
}
```

- **순환 참조**: 화면이 `Downloader`를 소유하면서 `add(self)`를 하면 서로를 강하게 잡는다.
- **수명 연장**: `Downloader`가 싱글톤처럼 오래 사는 객체라면, 거기에 `add`된 화면들이 **닫혀도 계속 살아남는다.** 화면이 100번 열리고 닫히면 100개가 쌓이고, 이미 닫힌 화면에도 이벤트가 호출된다.

배열에는 `weak`를 직접 쓸 수 없어서(`[weak DownloaderDelegate]` 같은 문법이 없다) 두 가지 방법을 쓴다.

**방법 A. 약한 래퍼로 감싸기**

```swift
final class Downloader {
    private struct WeakDelegate {
        weak var value: DownloaderDelegate?   // 래퍼 안에서 약하게 참조
    }
    private var delegates: [WeakDelegate] = []

    func add(_ delegate: DownloaderDelegate) {
        delegates.append(WeakDelegate(value: delegate))
    }

    func remove(_ delegate: DownloaderDelegate) {
        delegates.removeAll { $0.value === delegate }
    }

    func notifyFinished(_ data: Data) {
        delegates.removeAll { $0.value == nil }          // 이미 해제된 항목 정리
        delegates.forEach { $0.value?.downloader(self, didFinishWith: data) }
    }
}
```

**방법 B. `NSHashTable`의 약한 참조 모드 사용**

```swift
private let delegates = NSHashTable<AnyObject>.weakObjects()

func add(_ delegate: DownloaderDelegate) { delegates.add(delegate) }

func notifyFinished(_ data: Data) {
    for case let delegate as DownloaderDelegate in delegates.allObjects {
        delegate.downloader(self, didFinishWith: data)
    }
}
```

`NSHashTable`은 약하게 담은 객체가 해제되면 자동으로 목록에서 빠지는 장점이 있지만 **순서가 보장되지 않는다.** 호출 순서가 중요하면 방법 A를 쓰고, 이때는 해제된 항목을 직접 정리해야 한다.

**2) 델리게이트 메서드 안에서 쓰는 클로저가 `self`를 강하게 캡처하는 경우**

`delegate`를 `weak`로 선언해서 델리게이션의 순환은 끊었어도, 그 델리게이트 메서드 **안에서 만드는 클로저**가 `self`를 강하게 잡으면 별개의 소유 경로가 생긴다. 이건 델리게이션이 아니라 클로저의 문제다.

```swift
final class DownloadViewController: UIViewController, DownloaderDelegate {
    private let downloader = Downloader()
    private var retryAction: (() -> Void)?             // VC가 클로저를 소유

    func downloader(_ downloader: Downloader, didFailWith error: Error) {
        retryAction = { self.downloader.download(url: self.url) }   // 클로저가 self를 강하게 캡처
    }
}
```

```
VC ──소유──▶ retryAction(클로저) ──강하게 캡처──▶ VC     ← 순환 참조
```

`self`가 소유하지 않는 오래 사는 객체가 클로저를 붙잡는 경우도 있다.

```swift
func downloader(_ downloader: Downloader, didFinishWith data: Data) {
    Timer.scheduledTimer(withTimeInterval: 1, repeats: true) { _ in
        self.refresh()      // 타이머가 이 클로저를, 클로저가 self(VC)를 강하게 잡는다
    }
}
```

RunLoop가 타이머를 붙잡고, 타이머가 클로저를, 클로저가 VC를 붙잡아서, 타이머를 `invalidate`하기 전까지 화면이 해제되지 않는다. 두 경우 모두 캡처 리스트로 끊는다.

```swift
retryAction = { [weak self] in
    guard let self else { return }
    self.downloader.download(url: self.url)
}
```

### 2-5. 누수를 확인하는 방법

- **`deinit`에 로그**를 찍어서 화면을 닫을 때 호출되는지 확인한다. (제일 간단하다.)
- Xcode의 **Debug Memory Graph**에서 누수 표시와 참조 관계를 확인한다.
- Instruments의 **Leaks**를 쓴다.

## 3. 클로저의 캡처 리스트는 어떤 역할을 하나요?

### 3-1. 캡처란

클로저는 자기 바깥의 변수를 가져다 쓸 수 있고, 이걸 **캡처**라고 한다. 기본 동작은 이렇다.

- **변수 자체를 참조**로 잡는다(복사가 아니라 그 변수를 그대로 바라본다).
- 대상이 클래스 인스턴스면 **강한 참조**로 잡아서 해제를 막는다.

```swift
var count = 0
let increment = { count += 1 }   // count 변수 자체를 캡처
increment()
increment()
print(count)   // 2  (클로저가 바깥 변수를 직접 바꿨다)
```

### 3-2. 캡처 리스트 — 무엇을 어떻게 캡처할지 직접 정한다

클로저 맨 앞의 `[ ]`에 적는다. 용도가 두 가지다.

**① 값 타입을 "그 시점의 값으로 복사"해서 캡처**

```swift
var value = 10
let byReference = { print(value) }          // 변수를 참조
let byCopy = { [value] in print(value) }    // 캡처하는 시점의 값을 복사

value = 20
byReference()   // 20
byCopy()        // 10
```

**② 참조 타입을 "약하게 / 미소유로" 캡처해서 순환 참조를 끊는 것** (주로 쓰는 용도)

```swift
{ [weak self] in self?.refresh() }      // self는 Optional, 해제되면 nil
{ [unowned self] in self.refresh() }    // self는 Optional 아님, 해제 후 접근하면 크래시
```

### 3-3. `weak`와 `unowned`의 차이

| | `weak` | `unowned` |
|---|---|---|
| 클로저 안에서 타입 | **Optional** (`self?`) | **Non-optional** (`self`) |
| 대상이 해제된 뒤 접근 | 안전하게 `nil` | **크래시** (해제된 메모리를 가리킴) |
| 안전성 | 높음 | 낮음 (개발자가 수명을 보장해야 함) |
| 선택 기준 | 클로저가 대상보다 **오래 살 수 있을 때** | 클로저가 대상보다 **반드시 먼저(함께) 사라질 때** |

`unowned`가 위험한 순간은 이런 경우다.

```swift
var handler: (() -> Void)?

do {
    let viewController = ViewController()
    handler = { [unowned viewController] in viewController.refresh() }
}   // viewController 해제됨

handler?()   // 해제된 객체 접근 → 크래시
```

`unowned`는 클로저의 수명이 대상에 **확실히 종속**돼 있을 때만 쓴다. 예를 들어 `self`가 소유한 프로퍼티(`lazy var handler = { [unowned self] in ... }`)에 담긴 클로저는 `self`와 함께 사라지니 안전하다. 확신이 서지 않으면 `weak`가 기본이다.

### 3-4. 실무에서 쓰는 패턴과 주의점

**`[weak self]` + `guard let self`**

```swift
service.fetch { [weak self] result in
    guard let self else { return }      // 해제됐으면 그냥 종료
    self.update(result)                 // 이 아래부터는 self가 non-optional
}
```

`guard let self`는 Swift 5.7(Xcode 14)부터 쓸 수 있는 if-let 단축 표기다. 이전에는 `guard let self = self else { return }`로 썼고, 동작은 같다. 이렇게 올려놓은 `self`는 **클로저가 실행되는 동안만 강하게 유지**된다. 짧게 끝나는 작업이면 괜찮지만, 그 안에서 긴 비동기 작업을 또 하면 그동안 해제가 미뤄진다.

**중첩 클로저에서는 안쪽에도 다시 써야 한다.**

```swift
service.fetch { [weak self] result in
    guard let self else { return }
    // 여기서의 self는 강한 로컬 참조라서, 안쪽 escaping 클로저가 다시 강하게 잡는다
    self.timer = Timer.scheduledTimer(withTimeInterval: 1, repeats: true) { [weak self] _ in
        self?.tick()                    // 안쪽에도 [weak self]가 필요
    }
}
```

### 3-5. "무조건 `[weak self]`"가 정답은 아니다

| 상황 | 캡처 선택 |
|---|---|
| `map`, `forEach`처럼 즉시 끝나는 non-escaping 클로저 | 캡처 리스트 불필요 |
| 저장되거나 오래 사는 객체가 들고 있는 escaping 클로저 | `[weak self]` (순환 방지) |
| **반드시 끝까지 실행돼야 하는 작업** (예: 데이터 저장 완료 처리) | 일부러 **강하게** 잡는 게 맞을 수 있다. `weak`로 하면 화면이 사라질 때 작업이 통째로 스킵된다. |
| 값 타입(`struct`)의 `self` | `weak`/`unowned` 적용 자체가 불가 (클래스에만 가능). 순환도 생기지 않는다. |

판단은 두 가지 질문으로 한다. **클로저가 `self`를 소유한 쪽에 의해 오래 붙잡히는가?** 그리고 **`self`가 사라진 뒤에도 이 동작이 실행돼야 하는가?**

## 4. 델리게이션과 클로저를 함께 사용하는 경우의 장단점

두 방식을 함께 쓸 때는 **역할을 나눠서** 쓰는 게 자연스럽다.

- **델리게이트**: 화면 전체에서 공통으로 처리하는 **진행률 같은 상시 이벤트**
- **클로저**: 각 요청마다 맥락이 다른 **개별 결과 처리**

이 장에서는 `Downloader`를 이렇게 바꿔서 쓴다.

```swift
protocol DownloaderDelegate: AnyObject {
    func downloader(_ downloader: Downloader, didUpdateProgress progress: Double, for url: URL)
}

final class Downloader {
    weak var delegate: DownloaderDelegate?     // 상시 이벤트: 진행률

    func download(url: URL, completion: @escaping (Result<Data, Error>) -> Void) { /* ... */ }   // 이 요청의 결과
}
```

```swift
struct DownloadItem {
    let name: String
    let url: URL
}

final class DownloadListViewController: UIViewController {
    private let downloader = Downloader()
    private let statusLabel = UILabel()
    private let items: [DownloadItem] = [
        DownloadItem(name: "문서.pdf", url: URL(string: "https://example.com/a.pdf")!),
        DownloadItem(name: "사진.jpg", url: URL(string: "https://example.com/b.jpg")!),
    ]

    override func viewDidLoad() {
        super.viewDidLoad()
        downloader.delegate = self            // ① 진행률은 델리게이트로 받겠다고 등록
    }

    func startAll() {
        for item in items {
            // ② 항목마다 다운로드를 시작한다.
            //    클로저는 지금 실행되지 않고, "이 항목의 다운로드가 끝났을 때" 한 번 실행된다.
            downloader.download(url: item.url) { [weak self] result in
                self?.show(result, for: item)
            }
        }
    }

    private func show(_ result: Result<Data, Error>, for item: DownloadItem) {
        switch result {
        case .success: print("\(item.name) 완료")
        case .failure: print("\(item.name) 실패")
        }
    }
}

extension DownloadListViewController: DownloaderDelegate {
    // ③ 다운로드가 진행되는 동안 여러 번, 모든 항목의 진행률이 이 한 곳으로 들어온다.
    func downloader(_ downloader: Downloader, didUpdateProgress progress: Double, for url: URL) {
        statusLabel.text = "\(url.lastPathComponent): \(Int(progress * 100))%"
    }
}
```

```
startAll() 호출
 ├─ 문서.pdf 다운로드 시작
 ├─ 사진.jpg 다운로드 시작
 │
 │   (다운로드가 진행되는 동안)
 ├─ 델리게이트 호출: "a.pdf 10%"      ← ③, 어느 파일인지는 url 인자로 구분
 ├─ 델리게이트 호출: "b.jpg 30%"
 ├─ 델리게이트 호출: "a.pdf 60%" ...
 │
 │   (각 다운로드가 끝나는 순간)
 ├─ 문서.pdf 완료 → ②의 클로저 실행, 이 클로저는 item = 문서.pdf를 기억하고 있음
 └─ 사진.jpg 완료 → ②의 클로저 실행, 이 클로저는 item = 사진.jpg를 기억하고 있음
```

`for item in items`는 항목마다 `download`를 한 번씩 호출하고, 각 호출에 넘긴 클로저는 **그 호출 전용**이다. 클로저를 만들 때 그 순간의 `item`을 캡처해서 들고 있기 때문에, 완료됐을 때 "이 결과가 어느 항목의 것인지"를 따로 찾을 필요가 없다. 반면 진행률은 모든 다운로드에서 계속 들어오니 한 곳(`didUpdateProgress`)에서 한꺼번에 받고, 어느 파일의 것인지는 `url` 인자로 구분한다.

결과까지 델리게이트로만 받는다면 이렇게 된다.

```swift
func downloader(_ downloader: Downloader, didFinishWith data: Data, for url: URL) {
    guard let item = items.first(where: { $0.url == url }) else { return }   // 어느 요청인지 직접 찾아야 함
    show(.success(data), for: item)
}
```

### 장점

- **각 방식의 강점을 역할로 분담**한다. 공통이고 지속적인 이벤트는 델리게이트(계약, 메모리 관리 한 곳), 요청별 결과는 클로저(호출 자리에서 맥락을 캡처)로 처리한다.
- 델리게이트만 쓸 때의 **요청-응답 매칭 부담**(식별자로 직접 찾기)이 사라진다.

### 단점

- 결과를 받는 경로가 **두 갈래**라서 "이 이벤트는 델리게이트로 오나, 클로저로 오나"를 헷갈릴 수 있다. 어느 이벤트가 어느 경로로 가는지 **규약을 명확히** 해야 한다.
- **메모리 관리도 두 가지**를 챙겨야 한다. (`weak var delegate`와 클로저의 캡처 리스트)
- 같은 이벤트를 두 경로로 중복 전달하지 않도록 설계해야 한다.

`URLSession`이 이런 식이다. 인증 챌린지 같은 세션 단위 이벤트는 델리게이트로, 개별 요청의 결과는 completion handler로 받는 식으로 두 방식의 역할을 나눠 쓴다.

## 정리

- 델리게이션은 프로토콜로 책임을 위임하는 패턴이고, 클로저는 동작을 함수 값으로 넘기는 방식이다. 이벤트가 여러 개이면 델리게이션이 누락 방지와 메모리 관리 면에서 유리하지만, 클로저로도 가능하며 호출 횟수가 기준은 아니다.
- 델리게이션의 누수는 delegate를 강한 참조로 들고 있어서 생기는 순환 참조이고, `weak var delegate`(프로토콜은 `AnyObject`)로 해결한다. 델리게이트를 배열에 모으거나 델리게이트 메서드 안의 클로저가 `self`를 강하게 잡는 경우는 별도로 챙겨야 한다.
- 캡처 리스트는 클로저가 무엇을 어떻게 캡처할지 정하는 것이고, 순환 참조가 우려되면 `[weak self]`, 수명이 확실히 종속될 때만 `[unowned self]`를 쓴다. 모든 클로저에 `[weak self]`가 필요한 것은 아니다.
- 두 방식은 역할을 나눠 함께 쓸 수 있다. 상시 이벤트는 델리게이트로, 요청별 결과는 클로저로 처리하면 각자의 강점이 살지만, 결과 전달 경로가 둘이 되는 만큼 규약과 메모리 관리를 명확히 해야 한다.
