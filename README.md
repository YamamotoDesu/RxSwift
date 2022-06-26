# Yamamoto Note
RxSwift 研究読本1
今城 善矩

## share(replay: 1)  
### Cold Obserableのままだと
```swift
        let o: Observable<ValidationResult> = input.password
            .map { password in
                print("map:")
                return validationService.validatePassword(password)
            }
<!--             .share(replay: 1) -->
        
        _ = o
            .subscribe(onNext: { _ in
                print("onNext: s1") // map:
                                    // onNext: s1
            })
        
        _ = o
            .subscribe(onNext: { _ in
                print("onNext: s2") // map: 
                　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　// onNext: s2
            })

```

### share(replay: 1)によるHot変換
```swift
        let o: Observable<ValidationResult> = input.password
            .map { password in
                print("map:")
                return validationService.validatePassword(password)
            }
            .share(replay: 1)
        
        _ = o
            .subscribe(onNext: { _ in
                print("onNext: s1") // map:
                                    // onNext: s1
            })
        
        _ = o
            .subscribe(onNext: { _ in
                print("onNext: s2")  // onNext: s2
            })
```

## Observableを切り替えるflatMapLates
```swift
validatedUsername = input.username
    .flatMapLatest { username in
        return validationService.validateUsername(username)
            .observeOn(MainScheduler.instance)
            .catchErrorJustReturn(.failed(message: "Error contacting server"))
    }
    .share(replay: 1)
```
validationService.validateUsernameメソッドにより、入力されたユーザ名に対して通信した結果をObservable<ValidationResult>とします。
その結果の処理にメインスレッドを指定し、もしバリデート時にエラーが発生したらValidationResult.failed(message: String)に差し替えています。
  
flatMapLatestはユーザ名入力のイベントからユーザ名の文字列を取り出し、新しいObservable<ValidationResult>のイベントを作成して変換しているわけです。
さらにflatMapLatestの特徴として、最新のイベントを伝達することを目的としているため、すでに実行済みの最新でないイベントを破棄する特徴があります。バリデート用のHTTPS通信を順次行っている今回の場合を具体例に挙げると、その連続した処理が実行されるたび、古いリクエストはレスポンスを受け取る前に終了していきます。
  
注意しておかなければいけない事として、flatMapLatest自体で作成したストリームの 結果を重複して待ち受けずに済む という意味で無駄が省けるのであって、通信を実行する事自体の無駄 はflatMapLatestで省けるわけではありません。例えばRESTFullなWeb APIによりリソースをサーバ上に作成するような通信を行う場合、flatMapLatestでdisposeされる前にサーバ上へリクエストが到達し、リソース作成が完了してしまうというような場合の無駄は省けません。
  
  
<p align="center">
<img src="assets/RxSwift_Logo.png" width="35%" alt="RxSwift Logo" />
<br />

<a href="https://actions-badge.atrox.dev/ReactiveX/RxSwift/goto" target="_blank"><img src="https://github.com/ReactiveX/RxSwift/workflows/RxSwift/badge.svg?branch=main" alt="Build Status" /></a>
<img src="https://img.shields.io/badge/platforms-iOS%20%7C%20macOS%20%7C%20tvOS%20%7C%20watchOS%20%7C%20Linux-333333.svg" alt="Supported Platforms: iOS, macOS, tvOS, watchOS & Linux" />
<br />
<a href="https://cocoapods.org/pods/RxSwift" alt="RxSwift on CocoaPods" title="RxSwift on CocoaPods"><img src="https://img.shields.io/cocoapods/v/RxSwift.svg" /></a>
<a href="https://github.com/Carthage/Carthage" alt="RxSwift on Carthage" title="RxSwift on Carthage"><img src="https://img.shields.io/badge/Carthage-compatible-4BC51D.svg?style=flat" /></a>
<a href="https://github.com/apple/swift-package-manager" alt="RxSwift on Swift Package Manager" title="RxSwift on Swift Package Manager"><img src="https://img.shields.io/badge/Swift%20Package%20Manager-compatible-brightgreen.svg" /></a>
</p>

