# iOS 앱에서 SwiftUI와 UIKit을 함께 사용하는 방법은 무엇인가요?

## 1. 왜 함께 쓰는가 — 두 방향의 브리지

SwiftUI로 앱을 한 번에 갈아엎기는 어렵다. 이미 UIKit으로 만들어진 화면이 많고, SwiftUI에는 없는 UIKit 컴포넌트(`WKWebView`, UIKit 기반 서드파티 SDK 뷰 등)를 그대로 써야 하는 경우도 있다. 실무 프로젝트도 SnapKit 기반의 UIKit 앱이었다가 화면 단위로 SwiftUI로 옮겨가는 중이라, 두 프레임워크가 공존하는 구조가 되었다.

두 프레임워크를 잇는 방법은 방향에 따라 나뉜다.

| 방향 | 사용하는 타입 |
|---|---|
| UIKit 안에서 SwiftUI 뷰 사용 | `UIHostingController` |
| SwiftUI 안에서 UIKit 사용 | `UIViewRepresentable`, `UIViewControllerRepresentable` |

## 2. UIKit 뷰 컨트롤러에서 SwiftUI 뷰 호스팅하기

`UIHostingController`는 SwiftUI 뷰를 `rootView`로 받아서 **UIKit의 뷰 컨트롤러처럼 다룰 수 있게 감싸주는 컨테이너**다. 뷰 컨트롤러이기 때문에 `push`, `present`, 탭바 자식 등 UIKit의 화면 전환 방식을 그대로 쓸 수 있다.

```swift
let hosting = UIHostingController(rootView: HomeView())

navigationController?.pushViewController(hosting, animated: true)
// 또는
present(hosting, animated: true)
```

화면 전체가 아니라 일부 영역에만 넣고 싶다면 자식 뷰 컨트롤러로 추가한다.

```swift
addChild(hosting)
view.addSubview(hosting.view)
hosting.view.translatesAutoresizingMaskIntoConstraints = false
NSLayoutConstraint.activate([
    hosting.view.topAnchor.constraint(equalTo: view.topAnchor),
    hosting.view.leadingAnchor.constraint(equalTo: view.leadingAnchor),
    hosting.view.trailingAnchor.constraint(equalTo: view.trailingAnchor),
    hosting.view.bottomAnchor.constraint(equalTo: view.bottomAnchor),
])
hosting.didMove(toParent: self)
```

(iOS 16부터는 테이블뷰/컬렉션뷰 셀 안에 SwiftUI 뷰를 넣는 `UIHostingConfiguration`도 쓸 수 있다.)

실무 프로젝트에서도 `UITabBarController`의 각 탭을 `UIHostingController(rootView:)`로 구성해서, 탭바 같은 UIKit 컨테이너는 그대로 두고 화면 내용만 SwiftUI로 바꿨다.

## 3. SwiftUI 뷰에서 UIKit 사용하기

### 3-1. `UIViewRepresentable` — UIKit 뷰 감싸기

SwiftUI는 UIKit 뷰를 직접 쓸 수 없으므로, `UIViewRepresentable`을 채택한 래퍼 구조체로 감싼다.

- `makeUIView(context:)` — UIKit 뷰를 **한 번** 만든다.
- `updateUIView(_:context:)` — SwiftUI의 상태가 바뀔 때마다 호출되어, 바뀐 값을 UIKit 뷰에 반영한다.

```swift
struct WebView: UIViewRepresentable {
    let url: URL

    func makeUIView(context: Context) -> WKWebView {
        WKWebView()
    }

    func updateUIView(_ webView: WKWebView, context: Context) {
        // SwiftUI가 다시 그릴 때마다 호출되므로, 바뀐 경우에만 로드한다
        if webView.url != url {
            webView.load(URLRequest(url: url))
        }
    }
}
```

### 3-2. `Coordinator` — 델리게이트를 SwiftUI로 전달하기

UIKit 뷰는 이벤트를 **델리게이트**로 알려주는데, SwiftUI의 `View`는 값 타입(struct)이라 `NSObject`를 상속하는 델리게이트가 될 수 없다. 그래서 `makeCoordinator()`로 델리게이트 역할을 하는 클래스를 만들어 두고, 거기서 이벤트를 SwiftUI 쪽 클로저로 전달한다.

