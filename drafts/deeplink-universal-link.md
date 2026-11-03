# iOS 앱에서 Deep Link와 Universal Link의 차이점은 무엇인가요?

## 1. 두 링크의 개요

**Deep Link**는 원래 "앱 안의 특정 화면이나 콘텐츠로 바로 연결하는 링크" 전체를 가리키는 넓은 말이다. 다만 iOS에서 Universal Link와 짝으로 비교할 때는 보통 **커스텀 URL 스킴**(`example://...`)을 이용한 방식을 뜻하고, 이 글도 그 기준으로 설명한다.

| | Deep Link (커스텀 스킴) | Universal Link |
|---|---|---|
| 형태 | `example://product?id=123` | `https://example.com/product/123` |
| 소유권 검증 | 없음 (다른 앱이 같은 스킴 등록 가능) | 있음 (AASA로 도메인↔앱 연결 검증) |
| 앱 미설치 시 | 열리지 않음 (대체 경로 없음) | 웹페이지로 열림 |
| 설정 | Info.plist URL Types | 서버의 AASA 파일 + Associated Domains |
| 앱에서 받는 곳 | `openURLContexts` | `continue userActivity` |
| 주로 쓰는 곳 | 앱 간 호출, 로그인 콜백 | 공유 링크, 마케팅, 웹 → 앱 |

## 2. Deep Link를 구현하는 방법과 주의 사항

### 2-1. 받기 — URL Types 등록과 `openURLContexts`

앱이 받을 스킴을 Info.plist의 URL Types에 등록하면, iOS가 그 스킴의 URL을 이 앱으로 연결해준다. 서버나 별도 검증은 필요 없다.

```xml
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLName</key>
        <string>com.example.app</string>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>example</string>
        </array>
    </dict>
</array>
```

등록은 "연결"까지만 해준다. URL을 받은 뒤 무엇을 할지는 직접 구현해야 한다.

```swift
func scene(_ scene: UIScene, openURLContexts URLContexts: Set<UIOpenURLContext>) {
    guard let url = URLContexts.first?.url else { return }
    // example://product?id=123
    guard url.scheme == "example", url.host == "product" else { return }

    let items = URLComponents(url: url, resolvingAgainstBaseURL: false)?.queryItems
    guard let id = items?.first(where: { $0.name == "id" })?.value else { return }

    openProduct(id: id)
}
```

Scene을 쓰지 않는 앱에서는 `application(_:open:options:)`가 같은 역할을 한다. 앱이 **꺼져 있는 상태에서** 링크로 실행되면 이 메서드가 아니라 `willConnectTo`의 `connectionOptions.urlContexts`로 URL이 들어온다.

**SwiftUI에서 받기**

SwiftUI에서는 `.onOpenURL`로 받는다. 아래는 받은 URL의 상품 id를 `NavigationStack`의 경로에 추가해서 상품 화면으로 이동하는 예시다. (`NavigationStack`은 iOS 16 이상이다.)

```swift
struct ContentView: View {
    @State private var path: [String] = []   // 상품 id를 쌓는 내비게이션 경로

    var body: some View {
        NavigationStack(path: $path) {
            HomeView()
                .navigationDestination(for: String.self) { id in
                    ProductView(id: id)
                }
        }
        .onOpenURL { url in
            // example://product?id=123
            guard url.scheme == "example", url.host == "product",
                  let id = URLComponents(url: url, resolvingAgainstBaseURL: false)?
                      .queryItems?.first(where: { $0.name == "id" })?.value
            else { return }

            path.append(id)
        }
    }
}
```

URL을 검증하는 부분은 UIKit 예시와 똑같고, 달라지는 건 **받는 곳(`onOpenURL`)과 화면 이동 방식(경로 상태를 바꾸는 것)**뿐이다.

### 2-2. 보내기 — `open`과 `LSApplicationQueriesSchemes`

URL Types는 **내 앱으로 들어오는** 방향의 설정이다. 내 앱에서 다른 앱으로 나가는 방향은 다르다.

| 방향 | 하는 일 | 설정 |
|---|---|---|
| 들어옴 (다른 곳 → 내 앱) | 내 스킴 URL로 내 앱이 열림 | Info.plist의 URL Types |
| 나감 (내 앱 → 다른 앱) | `UIApplication.shared.open(url)`로 상대 앱을 엶 | 별도 등록 필요 없음 |
| 나가기 전 확인 | `canOpenURL(_:)`로 상대 앱이 설치돼 있는지 확인 | Info.plist의 `LSApplicationQueriesSchemes`에 스킴 등록 |

```swift
if let url = URL(string: "kakaotalk://"), UIApplication.shared.canOpenURL(url) {
    UIApplication.shared.open(url)
}
```

