# Chapter 7: Transforming Operators

- 구독자가 사용할 observable에서 온 데이터를 변형시키기위해 transforming 연산자를 사용할 것이다. `map(_:)`과 `flatMap(_:)` 처럼 RxSwift의 transforming 연산자와 Swift standard library 연산자는 비슷하게 생겼다. 이번 챕터에서는 모든 것을 변형시킬 것이다.

</br></br>

## 1. Transforming elements

### 1-1. toArray

<img width="531" src="https://user-images.githubusercontent.com/43217043/60964133-83c44c80-a34d-11e9-92bc-cc59d22f224f.png">

- Observable은 각각 요소들을 방출하지만, Observable을 table 또는 collection view와 바인딩 하는 것처럼 조합하기 원한다. Observable의 각각의 요소들을 배열로 전환하는 편리한 방법은 `toArray`를 사용하는 것이다. 위의 그림 처럼, `toArray`는 observable sequence의 요소들을 배열 형식으로 바꾸고, 배열 값을 가진 `.next` 이벤트를 subscriber에게 방출한다.
- playground에 새로운 예제를 추가해라.

```swift
let disposeBag = DisposeBag()
Observable.of("A", "B", "C")
	.toArray()
	.subscribe(onNext: { print($0) })
	.disposed(by: disposeBag)
// print ["A", "B", "C"]
```

</br></br>

### 1-2. map

<img width="576" src="https://user-images.githubusercontent.com/43217043/60963982-2b8d4a80-a34d-11e9-9992-d9cee1e9d269.png">

- RxSwift의 map 연산자는 Swift의 map와 같이 동작 한다. 단, RxSwift의 map은 observable을 연산한다. 위의 사진에서 보면 map은 각 요소들을 2씩 곱하는 클로저를 가진다.
- playground에 새로운 예시를 추가해라.

```swift
let disposeBag = DisposeBag()

let formatter = NumberFormatter()
formatter.numberStyle = .spellOut

Observable<NSNumber>.of(123, 4, 56)
	.map { formatter.string(from: $0) ?? "" }
	.subscribe(onNext: { print($0) })
	.disposed(by: disposeBag)
```

</br>

### 1-3. enumerated

- Chapter 5에서 enumerated와  map을 사용하는 방법에 대해 배웠다.
- 아래 코드를 playground에 추가해라.

```swift
let disposeBag = DisposeBag()

Observable.of(1, 2, 3, 4, 5, 6)
	.enumerated()
	.map { index, integer in
    index > 2 ? integer * 2 : integer
  }
	.subscribe(onNext: { print($0) })
	.disposed(by: disposeBag)

// 1
// 2
// 3
// 8
// 10
// 12
```

</br></br>

## 2. Transforming inner observables

- 다음 예제에서 사용하기 위해, playground에 아래 코드를 추가해라.

```swift
struct Student {
  var score: BehaviorSubject<Int>
}
```

- `Studenct` 는 `BehaviorSubject<Int>`의 `score` 프로퍼티를 가진 구조체이다. RxSwift는 observable에 접근하여 observable의 프로퍼티과 작업하는 몇 개의 연산자를 가지고 있다. 이번 챕터에서는 위와 같은 연산자를 두가지 배울 것이다.

</br></br>

### 2-1. flatMap

<img width="553" src="https://user-images.githubusercontent.com/43217043/60963984-2b8d4a80-a34d-11e9-8015-97afa02e9ddf.png">

- `flatMap`에 대한 문서에서 "observable sequence의 요소들을 각각 observable sequence로 만들고, observable sequence들을 한개의 observable sequence로 합친다"라고 설명한다.
- 위의 marble diagram을 살펴보자.
  - 첫번째 줄 source observable에서 마지막 줄 target observable까지의 전달 과정을 보여주고 있다. target observable은 나중에 subscriber에게 요소들을 전달 할 것이다.
  - 첫번째 줄 source observable은 `Int` 타입인 `value` 프로퍼티를 가졌다. `value` 프로퍼티의 초기값은 object의 숫자이다. ( `O1`의 초기값은 1, `O2`의 초기값은 2 그리고 `O3`의 초기값은 3)
  - `O1`부터 시작하면,  `faltMap`은 `value` 프로퍼티에 10을 곱한다. `10`를 `O1`의 새 observable(두번째 줄)에 투영하고, 마지막 줄 target observable에 전달한다.
  - 나중에 `O1`의 `value` 값이 4으로 변경된다. O1의 observable(두번째 줄)에 `40`을 투영하고 마지막 줄 target observable에 전달한다.
  - source observable의 다음 값인 `O2`는 `flatMap`에서 `20`으로 변환 후, `O2`의 새 observable(세번째)에 투영하고 target observable에 전달된다. 나중에 `O2`의 `value`가 5로 바뀌면, 50으로 변환후 투영하고 target observable에 전달한다.
  - 마지막 `O3` 또한 `flatMap`을 걸쳐 변환후 투영하고 target observable에 전달된다.

