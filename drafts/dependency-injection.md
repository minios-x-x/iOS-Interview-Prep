# iOS 앱에서 의존성 주입(Dependency Injection)은 어떤 목적으로 사용되나요?

## 1. DI란, 그리고 왜 쓰는가

### 1-1. 의존성이란

어떤 객체가 자기 일을 하려고 **다른 객체를 필요로 하면, 그게 의존성**이다. 이 글에서는 사용자 정보를 보여주는 `ProfileViewModel`이 사용자 정보를 가져오는 `UserService`를 필요로 하는 상황을 예시로 쓴다. `UserService`가 `ProfileViewModel`의 의존성이다.

### 1-2. DI 없이 짜면

의존 객체를 안에서 직접 만들면 이렇게 된다. (구현은 생략했다.)

```swift
struct User { let name: String }

final class NetworkUserService {
    func fetchUser(id: String) async throws -> User { /* 실제 네트워크 호출 */ }
}

final class ProfileViewModel {
    private let service = NetworkUserService()   // 안에서 직접 생성
    private(set) var user: User?

    func load(id: String) async {
        user = try? await service.fetchUser(id: id)
    }
}
```

동작은 하지만 문제가 세 가지 있다.

- **테스트가 안 된다.** `ProfileViewModel`을 테스트하면 `NetworkUserService`가 항상 진짜 네트워크를 호출한다. 서버 상태에 따라 결과가 달라지고, 느리고, 실패 상황을 만들어내기도 어렵다.
- **교체가 어렵다.** `NetworkUserService`를 `CachedUserService`로 바꾸려면 `ProfileViewModel` 코드를 직접 고쳐야 한다.
- **의존성이 숨어 있다.** `ProfileViewModel`을 사용하는 쪽에서는 이 클래스가 네트워크에 의존한다는 걸 알 수 없다. 내부 코드를 열어봐야 안다.

세 문제의 뿌리는 하나다. `ProfileViewModel`이 **"무엇을 쓸지"를 스스로 정하고 있다**는 것이다.

### 1-3. DI — 의존 객체를 밖에서 넣어준다

**의존 객체를 안에서 만들지 않고, 밖에서 받는 것**이 의존성 주입(DI)이다.

```swift
final class ProfileViewModel {
    private let service: NetworkUserService
    private(set) var user: User?

    init(service: NetworkUserService) {   // 밖에서 받는다
        self.service = service
    }
}

let viewModel = ProfileViewModel(service: NetworkUserService())   // 만드는 쪽이 정한다
```

`ProfileViewModel`은 이제 "무엇을 쓸지" 정하지 않고 받은 것을 쓰기만 한다. 객체를 만들고 연결하는 책임이 `ProfileViewModel` 밖으로 나갔다.

### 1-4. 프로토콜과 함께 써야 효과가 난다

위 코드는 DI이지만, `NetworkUserService`라는 **구체 타입**을 받고 있어서 테스트용으로 바꿔치기할 수 없다. 문제 중 테스트와 교체는 여전히 풀리지 않았다. 프로토콜로 받아야 풀린다.

```swift
protocol UserService {
    func fetchUser(id: String) async throws -> User
}

final class NetworkUserService: UserService { /* 실제 구현 */ }

final class MockUserService: UserService {   // 테스트용
    var result: Result<User, Error> = .success(User(name: "test"))
    func fetchUser(id: String) async throws -> User { try result.get() }
}

final class ProfileViewModel {
    private let service: UserService          // 프로토콜 타입으로 받는다
    private(set) var user: User?

    init(service: UserService) {
        self.service = service
    }

    func load(id: String) async {
        user = try? await service.fetchUser(id: id)
    }
}
```

상황에 따라 다른 구현을 넣을 수 있다.

```swift
// 앱에서: 진짜 구현
let viewModel = ProfileViewModel(service: NetworkUserService())

// 테스트에서: 가짜 구현
let viewModel = ProfileViewModel(service: MockUserService())
```

| 단계 | 의존 객체를 | 테스트/교체 |
|---|---|---|
| 직접 생성 | 안에서 만듦 | 불가 |
| 구체 타입 주입 | 밖에서 받음 | 불가 (구체 타입에 고정) |
| 프로토콜 주입 | 밖에서 받음 + 추상에 의존 | **가능** |

