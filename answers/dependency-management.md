# 의존성 관리 도구(CocoaPods, Carthage, Swift Package Manager)의 종류와 차이점

## 공통 목적

외부 라이브러리(third-party 코드)를 프로젝트에 가져오고, 버전을 관리하는 도구.

## 세 가지 도구의 핵심 차이

**CocoaPods**
`Podfile`에 의존성을 선언하면, 별도의 `.xcworkspace`를 만들어서 의존성을 프로젝트에 통합한다. 설정을 많이 자동화해주는 대신, 프로젝트 구조(워크스페이스 생성, 빌드 설정 변경)에 깊이 개입한다.

**Carthage**
의존성을 미리 빌드된 프레임워크 형태로만 받아오고, 프로젝트에 실제로 통합하는 작업은 개발자가 직접 한다. 프로젝트 파일 자체를 건드리지 않아 더 가벼운(탈중앙화된) 방식이다.

**Swift Package Manager (SPM)**
Apple 공식 도구로 Xcode에 완전히 통합돼 있어 별도 설치가 필요 없다. `Package.swift`로 선언만 하면 Xcode가 빌드·링크까지 알아서 처리한다.

## 실제 사용 경험 — CocoaPods vs SPM (Firebase SDK)

GoodPharm 프로젝트에서 Firebase SDK를 도입할 때 CocoaPods를 사용했다. 이유는:

1. **Firebase 공식 가이드가 오랫동안 CocoaPods 중심으로 작성되어 있었다.** SPM 지원은 이후에 추가된 것이라, 당시 문서·커뮤니티 자료가 CocoaPods 쪽이 더 풍부했다.
2. **Crashlytics 사용을 위해 Run Script Build Phase에 심볼(dSYM) 업로드 스크립트(`${PODS_ROOT}/FirebaseCrashlytics/run`)를 추가하고, Input Files에 dSYM 경로 등을 지정해야 했다.** 이 설정이 CocoaPods 기준으로 잘 정리되어 있어서 그 경로를 따라가는 게 수월했다. (참고로 이 심볼 업로드 스크립트 자체는 CocoaPods만의 요구사항은 아니고, SPM으로 Firebase를 쓸 때도 동일하게 필요하다 — 다만 스크립트 경로가 달라진다.)

반면 **SPM은 별도 워크스페이스나 도구 설치 없이 Xcode에 URL만 등록하면 돼서 훨씬 가볍게** 쓸 수 있었다. Apple 공식 지원 + Xcode 완전 통합이라는 장점 덕분에, 요즘은 특별한 이유(구버전 프로젝트 유지보수, SPM 미지원 라이브러리)가 없다면 SPM이 사실상 표준으로 자리잡았다.