</br>

- `flatMap`은 Observable의 Observable 값을 투영하고 변환한 다음 target observable에 전달된다. 실제로 사용 예시를 살펴보자. playground에 아래 코드를 추가해라.

```swift
let disposeBag = DisposeBag()

// 1
let ryan = Student(score: BehaviorSubject(value: 80))
let charlotte = Student(score: BehaviorSubject(value: 90))

// 2
let student = PublishSubject<Student>()

student
	.flatMap { $0.score } // 3
	.subscribe(onNext: { print($0) }) // 4
	.disposed(by: disposeBag)

// 5
student.onNext(ryan) // print "80"

// 6
ryan.score.onNext(85) // print "85"

// 7
student.onNext(charlotte) // print "90"

// 8
ryan.score.onNext(95) // print "95"

// 9
charlotte.score.onNext(100) // print "100"
```

- 위의 코드를 하나씩 살펴보자.
  1. 두개의 `Student` 인스턴트를 생성한다. (`ryan`과 `charlotte`)
  2. `Student` 타입의 source subject를 생성한다.
  3. `flatMap`을 사용하여 `student` subject의 `score` 값에 접근한다. `score` 값은 변형하지 않고 바로 통과 시킨다.
  4. subscription에서 `.next` 이벤트의 요소 값을 출력한다. 여기까지 아직 출력 되는 값은 없다.
  5. `student`에 `.next`이벤트를 통해 `ryan`이라는 `Student` 인스턴스를 추가하면 `ryan`의 `score` 값이 출력된다.
  6. `ryan`의 `score` 값을 변경하면, 새로운 `ryan`의 `score` 값이 출력된다.
  7. 또 다른 `Student` 인스턴트인 `charlotte`를 추가하면 `charlotte`의 `score` 값이 출된다.
  8. 이때 `ryan`의 점수를 다시 바꿔보면, 변경된 값이 여전히 출력된다.
     - 왜냐하면 `flatMap`은 source observable에 추가되는 각 요소의 observable을 계속 지켜본다.
  9. `charlotte`값을 바꿔보면 다시 `charlotte`의 `score` 값이 출력된다.

- 요약하자면, `flatMap`은 각 observable들의 변화를 계속 지켜본다.

</br></br>

### 2-2. flatMapLatest

<img width="469" src="https://user-images.githubusercontent.com/43217043/60963991-2d570e00-a34d-11e9-91f3-1a6c8ed50fa0.png">

- source observable에서 가장 최신 요소를 지켜보고 싶을때 `flatMapLatest`를 사용할 수 있다. `flatMapLatest`는 `switchLatest` + `map`을 조합한 것과 같다. `switchLatest`은 가장 최근의 observable으로 부터 값을 생산하고 이전 observable는 구독 해지 한다. 나머지는 **Chapter 9. Combining Operators**에서 자세히 배울 예정이다.
- 그래서 `flatMapLatest`는 observable sequence의 각 요소들을 observable sequence의 새로운 sequence으로 투영한다. 그런 다음 observable sequence들의 observable sequence에서 가장 최근의 observable sequence으로 부터 생성한 값들만 observable sequence로 변환한다.
- 위 marble diagram을 살펴보자.
  - `flatMapLatest`는 `O1`의 `value`를 10으로 변형시키고, `O1`의 새로운 observable으로 투영시킨다. 그런 후 target observable에 전달한다.
  - 여기서 `flatMapLatest`이 `O2`를 받으면 가장 최신의 observable은 `O2`의 observable이 된다.
  - `O1`의 `value`을 `3`으로 변경했을때, `flatMapLast`는 `O1`의 `value`를 `30`으로 변형 시킨다. 하지만 그 결과는 무시되어 target observable에 전달 되지 않는다.
  - `flatMapLatest`가 `O3`을 받으면 가장 최신의 observable은 `O3`의 observable이 된다. 그러면 `O2`은 무시 된다.

</br>

