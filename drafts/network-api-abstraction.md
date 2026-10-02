# 여러 개의 비슷한 네트워크 API 클라이언트를 프로토콜로 추상화하려면 어떻게 설계하시겠습니까?

## 문제 상황 — 왜 API 클라이언트들이 비슷해지는가

앱 하나가 백엔드 API를 하나만 호출하는 경우는 거의 없다. 로그인 API, 상품 조회 API, 결제 API처럼 기능별로 여러 API를 각각 호출하게 되는데, 이때 순진하게 짜면 API마다 따로 클라이언트를 만들게 된다.

```swift
class AuthAPIClient {
    func login(email: String, password: String, completion: @escaping (Result<User, Error>) -> Void) {
        guard let url = URL(string: "https://api.example.com/auth/login") else { return }
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.httpBody = try? JSONEncoder().encode(["email": email, "password": password])

        URLSession.shared.dataTask(with: request) { data, _, error in
            if let error = error { completion(.failure(error)); return }
            guard let data = data else { return }
            do {
                completion(.success(try JSONDecoder().decode(User.self, from: data)))
            } catch {
                completion(.failure(error))
            }
        }.resume()
    }
}
```

이런 클래스가 API 개수만큼 늘어나면, 하는 일(로그인 vs 상품 조회)은 다른데 **뼈대(`URLRequest` 생성 → `URLSession`으로 전송 → 디코딩 → 에러 처리)는 거의 판박이**가 된다. "공통 헤더를 추가해야 한다"는 요구사항이 하나 생기면, 클라이언트 개수만큼 코드를 다 찾아다니며 고쳐야 하는 문제가 생긴다.

## 프로토콜로 추상화하기

핵심은 "각 API마다 다른 부분"(URL, 메서드, 파라미터, 응답 타입)과 "모든 API가 공통으로 하는 일"(요청 전송, 디코딩, 에러 처리)을 분리하는 것이다.

```swift
protocol APIRequest {
    associatedtype Response: Decodable
    var url: URL { get }
    var method: String { get }
    var headers: [String: String] { get }
    var body: Data? { get }
}

struct NetworkClient {
    func send<Request: APIRequest>(
        _ request: Request,
        completion: @escaping (Result<Request.Response, Error>) -> Void
    ) {
        var urlRequest = URLRequest(url: request.url)
        urlRequest.httpMethod = request.method
        request.headers.forEach { urlRequest.setValue($1, forHTTPHeaderField: $0) }
        urlRequest.httpBody = request.body

        URLSession.shared.dataTask(with: urlRequest) { data, _, error in
            if let error = error { completion(.failure(error)); return }
            guard let data = data else { return }
            do {
                completion(.success(try JSONDecoder().decode(Request.Response.self, from: data)))
            } catch {
                completion(.failure(error))
            }
        }.resume()
    }
}
```

이제 각 API는 "이 요청이 어떤 모양인지"만 선언하는 작은 구조체가 된다.

```swift
struct LoginRequest: APIRequest {
    typealias Response = User
    let email: String
    let password: String

    var url: URL { URL(string: "https://api.example.com/auth/login")! }
    var method: String { "POST" }
    var headers: [String: String] { ["Content-Type": "application/json"] }
    var body: Data? { try? JSONEncoder().encode(["email": email, "password": password]) }
}
```

반복되는 로직(`URLSession` 호출, 디코딩, 에러 처리)은 `NetworkClient.send` 한 곳에만 있고, 새 API가 추가돼도 `APIRequest`를 채택한 작은 구조체 하나만 더 만들면 된다. "공통 헤더 추가" 같은 변경도 `NetworkClient.send` 한 곳만 고치면 모든 API에 자동으로 적용된다.

이 패턴을 라이브러리로 만든 게 **Moya**다. `TargetType`이 위 `APIRequest`에, `MoyaProvider<Target>`이 `NetworkClient`에 대응한다. Moya는 여기에 Alamofire 기반 전송, 테스트용 stub, 로깅/인증 토큰 첨부 같은 plugin 시스템을 추가로 얹은 것이다.

## 실전 경험 — ClientProtocol

실무 프로젝트에서는 Moya를 그대로 쓰지 않고, 그 위에 한 번 더 감싼 `ClientProtocol`을 만들어 사용했다.

```swift
protocol ClientProtocol {
    func codable<T: Decodable, E: TargetType>(_ target: E) async throws -> T
    func void<E: TargetType>(_ target: E) async throws
}
```

```swift
func codable<T: Decodable, E: TargetType>(_ target: E) async throws -> T {
    let provider: MoyaProvider<E> = .init(session: self.session)
    let response = await provider.request(target)

    switch response {
    case .success(let value):
        return value
    case .failure(let error):
        if error.response?.statusCode == 426 {
            // Should Update
        }
        throw error
    }
}
```

사용하는 쪽은 API 클라이언트(Moya의 `TargetType` case) + `ClientProtocol` 조합으로 호출한다.

