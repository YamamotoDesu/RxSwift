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
  
  
