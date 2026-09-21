# Combine 프레임워크란 무엇이며, 어떤 기능을 제공하나요?

## Combine이란?

"a declarative Swift API for processing values over time"
"시간에 따라 변화하는 값들을 선언적(declarative)으로 처리하는 Swift API"
애플 공식 문서에 적힌 Combine에 대한 설명이다.

여기서 우리는 선언적으로 처리된다는 점에 주목해보자.
선언형과 대비되는 것에는 명령형이 있는데 잠시 짧게 설명하고 가보자면

명령형은 함수 내부 구현이 그대로 보여지는 것이고, 선언형은 함수들을 가져다가 체이닝해서 사용하는 것이라고 생각하면 된다.
선언형에서 중요한 점은 내부 구현이 안 보여서 '해당 함수를 사용해줘' 정도만 알 수 있다는 점이다.

또한 명령형과 선언형의 성질을 가르는 것은 "코드 전체"를 보는 것이 아니라 "어느 레이어에서 보느냐"이다.
예를 들어 custom function을 하나 만들고 구현 내용이 보이는 곳에서는 해당 함수는 명령형이다.
하지만 해당 함수를 체이닝해서 사용하는 호출부에서 보면 선언형이라고 보면 된다.

이렇듯 Combine이 사용되는 곳에서는 각 함수들을 체이닝하여 선언형으로 사용하는 것이 핵심이고
버튼 탭/텍스트필드 입력/네트워크 응답/타이머 등 시간이 지나면서 반복적으로 발생하는 값들을 하나의 파이프라인으로 다루는 것이 중요하고, 서로 다른 이벤트 소스를 서로 다른 인터페이스(delegate, completion handler, Timer)들로 다룰 때 발생할 수 있는 실수를 하나의 통일된 인터페이스(Publisher-Operator-Subscriber)로 다루면, 어떤 값이 들어오든 항상 같은 방식으로 처리되기 때문에 개별 구현마다 실수할 여지가 줄어들고 코드의 안정성과 예측성을 높일 수 있게 된다.

## Publisher
"Declares that a type can transmit a sequence of values over time."
Publisher는 Combine에서 시간에 따라 값이 방출될 수 있음을 정의해둔 프로토콜이다.

무슨 값을 내보낼지(Output)와 실패한다면 어떤 에러 타입인지(Failure)를 연관 타입(associatedtype)으로 명시하고 있다. 실제로 어떤 타입이 들어갈지는 protocol 자체에는 정해져 있지 않고, 이를 채택하는 구체 타입마다 다르게 결정된다. 예를 들어 `URLSession.dataTaskPublisher(for:)`는 `Output = (data: Data, response: URLResponse)`, `Failure = URLError`를 가진다. 실패할 일이 없는 Publisher(예: `Just`)는 `Failure`가 `Never`이다.

```swift
protocol Publisher {
    associatedtype Output
    associatedtype Failure: Error
    
    ...
}
```

또한 Publisher는 `receive(subscriber:)` 함수를 구현하도록 요구하고 있다. 다만 이 함수는 Publisher를 채택한 쪽에서 각자 다르게 구현해야 하는 내부 구현 함수일 뿐, 우리가 직접 호출하는 함수는 아니다. 실제로 외부에서 구독을 걸 때는 `subscribe(_:)`라는 함수를 호출하는데, 이 함수는 protocol extension으로 모든 Publisher에 공통으로 딱 한 번 구현되어 있고, 내부적으로 각 타입의 `receive(subscriber:)`를 대신 호출해주는 창구 역할을 한다.

```swift
protocol Publisher {
    ...
    func receive<S>(subscriber: S)
        where S: Subscriber,
              Self.Failure == S.Failure,
              Self.Output == S.Input
}

extension Publisher {
    public func subscribe<S>(_ subscriber: S)
        where S: Subscriber,
              Self.Failure == S.Failure,
              Self.Output == S.Input
}
```