### 1-5. 헷갈리기 쉬운 용어

- **DI (의존성 주입)**: 의존 객체를 밖에서 넣어주는 **기법**이다.
- **IoC (제어의 역전)**: 객체 생성과 연결의 제어권을 밖으로 넘기는 **큰 개념**이고, DI는 그 구현 방법 중 하나다.
- **DIP (의존성 역전 원칙)**: SOLID의 원칙으로, 상위 모듈이 구체 구현이 아니라 **추상(프로토콜)에 의존**하라는 것이다. 위의 프로토콜 주입이 DIP를 따르는 모양이다.

`ProfileViewModel`이 직접 만들지 않으니, 누군가는 객체를 만들어서 연결해야 한다. 보통 앱의 진입점 근처처럼 **객체들을 조립하는 지점**에서 만들고, 이 지점을 Composition Root라고 부른다.

## 2. 의존성 주입의 세 가지 유형

세 유형은 **의존 객체를 언제, 어떤 경로로 넘기느냐**의 차이다.

### 2-1. Initializer Injection (생성자 주입)

객체를 **만드는 시점에** `init` 파라미터로 받는다.

```swift
final class ProfileViewModel {
    private let service: UserService

    init(service: UserService) {
        self.service = service
    }
}

let viewModel = ProfileViewModel(service: NetworkUserService())
```

- **장점**: `let`으로 선언할 수 있어 한 번 정해지면 바뀌지 않는다. 객체가 만들어졌다는 건 의존성이 반드시 들어있다는 뜻이라 컴파일러가 누락을 막아주고, `init` 시그니처에 의존성이 드러난다.
- **단점**: 의존성이 많아지면 `init` 파라미터가 길어진다. `init`을 직접 호출할 수 없는 상황(스토리보드로 만들어지는 뷰 컨트롤러 등)에서는 쓰기 어렵다.

### 2-2. Property Injection (프로퍼티 주입)

객체를 **만든 뒤에** 프로퍼티에 대입해서 넣는다.

```swift
final class ProfileViewController: UIViewController {
    var service: UserService!   // 만든 뒤에 외부에서 대입

    override func viewDidLoad() {
        super.viewDidLoad()
        Task { _ = try? await service.fetchUser(id: "1") }   // 대입 전에 쓰면 크래시
    }
}

let viewController = ProfileViewController()
viewController.service = NetworkUserService()
```

- **언제 쓰나**: `init`을 직접 부를 수 없는 경우, 의존성이 선택적인 경우, 순환 참조를 풀어야 하는 경우(`weak var delegate` 등)
- **단점**: 대입을 깜빡하면 **런타임에 크래시**가 난다. 객체가 만들어진 직후에는 의존성이 비어 있는 상태라서, 언제부터 써도 안전한지를 코드가 보장하지 않는다. `var`라서 중간에 바뀔 수도 있다.

기본값을 두고 테스트에서만 교체하는 변형도 많이 쓴다.

```swift
var service: UserService = NetworkUserService()   // 평소엔 기본값, 테스트에서만 교체
```

### 2-3. Method Injection (메서드 주입)

의존 객체를 **메서드를 호출할 때마다** 인자로 넘긴다.

```swift
final class ProfileViewModel {
    func load(id: String, using service: UserService) async -> User? {
        try? await service.fetchUser(id: id)
    }
}

let user = await viewModel.load(id: "1", using: NetworkUserService())
```

- **언제 쓰나**: 그 의존성이 이 메서드에서만 필요하거나, 호출할 때마다 다른 것을 쓰고 싶을 때
- **단점**: 호출하는 쪽이 매번 넘겨야 해서 번거롭고, 객체가 계속 들고 쓰는 의존성에는 어울리지 않는다.

### 2-4. 비교와 선택 기준