```swift
struct MapView: UIViewRepresentable {
    let onRegionChanged: (MKCoordinateRegion) -> Void

    func makeCoordinator() -> Coordinator {
        Coordinator(onRegionChanged: onRegionChanged)
    }

    func makeUIView(context: Context) -> MKMapView {
        let mapView = MKMapView()
        mapView.delegate = context.coordinator
        return mapView
    }

    func updateUIView(_ mapView: MKMapView, context: Context) {
        // Coordinator는 한 번만 만들어지므로, 클로저가 바뀔 수 있다면 여기서 갱신한다
        context.coordinator.onRegionChanged = onRegionChanged
    }

    final class Coordinator: NSObject, MKMapViewDelegate {
        var onRegionChanged: (MKCoordinateRegion) -> Void

        init(onRegionChanged: @escaping (MKCoordinateRegion) -> Void) {
            self.onRegionChanged = onRegionChanged
        }

        func mapView(_ mapView: MKMapView, regionDidChangeAnimated animated: Bool) {
            onRegionChanged(mapView.region)
        }
    }
}
```

실무 프로젝트에서는 UIKit 기반 서드파티 지도 SDK를 같은 방식으로 감쌌다. 카메라 변경이나 지도 탭 같은 델리게이트 이벤트는 `Coordinator`가 클로저로 전달하고, "카메라를 이 좌표로 이동"처럼 SwiftUI의 선언형 상태로 표현하기 어려운 **명령형 제어**는 지도 뷰를 약한 참조로 들고 있는 `Adapter` 객체를 따로 만들어서 처리했다.

### 3-3. `UIViewControllerRepresentable` — 뷰 컨트롤러 감싸기

뷰가 아니라 **뷰 컨트롤러 단위**로 감싸야 할 때는 `UIViewControllerRepresentable`을 쓴다. 실무 프로젝트에서는 SwiftUI 화면의 루트에서 `UINavigationController`를 감싸서, 화면 전환(push/present, 스와이프 뒤로가기)은 UIKit의 내비게이션을 그대로 쓰도록 했다.

```swift
struct NavigationRoot<Content: View>: UIViewControllerRepresentable {
    let rootView: Content

    func makeUIViewController(context: Context) -> UINavigationController {
        let hosting = UIHostingController(rootView: rootView)
        let navigationController = UINavigationController(rootViewController: hosting)
        navigationController.isNavigationBarHidden = true
        return navigationController
    }

    func updateUIViewController(_ navigationController: UINavigationController, context: Context) {}
}
```

SwiftUI가 `UINavigationController`를 감싸고, 그 안의 화면들은 다시 `UIHostingController`로 호스팅되는 구조라서, **두 방향의 브리지가 하나의 구조 안에 같이 들어있는 셈**이다.

## 4. 혼용할 때 주의할 점

- **환경 객체는 호스팅 경계를 넘지 않는다.** `UIHostingController`로 올린 SwiftUI 뷰는 별개의 SwiftUI 뷰 계층이라서, 위쪽에서 주입한 `environmentObject`가 전달되지 않는다. 각 호스팅 컨트롤러의 `rootView`마다 필요한 환경 객체를 직접 주입해야 한다.
- **`updateUIView`는 자주 호출된다.** SwiftUI 상태가 바뀔 때마다 호출될 수 있으므로, 매번 무조건 갱신하면 `WKWebView`가 다시 로드되는 것 같은 문제가 생긴다. 바뀐 값만 반영하도록 비교해야 한다.
- **호스팅 컨트롤러를 따로 강한 참조로 들고 있지 않는 게 안전하다.** `push`/`present`한 뷰 컨트롤러는 UIKit이 이미 소유하고 있어서, 같은 컨트롤러를 한 번 더 들고 있으면 해제 책임이 두 군데가 되고 잔존 참조가 생기기 쉽다.
- **화면 전환의 소유권을 명확히 해야 한다.** UIKit은 뷰 컨트롤러 라이프사이클(`viewDidAppear` 등)로, SwiftUI는 `onAppear`로 화면 상태를 다룬다. 둘이 섞이면 누가 화면 전환을 책임지는지 모호해지므로, 한쪽(UIKit 내비게이션)이 소유하고 다른 쪽은 요청만 하는 구조가 낫다.
- **`AnyView`는 가능하면 줄인다.** 타입 정보를 지우면 SwiftUI가 뷰 구조를 비교해서 최적화하는 이점이 줄어든다. 서로 다른 뷰를 한 컨트롤러 타입으로 다루려고 쓸 때는 비용을 알고 써야 한다.
- **레이아웃 방식이 다르다.** UIKit의 Auto Layout과 SwiftUI의 크기 결정 방식이 달라서, 뷰를 자식으로 임베드할 때는 제약과 safe area를 직접 챙겨야 한다.

## 정리

- UIKit → SwiftUI는 `UIHostingController`, SwiftUI → UIKit은 `UIViewRepresentable`/`UIViewControllerRepresentable`로 연결한다.
- `UIViewRepresentable`은 `makeUIView`로 한 번 만들고 `updateUIView`로 상태를 반영하며, 델리게이트 이벤트는 `Coordinator`가 SwiftUI로 전달한다.