그리고 이 `receive(subscriber:)`가 하는 일은 값을 바로 전달하는 게 아니라, Subscriber와의 연결(Subscription)을 맺어주는 것뿐이다. 즉, Publisher는 "값을 만들어낼 수 있는 설계도/인터페이스" 같은 거지, 구독자가 없으면 아무런 일도 일어나지 않는다. (RxSwift의 Observable도 비슷하게 subscribe하기 전엔 아무것도 실행 안 되는 것과 같은 맥락이다.)

물론 실무에서는 `subscribe(_:)`조차 직접 호출할 일은 거의 없다. `.sink`나 `.assign(to:on:)` 같은 함수들이 내부적으로 Subscriber를 대신 만들어서 `subscribe(_:)`를 호출해주기 때문이다.

> 💡 **Tip**
> Publisher 프로토콜을 직접 구현할 일은 거의 없고, 보통은 이런 것들을 사용한다:
>
> - `Just(1)` — 값 하나 즉시 방출하고 끝
> - `PassthroughSubject<String, Never>()` — 외부에서 `.send()`로 값을 흘려보낼 수 있는 Publisher
> - `@Published var text: String`의 `$text` — 프로퍼티가 바뀔 때마다 방출
> - `NotificationCenter.default.publisher(for: .someNotification)`
> - `urlSession.dataTaskPublisher(for: url)`

## Subscriber
"A protocol that declares a type that can receive input from a publisher."
Publisher로부터 입력값을 받을 수 있는 유형임을 정의해둔 프로토콜이다.

Subscriber는 어떤 값을 받을지(Input)와 어떤 에러를 받을지(Failure)를 연관 타입(associatedtype)으로 갖고 있다. 이 Input과 Failure는 연결되는 Publisher의 Output, Failure와 반드시 일치해야 한다.

연결은 Publisher의 `subscribe(_:)` 메서드를 통해 이루어진다. 이 메서드가 호출되면 Publisher는 Subscriber의 `receive(subscription:)`을 실행한다. 이 메서드를 통해 Subscriber는 Subscription 인스턴스를 얻을 수 있고, 이 인스턴스를 통해 Publisher로부터 값을 요청하거나 구독을 취소할 수 있다.

```swift
protocol Subscriber {
    associatedtype Input
    associatedtype Failure: Error
    
    ...
}
```

### Receiving Elements

Subscriber를 채택한 곳에서는 반드시 구현해야 하며, Subscriber에게 Publisher가 값이 발행되었음을 알려주는 함수이다. 리턴값은 `Subscribers.Demand`인데, 초기에 요청한 값에서 추가로 몇 개 더 받을지를 조절할 수 있도록 도와준다.

예를 들어 초기에 `receive(subscription:)`에서 `subscription.request(_:)`를 통해 `.max(2)`를 요청한 상황이라고 치자. 이것은 "나 지금 2개까지만 처리할 수 있는 여력이 있으니, 그만큼까지만 보내줘"라고 요청한 것과 같다. 이때 리턴값에 따라서 초기에 요청한 demand의 값을 조절할 수 있다.

a. `.none`을 리턴하는 경우.
"이번엔 추가로 더 요청하지 않겠다"는 뜻이며, 기존에 요청해둔 demand는 그대로 유지된다. 즉, 처음 요청한 2개 중 1개를 사용했으니, 추가 요청 없이 남은 demand는 1개가 된다.

b. `.max(1)`을 리턴하는 경우.
"방금 값 하나 처리했는데, 하나 더 받을 여유가 생겼어, 한 개 더 보내줘"라는 의미이며, 요청량을 늘려달라는 신호이다. 즉, 처음 요청한 2개 중 1개를 사용하고서 추가로 1개를 요청하여 남은 demand는 2개가 되는 것이다.

보통 `.none`을 사용하지만 정말 세밀한 backpressure 제어가 필요한 경우에 사용되곤 한다.
(ex. 한 번에 처리할 수 있는 양이 실시간으로 변하는 스트림)