| | Initializer | Property | Method |
|---|---|---|---|
| 넘기는 시점 | 객체 생성 시 | 생성 후 | 메서드 호출 시 |
| 불변(`let`) | 가능 | 불가 (`var`) | 해당 없음 |
| 누락 위험 | 컴파일러가 막음 | **런타임 크래시** | 컴파일러가 막음 |
| 적합한 의존성 | 필수 의존성 | 선택적, `init` 못 쓸 때 | 호출마다 달라지는 것 |

기본은 **Initializer Injection**이다. `init`을 쓸 수 없는 이유가 있을 때만 Property로 가고, 호출 단위로만 필요한 것은 Method로 넘긴다. 스토리보드를 쓰는 뷰 컨트롤러도 `UIStoryboard.instantiateViewController(identifier:creator:)`(iOS 13+)를 쓰면 `init`으로 주입받을 수 있어서, "스토리보드라 어쩔 수 없이 Property Injection"이라는 전제는 예전만큼 강하지 않다.

## 3. DI 컨테이너

### 3-1. 왜 필요한가

DI를 하면 누군가 객체를 **직접 조립**해야 한다. 의존성이 한두 개면 괜찮지만, 의존성의 의존성까지 이어지면 조립 코드가 길어진다.

```swift
// ProfileViewModel → UserService → APIClient → URLSession ...
let session = URLSession.shared
let client = APIClient(session: session)
let service = NetworkUserService(client: client)
let viewModel = ProfileViewModel(service: service)
```

화면마다 이런 조립을 반복하고, 구현을 바꾸려면(`NetworkUserService` → `CachedUserService`) 쓰는 곳마다 고쳐야 한다. **DI 컨테이너는 "무엇을 어떻게 만들지"를 한 곳에 등록해두고, 필요할 때 꺼내 쓰게 해주는 객체**다.

### 3-2. 컨테이너가 하는 일

- **등록(register)**: "`UserService`가 필요하면 `NetworkUserService`를 이렇게 만들어라"를 기록한다.
- **해결(resolve)**: 타입을 요청하면 등록된 방식대로 객체를 만들어서(필요한 하위 의존성도 채워서) 돌려준다.
- **수명 관리**: 요청할 때마다 새로 만들지(transient), 하나만 만들어 재사용할지(singleton)를 정한다.

### 3-3. 간단한 컨테이너 직접 만들어보기

```swift
final class Container {
    enum Scope { case transient, singleton }

    private struct Registration {
        let scope: Scope
        let factory: (Container) -> Any
    }

    private var registrations: [ObjectIdentifier: Registration] = [:]
    private var singletons: [ObjectIdentifier: Any] = [:]

    func register<T>(_ type: T.Type, scope: Scope = .transient, factory: @escaping (Container) -> T) {
        registrations[ObjectIdentifier(type)] = Registration(scope: scope, factory: factory)
    }

    func resolve<T>(_ type: T.Type) -> T {
        let key = ObjectIdentifier(type)
        guard let registration = registrations[key] else {
            fatalError("\(type)이(가) 등록되지 않았습니다")
        }
        if registration.scope == .singleton, let cached = singletons[key] as? T {
            return cached
        }
        let instance = registration.factory(self) as! T
        if registration.scope == .singleton { singletons[key] = instance }
        return instance
    }
}
```

팩토리 클로저가 `Container`를 **파라미터로 받는** 이유는, 컨테이너가 저장한 클로저가 컨테이너를 다시 캡처하면 순환 참조가 생기기 때문이다. 이 구현은 스레드 안전하지 않아서, 실제로 쓴다면 락 같은 보호가 필요하다.

### 3-4. 어디서 등록하고, 어디서 꺼내는가

`register`와 `resolve`는 둘 다 필요하지만 **하는 시점과 장소가 다르고, 클래스 안에서는 둘 다 하지 않는다.**

| | 언제 | 어디서 |
|---|---|---|
| `register` (등록) | 앱 시작 시 **한 번** | 조립 지점 (`SceneDelegate`, `App` 초기화 등) |
| `resolve` (꺼내기) | 객체가 필요한 순간마다 | 객체를 만들어서 쓰는 경계 (화면을 띄우는 코드 등) |
| 클래스 내부 | - | 컨테이너를 모른다. 그냥 `init(service:)`만 가진다. |