```swift
let target = Users.login(info)
let result = try await network.codable(target)
```

이렇게 감싼 이유는 두 가지였다.

1. **테스트 용이성** — `ClientProtocol`로 추상화해두면, 테스트에서 실제 네트워크 대신 가짜 구현체를 주입할 수 있을 거라 생각했다.
2. **공통 관심사(cross-cutting concern)의 중앙 집중화** — 426(업데이트 필요) 상태 코드 감지 같은 에러 처리 로직을 API 클라이언트마다 반복하지 않고 `codable` 한 곳에서 처리한다. 어떤 API를 호출하든 이 로직이 자동으로 적용되고, 나중에 공통 로직이 바뀌어도 이 함수 하나만 고치면 된다.

## 설계의 허점 — 제네릭 함수는 Mock하기 어렵다

위 설계에서 실제로 `ClientProtocol`을 mock으로 만들어 테스트해본 적은 없었는데, 시도해보면 바로 문제에 부딪힌다.

```swift
class MockClient: ClientProtocol {
    func codable<T: Decodable, E: TargetType>(_ target: E) async throws -> T {
        // T가 뭔지 이 함수 안에서는 알 수 없다.
        // 로그인 테스트에서는 User를, 상품 조회 테스트에서는 [Product]를 리턴해야 하는데
        // 이 함수 하나가 모든 경우를 다 커버해야 한다.
    }
}
```

`T`는 호출부의 타입 추론으로 결정되는 것이라, mock 내부에서는 미리 알 수 없다. 억지로 만들면 `T.self == User.self`처럼 타입을 하나씩 분기해서 강제 캐스팅(`as!`)하는 코드를 계속 늘려야 하고, 테스트가 늘어날수록 이 mock 함수는 점점 지저분한 `if-else` 덩어리가 된다.

## 학습 — TCA 학습 프로젝트에서 배운 테스트 가능한 설계

TCA 학습 프로젝트에서 `DependencyKey` 패턴을 써보면서, 이 문제를 자연스럽게 피해가는 구조를 접하게 됐다. 핵심은, 제네릭 네트워크 레이어를 외부에 직접 노출하지 않고 **구체적인 클로저 인터페이스 뒤에 숨긴 것**이다.

```swift
struct WeatherAdapter {
    var fetchCurrentWeather: (Locality.Coordinate) async throws -> Weather
    var fetchCurrentForecast: (Locality.Coordinate) async throws -> Forecast
}

extension WeatherAdapter: DependencyKey {
    static var liveValue: WeatherAdapter {
        .init(
            fetchCurrentWeather: { coord in
                try await NetworkClient.shared.codable(WeatherTarget.current(coord: coord))
            },
            fetchCurrentForecast: { coord in
                try await NetworkClient.shared.codable(WeatherTarget.forecast(coord: coord))
            }
        )
    }
}
```

`NetworkClient.shared.codable(...)`이라는 제네릭 호출은 `liveValue` 내부에만 존재한다. 바깥에 노출되는 `WeatherAdapter`의 `fetchCurrentWeather`는 `(Coordinate) async throws -> Weather`라는 **완전히 구체적인 타입**이다. 그래서 테스트에서는 이 클로저를 통째로 바꿔치기하기만 하면 된다.

```swift
withDependencies: {
    $0.weatherAdapter.fetchCurrentWeather = { _ in .mock }
    $0.weatherAdapter.fetchCurrentForecast = { _ in .mock }
}
```

타입을 분기할 필요도, 강제 캐스팅도 필요 없다. `Weather`라는 정해진 타입 하나만 다루는 클로저이기 때문이다.

## 결론 — 공통 로직의 분리, 그리고 테스트 가능한 설계

네트워크 레이어의 추상화는 기본적으로 **각 API마다 다른 부분과, 모든 API가 공통으로 하는 일을 분리하는 것**이다. 요청 전송, 디코딩, 에러 처리 같은 반복되는 로직을 제네릭 공통 실행기(`codable<T, E>`) 한 곳에 모아 코드 중복을 없애는 것까지가 기본 추상화다.

여기서 더 나아가 **테스트 가능한 설계**를 하려면, 이 제네릭 레이어를 내부 구현 디테일로 감춘 **구체적인 인터페이스(`fetchCurrentWeather`, `login(info:)` 등)를 하나 더 두고, 그 인터페이스를 통해 mock을 주입받을 수 있게** 만들어야 한다. 제네릭 레이어에 직접 mock을 주입하려 하면 타입 추론이 안 되어 테스트가 지저분해지지만, 기능 단위로 명확한 입출력 타입을 가진 구체적인 인터페이스에서는 클로저나 구현체를 통째로 갈아끼우기만 하면 되기 때문이다.

실무에서 쓴 `ClientProtocol`은 공통 로직 분리까지는 잘 해결했지만 이 테스트 가능한 설계까지는 미치지 못했다는 걸, TCA 학습 프로젝트에서 `DependencyKey` 기반 구조를 써보면서 깨달을 수 있었다.