`LSApplicationQueriesSchemes`에 등록하지 않으면 `canOpenURL`은 항상 `false`를 돌려주고, 등록할 수 있는 스킴은 최대 50개다.

실무 프로젝트의 카카오 로그인이 양방향을 모두 쓰는 예다. 카카오톡 앱으로 **나갈 때**는 설치 여부를 확인하려고 `LSApplicationQueriesSchemes`에 `kakaotalk` 등을 등록했고, 로그인이 끝나고 **돌아올 때**는 URL Types에 등록한 `kakao<앱키>` 스킴이 `openURLContexts`로 들어와서 SDK에 넘겨줬다.

### 2-3. 주의 사항

**1) 스킴은 소유권 검증이 없다.** 같은 스킴을 등록한 앱이 여러 개면 어느 앱이 열릴지 보장되지 않는다. 그리고 URL은 누구나 만들어서 호출할 수 있으므로, 받은 값은 신뢰하지 말고 검증해야 한다.

**2) 앱이 꺼진 상태(콜드 스타트)를 처리해야 한다.** 링크로 앱이 처음 실행되면 URL은 `willConnectTo`로 들어오는데, 이 시점에는 메인 화면이 아직 준비되지 않았다. 바로 화면을 열려고 하면 실패하거나 엉뚱한 곳으로 가기 때문에, 처리할 동작을 **저장해뒀다가 메인 화면이 준비된 뒤에 실행**해야 한다.

```swift
final class LinkRouter {
    static let shared = LinkRouter()

    private var isReady = false
    private var pendingAction: (() -> Void)?

    func route(_ action: @escaping () -> Void) {
        if isReady {
            action()
        } else {
            pendingAction = action   // 메인 화면이 준비될 때까지 보관
        }
    }

    // 메인 화면 준비가 끝난 시점에 호출
    func didBecomeReady() {
        isReady = true
        pendingAction?()
        pendingAction = nil
    }
}
```

**3) 처리 전에 전제 조건을 확인해야 한다.** 실무 프로젝트에서는 자체 스킴으로 식별값을 전달받아 해당 항목을 서버에 등록하는 기능이 있었는데, 쿼리에 필요한 값이 있는지 확인하고, 로그인 상태인지 토큰 재발급으로 확인한 뒤에야 등록을 진행했다. 링크는 로그인 여부와 상관없이 열리기 때문에, 링크가 열렸다는 사실이 "처리해도 되는 상태"를 보장하지 않는다.

## 3. Universal Link의 동작 원리와 설정 방법

### 3-1. 동작 원리 — 앱과 도메인의 양방향 선언

Universal Link는 내 도메인의 평범한 `https://` 링크인데, 앱이 설치돼 있으면 Safari 대신 앱이 열리는 방식이다. 이게 가능하려면 **앱과 도메인이 서로를 가리키고 있어야** 한다.

- **앱 → 도메인**: 앱의 Associated Domains에 `applinks:example.com`을 등록
- **도메인 → 앱**: 서버가 AASA 파일로 "이 앱이 이 도메인을 처리한다"고 선언

AASA는 `apple-app-site-association`을 줄인 말이다. 한쪽만 있으면 연결되지 않고, 이 양방향 확인 때문에 내 도메인을 다른 앱이 가로챌 수 없다.

앱을 설치하거나 업데이트할 때, iOS는 앱의 `applinks:` 도메인을 보고 그 도메인의 AASA를 Apple CDN에 요청한다. CDN은 필요할 때 내 서버의 `https://example.com/.well-known/apple-app-site-association`을 가져와 캐시한다. 즉 AASA를 Apple에 올리는 게 아니라, **내 서버에 올려두면 Apple CDN이 가져가는** 구조다. 이 캐시 때문에 AASA를 고쳐도 반영에 시간이 걸릴 수 있다.

### 3-2. 설정 방법

**1) 서버: AASA 파일 제공**

```
https://example.com/.well-known/apple-app-site-association
```

확장자 없는 JSON이고, HTTPS로 제공해야 하며, 리다이렉트가 있으면 안 된다.

```json
{
  "applinks": {
    "details": [
      {
        "appIDs": ["TEAMID.com.example.app"],
        "components": [
          { "/": "/_/*", "exclude": true },
          { "/": "/product/*" }
        ]
      }
    ]
  }
}
```

**2) 앱: Associated Domains 등록**

Xcode의 Signing & Capabilities에서 Associated Domains를 추가하고 `applinks:example.com`을 등록한다. 서브도메인은 별개의 도메인으로 취급되어 각각 등록해야 하고, 여러 서브도메인은 `applinks:*.example.com`처럼 와일드카드로 쓸 수 있다.

```xml
<key>com.apple.developer.associated-domains</key>
<array>
    <string>applinks:example.com</string>
</array>
```