```swift
// ① 클래스들은 컨테이너를 모른다. 평범한 initializer injection이다.
final class ProfileViewController: UIViewController {
    private let viewModel: ProfileViewModel

    init(viewModel: ProfileViewModel) {
        self.viewModel = viewModel
        super.init(nibName: nil, bundle: nil)
    }
    required init?(coder: NSCoder) { fatalError() }
}
```

```swift
// ② 앱 시작 시 한 번: 등록
func makeContainer() -> Container {
    let container = Container()
    container.register(UserService.self, scope: .singleton) { _ in NetworkUserService() }
    container.register(ProfileViewModel.self) { c in
        ProfileViewModel(service: c.resolve(UserService.self))   // ← 여기서 init 주입이 일어난다
    }
    return container
}
```

```swift
// ③ 화면을 만드는 지점: resolve해서 전달
let viewModel = container.resolve(ProfileViewModel.self)   // 이미 init 주입이 끝난 객체
let viewController = ProfileViewController(viewModel: viewModel)
navigationController.pushViewController(viewController, animated: true)
```

- **init 주입은 ②의 등록 클로저 안**(`ProfileViewModel(service: c.resolve(...))`)에서 일어난다. 컨테이너가 `init`을 대신 호출해주는 것이라서, `ProfileViewModel`은 자기가 컨테이너를 통해 만들어졌다는 것도 모른다.
- **`resolve`는 최상단 객체에만 한다.** ③에서는 `ProfileViewModel` 하나만 꺼냈고, 그 안의 `UserService`는 ②에서 등록한 대로 컨테이너가 채워줬다. 의존성이 몇 단계로 이어져도 맨 위 객체 하나만 요청하면 된다.
- **테스트에서는 컨테이너가 필요 없다.** 클래스가 컨테이너를 모르니, `ProfileViewModel(service: MockUserService())`처럼 직접 넣으면 된다.

### 3-5. 앱 시작 때 만든 컨테이너를 어떻게 다른 곳에서 쓰는가

컨테이너를 전역(`shared`)으로 만들지 않아도, **앱이 살아있는 동안 유지되는 객체가 들고 있다가 객체를 만드는 쪽에만 `init`으로 전달**하면 된다.

```swift
final class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?
    private var router: AppRouter?   // 앱이 사는 동안 유지되는 참조

    func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options: UIScene.ConnectionOptions) {
        guard let windowScene = scene as? UIWindowScene else { return }

        let container = makeContainer()                              // 등록은 여기서 한 번
        let factory = ViewControllerFactory(container: container)    // 컨테이너를 전달
        let navigationController = UINavigationController()

        let router = AppRouter(factory: factory, navigationController: navigationController)
        self.router = router
        router.showProfile()

        window = UIWindow(windowScene: windowScene)
        window?.rootViewController = navigationController
        window?.makeKeyAndVisible()
    }
}
```

```swift
final class ViewControllerFactory {
    private let container: Container
    init(container: Container) { self.container = container }

    func makeProfile() -> ProfileViewController {
        ProfileViewController(viewModel: container.resolve(ProfileViewModel.self))
    }
}

final class AppRouter {
    private let factory: ViewControllerFactory
    private let navigationController: UINavigationController

    init(factory: ViewControllerFactory, navigationController: UINavigationController) {
        self.factory = factory
        self.navigationController = navigationController
    }

    func showProfile() {
        navigationController.pushViewController(factory.makeProfile(), animated: true)
    }
}
```

```
SceneDelegate → (container) → ViewControllerFactory → (viewModel) → ProfileViewController
```

컨테이너를 아는 건 `SceneDelegate`와 `ViewControllerFactory`뿐이고, `ProfileViewController`와 `ProfileViewModel`은 결과물만 `init`으로 받는다. 화면에서 다른 화면을 열어야 한다면 컨테이너가 아니라 필요한 기능(`AppRouter`)만 `init`으로 주입받는다.