```swift
protocol Subscriber {
    func receive(_ input: Self.Input) -> Subscribers.Demand
}
```

그리고 `Input == Void`인 경우 `receive(())`라고 괄호를 채워 쓰는 대신 `receive()`라고 짧게 쓸 수 있도록 해주는 **편의 오버로드**가 있다. Apple이 미리 만들어둔 편의 함수이기 때문에 Subscriber 프로토콜을 채택한 구체 타입에서도 직접 구현할 필요는 없다.

보통 값 자체는 의미가 없고 이벤트가 발생했다는 사실만 중요한 Publisher인 경우에 자주 사용한다. 예를 들어 `Button`이 눌렸다는 이벤트만 전달받고 싶은 경우 또는 `Timer`처럼 "몇 시인지"보다 "타이머가 끝났다"는 게 중요한 경우 사용한다.

```swift
extension Subscriber where Input == Void {
    func receive() -> Subscribers.Demand
}
```

### Receiving life cycle events

Receiving Elements가 "값이 반복적으로 오가는 것"을 다뤘다면, 여기는 구독의 시작과 끝(lifecycle), 두 경계 지점에서 딱 한 번씩만 호출되는 함수들을 다룬다. 값 자체를 실어나르는 게 아니라 "지금 상태가 어떻게 바뀌었는지"를 통보하는 역할이기 때문에 리턴 타입도 `Subscribers.Demand`가 아니라 `Void`이다.

a. `receive(subscription:)` — 연결의 시작을 알림

```swift
func receive(subscription: Subscription)
```

`subscribe(_:)`가 호출된 직후, Publisher가 Subscription을 만들어서 건네줄 때 딱 한 번 호출된다. 흐름을 직접 확인해보자면:

`subscribe(_:)` → `receive(subscriber:)` → Subscription 생성 → `receive(subscription:)` (**여기!**) → `subscription.request(_:)` 호출

즉, 여기서 우리가 할 일은 보통 딱 하나 있다. 건네받은 `Subscription`에다 대고 `request(_:)`를 호출하여 **초기 demand를 정하는 것**이다. 이 함수를 호출하기 전까지는 값이 한 개도 안 넘어온다. 아무리 Publisher가 값을 만들 준비가 되어 있어도, Subscriber가 "나 받을 수 있어"라고 요청하기 전까지는 진행되지 않는다.

b. `receive(completion:)` — 연결의 끝을 알림

```swift
func receive(completion: Subscribers.Completion<Failure>)
```

스트림이 종료되었을 때 딱 한 번 호출된다. `Subscribers.Completion`은 case가 두 개뿐인 열거형 타입이다:

- `.finished` — 정상적으로 종료됨.
- `.failure(Failure)` — 에러 때문에 종료됨.

여기서 중요한 규약은, 이 함수가 호출된 이후로는 `receive(_:)`가 절대 다시는 호출되지 않도록 해야 한다는 점이다. "끝"이라고 선언했으면 진짜 끝인 것이다. — 이건 컴파일러가 강제하는 게 아니라, 잘 만들어진 Publisher라면 반드시 지켜야 하는 약속이다.

## Concrete Types

Publisher와 Subscriber를 따르는 구체 타입들이 있다. 이 타입들은 각각 `Publishers`, `Subscribers`라는 네임스페이스(빈 열거형) 아래에 모아져 있다 — 예를 들어 `Publishers.Map`, `Publishers.Filter`, `Subscribers.Sink`, `Subscribers.Assign` 같은 식이다.

이렇게 미리 구현해둔 타입들 덕분에, 우리는 `Publisher`와 `Subscriber` protocol을 직접 구현할 필요 없이 이미 만들어진 것들을 조합해서 쓸 수 있다. 그리고 이 타입들을 이어 붙일 수 있기 때문에 Combine에서 선언형으로 체이닝이 가능한 것이다.