개발 중에는 `applinks:example.com?mode=developer`로 등록하면 Apple CDN 캐시를 거치지 않고 서버에서 바로 받아와서 수정 사항을 빠르게 확인할 수 있다.

**3) 앱: `continue userActivity`로 받기**

```swift
func scene(_ scene: UIScene, continue userActivity: NSUserActivity) {
    guard userActivity.activityType == NSUserActivityTypeBrowsingWeb,
          let url = userActivity.webpageURL else { return }
    // https://example.com/product/123
    handle(url)
}
```

앱이 꺼진 상태에서는 `willConnectTo`의 `connectionOptions.userActivities`로 들어온다.

**SwiftUI에서 받기**

SwiftUI에서는 `.onContinueUserActivity(NSUserActivityTypeBrowsingWeb)`로 받는다. 위의 `ContentView`에 이어서 붙이면 된다.

```swift
struct ContentView: View {
    @State private var path: [String] = []

    var body: some View {
        NavigationStack(path: $path) {
            HomeView()
                .navigationDestination(for: String.self) { id in
                    ProductView(id: id)
                }
        }
        .onOpenURL { url in
            // example://product?id=123 (Deep Link)
            // ...
        }
        .onContinueUserActivity(NSUserActivityTypeBrowsingWeb) { activity in
            // https://example.com/product/123 (Universal Link)
            guard let url = activity.webpageURL,
                  url.host == "example.com",
                  url.pathComponents.count >= 3, url.pathComponents[1] == "product"
            else { return }

            path.append(url.pathComponents[2])
        }
    }
}
```

(SwiftUI 앱 라이프사이클에서는 Universal Link가 `.onOpenURL`로도 전달되는 경우가 있다. OS 버전에 따라 어느 쪽으로 들어오는지가 다를 수 있으니, 실제 기기에서 확인하고 같은 링크를 두 번 처리하지 않도록 주의해야 한다.)

### 3-3. AASA의 `paths`와 `components`

`applinks:example.com`은 "이 도메인과 이 앱이 연결돼 있다"까지만 선언한다. 그 도메인의 **어떤 경로를 앱에서 열지**는 AASA의 규칙이 결정한다.

| URL | 규칙 결과 |
|---|---|
| `https://example.com/product/123` | `/product/*`에 맞음 → 앱에서 열림 |
| `https://example.com/blog/post-1` | 어느 규칙에도 안 맞음 → 웹(Safari)에서 열림 |
| `https://example.com/_/internal` | `exclude` 규칙 → 앱으로 보내지 않음 |

- 규칙은 **위에서부터 처음 맞는 것**이 적용되므로 제외 규칙을 앞에 둬야 한다.
- `*`는 임의의 문자열, `?`는 한 글자를 뜻한다.

AASA의 규칙은 두 가지 형식으로 쓸 수 있고, **제외하는 방법이 서로 다르다.**

| | `paths` (iOS 13 이전부터 쓰던 형식) | `components` (iOS 13 이상) |
|---|---|---|
| 앱 ID | `"appID"` (단수) | `"appIDs"` (배열) |
| 포함 | 경로 문자열 `"/product/*"` | 객체 `{ "/": "/product/*" }` |
| **제외** | **`"NOT "` 접두어** `"NOT /_/*"` | **`"exclude": true`** `{ "/": "/_/*", "exclude": true }` |
| 매칭 대상 | 경로 | 경로(`"/"`), 쿼리(`"?"`), 프래그먼트(`"#"`) |

```json
"paths": ["NOT /_/*", "/product/*"]
```

```json
"components": [
  { "/": "/_/*", "exclude": true },
  { "/": "/product/*" }
]
```

두 형식 모두 위에서부터 처음 맞는 규칙이 적용되므로 제외 규칙을 앞에 둬야 한다. `components`는 쿼리와 프래그먼트까지 조건을 걸 수 있어서 더 정밀하다.

### 3-4. 언제 동작하고, 언제 동작하지 않는가

Universal Link는 사용자가 링크를 **탭**했을 때 동작한다.

| 상황 | 결과 |
|---|---|
| 메시지, 메모, 메일 등 다른 앱의 링크를 탭 | 앱에서 열림 |
| **다른 도메인**의 웹페이지 안 링크를 탭 | 앱에서 열림 |
| Safari 주소창에 직접 입력하거나 붙여넣기 | 웹에서 열림 |
| **같은 도메인** 안에서 링크를 탭 | 앱이 아니라 Safari 안에서 이동 |
| 앱 미설치, 또는 AASA 규칙에 안 맞음 | 웹에서 열림 |

"앱이 있는지"를 탭할 때마다 서버에 물어보는 게 아니라, 앱을 설치할 때 검증해 둔 연결 정보로 기기가 판단한다. 앱과의 연결이 없으면 원래의 `https` 링크대로 웹에서 열리기 때문에, 자연스러운 대체 경로가 된다.

