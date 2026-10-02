# 기존 completion handler 기반 코드를 async/await로 마이그레이션할 때 어떤 기준으로 우선순위를 정하시겠습니까?

## 우선순위를 정하는 기준 — 영향 범위가 적은 곳부터

마이그레이션 우선순위는 **앱에서 영향을 가장 적게 받는 곳부터** 정했다. 구체적으로는, 사용자 입장에서 핵심 플로우가 아닌 부가적인 기능부터 시작해서, 점차 핵심 플로우(중앙)로 들어가는 방식이었다. async/await로 먼저 만들어둔 네트워크 함수들을, 영향이 적은 화면에서부터 테스트하면서 점진적으로 확대해나갈 수 있다고 판단했다.

이렇게 "주변부"부터 시작한 이유는 단순히 안전해서만은 아니다. 처음 해보는 마이그레이션은 크고 작은 시행착오가 생길 수밖에 없는데, 그 시행착오의 비용을 영향이 적은 곳에서 먼저 치르고, 거기서 쌓은 패턴·에러 처리 경험을 바탕으로 핵심 플로우에 적용하면 실수할 확률을 줄일 수 있기 때문이다.

## 그 외에 고려할 수 있는 다른 기준들

영향 범위 외에도 우선순위를 정하는 기준은 여러 가지가 있을 수 있다.

- **콜백 중첩/복잡도** — completion handler가 여러 단계로 중첩돼서 콜백 지옥이 심한 곳일수록, async/await로 바꿨을 때 가독성·안정성 이득이 크다. 예를 들어 "로그인 성공 후 유저 정보 조회, 그 응답으로 설정값 조회"처럼 3단 이상 중첩된 코드가 있다면 그런 곳을 먼저 정리한다.
- **테스트 커버리지 유무** — 테스트가 있는 영역은 마이그레이션 후 회귀(regression)를 바로 검증할 수 있어 먼저 시작하기 안전하다. 테스트가 없는 곳은 리스크가 커서 신중해야 한다.
- **외부 노출 여부(API 경계)** — 다른 모듈·팀이 쓰는 public 인터페이스는 한 번에 바꾸면 파급력이 크므로, `withCheckedContinuation`으로 감싸서 "내부는 async, 겉은 여전히 completion handler"처럼 점진적으로 다리를 놓는 전략이 필요하다.
- **실제 버그가 있었던 영역** — race condition, 순환 참조로 인한 메모리 누수, 콜백 호출 누락 같은 기존 문제가 있던 곳이라면, 마이그레이션 자체가 버그 수정 효과까지 있어서 우선순위가 높아질 수 있다.
- **의존성의 레버리지 지점(leverage point)을 먼저 뚫기** — 어떤 코드를 바꾸느냐뿐 아니라, 얼마나 많은 코드가 의존하고 있는 공통 지점인가도 중요한 기준이 된다. 예를 들어 Moya처럼 많은 API 호출이 공통으로 거쳐가는 네트워크 레이어가 여전히 completion handler 기반이라면, 그 단 하나의 경계만 `withCheckedContinuation`으로 async화해도 그 위에 쌓인 모든 API 호출 지점이 한 번에 async/await로 작성 가능해진다. 개별 호출부를 하나하나 바꾸는 것보다, 가장 많은 코드가 공유하는 뿌리(root) 지점을 먼저 뚫는 게 훨씬 레버리지가 크다.

## 실제 마이그레이션 코드

실무 프로젝트에서 Moya 기반 네트워크 레이어를 async/await로 옮길 때, Moya의 completion 기반 `request` 함수만 `withCheckedContinuation`으로 감쌌다.

```swift
extension MoyaProvider {
    func request(_ target: Target) async -> Result<Response, MoyaError> {
        await withCheckedContinuation { continuation in
            self.request(target) { result in
                continuation.resume(returning: result)
            }
        }
    }
}
```

그 위에 쌓이는 공통 네트워크 함수는 처음부터 `async throws`로 작성했다.

```swift
protocol ClientProtocol {
    func codable<T: Decodable, E: TargetType>(_ target: E) async throws -> T
}

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

Moya라는 레거시 경계 한 곳에만 다리를 놓고, 그 위의 코드는 추가로 continuation을 쓸 필요 없이 자연스럽게 `await`로 이어 붙일 수 있었다.

## 이 코드에 대한 피드백 — 놓친 부분

이 코드엔 아쉬운 점이 하나 있다. **`withCheckedContinuation`만으로는 Task 취소가 전파되지 않는다.** Task는 협조적 취소(cooperative cancellation) 모델을 따르기 때문에, `task.cancel()`을 호출해도 그 신호에 반응하는 코드가 없으면 아무 일도 일어나지 않는다. `withCheckedContinuation`은 completion 기반 코드를 async로 이어주는 다리 역할만 할 뿐, 취소라는 개념 자체를 전혀 모른다. 그래서 사용자가 화면을 빨리 벗어나서 Task가 취소돼도, 실제 네트워크 요청은 그대로 끝까지 나가고 서버 리소스도 낭비된다.

취소까지 전파하려면 `withTaskCancellationHandler`를 추가로 결합해야 한다. 이때 `operation` 클로저와 `onCancel` 클로저가 같은 상태를 동시에 읽고 쓸 수 있기 때문에, 그 상태는 락으로 보호해야 한다.

```swift
final class Cancellation: @unchecked Sendable {
    private let lock = NSLock()
    private var cancellable: Cancellable?
    private var isCancelled = false