`Publisher`와 `Subscriber` 각각 대표적인 구체 타입 두 가지씩만 살펴보자.

### Publishers

**Map**

```swift
extension Publisher {
    public func map<T>(_ transform: @escaping (Output) -> T) -> Publishers.Map<Self, T>
}
```

`.map`은 사실 `Publisher`를 채택한 구체 타입인 `Publishers.Map`을 만들어서 리턴해주는 편의 함수일 뿐이다. `Publishers.Map`도 결국 `Publisher`이기 때문에, 그 뒤에 다시 다른 Operator를 이어 붙일 수 있다.

내부적으로는 Publisher가 값을 하나 내보낼 때마다, 그 값을 클로저에 통과시켜 다른 값(경우에 따라 다른 타입)으로 변환한 뒤 다시 흘려보낸다.

```swift
Just(1)
    .map { $0 * 2 }
    .sink { print($0) }   // 2
```

**Filter**

```swift
extension Publisher {
    public func filter(_ isIncluded: @escaping (Output) -> Bool) -> Publishers.Filter<Self>
}
```

`.filter`도 마찬가지로, `Publisher`를 채택한 `Publishers.Filter`를 리턴하는 편의 함수다. 클로저가 `true`를 리턴하면 그 값을 다음으로 흘려보내고, `false`면 그 값은 그냥 사라진다.

```swift
[1, 2, 3, 4, 5].publisher
    .filter { $0 % 2 == 0 }
    .sink { print($0) }       // 2, 4 만 출력됨
```

`.map`이랑 다른 점은, `Output` 타입이 안 바뀐다는 점이다.

### Subscribers

**Sink**

```swift
extension Publisher {
    public func sink(receiveValue: @escaping (Output) -> Void) -> AnyCancellable

    public func sink(
        receiveCompletion: @escaping (Subscribers.Completion<Failure>) -> Void,
        receiveValue: @escaping (Output) -> Void
    ) -> AnyCancellable
}
```

`.sink`도 마찬가지로, `Subscriber`를 채택한 구체 타입인 `Subscribers.Sink`를 만들어서 구독을 건 다음, 그 결과를 `AnyCancellable`로 감싸서 리턴해주는 편의 함수다. 체인의 끝에서 값을 실제로 소비하는 역할을 하며, 값이 올 때마다 넘겨준 클로저(`receiveValue`)를 실행하고, 실패할 수 있는 Publisher라면 `receiveCompletion`까지 처리한다.

```swift
let cancellable = Just(1).sink { print($0) }   // 1
```

**Assign**

```swift
extension Publisher where Failure == Never {
    public func assign<Root>(to keyPath: ReferenceWritableKeyPath<Root, Output>, on object: Root) -> AnyCancellable
}
```

`.assign(to:on:)`도 `Subscriber`를 채택한 구체 타입인 `Subscribers.Assign`을 만들어서 구독을 건 다음, `AnyCancellable`로 감싸서 리턴하는 편의 함수다. `.sink`가 클로저를 실행하는 거라면, `.assign`은 값이 올 때마다 특정 객체의 특정 프로퍼티에 바로 대입해버리는 더 좁은 용도다.

```swift
publisher.assign(to: \.text, on: label)
```

(`Failure == Never` 제약이 붙은 이유는, `.assign`엔 에러를 처리하는 클로저가 없어서 애초에 실패할 수 없는 Publisher에만 쓸 수 있게 막아둔 것이다.)

**왜 둘 다 `AnyCancellable`을 리턴하는가**

`.sink`, `.assign` 둘 다 리턴 타입이 `AnyCancellable`인 이유는 두 가지다.

