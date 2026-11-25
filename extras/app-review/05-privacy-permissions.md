# App Store 심사 전에 챙길 것 ④ 권한과 개인정보 (5/7)

[4편](https://velog.io/@jxxnnee/iOS-App-Store-심사-전에-챙길-것-스크린샷-메타데이터-카피캣-x4r7lcdt)에서는 사용자에게 보이는 스토어 페이지를 다뤘다. 이번 편은 앱 안으로 들어가서, 사용자의 데이터와 권한을 어떻게 다뤄야 하는지에 대한 규칙을 본다. 가이드라인 5.1(개인정보 보호)이 중심이고, 코드와 가장 가까운 편이다. 계정 로그인과 삭제(5.1.1(v))는 다음 편에서 따로 다룬다.

## 1. 개인정보 규칙의 구조

가이드라인 5.1은 사용자의 개인정보 보호를 "그 무엇보다 중요"하게 다룬다. 개발자는 고객의 기대, 개인정보 보호 모범 사례, 관련 법률, Apple Developer Program 사용권 계약을 모두 지켜야 한다. 구체적인 규칙은 두 묶음이다.

| 항목 | 다루는 것 |
|---|---|
| 5.1.1 데이터 수집 및 보관 | 처리방침, 동의, 데이터 최소화, 접근, 로그인, 규제 분야 |
| 5.1.2 데이터 사용 및 공유 | 제3자 공유, 추적, 수집 목적 외 사용, 시스템 기능 활성화 강요 금지 |

## 2. 개인정보 처리방침 (5.1.1(i))

모든 앱은 처리방침 링크를 **두 곳**에 둬야 한다.

- App Store Connect의 메타데이터 영역
- 앱 내부의 쉽게 볼 수 있는 위치

처리방침 내용에도 세 가지 요건이 있다.

1. 앱과 서비스가 수집하는 정보, 수집 방법, 사용 목적을 정의한다.
2. 사용자 정보를 공유하는 **제3자**(분석 도구, 광고 네트워크, 타사 SDK, 모회사와 자회사 등)가 가이드라인이 요구하는 것과 같거나 동등한 수준으로 보호한다는 점을 확인하고, 처리방침에 명시한다.
3. 정보 보유와 삭제 정책을 설명하고, 사용자가 동의를 철회하고 삭제를 요구하는 방법을 밝힌다.

SDK를 쓴다면 그 SDK가 어떤 정보를 어디로 보내는지가 내 처리방침에 반영되어야 한다는 뜻이다. 1편에서 말한 "SDK까지 내 책임"이 여기서도 이어진다.

## 3. 동의, 최소화, 거부했을 때의 대안

### 3-1. 세 가지 규칙

| 항목 | 내용 |
|---|---|
| 5.1.1(ii) 허가 | 사용자 데이터와 사용 내역을 수집하는 앱은 수집 직후 익명화되는 데이터라도 **동의**를 받는다. 유료 기능이 정보 접근 허용에 의존하면 안 된다. 동의를 쉽게 철회할 수단이 있어야 한다 |
| 5.1.1(iii) 데이터 최소화 | 핵심 기능과 관련된 **최소한의 데이터**만 요청하고 사용한다. 사진이나 연락처 같은 보호 리소스는 전체 접근 권한을 요청하기보다 가능하면 독립적인 선택 방법이나 공유 시트를 쓴다 |
| 5.1.1(iv) 접근 | 사용자의 권한 설정을 반영한다. 불필요한 데이터 접근에 대한 동의를 부당하게 유도하거나 조작하거나 강요하지 않는다. 가능하면 동의를 거부한 사용자에게 대체 수단을 준다 |

원문이 드는 예시가 규칙을 잘 보여 준다.

- 소셜 네트워크에 사진을 올리는 앱은 사진 업로드를 허용하기 전에 마이크 권한까지 요구하면 안 된다.
- 위치 공유를 거부한 사용자에게는 **수동 주소 입력** 같은 대체 기능을 제공한다.

### 3-2. 위치 권한을 거부했을 때

가상의 서점 찾기 기능을 예로 들어 보자. 위치 권한 상태에 따라 "내 주변 서점"과 "주소 직접 입력" 중 어떤 화면을 쓸지 정하는 코드다.

```swift
import CoreLocation

enum StoreSearchMode {
    case nearby          // 위치 권한 허용
    case manualAddress   // 위치 권한 거부 또는 제한 -> 주소 직접 입력
    case undetermined    // 아직 묻지 않음
}

final class LocationPermission {
    private let manager = CLLocationManager()

    func currentMode() -> StoreSearchMode {
        switch manager.authorizationStatus {
        case .authorizedWhenInUse, .authorizedAlways:
            return .nearby
        case .denied, .restricted:
            return .manualAddress
        case .notDetermined:
            return .undetermined
        @unknown default:
            return .manualAddress
        }
    }

    /// 사용자가 위치 기반 기능을 직접 눌렀을 때만 호출한다.
    func requestIfNeeded() {
        guard manager.authorizationStatus == .notDetermined else { return }
        manager.requestWhenInUseAuthorization()
    }
}
```

`.denied`와 `.restricted`일 때 막다른 길로 두지 않고 `.manualAddress`로 넘기는 것이 5.1.1(iv)가 말하는 대체 수단이다. 권한은 앱을 켜자마자 묻지 않고, 위치 기반 기능을 쓰려는 순간에 요청한다.

### 3-3. 사진은 전체 접근 대신 선택기로

5.1.1(iii)은 사진 같은 리소스에 전체 접근 권한을 요구하지 말고 독립적인 선택 방법을 쓰라고 한다. 사진이라면 `PHPickerViewController`가 그 방법이다. Apple 문서는 이 컨트롤러가 기존 이미지 선택 컨트롤러의 대안이라고 설명하고, 장점으로 **사진 보관함 접근 없이도 선택한 에셋을 받을 수 있다**는 점을 꼽는다.

```swift
import PhotosUI

var configuration = PHPickerConfiguration()
configuration.filter = .images
configuration.selectionLimit = 1

let picker = PHPickerViewController(configuration: configuration)
picker.delegate = self
present(picker, animated: true)
```

이 코드는 개념을 보여주려고 줄인 것이고 컴파일해 보지는 않았다. 델리게이트에서 선택 결과를 받는 부분도 생략했다.

## 4. 목적 문구

### 4-1. 목적 문구란

카메라, 위치, 마이크, 건강 데이터 같은 보호된 리소스에 접근하려면 시스템이 사용자에게 권한을 묻는다. 이때 팝업에 함께 보여주는 설명이 **목적 문구(purpose string)**, 다른 말로 usage description이다. App Review Help의 [5.1.1(ii) 문서](https://developer.apple.com/help/app-review/guideline-reference/5-1-1-purpose-strings)가 이 규칙을 해설한다.

문구는 `Info.plist`에 권한별 키로 넣는다. Apple 문서에서 확인한 키는 다음과 같다.

| 리소스 | Info.plist 키 |
|---|---|
| 카메라 | `NSCameraUsageDescription` |
| 마이크 | `NSMicrophoneUsageDescription` |
| 위치(앱 사용 중) | `NSLocationWhenInUseUsageDescription` |
| 사진 보관함 | `NSPhotoLibraryUsageDescription` |
| 추적 | `NSUserTrackingUsageDescription` |

Xcode에서는 앱 타깃의 **Signing and Capabilities** 편집기에서 권한(capability)을 추가하고 문구를 입력하면 된다. Apple 문서에 따르면 이 방식은 해당 키를 빌드 설정(예: `INFOPLIST_KEY_NSLocationWhenInUseUsageDescription`)으로 구성해 준다.

알림 권한은 이 흐름과 다르다. 알림 권한을 요청하는 `UNUserNotificationCenter`의 `requestAuthorization(options:completionHandler:)`는 받는 인자가 허용할 상호작용 종류(알림창, 소리, 배지 등)뿐이고, 사용자에게 보여줄 설명 문구를 넘기는 자리가 없다. Apple 문서가 이 메서드에서 안내하는 것은 문구가 아니라 **시점**이다.

- 알림 예약이나 APNs 등록 전에 이 메서드를 호출해야 한다.
- 사용자가 앱이 왜 권한을 필요로 하는지 이해할 수 있는 **맥락에서** 요청한다. 예를 들어 할 일 앱이라면 사용자가 첫 할 일을 등록한 뒤에 요청한다. 앱 첫 실행 때 자동으로 요청하는 것보다 낫다고 설명한다.
- 처음 호출할 때만 시스템이 팝업을 띄우고, 이후 호출은 다시 묻지 않는다.
- 사용자가 판단할 정보가 부족하다면 **임시 인증(provisional authorization)**으로 알림을 시범 삼아 보내 볼 수 있다. 임시 알림은 소리나 배너 없이 알림 센터에만 조용히 쌓이고, 사용자가 유지하거나 끌지 선택한다.

### 4-2. 좋은 문구의 기준

Help 문서의 기준을 모으면 이렇다.

- 리소스 이름만 반복하거나 막연한 이점만 말하면 통과하지 못한다.
- 구체적인 기능이나 결과를 말하고, 보호 리소스마다 **따로** 쓴다.
- 구체적인 예시를 들어 한 문장으로 짧고 완전하게, 능동태로, 마침표로 끝낸다.
- 마케팅 문구나 기능 목록을 채우지 않는다.
- "데이터가 더 나은 경험을 위해 쓰일 것입니다"처럼 누가 접근하는지를 흐리는 수동태나 막연한 이점은 쓰지 않는다.

Help 문서의 예시는 영어다. 같은 기준을 가상의 책 기록 앱 "북로그"에 적용해 한국어로 써 보면 다음과 같다. 규칙을 이해하려고 내가 만든 예시다.

| 부족한 문구 | 명확한 문구 |
|---|---|
| 카메라를 사용합니다. | 책 뒷면의 바코드를 스캔해 도서 정보를 불러오기 위해 카메라를 사용합니다. |
| 더 나은 경험을 위해 위치 정보를 사용합니다. | 근처 서점을 지도에 보여주기 위해 현재 위치를 사용합니다. |

### 4-3. 현지화

앱이 여러 언어를 지원하면 목적 문구도 **현지화해야 한다.** Help 문서는 각 현지화로 앱을 설정해 테스트하고 올바른 문구가 팝업에 나오는지 확인하라고만 하고 방법은 설명하지 않는다. 방법은 Help가 관련 문서로 링크한 Apple 문서 [Requesting access to protected resources](https://developer.apple.com/documentation/UIKit/requesting-access-to-protected-resources)에 나온다.

1. Signing and Capabilities 편집기에서 기본 언어의 목적 문구를 입력한다.
2. **`InfoPlist.xcstrings`**라는 이름의 string catalog 파일을 만든다.
3. 앱을 빌드해서 string catalog에 앱의 사용 설명(usage description) 키들이 채워지게 한다.
4. string catalog에서 각 언어의 번역을 추가한다.

현지화한 문구도 4-5의 요건을 똑같이 만족해야 한다.

참고로 Apple의 보관(archive) 문서에는 예전 방식으로, 앱 이름 같은 `Info.plist` 값을 언어별 `InfoPlist.strings` 파일로 현지화한다는 설명이 있다(`CFBundleDisplayName`을 예로 든다). 다만 이 보관 문서는 목적 문구를 직접 다루지는 않으므로, 목적 문구를 현지화할 때는 위의 string catalog 방식을 기준으로 삼는 편이 안전하다.

### 4-4. SDK가 요청하는 권한

Help의 FAQ에 실무에서 놓치기 쉬운 항목이 있다. **서드파티 SDK가 요청하는 권한도 내 책임이다.**

- SDK는 위치 같은 보호 리소스에 접근하는 API를 앱 개발자가 모르는 채로 참조할 수 있다.
- 내 앱에 그 리소스를 쓸 타당한 이유가 있다면, 내 `Info.plist`에 구체적인 사용 사례를 설명하는 목적 문구를 정의한다.
- 이유가 없다면 **바이너리에서 그 API 참조를 제거한다.**
- 제출 전에 SDK가 접근하려는 리소스가 무엇인지 파악해 둔다.

위에서 언급한 Apple 문서는 한 가지를 더 짚는다. 내 앱이 그 API를 실제로 쓰지 않더라도 SDK가 참조하고 있으면 **App Review를 위해 목적 문구가 여전히 필요하다.** SDK가 어떤 보호 리소스를 쓰는지 SDK 개발자에게 문의하거나, 해당 API가 없는 버전을 요청할 수 있다. 외부 SDK와 라이브러리의 접근까지 포함해 보호 리소스 접근은 모두 개발자의 책임이다.

### 4-5. 문구가 갖춰야 할 기술적 요건

Apple 문서가 정리한 목적 문구의 기본 요건이다. 현지화한 문구에도 모두 적용된다.

- 비어 있거나 공백 문자로만 이루어져 있지 않다.
- **4,000바이트보다 짧다.** 보통은 한 문장이지만, 사용자가 올바르게 판단하도록 돕는 추가 정보를 넣을 수 있다.
- 해당 키가 요구하는 타입(보통 문자열)이다.
- 앱이 보호 리소스에 접근하는 이유를 정확하고 의미 있고 구체적으로 설명한다.

문구를 아예 넣지 않았을 때의 결과도 문서에 있다. 보호 리소스를 쓰는데 목적 문구가 없으면 **접근이 실패하고 앱이 크래시할 수도 있다.** Xcode가 이런 크래시를 감지해 문구를 추가하라는 이슈를 보여준다. 또 **App Review는 목적 문구 없이 보호 리소스에 접근하는 코드가 있는 앱을 리젝한다.** 해결 방법은 이유를 설명하는 목적 문구를 제공하거나 해당 리소스에 접근하는 코드를 제거하는 것이다.

## 5. 데이터 사용과 공유 (5.1.2)

### 5-1. 규칙 정리

| 항목 | 내용 |
|---|---|
| (i) 공유와 추적 | 사전 동의 없이 개인정보를 사용, 전송, 공유하지 않는다. **타사 AI를 포함한 제3자**와 공유할 때는 명확히 공개하고 공유 전에 명시적 허가를 받는다. 활동 추적은 App Tracking Transparency API로 명시적 허가를 받는다 |
| (ii) 목적 외 사용 | 특정 목적으로 수집한 정보는 관련 법률이 허용하지 않는 한 추가 동의 없이 다른 목적으로 쓰지 않는다 |
| (iii) 프로필 구축 | 수집한 데이터로 몰래 사용자 프로필을 만들거나, 익명 데이터에서 개인을 식별하려 하지 않는다 |
| (iv) 연락처와 설치 앱 | 연락처, 사진 등의 정보로 DB를 구축해 개인적으로 쓰거나 제3자에게 팔지 않는다. 분석, 광고, 마케팅 목적으로 기기에 설치된 다른 앱 정보를 수집하지 않는다 |
| (v) 연락처로 연락 | 개별적이고 명시적인 동의 없이 연락처나 사진으로 수집한 정보를 써서 타인에게 연락하지 않는다. "전체 선택"이나 기본 전체 선택을 두지 않는다 |
| (vi) 민감 API | HomeKit, HealthKit, ClassKit, ARKit, 카메라, 사진 API 등에서 얻은 데이터를 마케팅, 광고, 데이터 마이닝에 쓰지 않는다. 제3자가 쓰는 경우도 포함한다 |
| (vii) Apple Pay | Apple Pay로 수집한 정보는 상품과 서비스 제공 및 개선 목적으로만 제3자와 공유한다 |

### 5-2. 추적은 ATT로

(i)의 추적 규칙은 코드에서 `ATTrackingManager`로 이어진다. Apple 문서는 `requestTrackingAuthorization(completionHandler:)`가 사용자 또는 기기를 추적하는 데 쓸 수 있는 데이터에 접근하는 권한을 묻는 모달 UI를 띄운다고 설명한다. 이때 팝업에 보이는 설명은 `Info.plist`의 `NSUserTrackingUsageDescription`에 쓴다.

### 5-3. 시스템 기능을 강요하지 않는다

(i)의 마지막 부분이 지난 알림 글과 직접 이어진다. 사용자가 기능이나 콘텐츠를 쓰거나 금전 또는 기타 보상을 받기 위해 **푸시 알림, 위치 서비스, 추적 같은 시스템 기능을 켜도록 요구하면 안 된다.**

[로컬 푸시 알림 글](https://velog.io/@jxxnnee/Swift-로컬-푸시-알림Local-Push-Notification을-구현하는-방법은-무엇인가요)에서 알림 권한 요청을 다뤘는데, 그 요청을 앱 사용의 조건으로 만들면 이 규칙에 걸린다. 알림을 허용하지 않아도 앱의 기능은 써야 하고, 알림 허용을 대가로 보상을 주는 설계도 피해야 한다.

## 6. 그 밖의 5.1.1 항목

| 항목 | 내용 |
|---|---|
| (vi) | 사용자 몰래 암호나 기타 개인정보를 알아내는 데 앱을 악용하면 Developer Program에서 제명된다 |
| (vii) | `SafariViewController` 사용 방식을 정한다. 아래 6-1에서 다룬다 |
| (viii) | 사용자에게 직접 받지 않은 개인정보를 명시적 동의 없이 컴파일한 앱은 등록할 수 없다. 출처가 공용 데이터베이스여도 마찬가지다 |
| (ix) | 금융, 의료, 도박, 합법 대마초, 항공 여행, 암호화폐 거래처럼 규제가 엄격한 분야이거나 민감한 사용자 정보가 필요한 앱은 개별 개발자가 아니라 서비스를 제공하는 **법인**이 제출해야 한다 |
| (x) | 이름, 이메일 같은 기본 연락처 정보는 요청할 수 있다. 다만 사용자가 선택적으로 제공할 수 있어야 하고, 제공하는 조건으로 기능을 막으면 안 된다 |

(ix)는 3편의 증빙 서류와 같이 봐야 한다. 규제 분야 앱은 서류를 내는 것에 더해 제출하는 주체도 법인이어야 한다.

### 6-1. SafariViewController

`SFSafariViewController`는 앱을 떠나지 않고 웹 페이지를 보여 주는 화면이다. Apple 문서는 Reader, AutoFill, 사기 웹사이트 경고, 콘텐츠 차단 같은 Safari 기능을 지원하고, 이 화면 안에서의 **상호작용은 앱에 보이지 않는다**고 설명한다. 앱은 AutoFill 데이터, 방문 기록, 웹사이트 데이터에 접근할 수 없다. 화면을 꾸미거나 웹 콘텐츠와 상호작용해야 하면 `WKWebView`를 쓰라고 안내한다.

5.1.1(vii)은 이 화면의 사용 방식을 세 가지로 정한다.

- 정보를 사용자에게 **시각적으로 표시하는 용도**로 쓴다.
- 다른 항목이나 레이어로 **숨기거나 가리지 않는다.**
- 사용자가 인지하고 동의하지 않은 채 쓰지 않는다.

`SFSafariViewController`의 Apple 문서에도 같은 취지의 문장이 있다. 이 화면은 사용자에게 정보를 눈에 띄게 보여줘야 하고, 다른 뷰나 레이어 뒤에 숨기거나 가리면 안 되며, 사용자 모르게 추적하는 데 쓰면 안 된다. 앱에서 보이지 않는 화면이라는 특성 때문에, 사용자에게 안 보이게 만들어 몰래 쓰는 일을 막으려는 규칙으로 읽힌다. 다만 이 이유는 내 해석이고 문서가 직접 설명한 것은 아니다.

규칙에 맞는 사용은 사용자 동작에 맞춰 화면에 제대로 보이게 띄우는 것이다.

```swift
import SafariServices

func showTerms(from presenter: UIViewController) {
    guard let url = URL(string: "https://example.com/terms") else { return }
    presenter.present(SFSafariViewController(url: url), animated: true)
}
```

이 코드도 컴파일해 보지는 않았다.

## 정리

- 처리방침 링크는 App Store Connect와 앱 내부 두 곳에 두고, 쓰는 SDK가 공유하는 정보까지 반영한다.
- 데이터 수집은 동의를 받고, 핵심 기능에 필요한 최소한만 요청한다. 거부한 사용자에게는 대체 수단을 준다.
- 목적 문구는 구체적인 기능을 한 문장으로 쓰고, 권한마다 따로 쓰고, `InfoPlist.xcstrings`로 현지화하고, SDK가 요청하는 권한까지 확인한다. 문구가 없으면 크래시와 리젝의 원인이 된다.
- 제3자(타사 AI 포함)와 공유하려면 공유 전에 명시적 허가가 필요하고, 추적은 ATT로 받는다.
- 푸시 알림, 위치, 추적 같은 시스템 기능을 켜도록 요구해서는 안 된다.
- `SFSafariViewController`는 숨기거나 가리지 않고, 사용자가 인지한 상태에서 쓴다.

### 제출 전 점검표

- [ ] 처리방침 링크가 App Store Connect와 앱 안에 모두 있는가
- [ ] 처리방침에 수집 정보, 목적, 제3자 공유, 삭제 방법이 들어 있는가
- [ ] 권한마다 목적 문구가 따로 있고 구체적인가
- [ ] 지원하는 모든 언어에서 목적 문구가 올바르게 나오는가(`InfoPlist.xcstrings`)
- [ ] 쓰는 SDK가 접근하는 권한을 파악했고, 필요 없는 API 참조는 제거했는가
- [ ] 권한을 거부해도 대체 수단이나 기본 기능이 동작하는가
- [ ] 권한 요청이 앱 실행 직후가 아니라 해당 기능을 쓰는 순간에 일어나는가
- [ ] 제3자(AI 포함)와 개인정보를 공유하기 전에 명시적 허가를 받는가
- [ ] 알림, 위치, 추적을 켜지 않으면 앱을 못 쓰게 만들지 않았는가
- [ ] `SFSafariViewController`를 숨기거나 가리지 않았는가

다음 편에서는 계정을 다룬다. 앱 내 계정 삭제(5.1.1(v)), Sign in with Apple, 자동 갱신 구독을 가진 사용자의 계정 삭제 처리다.

## 참고

- [5.1.1(ii) - Write clear purpose strings](https://developer.apple.com/help/app-review/guideline-reference/5-1-1-purpose-strings)
- [App Review Guidelines](https://developer.apple.com/kr/app-store/review/guidelines/)
- [Requesting access to protected resources](https://developer.apple.com/documentation/UIKit/requesting-access-to-protected-resources)
- [SFSafariViewController](https://developer.apple.com/documentation/safariservices/sfsafariviewcontroller)
- [PHPickerViewController](https://developer.apple.com/documentation/photosui/phpickerviewcontroller)
