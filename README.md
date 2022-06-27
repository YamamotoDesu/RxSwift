# Yamamoto Note
RxSwift 研究読本1
今城 善矩

![image](https://user-images.githubusercontent.com/47273077/175798093-910ebdf9-ace2-4202-b7d7-9522b5f37021.png)

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

------ 
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
        
        
------
        
## Observableを合成するcombineLatest

<table>
  <tr>
    <th width="30%">combineLatest</th>
    <th width="30%">In Action</th>
  </tr>
  <tr>
    <td>combineLatestの例</td>
    <th rowspan="9"><img src="https://user-images.githubusercontent.com/47273077/175798414-610749c6-bdcd-410b-8d5a-f1a92c3fd284.gif"></th>
  </tr>
  <tr>
    <td><div class="highlight highlight-source-swift"><pre>
  validatedPasswordRepeated = Observable.combineLatest(
      input.password,
      input.repeatedPassword,
      resultSelector: validationService.validateRepeatedPassword
  )
  .share(replay: 1)
    </pre>
     </div></td>
  </tr>
  <tr>
    <td width="30%"><div class="highlight highlight-source-swift">
ombineLatestは引数により2つの入力を受け取り、引数resultSelectorにより合成する処理方法を受け取ることができるオペレータです。resultSelectorに渡しているvalidationService.validateRepeatedPasswordは、DefaultImplementations.swiftにて、passwordとrepeatedPassowrdが同じかを比較しValidationResultにして返します。入力されたパスワードに関する2つのストリームから文字列を取り出して比較し、それぞれの文字列が同じならValidationResult.ok(message:)を持つストリームを返す、というわけです。
    </div></td>
  </tr>
</table>
        
### combineLatestでこんなこともできる
```swift
 let usernameAndPassword
  = Observable.combineLatest(input.username, input.password) {
     (username: $0, password: $1)
}
```
クロージャの処理内容としては、引数に取る2つのストリームの文字列をタプルとして作成し、ユーザ名の文字列とパスワードの文字列を1つのusernameAndPassword: Observable<(username: String, password: String)>ストリームとしているだけです。このように、複数ストリームを合成したストリームにすることは、データをまとめて利便性を上げることもできます。

-----
## 最新の値を取得するwithLatestFrom
合成されたパスワードの文字列とパスワード確認文字列のストリームはリスト4.2における「ViewModelの実装2_5」にて、サインアップボタンのイベントを伝達するloginTaps: Observable<Void>ストリームによって、withLatestFromオペレータでその最新のデータを取得されます。
すなわち、ボタンを押されたタイミングをきっかけに、ユーザ名とパスワードの文字列を使って当初の目的であったサインアップ処理を行うイベントを作成したいわけです。
```swift
signedIn = input.loginTaps.withLatestFrom(usernameAndPassword)
    .flatMapLatest { pair in
        return API.signup(pair.username, password: pair.password)
            .observeOn(MainScheduler.instance)
            .catchErrorJustReturn(false)
            .trackActivity(signingIn)
    }
    .flatMapLatest { loggedIn -> Observable<Bool> in
        let message = loggedIn ? "Mock: Signed in to GitHub." : "Mock: failed"
        return wireframe.promptFor(message, cancelAction: "OK", actions: [])
            // propagate original value
            .map { _ in
                loggedIn
            }
    }
    .share(replay: 1)
```

------
## Driverの特性について
* エラーを伝達しないことを前提としている（onErrorが通知されない）
* イベントはメインスレッドで観測されることを約束する
* ストリームはHotに変換される（副作用が共有される）

■ DriverとSignalの違い  
Driverは購読直後に過去のイベントを取得できる
UITextFieldなどの現在の文字列をイベントとして取得

Signalは購読後にイベントが発生してからでないと取得できない

### IBOutletなプロパティのDriverへの変換
```swift
        // usernameOutlet.rx.text.orEmpty.asObservable()
        username: usernameOutlet.rx.text.orEmpty.asDriver()
```

### IBOutletなプロパティのSignalへの変換
```swift
        // loginTaps: signupOutlet.rx.tap.asObservable()
        loginTaps: signupOutlet.rx.tap.asSignal()
```
Signalの特性としては、Driverの特性にさらにreplayされないという特性を持っています。
replayされないというのは、過去のイベントを一切保持せず、その値も保持していません。

具体的にはDriverは購読直後にもし最新のイベントがあれば、そのイベントを流そうとしますが、Signalはそのような動作をしません。そのためUIButtonのタップイベントに向いているのです。replayしないという挙動があることを型で表現することは、コードの意図を人に伝えるという点においてとても意味のあることでしょう。

### Driverのmap変換
```swift

        <!--        
            validatedPassword = input.password
            .map { password in
                return validationService.validatePassword(password)
            }
            .share(replay: 1) 
        -->
            validatedPassword = input.password
            .map { password in
                return validationService.validatePassword(password)
            }
 ```
このコードでのDriverとObservableの違いは、share(replay: 1)メソッドを呼び出さずに済んでいる点だけで、これはDriverとして変換された時点ですでにHot Observable済みだからです。

### Driverを切り替えるflatMapLatest
```swift
        
           <!--
            validatedUsername = input.username
            .flatMapLatest { username in
                return validationService.validateUsername(username)
                    .observe(on:MainScheduler.instance)
                    .catchAndReturn(.failed(message: "Error contacting server"))
            }
            .share(replay: 1) 
            -->
        
            validatedUsername = input.username
            .flatMapLatest { username in
                return validationService.validateUsername(username)
                    .asDriver(onErrorJustReturn: .failed(message: "Error contacting server"))
            }

```
------
## combineLatestでの合成
```swift
let password = PublishSubject<String>()
let repeatedPassword = PublishSubject<String>()

_ = Observable.combineLatest(password, repeatedPassword) { "\($0), \($1)"}
      .subscribe(onNext: { print("onNext: ", $0) })

password.onNext("a")
password.onNext("ab")

repeatedPassword.onNext("A")
repeatedPassword.onNext("AB")
repeatedPassword.onNext("ABC")
 
```

出力結果
```
onNext: ab, A
onNext: ab, AB
onNext: ab, ABC
```
注目すべき点は、repeatedPassword: PublishSubject<String>の値が変わっても、常にpassword: PublishSubject<String>の最新の値abのみを使っている点です。
        

-----

## Zip
```swift
let intSubject = PublishSubject<Int>()
let stringSubject = PublishSubject<String>()

_ = Observable.zip(intSubject, stringSubject) {
        "\($0) \($1)"
    }
    .subscribe(onNext: { print($0) })

intSubject.onNext(1)
intSubject.onNext(2)

stringSubject.onNext("A")
stringSubject.onNext("B”)
stringSubject.onNext("C")
stringSubject.onNext("D")

intSubject.onNext(3)
intSubject.onNext(4)
```

出力結果
```
1 A
2 B
3 C
4 D
```
注目すべきことは、入力として実行する1,2,3,4の順序とA,B,C,Dのそれぞれの順序が、出力時にも揃っているということです。
2のあとにAのデータが発生させていても、それは別々の順序のため関係ありません。ここで読者のみなさんが気になるのは、「zipは最新の値でないことで微妙に使いづらそうなんだけど何に使うの？」ということでしょう。
もちろん、どちらかの値が最新ではない可能性があることを気にかける必要があります。そのためzipは最新結果の状態をチェックするのには向きません。
実際は「イベントが揃ったら動作する仕組みとして割り切って使う」というのがよくある使われ方でしょう。

-------


![image](https://user-images.githubusercontent.com/47273077/175941213-d6aee886-5750-4f2d-8204-5c0eee05de70.png)


## エラー通知時に自動で再度購読を行うretryオペレータ
```swift
let sequenceThatErrors = Observable<String>.create { observer in
    observer.onNext("A")

    observer.onError(TestError.test)
    print("Error encountered")

    observer.onNext("B")
    observer.onCompleted()

    return Disposables.create()
 }

_ = sequenceThatErrors
    .retry(3) // 1.
    .subscribe(onNext: {
         print("onNext: \($0)")
     }, onError: {
         print("onError: \($0)")
     }, onCompleted: {
         print("onCompleted:")
     }, onDisposed: {
         print("onDisposed:")
     })
```

ログ出力結果
```
onNext: A
Error encountered
onNext: A
Error encountered
onNext: A
Error encountered
onError: test
onDisposed:
```

## エラーを任意のデータによって(catchError)
```swift
let sequenceThatErrors = Observable<String>.create { observer in
    observer.onNext("A")
    observer.onError(TestError.test)
    observer.onNext("B")
    observer.onCompleted()

    return Disposables.create()
}

_ = sequenceThatErrors
     .catchError { error in
         if error is TestError {
            return Observable.just("Z")
        } else {
            return Observable.empty()
        }
    }
    .subscribe(onNext: {
        print("onNext: \($0)")
     }, onError: {
         print("onError: \($0)")
     }, onCompleted: {
         print("onCompleted:")
     }, onDisposed: {
         print("onDisposed:")
})
```

出力結果
```
onNext: A
onNext: Z
onCompleted:
onDisposed:
```
        
## catchErrorJustReturn
```swift
 let sequenceThatErrors = Observable<String>.create { observer in
    observer.onNext("A")
    observer.onError(TestError.test)
    observer.onNext("B")
    observer.onCompleted()
 
    return Disposables.create()
 }

 _ = sequenceThatErrors
    .catchErrorJustReturn("Z") 
    .subscribe(onNext: {
        print("onNext: \($0)")
    }, onError: {
        print("onError: \($0)")
    }, onCompleted: {
        print("onCompleted:")
   }, onDisposed: {
       print("onDisposed:")
   })
```
        
出力結果
```
onNext: A
onNext: Z
onCompleted:
onDisposed:
```

## catchErrorJustReturnとcatchErrorの違い
errorに応じて自由にストリームを返したい場合や要素を何も返したくない場合はcatchErrorを使い、任意の要素に置き換えたい場合はcatchErrorJustReturnを使うというのがユースケースでしょう。
しかしcatchError/catchErrorJustReturnどちらを使っても、結果はエラー通知の置き換え後に正常終了します。購読側で細かくハンドリングできるというものではありません。多くの場合、プログラマ側があらかじめ予期するエラーに対して、ストリームの異常終了を防ぐために使われます。