1. **자동 취소** — `Cancellable`은 `cancel()`만 정의된 단순한 protocol이라, 메모리에서 해제될 때 자동으로 `cancel()`이 호출되는 걸 보장하지 않는다. `AnyCancellable`은 구체 클래스로 `deinit`에서 감싸둔 대상의 `cancel()`을 직접 호출하도록 구현되어 있다.
2. **타입 통일(Hashable)** — `Cancellable`은 protocol이라 `Hashable`을 보장하지 않기 때문에, 서로 다른 구체 타입(`Subscribers.Sink`, `Subscribers.Assign` 등)을 그대로 `Set`에 저장할 수 없다. `AnyCancellable`은 구체 타입으로 `Hashable`을 직접 구현해뒀기 때문에, 서로 다른 `Cancellable` 구현체들을 하나의 통일된 타입으로 감싸서 `Set<AnyCancellable>`처럼 한 곳에 모아 저장할 수 있게 해준다.

## Subject

`Subject`는 `Publisher`를 상속하면서, `send`라는 이름의 함수 3개를 추가로 구현하도록 요구하는 protocol이다.

```swift
protocol Subject: AnyObject, Publisher {
    func send(_ value: Output)
    func send(completion: Subscribers.Completion<Failure>)
    func send(subscription: Subscription)
}
```

일반 Publisher는 값이 언제, 어떻게 나올지가 전부 내부 구현에 캡슐화되어 있다. `Just`, `URLSession`, `NotificationCenter`도 똑같이 `Publisher`를 채택한 타입(정확히는 그 메서드들이 리턴하는 타입)이지만 `send` 같은 공개 함수는 없다. 대신 각자 다른 내부 로직(`Just`는 생성 시점에 고정된 값, `URLSession`은 네트워크 콜백, `NotificationCenter`는 알림 옵저버)을 통해 자체적으로 값을 방출한다.

`Subject`는 이런 일반적인 Publisher의 한계에 대한 예외다. `send(_:)`라는 공개 함수를 통해, 외부에서 원하는 시점에 직접 값을 주입할 수 있게 해준다. 즉 명령형 세계(버튼 탭, 사용자 입력 등)의 이벤트를 Combine의 선언형 파이프라인 안으로 끌어들이는 다리 역할을 한다.

### PassthroughSubject / CurrentValueSubject

`Subject`를 채택한 대표적인 구체 타입은 두 가지다.

- `PassthroughSubject` — 구독 이전의 값은 전달하지 않고, 구독 이후에 `send`로 주입되는 값들만 방출한다.
- `CurrentValueSubject` — 현재 값을 계속 들고 있다가, 구독하는 순간 바로 그 현재 값을 방출한다.

### send 함수 세 개의 역할

이름이 `receive`가 아니라 `send`일 뿐, 사실 Subscriber의 세 함수(`receive(_:)`, `receive(completion:)`, `receive(subscription:)`)와 정확히 대응된다.

- `send(_ value:)` — 호출하면 지금 구독 중인 모든 Subscriber한테 그 값을 전달한다.
- `send(completion:)` — "끝났다"를 알린다. 호출하면 구독 중인 모든 Subscriber한테 완료가 전달되고, 그 이후로는 값이 다시 오면 안 된다.
- `send(subscription:)` — Subject가 **다른 Publisher의 구독 대상이 될 때** 쓰인다. (바로 아래에서 이어서 설명한다.)

### Subject를 구독 대상으로 쓰는 경우

`Publisher`에는 `Subject`를 받는 별도의 `subscribe(_:)` 오버로드가 있다.

```swift
extension Publisher {
    public func subscribe<S: Subject>(_ subject: S) -> AnyCancellable
        where Failure == S.Failure, Output == S.Output
}
```

`Subject` 자체는 `Subscriber`를 채택한 게 아니다. 이 함수 내부에서 Combine이 `Subscriber`를 채택한 숨겨진 어댑터를 하나 만들어서, 그 어댑터가 upstream Publisher를 실제로 구독하고, 받은 값을 `subject.send(...)`로 전달해주는 방식으로 동작한다. (`Publishers.Map`이 내부에 `Subscriber`를 채택한 `Inner` 헬퍼를 숨겨두고 있던 것과 같은 패턴이다.)