| | `Container.shared` | 조립 지점이 들고 전달 |
|---|---|---|
| 접근 | 앱 어디서든 | 전달받은 객체만 |
| 장점 | 전달 경로를 신경 쓰지 않아도 됨 | 누가 컨테이너를 쓰는지 코드에 드러남 |
| 단점 | 어디서든 `resolve`할 수 있어서 서비스 로케이터로 변질되기 쉬움 | 전달 경로가 길어지면 번거로움 |

`shared` 자체가 금지된 건 아니다. 문제는 어디서든 꺼내 쓸 수 있다는 점이라서, `shared`를 쓰는 프로젝트도 "조립 지점에서만 `resolve`한다"는 규칙으로 통제한다.

### 3-6. 주의점 — 서비스 로케이터로 변질되기 쉽다

컨테이너를 **클래스 안에서 꺼내 쓰기 시작하면** 문제가 생긴다.

```swift
final class ProfileViewModel {
    private let service = Container.shared.resolve(UserService.self)   // 안에서 직접 꺼내 씀
}
```

이건 DI가 아니라 **서비스 로케이터 패턴**이고, 처음에 풀려던 문제로 되돌아간다.

- **의존성이 다시 숨는다.** `ProfileViewModel`이 `UserService`를 필요로 한다는 게 `init`에 보이지 않는다.
- **전역 컨테이너에 의존**하게 되어, 테스트할 때마다 전역 컨테이너를 설정하고 초기화해야 한다.

규칙은 단순하다. **컨테이너는 조립 지점에서만 쓰고, 각 클래스는 `init`으로 주입받는다.** 그리고 등록하지 않은 타입을 요청하면 **런타임에 실패**한다(위 코드의 `fatalError`). `init` 주입은 컴파일러가 누락을 막아주는 것과 대조적이다.

### 3-7. 실제로 쓰이는 도구

직접 만들기보다 검증된 도구를 쓰는 경우가 많다. 대표적으로 Swinject, Factory 같은 서드파티 라이브러리가 있고, 프레임워크가 제공하는 컨테이너 성격의 구조도 있다. TCA의 `DependencyValues`(`@Dependency`)나 SwiftUI의 `Environment`가 "키로 등록해두고 꺼내 쓰는" 구조다.

## 4. 의존성 주입을 사용함으로써 얻는 이점

**1) 테스트가 쉬워진다.** 의존성을 가짜로 바꿔 넣을 수 있어서, 네트워크 없이 성공과 실패 상황을 모두 만들 수 있다.

```swift
@Test
func load_success() async {
    let service = MockUserService()
    service.result = .success(User(name: "tester"))
    let viewModel = ProfileViewModel(service: service)

    await viewModel.load(id: "1")

    #expect(viewModel.user?.name == "tester")
}

@Test
func load_failure() async {
    struct TestError: Error {}
    let service = MockUserService()
    service.result = .failure(TestError())
    let viewModel = ProfileViewModel(service: service)

    await viewModel.load(id: "1")

    #expect(viewModel.user == nil)
}
```

**2) 구현을 바꾸기 쉽다.** `NetworkUserService`를 `CachedUserService`로 바꿔도 `ProfileViewModel`은 수정하지 않고, 조립 지점의 등록 한 줄만 바꾸면 된다.

**3) 결합도가 낮아지고 의존성이 드러난다.** 클래스가 구체 타입이 아니라 프로토콜에 의존하게 되고, `init` 시그니처만 봐도 그 클래스가 무엇을 필요로 하는지 알 수 있다.

## 정리

- DI는 의존 객체를 안에서 만들지 않고 밖에서 넣어주는 기법이고, 프로토콜로 받을 때 테스트와 교체라는 효과가 생긴다.
- 주입 유형은 Initializer(기본), Property(`init`을 못 쓸 때), Method(호출마다 달라질 때) 세 가지이고, 누락을 컴파일러가 막아주는 Initializer Injection이 기본 선택이다.
- DI 컨테이너는 객체 조립을 한 곳에 모아주는 도구이고, 등록은 앱 시작 시 조립 지점에서 한 번, 꺼내기는 객체를 만드는 경계에서만 한다.
- 클래스 안에서 컨테이너를 직접 꺼내 쓰면 서비스 로케이터가 되어 의존성이 다시 숨으므로, 컨테이너는 조립 지점에만 둔다.
