# Chapter 3 - Subjects
## 요약
1. Subject = Observerable + Observer
2. `PublishSubject`<br/>
	<img src="https://user-images.githubusercontent.com/43217043/58745827-5e603b00-8491-11e9-851c-ea3c6c560b18.png"><br/>
3. `BehaviorSubject`<br/>
	<img src="https://user-images.githubusercontent.com/43217043/58745818-48eb1100-8491-11e9-9a9a-1fb63aa55df5.png"><br/>
4. `ReplaySubject`<br/>
	<img src="https://user-images.githubusercontent.com/43217043/58745802-ec87f180-8490-11e9-8728-ac2d036b15ed.png"><br/>

## What are subjects?
1. Subject = Observable + Observer
2. Subject type
	1) `PublishSubject`
    	- 빈 상태로 시작한 후, subscriber에게 새로운 element들을 방출함
    2)  `BehaviorSubject`
    	- 초기값을 가진 상태로 시작한 후, subscriber에게 초기값 또는 최신 element를 방출함
    3) `ReplaySubject`
    	- 버퍼 사이즈로 초기화하고, subscriber에게 버퍼 사이즈 만큼 element들을 유지 및 방출함
    4) <s>(Deprecated) `Varible`</s>
    	- BehaviorSubject를 랩핑
        - 현재 값을 상태로 유지하고, subscriber에게 초기값 또는 최신 element을 방출함

## Working with publish subjects
1. 예시
	1. 다음 코드의 결과를 알아보자
	```swfit
    	let subject = PublishSubject<String>()
        subject.onNext("Is anyone listenling?") // print 안됨, 왜냐하면 observer가 없기 때문
        
        // 2
        let subscriptionOne = subject
        .subscribe(onNext: { string in
        	print(string)
        })
        subject.on(.next("1")) //print "1"
        subject.onNext("2") // print "2"
        
        // 3
        let subscriptionTwo = subject
        .subscribe { event in
        	print("2)", event.element ?? event)
        }
        subject.onNext("3") // print "3", "2) 3"
        subscriptionOne.dispose()
        subject.onNext("4") // print "2) 4"
    ```
	2. `PublishSubject`가 `.completed` 나 `.error` 이벤트를 받는다면, subscriber에게 stop 이벤트를 방출하고 더이상 `.next` 이벤트를 방출하지 않음
	```swift
    	subject.onCompleted() // print "2) completed"
        subject.onNext("5") // not print
        subscriptionTwo.dispose()
        subscriptionTwo.dispose()
        let disposeBag = DisposeBag()
        
        // *** 이후 결과 자세히 보기
        subject
        	.subscribe {  // print "3) completed", 한번 종료하면 새 subscription이 생겨도 바로 .completed 이벤트를 방출하고 바로 종료
            	print("3)", $0.element ?? $0)
            }
            .disposed(by: disposeBag)
        subject.onNext("?") // not print
    ```
2. 어디에 적용할까
	- 시간에 민감한 데이터를 모델링할 때

## Working with behavior subjects
1. 예시
	1. 다음 코드의 결과를 알아보자
	```swift
    	func print<T: CustomStringConverible>(label: String, event: Event<T>) {
        	print(label, event.element ?? event.error ?? event)
        }
        let subject = BehaviorSubject(value: "Intial value")
        let disposeBag = DisposeBag()
        subject.onNext("X")
        subject
        	.subscribe { // print "1) X", not print "1) Intial value"
            	print(label: "1)", event: $0)
            }
            .disposed(by: disposeBag)
        subject.onError(MyError.anError) // print "1) anError"
        subject
        	.subscribe { // print "2) anError"
            	print(label: "2)", event: $0)
            }
            .disposed(by: disposeBag)
    ```
	2. `BehaviorSubject`는 초기값 없이 생성 할 수 없음. 만약 초기값 없이 생성하고 싶으면, `PublishSubject`를 대신 사용
2. 어디에 적용할까
	- 뷰를 가장 최신의 데이터로 미리 채울때 용이

## Working with replay subjects
1. 예시
	1. 다음 코드의 결과를 알아보자
	```swift
    	let subject = ReplaySubject<String>.create(bufferSize: 2)
        let disposeBag = DisposeBag()
        subject.onNext("1")
        subject.onNext("2")
        subject.onNext("3")
        
        subject
        	.subscribe { // print "1) 2", "1) 3"
            	print(label: "1)", event: $0)
            }
            .disposed(by: disposeBag)
        subject
        	.subscribe { // print "2) 2", "2) 3"
            }
        	.disposed(by: disposeBag)
    ```
	2. `ReplaySubject`가 에러와 함께 종료되면, subscriber에게 `.error` 이벤트를 방출. 하지만 버퍼는 아직 살아 있기 때문에, 새로운 subscriber에게 버퍼에 있는 값을 방출.
	```swift
    	subject.onNext("4") // print "1) 4", "2) 4"
        subject.onError(MyError.anError) // print "1) anError", "2) anError"
        
        // *** 이후 결과 자세히 보기
        subject
        	.subscribe { // print "3) 3", "3) 4", "3) anError"
            	print(label: "3)", event: $0)
            }
            .disposed(by: disposeBag)
    ```
	3. 그러므로 `.error` 이벤트 이후에 반드시 dispose를 하여 재방출을 막아야 함.
	```swift
    	subject.onNext("4") // print "1) 4", "2) 4"
        subject.onError(MyError.anError) // print "1) anError", "2) Error"
        subject.dispose() // dispose를 해줘야 위에 같은 문제가 발생하지 않음
        
        subject
        	.subscribe { // "3) anError"
            	print(label: "3)", event: $0)
            }
            .disposed(by: disposeBag)
    ```
2. 어디에 적용할까
	- 최근 검색어 보기를 n개 보여주는 것처럼, 최신의 값 외에 이전의 값 n개를 보여줄 때 용이

## <s> (Deprecated) Working with varialbes </s>
1. 예시
	1. `BehaviorSubject`를 랩핑하고 현재 값을 상태로 보유하는데, subject나 observable과 다르게 `onNext(_:)`을 사용할 수 없어서 새로운 방법으로 element를 추가 해야함.
	```swift
    	let variable = Variable("Initial value")
        let disposeBag = DisposeBag()
        variable.value = "New initial value" // element를 추가 하는 방법
        variable.asObservable() // Variable를 subscribe하려면 asObservable()를 호출해줘야함
        	.subscribe { // print "1) New inital value"
            	print(label: "1)", event: $0)
            }
            .disposed(by: disposeBag)
        variable.value = "1" // print "1) 1"
        variable.asObservable()
        	.subscribe { // print "2) 1"
            	print(label: "2)", event: $0)
            }
            .disposed(by: diposeBag)
        variable.value = "2" // print "1) 2", "2) 2"
    ```
    2. `Variable`의 특징은, 에러가 발생하지 않는 것을 보장함. 그러므로 `.error`를 추가할 수 없음
	3. `Varialbe`는 할당 해제 되었을때 자동으로 종료 되기 때문에, `.completed`를 추가할 수 없음
<br/>
<br/>
<br/>