이게 왜 필요하냐면, **하나의 작업 결과를 여러 구독자한테 나눠주는 멀티캐스팅**이 가능해지기 때문이다. 일반 Publisher는 구독할 때마다 처음부터 다시 작업이 실행되기 때문에, 예를 들어 네트워크 요청 Publisher를 두 군데서 각각 `sink`로 구독하면 네트워크 요청이 두 번 나간다.

```swift
let networkPublisher = URLSession.shared.dataTaskPublisher(for: url)

networkPublisher.sink { data in /* 화면 A 업데이트 */ }.store(in: &cancellables)
networkPublisher.sink { data in /* 화면 B 업데이트 */ }.store(in: &cancellables)
// 네트워크 요청이 2번 나감
```

이걸 `Subject`를 중간에 끼워서 한 번만 구독하도록 바꾸면, 마치 **멀티탭**처럼 하나의 연결(벽 콘센트)에 여러 소비자(기기)를 나눠 꽂을 수 있게 된다.

```swift
let subject = PassthroughSubject<Data, URLError>()

networkPublisher.subscribe(subject)   // 네트워크 요청은 이 한 줄로 딱 1번만 실행됨

subject.sink { data in /* 화면 A 업데이트 */ }.store(in: &cancellables)
subject.sink { data in /* 화면 B 업데이트 */ }.store(in: &cancellables)
```

실무에서 이 패턴을 직접 이렇게 짜는 경우는 드물고, 이걸 더 편하게 해주는 `.share()`, `.multicast(subject:)`라는 전용 Operator를 대신 쓴다. 다만 그 Operator들도 내부적으로는 지금 설명한 "Publisher를 Subject에 구독시켜서 멀티캐스팅한다"는 원리를 그대로 쓰고 있다.

## Scheduler

Scheduler는 작업을 실행할 수 있는 컨텍스트(스레드, 큐, 런루프)를 protocol로 추상화해둔 것이다. `DispatchQueue`, `RunLoop`, `OperationQueue` 모두 이 `Scheduler`를 채택하고 있어서, Combine 체인에 "이 지점부터는 이 스케줄러에서 실행해달라"고 지정할 수 있다.

핵심 Operator는 두 개다.

**`.receive(on:)`**

이 지점 이후로 **downstream에 값이 전달되는 스케줄러**를 바꾼다. 체인에서의 위치가 중요하다 — "그 지점 이후"에만 영향을 주기 때문에, 체인 앞쪽에 쓰면 그 뒤 전체가, 뒤쪽에 쓰면 그 뒤만 영향을 받는다.

```swift
publisher.receive(on: queue).map { ... }.sink { ... }
// map, sink 둘 다 queue에서 실행됨

publisher.map { ... }.receive(on: queue).sink { ... }
// map은 원래 스케줄러에서, sink만 queue에서 실행됨
```

**`.subscribe(on:)`**

**구독이 실제로 시작되는(upstream 쪽 작업이 실행되는) 스케줄러**를 바꾼다. 애초에 "이 Publisher가 일을 시작하는 지점" 자체를 바꾸는 것이라, 체인 어디에 넣든 결과가 동일하다.

```swift
publisher.subscribe(on: queue).map { ... }.sink { ... }
publisher.map { ... }.subscribe(on: queue).sink { ... }
// 위 둘 다 upstream(publisher) 작업이 실행되는 스케줄러는 동일하게 적용됨
```

실무에서 제일 많이 쓰는 패턴은, 네트워크 응답(백그라운드 스레드)을 받은 다음 `.receive(on: DispatchQueue.main)`을 넣고 `.sink`에서 UI를 업데이트하는 것이다.

```swift
networkPublisher
    .receive(on: DispatchQueue.main)
    .sink { data in
        label.text = data  // 메인 스레드에서 안전하게 UI 업데이트
    }
    .store(in: &cancellables)
```