### 3-5. 실무 사례 — Firebase Dynamic Links

실무 프로젝트에서는 Firebase Dynamic Links를 사용했다. 앱 쪽에서는 Associated Domains 등록과 `continue userActivity` 처리를 구현했고, **AASA는 직접 올리지 않았다.** Dynamic Link 도메인은 Firebase가 호스팅하는 도메인이라, AASA도 Firebase가 대신 제공했기 때문이다. 받아본 AASA에는 Firebase 프로젝트에 등록된 앱들이 한 파일에 모여 있었다.

링크를 받은 뒤에는 `DynamicLinks.dynamicLinks().handleUniversalLink(url)`로 짧은 링크를 원래 목적지 URL로 풀고, 그 URL로 화면을 이동했다. 이때도 토큰 재발급으로 로그인 상태를 확인한 뒤에 진행했고, 앱이 꺼진 상태라면 처리할 동작을 보관했다가 메인 화면이 준비된 후 실행했다.

```
짧은 링크(Universal Link)로 앱이 열림 → continue userActivity로 URL 수신
→ handleUniversalLink가 짧은 링크를 원래 목적지 URL로 풀어줌 → 앱이 그 URL로 화면 이동
```

즉 Dynamic Link는 Universal Link에 "짧은 링크를 원래 목적지로 풀어주는 서비스"를 얹은 것이다. (Firebase Dynamic Links는 이후 서비스가 종료되었으므로, 지금 새로 구성한다면 AASA 호스팅과 링크 해석을 직접 해야 한다.)

## 4. Deep Link와 Universal Link를 함께 사용하는 경우의 장점

**1) 상황에 맞게 역할을 나눌 수 있다.** 같은 앱 안에서도 용도가 다르다. 앱 간 호출이나 로그인 콜백처럼 **앱끼리 직접 주고받는** 경우는 서버 설정이 필요 없는 커스텀 스킴이 간단하고, 외부에 공유하거나 마케팅에 쓰는 링크는 앱이 없을 때 웹으로 연결되는 Universal Link가 적합하다. 실무 프로젝트에서도 카카오 로그인 콜백과 값 전달 기능은 스킴으로, 프로모션 링크는 Universal Link로 나눠서 썼다.

**2) 한 화면에 여러 진입 경로를 열어둘 수 있다.** 같은 상품 화면이라도 `example://product?id=123`과 `https://example.com/product/123` 두 가지로 들어올 수 있게 하고, 앱 안에서는 두 URL을 **하나의 목적지로 해석**하는 라우팅 지점을 둔다.

```swift
enum DeepLink {
    case product(id: String)

    init?(url: URL) {
        if url.scheme == "example", url.host == "product",
           let id = URLComponents(url: url, resolvingAgainstBaseURL: false)?
                .queryItems?.first(where: { $0.name == "id" })?.value {
            self = .product(id: id)
        } else if url.scheme == "https", url.host == "example.com",
                  url.pathComponents.count >= 3, url.pathComponents[1] == "product" {
            self = .product(id: url.pathComponents[2])
        } else {
            return nil
        }
    }
}
```

`openURLContexts`와 `continue userActivity`는 들어오는 경로만 다를 뿐이므로, 둘 다 `DeepLink(url:)`로 해석해서 같은 화면 이동 코드로 보내면 중복 없이 관리된다.

**3) 서로의 약점을 보완한다.** 스킴은 앱이 없으면 열리지 않고 소유권 검증이 없다는 약점이 있고, Universal Link는 주소창 입력이나 같은 도메인 내 이동에서 앱이 열리지 않는 경우가 있다. 두 방식을 함께 열어두면 한쪽이 동작하지 않는 상황을 다른 쪽이 채워줄 수 있다.

## 정리

- Deep Link(커스텀 스킴)는 URL Types에 등록만 하면 iOS가 기기 안에서 연결해주지만, 소유권 검증이 없고 앱이 없으면 열리지 않는다.
- Universal Link는 앱의 `applinks:` 선언과 서버의 AASA 파일이 서로를 가리킬 때 동작하고, 앱이 없으면 웹으로 열린다. AASA는 내 서버에 올려두면 Apple CDN이 가져가서 캐시한다.
- AASA의 `paths`/`components`는 도메인의 어떤 경로를 앱에서 열지를 정하는 규칙이고, 위에서부터 처음 맞는 규칙이 적용된다.
- 두 링크 모두 앱이 꺼진 상태(콜드 스타트)와 로그인 같은 전제 조건을 직접 처리해야 한다.
- 함께 쓰면 용도별로 역할을 나누고, 여러 진입 경로를 하나의 라우팅으로 합칠 수 있다.
