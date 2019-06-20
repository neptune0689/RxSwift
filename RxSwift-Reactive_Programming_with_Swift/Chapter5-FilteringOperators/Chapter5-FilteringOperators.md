# Chapter 5: Filtering Operators


## 1. Ignoring operators
### 1-1. ignoreElements

(사진)

`ignoreElements`는 `.next` 이벤트를 무시하고, 종료 이벤트인 `.completed`와 `.error` 는 통과 시킨다.

```swift
let strikes = PublishSubject<String>()
let disposeBag = DisposeBag()
strikes
    .ignoreElements()
    .subscribe { _ in print("You're out!") }
    .disposed(by: disposeBag)

strikes.onNext("X") // no print
strikes.onNext("X") // no print
strikes.onNext("X") // no print
strikes.onCompleted() // print "You're out!"
```
- subscriber는 `.completed` 이벤트를 받을 것이고, "You're out!"을 출력 할 것이다.

</br></br>

### 1-2. elementAt

(사진)

observable에 의해 방출되는 n번째 요소를 처리할고 싶을 때가 있다. `elementAt`을 사용하면 원하는 n번째 요소를 통과시키고, 나머지 요소들은 무시할 수 있다. 

```swift
let strikes = PublishSubject<String>()
let disposeBag = DisposeBag()

strikes
	.elementAt(2)
  .subscribe(onNext: { _ in print("You're out!") })
	.disposed(by: disposeBag)

strikes.onNext("X") // no print
strikes.onNext("X") // no print
strikes.onNext("X") // print
```
- `.next` 이벤트를 구독한다. 하지만 (index가 2인) 세번째 `.next` 이벤트까지 무시한다

</br></br>

### 1-3. filter

(사진)

`ignoreElements`와 `elementAt`은 observable에 의해 방출되는 요소들을 필터링 했다.  필터링이 필요한 경우가 하나 이상일때, `filter`를 사용할 수 있다. `filter`는 조건문 클로저를 갖는다. 이 조건문은 요소들마다 적용되어, 조건문이 `true`일때만 요소를 통과 시킨다.

```swift
let disposeBag = DisposeBag()
Observable.of(1, 2, 3, 4, 5, 6)
	.filter { $0 % 2 == 0 } // 짝수 요소만 통과
  .subscribe(onNext: { print($0) }) // print "2", "4", "6"
  .disposed(by: disposeBag)
```
</br></br>


## 2. Skipping operators
### 2-1. skip

(사진)

`skip`은 첫번째부터 n번째까지 요소를 무시한다. (index가 아님!)

```swift
let disposeBag = DisposeBag()
Observable.of("A", "B", "C", "D", "E", "F")
	.skip(3) // 세번째 요소인 "C"까지 무시, 이후 "D"부터는 통과
  .subscribe(onNext: { print($0) }) // print "D", "E", "F"
  .disposed(by: disposeBag)
```
</br></br>

### 2-2. skipWhile

(사진)

subscription가 살아 있는 동안 계속 요소들을 필터링하는 `filter` 와 다르게, `skipWhile`은 조건문이 `false` 되기 전까지만 요소 값을 무시하는 연산자이다. `skipWhile`의 클로저는 ''어떤 조건까지 요소 값을 무시할지'에 대한 로직을 갖는다. 해당 로직 값이 false 된 이후에 요소들이 통과된다.

```swift
let disposeBag = DisposeBag()
Observable.of(2, 2, 3, 4, 4)
	.skipWhile { %0 % 2 == 0 } // 두번째 2까지 무시, 3 이후 부터는 다 통과
  .subscribe(onNext: { print($0) }) // print "3", "4", "4"
  .disposed(by: disposeBag)
```
- 보험 금액 청구 앱을 개발한다고 가정하면, 공제액이 충족될 때까지 보험금 지급을 막기위해 `skipWhile`을 사용 할 수 있다.

</br></br>

### 2-3. skipUntil

(사진)

지금 필터링은 정적인 조건을 가졌다. 다른 observable을 기반으로 동적이게 요소 값을 필터링하기를 원한다면? `skipUntil`은 trigger observable이 이벤트를 방출하기 전까지 source observable (구독하게 될 observable)으로 온 요소를 무시한다. 

```swift
let disposeBag = DisposeBag()

let subject = PublishSubject<String>()
let trigger = PublishSubject<String>()

subject
	.skipUntil(trigger) // trigger observable이 .next 이벤트를 방출하기 전까지 요소들 무시
	.subscribe(onNext: { print($0) })
  .disposed(by: disposeBag)

subject.onNext("A") // no print
subject.onNext("B") // no print
trigger.onNext("X")
subject.onNext("C") // print "C"
```
</br></br>

## 3. Taking operators
### 3-1. take

(사진)

taking은 skipping과 반대되는 개념이다. `task`는 n개까지 요소들 가져가고 싶을 때 사용한다.

```swift
let disposeBag = DisposeBag()
Observable.of(1, 2, 3, 4, 5, 6)
	.take(3) // 3번째 요소까지만 통과
  .subscribe(onNext: { print($0) }) // print "1", "2", "3"
  .disposed(by: disposeBag)
```
</br></br>

### 3-2. takeWhile

(사진)

takeWhile은 skipWhile처럼 동작한다. </br>