Rx is a [generic abstraction of computation](https://youtu.be/looJcaeboBY) expressed through `Observable<Element>` interface, which lets you broadcast and subscribe to values and other events from an `Observable` stream.  

RxSwift is the Swift-specific implementation of the [Reactive Extensions](http://reactivex.io) standard.

<p align="center"><img src="assets/example.png" width="55%" alt="RxSwift Observable Example of a price constantly changing and updating the app's UI" /></p>

While this version aims to stay true to the original spirit and naming conventions of Rx, this projects also aims to provide a true Swift-first API for Rx APIs.

Cross platform documentation can be found on [ReactiveX.io](http://reactivex.io/).

Like other Rx implementation, RxSwift's intention is to enable easy composition of asynchronous operations and streams of data in the form of `Observable` objects and a suite of methods to transform and compose these pieces of asynchronous work.

KVO observation, async operations, UI Events and other streams of data are all unified under [abstraction of sequence](Documentation/GettingStarted.md#observables-aka-sequences). This is the reason why Rx is so simple, elegant and powerful.

## I came here because I want to ...

###### ... understand

* [why use rx?](Documentation/Why.md)
* [the basics, getting started with RxSwift](Documentation/GettingStarted.md)
* [traits](Documentation/Traits.md) - what are `Single`, `Completable`, `Maybe`, `Driver`, and `ControlProperty` ... and why do they exist?
* [testing](Documentation/UnitTests.md)
* [tips and common errors](Documentation/Tips.md)
* [debugging](Documentation/GettingStarted.md#debugging)
* [the math behind Rx](Documentation/MathBehindRx.md)
* [what are hot and cold observable sequences?](Documentation/HotAndColdObservables.md)

###### ... install

* Integrate RxSwift/RxCocoa with my app. [Installation Guide](#installation)

###### ... hack around

* with the example app. [Running Example App](Documentation/ExampleApp.md)
* with operators in playgrounds. [Playgrounds](Documentation/Playgrounds.md)

###### ... interact

* All of this is great, but it would be nice to talk with other people using RxSwift and exchange experiences. <br />[Join Slack Channel](http://slack.rxswift.org)
* Report a problem using the library. [Open an Issue With Bug Template](.github/ISSUE_TEMPLATE.md)
* Request a new feature. [Open an Issue With Feature Request Template](Documentation/NewFeatureRequestTemplate.md)
* Help out [Check out contribution guide](CONTRIBUTING.md)

###### ... compare

* [with Combine and ReactiveSwift](Documentation/ComparisonWithOtherLibraries.md).

###### ... understand the structure

RxSwift is as compositional as the asynchronous work it drives. The core unit is RxSwift itself, while other dependencies can be added for UI Work, testing, and more.

It comprises five separate components depending on each other in the following way:

```none
┌──────────────┐    ┌──────────────┐
│   RxCocoa    ├────▶   RxRelay    │
└───────┬──────┘    └──────┬───────┘
        │                  │        
┌───────▼──────────────────▼───────┐
│             RxSwift              │
└───────▲──────────────────▲───────┘
        │                  │        
┌───────┴──────┐    ┌──────┴───────┐
│    RxTest    │    │  RxBlocking  │
└──────────────┘    └──────────────┘
```

* **RxSwift**: The core of RxSwift, providing the Rx standard as (mostly) defined by [ReactiveX](https://reactivex.io). It has no other dependencies.
* **RxCocoa**: Provides Cocoa-specific capabilities for general iOS/macOS/watchOS & tvOS app development, such as Shared Sequences, Traits, and much more. It depends on both `RxSwift` and `RxRelay`.
* **RxRelay**: Provides `PublishRelay`, `BehaviorRelay` and `ReplayRelay`, three [simple wrappers around Subjects](https://github.com/ReactiveX/RxSwift/blob/main/Documentation/Subjects.md#relays). It depends on `RxSwift`. 
* **RxTest** and **RxBlocking**: Provides testing capabilities for Rx-based systems. It depends on `RxSwift`.

## Usage

<table>
  <tr>
    <th width="30%">Here's an example</th>
    <th width="30%">In Action</th>
  </tr>
  <tr>
    <td>Define search for GitHub repositories ...</td>
    <th rowspan="9"><img src="https://raw.githubusercontent.com/kzaher/rxswiftcontent/master/GithubSearch.gif"></th>
  </tr>
  <tr>
    <td><div class="highlight highlight-source-swift"><pre>
let searchResults = searchBar.rx.text.orEmpty
    .throttle(.milliseconds(300), scheduler: MainScheduler.instance)
    .distinctUntilChanged()
    .flatMapLatest { query -> Observable&lt;[Repository]&gt; in
        if query.isEmpty {
            return .just([])
        }
        return searchGitHub(query)
            .catchAndReturn([])
    }
    .observe(on: MainScheduler.instance)</pre></div></td>
  </tr>
  <tr>
    <td>... then bind the results to your tableview</td>
  </tr>
  <tr>
    <td width="30%"><div class="highlight highlight-source-swift"><pre>
searchResults
    .bind(to: tableView.rx.items(cellIdentifier: "Cell")) {
        (index, repository: Repository, cell) in
        cell.textLabel?.text = repository.name
        cell.detailTextLabel?.text = repository.url
    }
    .disposed(by: disposeBag)</pre></div></td>
  </tr>
</table>


## Requirements

* Xcode 12.x
* Swift 5.x

For Xcode 11 and below, [use RxSwift 5.x](https://github.com/ReactiveX/RxSwift/releases/tag/5.1.1).

## Installation

RxSwift doesn't contain any external dependencies.

These are currently the supported installation options:

### Manual

Open Rx.xcworkspace, choose `RxExample` and hit run. This method will build everything and run the sample app

### [CocoaPods](https://guides.cocoapods.org/using/using-cocoapods.html)

```ruby
# Podfile
use_frameworks!

target 'YOUR_TARGET_NAME' do
    pod 'RxSwift', '6.5.0'
    pod 'RxCocoa', '6.5.0'
end

# RxTest and RxBlocking make the most sense in the context of unit/integration tests
target 'YOUR_TESTING_TARGET' do
    pod 'RxBlocking', '6.5.0'
    pod 'RxTest', '6.5.0'
end
```

Replace `YOUR_TARGET_NAME` and then, in the `Podfile` directory, type:

```bash
$ pod install
```

### XCFrameworks

Each release starting with RxSwift 6 includes `*.xcframework` framework binaries.

Simply drag the needed framework binaries to your **Frameworks, Libraries, and Embedded Content** section under your target's **General** tab.

> **Note**: If you're using `RxCocoa`, be sure to also drag **RxCocoaRuntime.xcframework** before importing `RxCocoa`.

<img src="https://raw.githubusercontent.com/ReactiveX/RxSwift/main/assets/xcframeworks.png" alt="XCFrameworks instructions" width="65%">

### [Carthage](https://github.com/Carthage/Carthage)

Add this to `Cartfile`

```
github "ReactiveX/RxSwift" "6.5.0"
```

```bash
$ carthage update
```

#### Carthage as a Static Library

Carthage defaults to building RxSwift as a Dynamic Library. 

If you wish to build RxSwift as a Static Library using Carthage you may use the script below to manually modify the framework type before building with Carthage:

```bash
carthage update RxSwift --platform iOS --no-build
sed -i -e 's/MACH_O_TYPE = mh_dylib/MACH_O_TYPE = staticlib/g' Carthage/Checkouts/RxSwift/Rx.xcodeproj/project.pbxproj
carthage build RxSwift --platform iOS
```

### [Swift Package Manager](https://github.com/apple/swift-package-manager)

> **Note**: There is a critical cross-dependency bug affecting many projects including RxSwift in Swift Package Manager. We've [filed a bug (SR-12303)](https://bugs.swift.org/browse/SR-12303) in early 2020 but have no answer yet. Your mileage may vary. A partial workaround can be found [here](https://github.com/ReactiveX/RxSwift/issues/2127#issuecomment-717830502).

Create a `Package.swift` file.

```swift
// swift-tools-version:5.0

import PackageDescription

let package = Package(
  name: "RxTestProject",
  dependencies: [
    .package(url: "https://github.com/ReactiveX/RxSwift.git", .exact("6.5.0"))
  ],
  targets: [
    .target(name: "RxTestProject", dependencies: ["RxSwift", "RxCocoa"])
  ]
)
```

```bash
$ swift build
```

To build or test a module with RxTest dependency, set `TEST=1`.

```bash
$ TEST=1 swift test
```

### Manually using git submodules

* Add RxSwift as a submodule

```bash
$ git submodule add git@github.com:ReactiveX/RxSwift.git
```

* Drag `Rx.xcodeproj` into Project Navigator
* Go to `Project > Targets > Build Phases > Link Binary With Libraries`, click `+` and select `RxSwift`, `RxCocoa` and `RxRelay` targets

## References

* [http://reactivex.io/](http://reactivex.io/)
* [Reactive Extensions GitHub (GitHub)](https://github.com/Reactive-Extensions)
* [RxSwift RayWenderlich.com Book](https://store.raywenderlich.com/products/rxswift-reactive-programming-with-swift)
* [RxSwift: Debunking the myth of hard (YouTube)](https://www.youtube.com/watch?v=GdvLP0ZAhhc)
* [Boxue.io RxSwift Online Course](https://boxueio.com/series/rxswift-101) (Chinese 🇨🇳)
* [Expert to Expert: Brian Beckman and Erik Meijer - Inside the .NET Reactive Framework (Rx) (video)](https://youtu.be/looJcaeboBY)
* [Reactive Programming Overview (Jafar Husain from Netflix)](https://youtu.be/-8Y1-lE6NSA)
* [Subject/Observer is Dual to Iterator (paper)](http://csl.stanford.edu/~christos/pldi2010.fit/meijer.duality.pdf)
* [Rx standard sequence operators visualized (visualization tool)](http://rxmarbles.com/)
* [Haskell](https://www.haskell.org/)
