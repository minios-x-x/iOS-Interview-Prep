# Swift Concurrency에서 Task는 어떻게 취소되는가

## `task.cancel()`은 깃발만 꽂는 것

```swift
let task = Task {
    await doSomething()
}

task.cancel()
```

`task.cancel()`을 호출해도 `doSomething()`이 그 즉시 멈추지 않는다. Swift의 Task 취소는 **협조적 취소(cooperative cancellation)** 모델을 따르기 때문이다. `cancel()`은 그 Task의 `isCancelled` 플래그를 `true`로 바꿀 뿐, 실행 중인 코드를 강제로 중단시키지 않는다. 실제로 멈추는 동작은 그 코드 내부에서 "지금 취소됐나?"를 직접 확인하고 반응하도록 **개발자가 직접 구현**해야 한다.

## 구조화된 작업 vs 비구조화된 작업 — 취소가 전파되는 범위가 다르다

Swift Concurrency는 크게 두 가지로 나뉜다.

- **구조화된 작업(Structured Concurrency)** — `async let`, `TaskGroup`. 작업이 선언된 스코프 안에서만 살아있고, 스코프를 벗어나면 자동으로 정리된다.
- **비구조화된 작업(Unstructured Concurrency)** — `Task { }`, `Task.detached { }`. 스코프와 수명이 묶여있지 않고 독립적으로 존재한다.

**취소 전파는 구조화된 작업에서만 잘 일어난다.** 이걸 확인하기 위해, 취소 여부를 직접 체크하면서 1초씩 5번 도는 함수를 하나 만들어보자.

```swift
func work(prefix: String) async {
    for i in 0..<5 {
        if Task.isCancelled {
            print(prefix, "cancelled")
            return
        }
        try? await Task.sleep(for: .seconds(1))
        print(prefix, i)
    }
}
```

이 함수를 구조화된 작업(`async let`)과 비구조화된 작업(`Task { }`) 양쪽에서 동시에 실행시키고, 바깥에서 전체를 취소해보자.

```swift
func execute() async {
    Task {
        await work(prefix: "🔴 nested task")   // 비구조화
    }

    async let _ = work(prefix: "🟡 async let")  // 구조화

    await work(prefix: "🟣 task")               // 구조화 — 같은 Task 안에서 직접 await
}

let task = Task { await execute() }
task.cancel()
```

```
🟡 async let cancelled
🟣 task cancelled
🔴 nested task 0
🔴 nested task 1
🔴 nested task 2
🔴 nested task 3
🔴 nested task 4
```

`task.cancel()`을 호출하자마자 **`🟡 async let`과 `🟣 task`는 바로 취소됐지만, `🔴 nested task`는 취소 신호를 전혀 못 받고 5번을 끝까지 다 돈다.** `🟣 task`는 별도의 Task 경계를 만들지 않고 `execute()`를 감싸는 Task 안에서 그대로 실행되기 때문에, 그 Task가 취소되면 `isCancelled`도 바로 같이 `true`가 된다. 부모 Task(`execute()`를 감싸는 Task)를 취소해도, 내부에 새로 만든 `Task { }`는 **별도의 작업으로 취급돼서 취소가 전파되지 않기 때문**이다. `Task`가 생성 시점의 context(우선순위 등)는 상속하지만, **취소 상태까지 상속하는 건 아니다.** 중첩된 `Task { }`를 취소하고 싶다면 그 Task를 따로 들고 있다가 직접 `cancel()`을 호출해야 한다.

반면 `async let`은 구조화된 작업이라서, 부모가 취소되면 그 취소가 자동으로 전파된다. `TaskGroup`도 마찬가지다.

```swift
func execute() async {
    await withTaskGroup(of: Void.self) { group in
        group.addTask { await work(prefix: "🔵 group task 1") }
        group.addTask { await work(prefix: "🔵🔵 group task 2") }
    }
}

let task = Task { await execute() }
task.cancel()
```

```
🔵 group task 1 cancelled
🔵🔵 group task 2 cancelled
```

`group.addTask { }`로 추가한 자식 작업들도 `TaskGroup` 안에 구조화돼있기 때문에, 바깥 Task가 취소되면 그 취소가 그대로 전파돼서 두 작업 모두 즉시 취소된다.

