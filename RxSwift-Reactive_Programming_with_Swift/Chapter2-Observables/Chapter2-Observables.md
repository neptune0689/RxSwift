# Chapter 2 - Observables
## 요약
1) Observable
	1) 이벤트 방출자이고, `.next(Element)`, `error(Swift.Error)`, `completed` 이벤트를 sequency하게 방출
	2) Observable이 `error(Swift.Error)` 또는 `completed` 이벤트에 의해 종료되면, 더이상 이벤트를 발생하지 않음
2) Subscribe
	1) Observable이 방출한 이벤트를 처리
	2) Subscribe를 dispose 하지 않으면, 메모리 릭 발생
3) Traits: 작은 범위의 Observable
	1) Single: `.success(value)` 또는 `.error` 이벤트 방출
    2) Completable: `.completed` 또는 `.error` 이벤트 방출
    3) Maybe: `.success(value)`, `.completed`, `.error` 이벤트 방출

## What is an observable?
1) '관찰 할 수 있는'
2) sequence

## Life cycle of an observable
```swift

public enum Event<Element> {
	// Next element is produced
    case next(Element)
    
    // Sequence terminated with a error
    case error(Swift.Error)
    
    // Sequence completed successfully
    case completed
}

```

1) next event
	- 이벤트가 정상적으로 방출
	- 다음 이벤트 있음
2) error event
	- 오류 발생하면 방출
	- 다음 이벤트 없음
3) completed event
	- 정상적으로 종료 되었을때 방출
	- 다음 이벤트 없음
+ 관측이 종료되면 이벤트가 발생하지 않음

## Creating observables
1) `Observable<Int>.just(1)`
2) `Observable.of(1, 2, 3)`
	- `Int` Observable 타입
3) `Observable.of([1, 2, 3])`
	- `[Int]` Observable 타입
	- Array를 하나의 이벤트로 받음
4) `Observable.from([1, 2, 3])`
	- Array의 요소들을 각각의 이벤트로 받음
	- Observable의 타입은 Int
	- from은 오직 Array만 가능

## Subscribing to observables
1) Observable은 Subscriber를 소유할 때까지 이벤트를 방출하지 않음 (= Cold Observable)
2) Observable은 sequence
3) Observable를 `subscribe`(구독)하면, Iterator가 `next()`를 호출함
4) Observable은 `.next`, `.error`, `.completed` 이벤트를 발생함
5) subscribe은 `Disposable`을 리턴
	```swift
    func subscribe(_ on @escaping(Event<Int>) -> Void) -> Dispoable
    ```
6) 아래 코드는 `.next` 이벤트를 핸들링하고, 다른 이벤트들은 무시하는 코드
	```swift
    	observable.subscribe(onNext: element in { print(element) })
	```
7) 빈 Operator는 빈 Observable sequence를 생성; `.completed` 이벤트만 방출하고 바로 종료됨

## Disposing and termination
1) Subscription을 취소하려면, `dispose()`를 호출
2) Dispose Bag은 deallocate 될때, 각각의 `dispose()`를 호출함
3) Subscription의 Dipose Bag을 추가하는 것을 잊으면, memory leak이 발생
4) 예시
	1) Create Operator는 `subscribe`라는 하나의 매개변수를 갖음
	```swift
    static func create(_ subscribe: @escaping(AnyObserve<String>) -> disposable) -> Observable<String> 
    ```
    2) 아래 코드는 `.next` 이벤트의 element인 '1', 'Completed'와 'Disposed'가 print됨. 두번째 `.next` 이벤트의 element인 '?'는 print 되지 않는데, `.onNext("?")`가 호출되기 전에 `.onCompleted()`가 호출되어 이미 Observable이 종료되었기 때문
    ```swift
    Observable<String>.create { observer in
		observer.onNext("1")
		observer.onCompleted() // *** 이후 결과 자세히 보기
        observer.onNext("?")
        return Disposables.create()
	}
    .subcribe(
    	onNext: { print($0) },
        onError: { print($0) },
        onCompleted { print("Completed") },
        onDisposed: { print("Disposed") }
    )
    .disposed(by: disposeBage)
    
    // 1
    // Completed
    // Disposed
    ```
    3) `.error` 이벤트에 의해 Observable이 종료되어도, 위와 결과가 같음
    ```swift
    Observable<String>.create { observer in
		observer.onNext("1")
		observer.onError(MyError.anError) // *** 이후 결과 자세히 보기
		observer.onNext("?")
		return Disposables.create()
	}
	.subcribe(
    	onNext: { print($0) },
        onError: { print($0) },
        onCompleted: { print("Completed") },
        onDisposed: { print("Disposed") }
	)
	.disposed(by: disposeBag)

	// 1
	// anError
	// Disposed
    ```
    4) 만약에 `.completed` 또는 `.error` 이벤트를 방출하지 않거나 subscription에 DisposeBag를 추가하지 않는 다면, memory leak 발생. Observable은 영원히 끝나지 않고, disposable은 영원히 처리 되지 않음
    ```swift
    Observable<String>.create { observer in
    	observer.onNext("1")
    	// observer.onCompleted()
    	// observer.onError(MyError.anError)
    	observer.onNext("?")
    	return Disposables.create()
    }
    .subcribe(
    	onNext: { print($0) },
    	onError: { print($0) },
    	onCompleted: { print("Completed") },
    	onDisposed: { print("Disposed") }
    )
    // .disposed(by: disposeBag)
    
    // 1
    // ?
    ```
    
## Creating observable fatories
- `Observable<Int>.create`는 하나의 `Int` 타입 Observable를 생성해주는데, `Observable.deferred`는 subscribe가 호출될 때마다 Observable를 여러번 생성
    ```swift
    var flip = false
    let factory: Observable<Int> = Observable.deferred {
    	flip = !flip
        if filp {
        	return Observable.of(1, 2, 3)
        } else {
        	return Observable.of(4, 5, 6)
        }
 
    }
    
    for _ in 0...3 {
    	factory.subscribe(onNext: {
        	print($0, terminator: "")
        })
        .disposed(by: disposeBag)
        print()
    }
    // 123
    // 456
    // 123
    // 456
    ```
    
## Using Traits
1) Traits는 Observable보다 작은 범위로 사용되고, 코드의 가독성을 높혀줌
2) Traits 종류
	1) Single
		1) `.success(value)` 또는 `.error` 이벤트를 방출
		2) `.success(value)` = `.next` + `.completed`
		3) '데이터 다운', '디스크 로드'와 같이, 성공(+ 결과 값) 또는 실패를 갖는 one-time process에 유용
	2) Completable
    	1) `.completed` 또는 `error` 이벤트를 방출
        2) '파일 쓰기'와 같이, 연산이 성공적으로 완료되었는지만 확인할때 유용
	3) Maybe  
		1) Single + Completeable의 결합체; `.success(value)`, `.completed`, `.error`를 방출
        2) 성공 또는 실패 결과와, 선택적으로 성공했을때의 결과 값을 확인해야할 때 유용