    func set(_ cancellable: Cancellable) {
        lock.withLock {
            if isCancelled {
                cancellable.cancel()
            } else {
                self.cancellable = cancellable
            }
        }
    }

    func cancel() {
        lock.withLock {
            isCancelled = true
            cancellable?.cancel()
        }
    }
}

extension MoyaProvider {
    func request(_ target: Target) async -> Result<Response, MoyaError> {
        let box = Cancellation()

        return await withTaskCancellationHandler {
            await withCheckedContinuation { continuation in
                let token = self.request(target) { result in
                    continuation.resume(returning: result)
                }
                box.set(token)
            }
        } onCancel: {
            box.cancel()
        }
    }
}
```

`withTaskCancellationHandler`가 `operation`(실제로 suspend되는 구간 전체)을 감싸고 있어야, 그 구간에서 Task가 취소될 때 `onCancel`이 호출되어 Moya의 실제 요청을 끊을 수 있다. 다만 `onCancel`은 `operation`과 동시에, 서로 다른 스레드에서 실행될 수 있어서 `cancellable` 프로퍼티를 그냥 공유하면 데이터 레이스가 된다. `Cancellation`은 이 접근을 `NSLock`으로 직렬화하고, `isCancelled` 플래그로 "토큰이 세팅되기 전에 취소가 먼저 들어온" 경우까지 처리한다 — 그 경우 `set`이 뒤늦게 호출돼도 방금 만든 요청을 바로 취소시킨다.

호출하는 쪽에서는 이 함수를 `Task`로 감싸서 쓰고, 그 `Task` 자체를 취소 핸들로 사용한다.

```swift
let task = Task {
    let result = await provider.request(target)
}

// 화면을 벗어나는 등 결과가 더 이상 필요 없어지면
task.cancel()
```

`task.cancel()`을 호출하면 그 취소 신호가 `withTaskCancellationHandler`의 `onCancel`로 전달되고, 거기서 `Cancellation.cancel()`이 실행되어 실제 Moya 네트워크 요청까지 끊긴다. 함수가 별도로 `Cancellable`을 리턴하지 않아도, 호출부의 `Task`가 그 역할을 대신하는 구조다.

## `withCheckedContinuation` vs `withUnsafeContinuation`

당시엔 일반적으로 권장되는 `withCheckedContinuation`(논스로잉)과 `withCheckedThrowingContinuation`(스로잉) 두 가지를 기준으로 공부하고 사용했는데, 이번에 다시 살펴보면서 `withUnsafeContinuation`/`withUnsafeThrowingContinuation`이라는 비검증 버전도 있다는 걸 알게 됐다. 두 버전의 차이는, `Checked`는 continuation이 정확히 한 번만 resume되는지 런타임에 검증해주고, `Unsafe`는 그 검증 없이 조금 더 빠르게 동작한다는 점이다. 일반적으로는 `Checked`를 기본으로 쓰고, 성능이 중요한 핫패스에서 이미 안전성을 검증한 경우에만 `Unsafe`로 바꾸는 걸 권장한다. 네트워크 요청처럼 호출 빈도가 성능에 큰 영향을 주지 않는 지점에서는 `Checked`가 더 안전한 기본 선택이라고 생각한다.

## 결론

마이그레이션 우선순위는 영향 범위, 콜백 복잡도, 테스트 커버리지, API 경계, 기존 버그 여부 같은 여러 기준으로 정할 수 있는데, 실제로는 "영향이 적은 곳에서 패턴을 검증하고 핵심 플로우로 확장한다"는 전략을 택했다. 그리고 레거시 completion 기반 라이브러리(Moya)와의 경계는 `withCheckedContinuation`으로 최소한의 지점에만 다리를 놓아, 그 위의 코드는 자연스럽게 async/await로 작성할 수 있었다. 다만 이 다리는 Task 취소까지는 자동으로 전파해주지 않기 때문에, 완전한 마이그레이션을 위해서는 `withTaskCancellationHandler`로 취소 신호까지 명시적으로 연결해줘야 한다는 걸 이번에 다시 점검하며 깨달았다.