```swift
let disposeBag = DisposeBag()

let ryan = Student(score: BehaviorSubject(value: 80))
let charlotte = Student(score: BehaviorSubject(value: 90))

let student = PublishSubject<Student>()

student
	.flatMapLatest { $0.score }
	.subscribe(onNext: { print($0) })
	.disposed(by: disposeBag)

student.onNext(ryan) // print "80"
ryan.score.onNext(85) // print "85"
student.onNext(charlotte) // print "90"
ryan.score.onNext(95) // not print ***
charlotte.score.onNext(100) // print "100"
```

- `ryan`의 `score`가 변경되지 않는다. 왜냐하면 변경 이전에 `flatMapLatest`는 최근 observable이 변경 되었기 때문이다.

</br>

- `flatMapLatest`는 네트워크 연산에 가장 많이 쓰인다.
- 간단한 예시로 사전으로 단어를 찾는 것을 생각해보자. 사용자가 각 문자 s, w, i, f, t를 입력한다면 가장 최신 단어로 검색을 실행 해야한다. 이전 검색 결과 (s, sw, swi, swif로 검색한 값)은 무시해야할 때 사용 할 수 있다.

</br></br>

## 3. Observing events

- observable을 observable 이벤트로 전환해야 할 때가 있다. 프로퍼티들을 관찰하는 observable을 통제할 수 없고, 외부 sequence들이 종료되는 것을 막기 위해 error 이벤트를 처리할 때 사용한다.
- 아래 코드를 playground에 추가해라.

```swift
// 1
enum MyError: Error {
  case anError
}

let disposeBag = DisposeBag()

let ryan = Student(score: BehaviorSubject(value: 80))
let charlotte = Student(score: BehaviorSubject(value: 100))
let student = BehaviorSubject(value: ryan) // 2

// 3
let studentScore = student.flatMapLatest { $0.score }

// 4
studentScore
	.subscribe(onNext: { print($0) }) // print "80"
	.disposed(by: disposeBag)

// 5
ryan.score.onNext(85) // print "85"
ryan.score.onError(MyError.anError) // print "Unhandled error happened:"
ryan.score.onNext(90) // not print

// 6
student.onNext(charlotte) // not print
```

- 코드를 분석해보자.
  1. error 타입을 생성한다.
  2. 두개의 `Student` 인스턴트를 생성하고  `ryan`을 초기값으로하는 `student`이라는 `BehaviorSubject`를 생성한다.
  3. `student` observable의 `score`를 `Observable<Int>`으로 변형시키기 위해, `flatMapLast`을 사용하여 `studentScore`을 생성한다.
  4. `studentScore`을 구독하여 `score` 값을 출력한다.
  5.  `ryan`에 `score`, `error`, 다른 `score`를 추가한다.
  6. `student` observable에 `charlotte`를 추가한다.

</br>

<img width="533" src="https://user-images.githubusercontent.com/43217043/60963992-2defa480-a34d-11e9-939f-8241a3ab6562.png">

- 여기서 `materialize`를 사용하여, 각각의 방출되는 이벤트를 observable로 만들 수 있다.
- `studentScore` 코드 부분을 아래 코드로 바꿔라.

```swift
let studentScore = student.flatMapLatest { $0.score.materialize() }

studentScore
	.subscribe(onNext: { print($0) }) // print "next(80)"
	.disposed(by: disposeBag)

ryan.score.onNext(85) // print "next(85)"
ryan.score.onError(MyError.anError) // print "error(anError)"
ryan.score.onNext(90) // not print
student.onNext(charlotte) // print "next(100)"
```

- `studentSocre`의 타입이 `Observable<Event<Int>>`인 것을 볼 수 있다. 그리고 `studentSocre`의 subscription는 방출되는 이벤트를 구독한다. 에러는 `studentScore`를 종료 시키지만, 새로운 observable인 `charlotte`를 추가했을 때 다시 이벤트를 받아 출력한다.

</br>

<img width="532" src="https://user-images.githubusercontent.com/43217043/60963993-2defa480-a34d-11e9-92c4-9161f081d1d0.png">

- 지금은 요소가 아니라 event를 받아 출력하고 있다. `dematerialize`를 이용하면 원래 형태로 바꿀 수 있다.

```swift
studentScore
	.filter {
    guard $0.error == nil else {
      print($0.error!)
      return false
    }
    return true
  }
.dematerialize()
.subscribe(onNext: { print($0) }) // print "80"
.disposed(by: disposeBag)

ryan.score.onNext(85) // print "85"
ryan.score.onError(MyError.anError) // print "anError"
ryan.score.onNext(90) // not print
student.onNext(charlotte) // print "100"
```

</br></br>