## `async let`의 특이한 동작 — 즉시 시작하지만, 결과를 안 쓰면 취소된다

`async let`은 선언되는 즉시(=`await`를 기다리지 않고) 자식 Task로 등록되어 백그라운드에서 실행된다.

```swift
func execute() async {
    async let result = work(prefix: "async let")
    await someOtherWork()   // 이 await가 스레드에 틈을 만들어줘야 async let도 실제로 돌 기회를 얻는다
    print(await result)
}
```

여기서 `async let`이 "즉시 시작된다"는 건 Task로 **등록**된다는 뜻이지, 그 순간 바로 스레드 위에서 **실행**된다는 보장은 아니다. 실제로 코드가 돌려면 스케줄러가 틈을 낼 기회(다른 suspend 지점)가 있어야 한다.

그리고 더 중요한 규칙 하나 — **`async let`으로 만든 결과값을 한 번도 `await`하지 않은 채로 그 스코프를 벗어나면, Swift는 그 작업을 암묵적으로 취소시킨다.**

```swift
func execute() async {
    async let _ = work(prefix: "async let")
    print("end")   // 바로 다음 줄에서 스코프가 끝남
}
```

이 코드는 `work`가 단 한 번도 제대로 실행되지 못하고 취소된다. `async let`은 "자기를 만든 스코프보다 더 오래 살아남으면 안 된다"는 구조화된 작업의 규칙을 따르기 때문에, 결과를 안 쓰고 스코프가 끝나버리면 Swift가 정리 차원에서 취소시켜버리는 것이다.

## 전파된 취소를 실제로 이용하는 두 가지 방법

`isCancelled` 플래그가 켜졌다는 걸 알았다고 해도, 그걸 코드에서 확인하는 방법은 성격이 다른 두 가지로 나뉜다.

**폴링(polling) 방식 — `Task.checkCancellation()` / `Task.isCancelled`**

코드 중간중간에서 "지금 취소됐나?"를 직접 확인해야 반응할 수 있다. 위에서 본 `work(prefix:)`의 `if Task.isCancelled { ... }`가 바로 이 방식이다.

`Task.checkCancellation()`은 같은 역할을 하되, `Bool`을 리턴하는 대신 취소 상태면 `CancellationError`를 던진다(throw). 반복문이나 시간이 걸리는 작업 중간중간에 넣어서, 비싼 작업을 하기 전에 취소 여부를 확인하는 용도로 쓴다.

**이벤트 기반(event-driven) 방식 — `withTaskCancellationHandler`**

폴링과 다르게, 취소되는 "순간" 자동으로 콜백이 호출된다.

```swift
await withTaskCancellationHandler {
    await work(prefix: "handler")
} onCancel: {
    print("cancelled immediately")
}
```

`onCancel`은 `operation`이 실행되는 동안 Task가 취소되는 그 즉시 호출된다. 중간중간 직접 체크하지 않아도 되기 때문에, completion handler 기반의 레거시 API처럼 **취소라는 개념 자체를 모르는 코드를 Swift Concurrency의 취소 체계에 끼워 넣을 때** 특히 유용하다.

## 정리

- `task.cancel()`은 즉시 멈추는 명령이 아니라, `isCancelled` 플래그를 세우는 신호일 뿐이다.
- 그 신호가 전파되는 범위는 구조화된 작업(`async let`, `TaskGroup`)인지 비구조화된 작업(`Task { }`, `Task.detached { }`)인지에 따라 다르다. 비구조화된 작업은 취소를 상속하지 않는다.
- `async let`은 즉시 등록되어 백그라운드에서 돌지만, 결과를 한 번도 `await`하지 않고 스코프를 벗어나면 암묵적으로 취소된다.
- 전파된 취소 신호를 코드에서 실제로 이용하려면, 중간중간 직접 확인하는 폴링 방식(`checkCancellation`/`isCancelled`)과 취소되는 순간 자동으로 반응하는 이벤트 기반 방식(`withTaskCancellationHandler`) 중 상황에 맞는 걸 골라 써야 한다.