(만약에 observable에 의해 방출된 요소와 인덱스 값을 얻고 싶다면 `enumberated`를 사용하면 된다. Swift Standard Library의 메소드인 `enumberated`와 똑같이 동작한다.)

```swift
let disposeBag = DisposeBag()
Observable.of(2, 2, 4, 4, 6, 6)
	.enumerated()
  .takeWhile { index, integer in
  	integer % 2 == 0 && index < 3 // 짝수이고 index가 3보다 작은 요소들만 통과
	}
  .map { $0.element }
  .subscribe(onNext: { print($0) }) // print "2", 2", "4"
  .disposed(by: disposeBag)
```

- 조건값이 false 되기전까지 요소를 통과 시킨다.
- 위의 `map`은 Swift Standard Library의 `map` 같이 동작하지만, observable에서만 사용할 수 있다.

</br></br>

### 3-3. takeUntil

(사진)

skipUntil와 비슷하게, taskUtil도 trigger observable이 요소를 방출하기 전까지 source observable에서 방출된 요소들을 통과시킨다.

```swift
let disposeBag = DisposeBag()

let subject = PublishSubject<String>()
let trigger = PublishSubject<String>()

subject
.takeUntil(trigger)
.subscribe(onNext: { print($0) })
.disposed(by: disposeBag)

subject.onNext("1") // print "1"
subject.onNext("2") // print "2"
trigger.onNext("X")
subject.onNext("3") // no print
```

</br>

RxCocoa의 API에서 dispose bag를 추가하는 대신 `takeUntil`을 사용하여 구독을 해지를 할 수 있다.  `taskUntil`가 요소를 통과시키는 것을 멈추게할 trigger는 `self`의 할당 해지이다. 여기서 `self`는 대게 view controller 나 view model이 된다.

```swift
someObservable
	.takeUntil(self.rx.deallocated) // self가 할당 해지하기 전까지 요소를 통과  
  .subscribe(onNext: { print($0) })
```

</br></br>

## 4. Distinct operators

### 4-1. distinctUntilChanged

(사진)

연속으로 중복된 item을 통과시키지 않을 수 있는 연산자를 알아보자. `distinctUntilChanged`은 바로 옆에 있는 중복만 방지한다.

```swift
let disposeBag = DisposeBag()

Observable.of("A", "A", "B", "B", "A")
	.distinctUntilChanged() // 연속으로 중복되는 두번째 "A"와 네번째 "B"는 무시
  .subscribe(onNext: { print($0) }) // print "A", "B", "A"
  .disposed(by: disposeBag)
```

</br>

(사진)

`String` 타입은 `Equatable`을 따른다. 그래서 `String`의 요소들은 `Eqauatable`의 구현체를 이용하여 값이 같은지 비교한다. 이와 다르게 `distinctUntilChanged(_:)`은 직접 구현한 비교 로직을 파라미터로 넘겨 사용할 수 있다.

```swift
let disposeBag = DisposeBag()

// 1
let formatter = NumberFormatter()
formatter.numberStyle = .spellOut

// 2
Observable<NSNumber>
	.of(10, 110, 20, 200, 210, 310)
	// 3
	.distinctUntilChanged { a, b in
    // 4
		guard let aWords = formatter.string(from: a)?.components(separatedBy: " "),
			let bWords = formatter.string(from: b)?.components(separatedBy: " ") else { return }
    var containsMatch = false
    
    // 5
    for aWord in aWords {
      for bWord in bWords {
        if aWord == bWord {
          containsMatch == true
          break
        }
      }
    }
    return containsMatch
	}
	// 6
	.subscribe(onNext: { print($0) })
	.disposed(by: diposeBag)
                         
```

1. 번호를 spell out하는 `NumberFormatter`를 생성한다.
   - ex) 10 is "ten", 110 is "one hundred ten"
2. `NSNumber`의 observable을 생성한다. (이러면 formatter를 사용한 이후에 integer로 전환할 필요가 없다.)
3. 각각의 순차적인 요소 쌍을 받는 `distinctUntilChanged(_:)`을 사용한다.
   - ex) 순차적인 요소 쌍:  (10, 110), (110, 20), (20, 200)
4. 요소의 구성요소들을 " " 기준으로 분리한다.
   - ex) "one hundred ten" 를 ["one", "hundred", "ten"]
5. 이중 for-in 문을 돌면서 요소의 구성요소들 중에서 서로 값이 있는지 확인한다.
6. 비교 로직을 만족하는 요소들인 "10", "20", "200" 값을 출력

</br></br>

## Challenges

### Challenge 1: Create a phone number lookup

조건

1. 전화번호는 0으로 시작 불가 - `skipWhile` 사용
2. 전화번호는 일의 자리 숫자 (10보다 작음) - `filter`사용
3. 10번째의 숫자만 받도록 - `take`와 `toArray` 사용

```sw
input
	.skipWhile { $0 == 0 }
	.filter { $0 < 10 }
	.take(10)
	.toArray()
	.subscribe(onNext: {
		let phone = phoneNumber(from: $0)
		if let contact = contacts[phone] {
    	print("Dialing \(contact) (\(phone))...")
    } else {
      print("Contact not found")
    }
	})
	.disposed(by: disposeBag)
```

