# App Store 심사 전에 챙길 것 ⑤ 계정과 로그인 (6/7)

[5편](https://velog.io/@jxxnnee/iOS-App-Store-심사-전에-챙길-것-권한과-개인정보)에서는 권한과 개인정보 규칙을 다뤘다. 이번 편은 그 가운데 계정에 관한 부분이다. 가이드라인 5.1.1(v)가 정한 계정 로그인과 **앱 안에서의 계정 삭제**, 그리고 Sign in with Apple을 쓰는 앱과 구독 사용자가 있을 때 계정 삭제를 어떻게 처리하는지를 본다.

## 1. 계정에 대한 규칙의 구조

가이드라인에서 계정을 다루는 곳은 5.1.1(v)와 4.8 두 군데다. 5.1.1(v)는 한 문단 안에 서로 다른 규칙이 들어 있어서, 규칙마다 어떤 앱에 적용되는지를 나눠서 보면 다음과 같다.

| 항목 | 규칙 | 적용되는 앱 |
|---|---|---|
| 5.1.1(v) 로그인 | 로그인 없이 쓸 수 있게 한다 | 주요 기능을 쓰는 데 반드시 계정이 필요하지는 않은 앱 |
| 5.1.1(v) 계정 삭제 | 앱 안에서 계정 삭제를 제공한다 | 계정 생성을 지원하는 앱 |
| 5.1.1(v) 소셜 자격 증명 | 앱 안에서 자격 증명을 철회하거나 접근을 끄는 수단을 둔다 | 소셜 네트워크 로그인을 쓰는 앱 |
| 4.8 로그인 서비스 | 다른 로그인 서비스를 동등한 옵션으로 함께 제공한다 | 타사 또는 소셜 로그인으로 기본 계정을 설정하거나 인증하는 앱 |

이 규칙들은 이어진 조건이 아니라 각각 따로 적용된다. 이메일 가입이 있는 앱은 계정 삭제 규칙만 해당되고, 소셜 로그인으로 가입하는 앱은 계정 삭제와 소셜 자격 증명 규칙이 둘 다 해당된다. 계정을 만들 수 있게 해 둔 앱이라면 로그인을 강제하지 않더라도 계정 삭제는 제공해야 한다.

5.1.1(v)는 App Review Help에 [해설 문서](https://developer.apple.com/help/app-review/guideline-reference/5-1-1-account-deletion)가 따로 있다. 이 글의 대부분은 그 문서를 기준으로 한다.

## 2. 로그인 없이 쓸 수 있어야 한다

5.1.1(v)의 앞부분은 계정 삭제가 아니라 **로그인을 요구해도 되는 범위**를 정한다.

| 요구 | 내용 |
|---|---|
| 로그인 없이 사용 | 주요 기능을 쓰는 데 반드시 계정이 필요하지 않은 앱은 로그인하지 않아도 쓸 수 있게 한다 |
| 개인정보 입력 | 핵심 기능과 직접 관련되거나 법적으로 필요한 경우가 아니면, 앱을 작동하는 데 사용자 개인정보 입력을 요구하지 않는다 |
| 소셜 로그인 | 핵심 기능이 특정 소셜 네트워크와 관련 없으면, 로그인이나 비슷한 메커니즘 없이 바로 기능을 쓸 수 있게 한다. 기본 프로필 정보를 가져오거나 소셜 네트워크에 공유하거나 친구를 초대하는 것은 핵심 기능으로 보지 않는다 |
| 소셜 자격 증명 | 앱 안에서 소셜 네트워크 자격 증명을 철회하거나 데이터 접근을 끄는 수단을 둔다. 자격 증명이나 토큰은 기기 밖 소셜 네트워크에 저장하지 않고, 앱이 소셜 네트워크에 직접 연결하는 용도로만 쓴다 |

앱을 열자마자 로그인 화면부터 띄우는 구조를 만들기 쉬운데, 이 규칙은 계정이 정말 필요한 기능이 아니라면 먼저 보여 주지 말라는 뜻이다. 둘러보기와 계정이 필요한 기능을 나누는 설계가 안전하다.

## 3. 앱 안에서 계정을 삭제할 수 있어야 한다

### 3-1. 규칙

가이드라인 5.1.1(v)는 앱이 계정 생성을 지원하면 **앱 안에서 계정 삭제도 제공**해야 한다고 한다. Help 문서는 계정 삭제를 이렇게 정의한다. 개발자의 기록에서 계정을 지우고, 법적으로 보관할 의무가 없는 관련 데이터도 함께 삭제하는 것이다. 사용자가 공유한 개인 데이터를 더 잘 통제하게 하려는 기능이라고 설명한다.

### 3-2. 삭제 흐름이 지켜야 할 것

Help 문서는 삭제를 시작하고 확인하는 과정이 **간단하고 투명해야** 한다고 하며, 네 가지를 든다.

- 삭제 옵션을 앱에서 쉽게 찾을 수 있게 한다. 보통 계정 설정에 둔다.
- 계정 기록 전체와 관련 개인정보 삭제를 제공한다. 추가 옵션을 둘 수는 있지만, **일시 비활성화만 제공하는 것은 부족하다.**
- 사용자가 웹사이트에서 삭제를 마쳐야 한다면, 삭제를 완료할 수 있는 **페이지로 바로 가는 링크**를 앱에 넣는다. tvOS에서는 고정 URL이나 QR 코드를 보여줄 수 있다.
- 사용자에게 진행 상황을 알린다. 삭제에 시간이 걸리면 알리고, 앱 내 구입이 있으면 청구와 취소가 어떻게 처리되는지 이해하게 하고, 법적으로 일부 정보를 보관해야 하면 무엇을 보관하고 무엇을 지우는지 알린다.

### 3-3. FAQ에서 확인한 판단 기준

| 질문 | 답 |
|---|---|
| 고객센터 흐름으로 삭제를 안내해도 되나 | 규제가 엄격한 업종(5.1.1(ix))은 추가 고객센터 흐름을 쓸 수 있다. 그렇지 않은 앱은 **전화, 이메일, 채팅 지원 등을 요구하면 안 된다** |
| 재인증이나 확인 단계를 넣어도 되나 | 된다. 본인 확인이나 삭제 의사 확인(예: 계정에 연결된 이메일이나 번호로 받은 코드 입력)은 적절하다. 다만 삭제를 불필요하게 어렵게 만들면 심사를 통과하지 못한다 |
| 웹 브라우저로 가입을 유도하는 앱도 앱 안 삭제가 필요한가 | 필요하다 |
| 자동으로 생성되는 계정(게스트 계정)도 삭제 옵션이 필요한가 | 필요하다. 자동 생성 계정과 그 데이터도 삭제할 수 있어야 한다 |
| 수동으로 삭제하는데, 즉시 자동이어야 하나 | 아니다. 합리적인 시간이 걸려도 된다. 걸리는 시간을 알리고 완료되면 확인을 준다. 소요 시간은 지역 법을 따라야 한다 |
| 사용자 생성 콘텐츠도 지워야 하나 | 지워야 한다. 다른 사람과 공유한 사진, 영상, 게시글, 리뷰도 포함한다. 법적으로 보관해야 하는 데이터가 있으면 알린다 |
| CCPA나 GDPR 때문에 일부 지역에만 삭제를 제공해도 되나 | 안 된다. 위치와 상관없이 **모든 사용자**가 삭제할 수 있어야 한다. 법 준수용으로 만든 흐름을 모든 사용자에게 열어도 된다 |

## 4. Sign in with Apple을 쓰는 앱

### 4-1. 토큰을 revoke해야 한다

Help 문서는 Sign in with Apple로 계정 생성과 인증을 하는 앱이 계정을 삭제할 때 **Sign in with Apple REST API로 사용자 토큰을 revoke**해야 한다고 한다. 변경 사항은 server-to-server 알림 엔드포인트로 처리하라고 하며, 대한민국에 기반을 둔 개발자는 새 Services ID를 등록할 때 이 서버 간 알림 엔드포인트를 반드시 제공해야 한다고 덧붙인다.

revoke API를 Apple 문서에서 확인했다.

- `POST https://appleid.apple.com/auth/revoke`, 입력은 form-data다.
- `client_id`: 앱의 App ID 또는 Services ID. 인증 요청 때 쓴 값과 같아야 하고 Team ID를 포함하면 안 된다.
- `client_secret`: Sign in with Apple 개인 키로 서명한 JWT.
- `token`: revoke할 refresh token 또는 access token.
- `token_type_hint`: `refresh_token` 또는 `access_token`.
- 성공하면 본문 없이 200을 돌려준다. 토큰이 이미 유효하지 않았던 경우에도 200이다.
- 사용자의 인증을 revoke하려면 유효한 refresh token이나 access token이 필요하다. 없으면 authorization code를 검증할 때 토큰을 발급받을 수 있다.

서버에서 보내는 요청의 모양은 다음과 같다. 값은 모두 자리표시자다. 실제 서버에 보내 본 것은 아니고, 문서의 파라미터를 옮긴 것이다.

```bash
curl -X POST "https://appleid.apple.com/auth/revoke" \
  --data-urlencode "client_id=<앱의 App ID 또는 Services ID>" \
  --data-urlencode "client_secret=<Sign in with Apple 개인 키로 서명한 JWT>" \
  --data-urlencode "token=<사용자의 refresh token>" \
  --data-urlencode "token_type_hint=refresh_token"
```

이 호출은 `client_secret`을 만들 개인 키가 필요하므로 **앱이 아니라 서버에서** 해야 한다. 앱은 삭제 요청만 서버에 보내고, 서버가 토큰을 revoke하고 계정 데이터를 지우는 구조가 자연스럽다. 이 구조는 내 해석이고 문서가 지정한 것은 아니다.

## 5. 구독이 있는 사용자의 계정 삭제

자동 갱신 구독이 있는 사용자가 계정을 지우면, 계정은 없어졌는데 구독 청구는 계속될 수 있다. Help 문서의 해법은 이렇다.

- 청구가 **Apple을 통해 계속된다**고 사용자에게 알리고, 계속하기 전에 구독을 취소하라고 안내한다.
- 구독 상태는 App Store Server Notifications로 실시간 확인하거나, Subscription Status API로 알아낼 수 있다.
- 구독 관리와 환불 요청은 StoreKit이 시트를 제공한다.
- 삭제를 **구독 만료일에 맞춰 예약**하는 선택지를 줄 수도 있다. 단 즉시 삭제하는 옵션이 같이 있어야 한다.

### 5-1. 구독 관리와 환불 요청

Apple 문서에서 확인한 API다.

| 기능 | API | 설명 |
|---|---|---|
| 구독 관리 | `AppStore.showManageSubscriptions(in:)` (UIKit) | 사용자가 App Store에서 보는 구독 관리 시트와 같은 시트를 앱 안에서 띄운다. 현재 구독을 보고 업그레이드, 다운그레이드, 취소할 수 있다 |
| 구독 관리 (SwiftUI) | `manageSubscriptionsSheet(isPresented:)` | 같은 시트를 불리언 바인딩으로 띄운다. 사용자가 닫거나 표시에 실패하면 시스템이 값을 `false`로 되돌린다 |
| 환불 요청 | `Transaction.beginRefundRequest(in:)` | 구매 내역과 사유 코드 목록이 있는 환불 시트를 띄운다. 계정 설정이나 도움말 메뉴에서 호출하라고 안내한다 |

Help 문서는 구독 관리 링크(`https://apps.apple.com/account/subscriptions/`)와 환불 요청 링크(`https://support.apple.com/118223`)도 제공한다. 두 API는 iOS 15와 iPadOS 15 이상에서 쓸 수 있다.

UIKit에서 구독 관리 시트를 띄우는 모양이다. 개념을 보여주려고 줄인 예시다.

```swift
import StoreKit

func showManageSubscriptions(in scene: UIWindowScene) async {
    do {
        try await AppStore.showManageSubscriptions(in: scene)
    } catch {
        // 시트를 띄우지 못한 경우: 구독 관리 링크를 대신 안내한다
    }
}
```

## 6. 삭제 화면 흐름 예시

규칙을 코드로 옮겨 보면 어떤 모양이 될지 가상의 앱으로 만들어 봤다. **Help 문서의 요건(삭제 옵션, 구독 안내, 소요 시간 안내)을 반영해 본 설계 예시일 뿐이고, 이렇게 구현하면 심사를 통과한다는 보장은 없다.** Apple이 정한 구현 형식은 없다.

### 6-1. 상태와 분기 로직

화면은 `idle`에서 시작해 확인, 구독 안내, 요청, 완료 순서로 흐른다. 활성 구독이 있으면 요청 전에 안내 단계를 한 번 거친다. UI 프레임워크와 분리한 로직이다.

```swift
import Foundation
import Combine

struct DeletionResult: Equatable {
    let completedImmediately: Bool
    let expectedDays: Int?
}

protocol AccountService {
    func hasActiveSubscription() async throws -> Bool
    func requestDeletion() async throws -> DeletionResult
}

enum DeletionStep: Equatable {
    case idle
    case confirming
    case subscriptionNotice
    case requesting
    case completed(DeletionResult)
    case failed
}

@MainActor
final class AccountDeletionModel: ObservableObject {
    @Published private(set) var step: DeletionStep = .idle
    private let service: AccountService

    init(service: AccountService) {
        self.service = service
    }

    func start() {
        step = .confirming
    }

    /// 사용자가 "계정 삭제"를 확인했을 때
    func confirm() async {
        do {
            if try await service.hasActiveSubscription() {
                step = .subscriptionNotice
            } else {
                await requestDeletion()
            }
        } catch {
            step = .failed
        }
    }

    /// 구독 안내를 읽고 계속 진행할 때
    func continueAfterNotice() async {
        await requestDeletion()
    }

    private func requestDeletion() async {
        step = .requesting
        do {
            step = .completed(try await service.requestDeletion())
        } catch {
            step = .failed
        }
    }
}
```

### 6-2. SwiftUI 화면

위 모델을 화면에 연결한 모양이다. 구독 안내 단계에서 구독 관리 시트를 띄울 수 있게 했다.

```swift
import SwiftUI
import StoreKit

struct AccountDeletionView: View {
    @StateObject var model: AccountDeletionModel
    @State private var showManageSubscriptions = false

    var body: some View {
        VStack(spacing: 16) {
            switch model.step {
            case .idle:
                Button("계정 삭제") { model.start() }
            case .confirming:
                Text("계정과 관련된 모든 데이터가 삭제됩니다.")
                Button("삭제 진행", role: .destructive) { Task { await model.confirm() } }
            case .subscriptionNotice:
                Text("구독은 Apple을 통해 계속 청구됩니다. 먼저 구독을 취소해 주세요.")
                Button("구독 관리") { showManageSubscriptions = true }
                Button("계속 삭제", role: .destructive) { Task { await model.continueAfterNotice() } }
            case .requesting:
                ProgressView()
            case .completed(let result):
                Text(result.completedImmediately
                     ? "계정이 삭제되었습니다."
                     : "삭제 요청이 접수되었습니다. 약 \(result.expectedDays ?? 0)일 안에 처리됩니다.")
            case .failed:
                Text("삭제하지 못했습니다. 잠시 후 다시 시도해 주세요.")
            }
        }
        .manageSubscriptionsSheet(isPresented: $showManageSubscriptions)
    }
}
```



| 요건 | 예시에서의 반영 |
|---|---|
| 삭제 확인 단계 | `confirming` 단계 |
| 구독 청구가 계속된다는 안내와 취소 유도 | `subscriptionNotice` 단계와 구독 관리 버튼 |
| 소요 시간 안내 | 완료 단계에서 즉시 삭제와 접수 후 처리를 구분 |
| 삭제 옵션을 쉽게 찾기 | 이 예시에는 없다. 화면을 계정 설정에 어떻게 배치할지는 앱마다 다르다 |

즉시 삭제 옵션과 구독 만료일에 맞춘 예약 삭제를 둘 다 두는 것도 Help가 말하는 방식인데, 이 예시는 즉시 삭제 쪽만 구현했다.

## 7. 로그인 서비스 (4.8)

가이드라인 4.8은 **타사나 소셜 로그인으로 기본 계정을 설정하거나 인증하는 앱**이 다른 로그인 서비스를 동등한 옵션으로 함께 제공하라고 한다. 그 서비스는 세 가지를 만족해야 한다.

- 수집 범위가 사용자의 이름과 이메일 주소로 제한된다.
- 사용자가 계정을 설정할 때 이메일 주소를 비공개로 유지할 수 있다.
- 동의 없이 광고 목적으로 앱과의 상호작용을 수집하지 않는다.

예외는 다섯 가지다.

- 회사 자체 계정 설정과 로그인 시스템만 쓰는 앱
- 외부 앱 마켓플레이스 전용 로그인을 쓰는 앱
- 기존 교육 또는 기업 계정으로 로그인해야 하는 교육, 기업, 비즈니스 앱
- 정부 또는 업계 지원 주민 확인 시스템이나 전자 ID로 사용자를 인증하는 앱
- 특정 타사 서비스의 클라이언트로, 콘텐츠에 접근하려면 그 서비스 계정으로 직접 로그인해야 하는 앱

가이드라인 4.8의 한국어 원문에는 Sign in with Apple이라는 이름이 나오지 않는다. 위 세 조건이 설명으로만 적혀 있어서, 어떤 서비스가 이 조건을 만족하는지는 이 글에서 단정하지 않는다.

## 정리

- 계정이 꼭 필요한 기능이 없다면 로그인 없이 쓸 수 있게 한다.
- 앱이 계정 생성을 지원하면 **앱 안에서** 계정 삭제를 시작할 수 있어야 하고, 비활성화만으로는 부족하다.
- 전화나 이메일, 채팅 지원을 거쳐야 삭제되는 구조는 규제 업종이 아니면 안 된다.
- 게스트 계정, 사용자 생성 콘텐츠, 모든 지역의 사용자가 삭제 대상이다.
- Sign in with Apple을 쓰면 계정을 지울 때 서버에서 토큰을 revoke한다.
- 구독이 있는 사용자에게는 청구가 계속된다는 점을 알리고 구독 관리와 환불 요청 수단을 준다.

### 제출 전 점검표

- [ ] 계정이 꼭 필요하지 않은 기능은 로그인 없이 쓸 수 있는가
- [ ] 계정 설정 같은 쉽게 찾을 곳에 삭제 옵션이 있는가
- [ ] 비활성화가 아니라 계정과 관련 개인정보 전체를 삭제하는가
- [ ] 전화, 이메일, 채팅 지원을 거치지 않고 앱 안(또는 삭제 페이지로 가는 직접 링크)에서 끝나는가
- [ ] 게스트 계정과 사용자 생성 콘텐츠도 삭제되는가
- [ ] 삭제에 시간이 걸리면 소요 시간과 완료 확인을 알리는가
- [ ] 법적으로 보관하는 정보가 있으면 무엇인지 알리는가
- [ ] 모든 지역의 사용자에게 같은 삭제 기능을 제공하는가
- [ ] Sign in with Apple을 쓰면 삭제 시 토큰을 revoke하는가
- [ ] 구독이 있는 사용자에게 청구가 계속된다고 알리고 취소 수단을 제공하는가

마지막 7편에서는 제출 후를 다룬다. 심사 상태 확인, 빠른 심사 요청, 심사팀에 문의, 이의 신청, 버그 수정 제출이다.

## 참고

- [5.1.1(v) - Offering account deletion in your app](https://developer.apple.com/help/app-review/guideline-reference/5-1-1-account-deletion)
- [App Review Guidelines](https://developer.apple.com/kr/app-store/review/guidelines/)
- [Token revocation](https://developer.apple.com/documentation/signinwithapplerestapi/revoke-tokens)
- [showManageSubscriptions(in:)](https://developer.apple.com/documentation/storekit/appstore/showmanagesubscriptions(in:))
- [manageSubscriptionsSheet(isPresented:)](https://developer.apple.com/documentation/swiftui/view/managesubscriptionssheet(ispresented:))
