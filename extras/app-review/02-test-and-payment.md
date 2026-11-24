# App Store 심사 전에 챙길 것 ① 기기 테스트와 결제 (2/7)

[1편](https://velog.io/@jxxnnee/App-Store-심사-전체-흐름부터-짚어보자-1-wtnyg9om)에서 심사의 전체 흐름을 봤다. 이번 편부터는 제출 전에 챙겨야 할 것들을 본다. 제출 전 준비는 분량이 많아서 두 편으로 나눴고, 이 글은 그 첫 번째다. 앱이 실제 기기에서 제대로 도는지, 베타 테스트는 어떻게 하는지, 결제는 어떻게 검증받는지를 다룬다.

## 1. 심사는 실기기에서 이뤄진다

App Review Help의 [Test your app on compatible devices](https://developer.apple.com/help/app-review/before-submitting-for-review/test-compatible-devices)는 첫 문장부터 분명하다. 심사는 사용자가 쓰는 방식 그대로, **실기기에 설치하고 실제 네트워크 환경에 연결해서** 이뤄진다. 시뮬레이터에서만 확인하고 제출하면 안 된다는 뜻이다.

### 1-1. 호환 모드를 알아야 한다

내가 만든 앱이 의도하지 않은 기기에서 돌아갈 수도 있다. Help 문서가 드는 예는 다음과 같다.

- iPhone용으로 만든 앱은 iPhone 기능에 의존하지 않으면 iPad에서 호환 모드로 실행될 수 있다.
- 호환되는 iPhone, iPad 앱은 Apple silicon Mac에서 수정 없이 실행될 수 있다.
- 호환되는 iPhone, iPad 앱은 Apple Vision Pro에서도 수정 없이 실행될 수 있다.

여기서 놓치기 쉬운 점이 하나 있다. Xcode의 **Supported Destinations**에서 기기를 빼거나 더해도, 이미 가능한 배포가 막히지는 않는다. iPhone용 앱이 iPhone에만 있는 기능에 의존하지 않는다면, Xcode에서 iPad를 지원 대상으로 넣었든 뺐든 App Store에서 iPad로 내려받을 수 있다.

그래서 Help 문서는 호환 모드에서의 동작을 테스트하지 않고 제출하지 말라고 하고, 지원 대상을 뺐다고 해서 그 기기로 배포되지 않는다고 가정하지 말라고 한다.

<video src="https://developer.apple.com/help/app-review/before-submitting-for-review/test-compatible-devices/videos/xcode-supported-destinations.mp4" controls muted loop playsinline width="100%"></video>

*Xcode에서 Supported Destinations를 설정하는 화면 (출처: [Apple App Review Help](https://developer.apple.com/help/app-review/before-submitting-for-review/test-compatible-devices))*

### 1-2. 테스트 방법

| 방법 | 용도 |
|---|---|
| Xcode 시뮬레이터 | 여러 기기 유형에서 기본 기능 확인 |
| TestFlight 외부 테스터 | 다양한 실기기에서 확인 |

시뮬레이터는 기본 기능 확인용이고, 실기기 확인은 TestFlight 외부 테스터를 활용하라는 것이 문서의 권장이다.

### 1-3. 적응형 UI를 만든다

- 화면 크기와 방향 변화에 대응하는 인터페이스를 만든다.
- 하드웨어에 따라 기능을 조절하고, 없는 경우에는 대안을 제공한다.
- 한 가지 기기나 입력 방식만 가정한 고정된 UI는 피한다.
- 선택적인 하드웨어가 없어도 앱이 크래시하거나 못 쓰게 되면 안 된다.

마지막 항목이 실무에서 특히 걸린다. 예를 들어 카메라나 위치 같은 장치가 없는 기기에서 해당 기능만 비활성화되는 것이 아니라 화면 전체가 멈추면 문제가 된다.

### 1-4. 배포를 제한해야 한다면

가능하면 여러 플랫폼에서 쓸 수 있게 하라는 것이 문서의 기본 입장이다. 다만 제한이 합리적인 경우도 있다고 하며 예시를 든다.

- iPhone 전용 하드웨어에 의존하는 앱은 iPad에서 기대한 대로 동작하지 않는다.
- 내비게이션이나 카메라 중심 앱은 visionOS에 잘 맞지 않는다.
- 키보드로 대체할 수 없는 터치 입력에 크게 의존하는 앱은 macOS에 잘 맞지 않는다.

하드웨어 의존성을 선언하는 방법은 Info.plist의 `UIRequiredDeviceCapabilities` 키다. Apple 문서에 따르면 이 키로 선언한 기능이 없는 기기에는 App Store에서 앱 설치 자체가 막힌다. 값은 배열이나 딕셔너리로 줄 수 있다.

```xml
<key>UIRequiredDeviceCapabilities</key>
<array>
    <string>location-services</string>
    <string>gps</string>
</array>
```

배열에 값이 있으면 그 기능을 요구한다는 뜻이다. 반대로 값을 생략하면 그 기능을 요구하지는 않지만 있어도 된다는 뜻이다. 위 예시에서 `gps`와 `location-services`를 같이 쓴 이유는 Apple 문서가 `gps`를 포함하면 `location-services`도 함께 포함하라고 안내하기 때문이다.

딕셔너리로 쓰면 기능마다 Boolean을 줄 수 있다. `true`는 그 기능이 필수라는 뜻이고, `false`는 그 기능이 기기에 **없어야 한다**는 뜻이다. 딕셔너리에 넣지 않은 기능은 요구도 금지도 하지 않는다.

```xml
<key>UIRequiredDeviceCapabilities</key>
<dict>
    <key>still-camera</key>
    <true/>
    <key>telephony</key>
    <false/>
</dict>
```

이 예시는 사진을 찍을 수 있는 카메라가 있어야 하고(`still-camera`), 전화 기능이 없는 기기여야 한다(`telephony`가 `false`)는 선언이다. `false`가 어떻게 동작하는지 보여주려는 예시일 뿐, 이렇게 선언하면 iPhone이 배포 대상에서 빠지므로 실제 앱에서는 신중해야 한다.

Apple 문서에 설명된 값 중 몇 가지를 옮기면 다음과 같다. 전체 목록은 문서에서 확인해야 한다.

| 값 | 의미 |
|---|---|
| `gps` | 위치 추적용 GPS(또는 AGPS) 하드웨어. 셀룰러나 Wi-Fi보다 정확한 위치 데이터가 필요할 때만 요구하라고 안내한다 |
| `still-camera` | 사진을 찍을 수 있는 카메라 |
| `video-camera` | 영상을 찍을 수 있는 카메라 |
| `telephony` | Phone 앱. `tel` 스킴 URL을 여는 앱이 요구할 수 있다 |
| `sms` | Messages 앱. `sms` 스킴 URL을 여는 앱이 요구할 수 있다 |
| `wifi` | Wi-Fi 네트워크 기능 |
| `peer-peer` | 블루투스 네트워크를 통한 피어 투 피어 연결 |

한 가지 예외가 있다. 앞서 "설치 자체가 막힌다"고 했지만, `peer-peer`는 문서에 따르면 App Store가 지원하지 않는 기기로의 배포를 막지는 않고, 대신 iOS가 앱 실행을 막는다. 값마다 동작이 조금씩 다를 수 있으니 쓰려는 값의 설명을 확인해 두는 편이 안전하다.

Help 문서는 이 키를 **진짜 하드웨어 의존성에만** 쓰라고 못 박는다. "이 기기에는 배포하고 싶지 않다"는 배포 선호를 표현하는 용도로 쓰면 안 된다.

### 1-5. 관련 가이드라인

| 가이드라인 | 내용 |
|---|---|
| 2.1(a) | 충돌이나 명백한 기술 문제가 드러나는 불완전한 앱은 거부된다 |
| 2.4.1 | 가능하면 iPhone 앱은 iPad에서도 실행되도록 한다 (권장) |
| 2.4.2 | 배터리를 빠르게 소모하거나 과도한 열을 내지 않는다 |
| 2.5.5 | IPv6 전용 네트워크에서 정상 동작해야 한다 |

2.5.5는 "실제 네트워크 환경"에서 심사한다는 Help 문서의 설명과 맞닿아 있다. 개발 환경의 네트워크에서 잘 된다고 해서 IPv6 전용 환경에서도 잘 된다고 보장할 수 없다.

## 2. TestFlight로 미리 검증한다

### 2-1. 규모

Help 문서에 나온 TestFlight 테스트 규모다.

- 내부 테스터는 Apple Developer Program 계정에서 최대 100명까지 추가할 수 있다.
- 외부 테스터는 이메일 초대나 공개 링크로 최대 10,000명까지 초대할 수 있다.

### 2-2. 심사를 거치는 범위

외부 테스터 그룹에 **처음 추가하는 빌드는 App Review로 전달**되어 가이드라인 준수 여부를 확인받는다. 같은 그룹에 이후 추가하는 빌드는 전체 심사가 필요 없을 수 있다.

"필요 없을 수 있다"가 어디까지인지는 가이드라인 2.2가 알려 준다. 베타 빌드에서 **중요한 업데이트**는 테스터에게 배포하기 전에 TestFlight 심사팀에 제출해야 한다.

### 2-3. 가이드라인 2.2

TestFlight와 관련해서 가이드라인 2.2가 정한 규칙은 세 가지다.

- 데모, 베타, 체험판 버전의 앱은 App Store에 등록할 수 없다. 그런 용도는 TestFlight를 써야 한다.
- TestFlight로 제출하는 앱도 공개 배포를 전제로 하며 심사 지침을 지켜야 한다.
- 테스터에게 크라우드 소스 펀딩에 대한 보상을 포함해 어떤 종류의 보상이나 대가도 제공해서는 안 된다.

TestFlight를 "심사 전 리허설"로만 생각하기 쉬운데, 이 규칙들을 보면 TestFlight 자체도 심사 대상이다.

## 3. 앱 내 구입은 샌드박스에서

### 3-1. 심사도 샌드박스에서 이뤄진다

[Configure In-App Purchases for review in the sandbox](https://developer.apple.com/help/app-review/before-submitting-for-review/configure-in-app-purchases)에 따르면 App Review는 개발자가 테스트할 때 쓰는 것과 같은 샌드박스 환경에서 앱 내 구입을 평가한다. 샌드박스에서는 실제 상품 데이터와 서버 간 거래를 쓰면서도 금전적 청구는 발생하지 않는다.

그리고 앱 내 구입은 **심사 중에 동작하기 위해 사전 승인이 필요하지 않다**고 명시돼 있다. 심사를 받기 전에 IAP부터 승인받아야 하나 걱정할 필요는 없다.

### 3-2. 준비 순서

Help 문서가 안내하는 순서다.

1. Account Holder가 App Store Connect의 비즈니스 섹션에서 **Paid Apps Agreement**를 수락하고, 계약이 유효한 상태인지 확인한다. 이 계약이 활성화되기 전에는 유료 앱 내 구입이 심사에서 동작하지 않는다.
2. 샌드박스 Apple 계정을 만들고, App Store Connect에서 상품 정보를 설정한다.
3. 앱의 UI에 앱 내 구입을 보여주기 전에 StoreKit API로 상품 정보를 가져와서, 화면에 표시되는 내용이 App Store Connect에 설정한 것과 일치하는지 확인한다.
4. 제출 전에 샌드박스에서 직접 테스트한다. 표시되지 않는 상품이 있으면 Apple의 [TN3186](https://developer.apple.com/documentation/technotes/tn3186-troubleshooting-in-app-purchases-availability-in-the-sandbox)을 따라 원인을 찾는다.
5. storefront나 기기 구성에 따라 앱 내 구입을 제한한다면, App Store Connect의 App Review Information 섹션 Notes 필드에 적는다.
6. 앱 내 구입이 **Ready for Review** 상태가 되면 App Store Connect에서 함께 제출한다.

### 3-3. 상품 정보를 가져와서 보여주기

3번 항목은 StoreKit 2의 `Product.products(for:)`로 구현할 수 있다. 상품 식별자를 넘기면 App Store에서 상품 정보를 가져온다.

```swift
import StoreKit

struct ProductItem {
    let id: String
    let name: String
    let price: String
}

func loadProductItems() async throws -> [ProductItem] {
    let identifiers = [
        "com.example.app.premium.monthly",
        "com.example.app.premium.yearly"
    ]
    let products = try await Product.products(for: identifiers)
    return products.map {
        ProductItem(id: $0.id, name: $0.displayName, price: $0.displayPrice)
    }
}
```

핵심은 이름과 가격을 코드에 하드코딩하지 않고 `displayName`과 `displayPrice`처럼 App Store에서 가져온 값을 화면에 보여주는 것이다. 그래야 App Store Connect에 설정한 내용과 UI에 보이는 내용이 어긋나지 않는다. 식별자는 예시 값이고, 실제로는 App Store Connect에 만든 상품 ID를 써야 한다.

### 3-4. 관련 가이드라인

가이드라인 2.1(b)는 앱 내 구입이 **완전하고 최신 상태이며, 심사자가 볼 수 있고 제대로 작동해야 한다**고 요구한다. 구성해 둔 앱 내 구입을 앱에서 찾을 수 없거나 심사할 수 없다면, 그 이유를 심사 메모에 적어야 한다. 3-2의 5번 항목(storefront나 기기별 제한을 Notes에 적는 것)이 바로 이 규칙에 대응한다.

## 정리

- 심사는 시뮬레이터가 아니라 실기기와 실제 네트워크 환경에서 이뤄진다.
- Xcode에서 지원 대상을 빼도 배포는 막히지 않으므로, iPad 호환 모드 등을 직접 테스트한다.
- 배포 제한은 `UIRequiredDeviceCapabilities`로 하되, 실제 하드웨어 의존성에만 쓴다.
- TestFlight는 외부 그룹의 첫 빌드가 심사를 거치고, 중요한 업데이트는 배포 전에 심사팀에 제출해야 한다. 보상을 대가로 제공하면 안 된다.
- 앱 내 구입은 샌드박스에서 심사되며, Paid Apps Agreement 활성화와 StoreKit 상품 조회, Ready for Review 상태 확인이 선행돼야 한다.

다음 편에서는 제출 전 준비의 두 번째로, 심사 정보(Notes)와 데모 계정, 증빙 서류, 1:1 상담을 다룬다.

## 참고

- [Test your app on compatible devices](https://developer.apple.com/help/app-review/before-submitting-for-review/test-compatible-devices)
- [Configure In-App Purchases for review in the sandbox](https://developer.apple.com/help/app-review/before-submitting-for-review/configure-in-app-purchases)
- [App Review Guidelines](https://developer.apple.com/kr/app-store/review/guidelines/)
- [UIRequiredDeviceCapabilities](https://developer.apple.com/documentation/bundleresources/information-property-list/uirequireddevicecapabilities)
- [Product.products(for:)](https://developer.apple.com/documentation/storekit/product/products(for:))
