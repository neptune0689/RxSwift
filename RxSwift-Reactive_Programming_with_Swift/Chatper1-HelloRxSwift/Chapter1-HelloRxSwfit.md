# Chapter 1: Hello RxSwfit!
## 1. Intro 
```
RxSwift is a library for composing asynchronous and event-based code
by using observable and functional style operators, allowing for parameterized execution via schedulers
```
RxSwift는 비동기와 이벤트 기반 코드로 이뤄진 라이브러리이다.</br>
관찰할 수 있는 함수 스타일 연산자이며, 스케쥴러를 통해 매개 변수화된 실행을 허용한다

```
RxSwift, in its essence, simplifies developing asynchronous programs
by allowing your code to react to new data and progress it in a sequential, isolated manner
```
RxSwift는 비동기 프로그램을 개발하는 것을 쉽게 한다.
코드가 새로운 데이터에 반응하고 그것을 순차적으로 처리하는 것을 허용한다</br></br>

## 2. Introduction to asynchronous programming
### 2-1. Cocoa and UIKit Asynchronous APIs
아래는 기존 iOS SDK에서 제공하는 비동기 코드이다
- NotificationCenter
- The delegate pattern
- Grand Central Dispatch
- Closures

하지만, 비동기 코드를 작성하는 아래와 같은 이슈가 생긴다
- the order in which pieces of work are performed (작업 순서)
- shard mutable data (변경 가능한 데이터 공유)

(자세한 것은 아래 내용을 보며 이해해 보자)</br></br>

### 2-2. Asynchronous programming glossary
<b>2-2-1. State and specifically, shared mutable state</b></br>
아래 예제를 보며 <b>state</b>를 이해해보자 
1. 처음 랩탑을 사용할 때 잘 작동한다
2. 하지만, 며칠이 지나면 갑자기 반응이 이상하거나 반응이 하지 않는 상황이 발생한다
3. 하드웨드와 소프트웨어는 같지만, state가 바뀌었기 때문이다
4. 메모리에 있는 데이터, 디스크에 저장된 데이터, 사용자의 입력에 반응한 모든 결과물 등, 이것들을 모두 합친 것이 랩탑의 state이다</br>

특히 여러 비동기 구성 요소들 간에 상태를 공유 할 때,  앱의 상태를 관리하는 것은 이 책에서 배우는 중요한 이슈 중 하나이다</br>

<b>2-2-2. Imperative programming</b></br>
명령형 프로그래밍은 프로그램의 상태를 바꾸기 위해 사용하는 프로그래밍 패러다임이다. 하지만 복잡하고 비동기적인 앱에서 명령형 코드를 작성하는 것은 어려운 일이다</br>
아래는 예제를 보며 이해해보자
```swift
override func viewDidAppear(_ animated: Bool) {
	super.viewDidAppear(animated)
	setupUI()
	connectUIControls()
	createDataSource()
	listenForChange()
}
```
- 이 메소드들이 무엇을 하는지 알 수 없다
- view controller의 프로퍼티들을 업데이트 하는지, 올바른 순서대로 메소드를 호출하는지도 알 수 없다
- 만약 누군가가 메소드 호출 순서를 바꿨다면, 앱이 완전 다르게 동작할 수 있다</br></br>

<b>2-2-3. Side effects</b>
Side effect (부작용)는 현재 범위 외의 상태에 대한 모든 변화이다
아래 예시를 보며 이해해보자
1. `connectUIControls` 는 UI 구성 요소들의 이벤트 핸들러를 연결하는 코드이다
2. 만약 디스크에 저장된 데이터를 수정하거나 화면 상의 text label를 갱신하면, side effects가 일어난다
3. side effects가 나쁜 것은 아니다
4. side effect가 발생하는 코드인지, 아니면 데이터를 처리하고 출력하는 코드인지 결정할 수 있어야 한다
5. 즉, side effect의 이슈는 컨트롤하는 방법이다
6. RxSwift는 이러한 이슈를 추적하게 해준다</br></br>

<b>2-2-4. Declarative code</b></br>
명령형 프로그래밍에서는 state를 변경하고, 함수형 프로그래밍은 어떠한 side effect가 없다</br>
Rxswift는 명령형 프로그래밍 장점 (= 상태를 변화 시킴)과 함수형의 프로그래밍의 장점 (= 값을 추적 가능함)을 결합했다</br>
선언형 코드에서 동작을 정의한다</br>
RxSwift는 관련 이벤트가 발생할때 마다 동작을 실행하고, 변경 불가능하고 작업과 격리된 데이터를 제공한다</br></br>

<b>2-2-5. Reactive systems</b></br>
Reactive system은 아래와 같은 추상적인 단어이며, 아래와 같은 특성을 iOS앱에서 다룰 것이다
- Responsive (반응하는): 최신의 UI를 유지하고, 최신의 앱 상태를 표시
- Resilient (탄력있는): 각 행위를 독립적으로 정의하고, 유연한 에러 복구를 제공
- Elastic (유연한): 다양한 워크로드를 다루고, lazy pull driven 데이터 수집, 이벤트 제한, 리소스 공유와 같은 것을 실행한다
- Message driven (메세지 전달): 재사용성과 독립성을 향상시키기 위해, 구성 요소는 메세지 기반 커뮤니케이션을 사용한다. 이는 생명주기와 클래스 구현을 분리한다</br></br>

## 3. Foundation of RxSwift
RxSwift의 세가지 구성 요소 - Observables, Operators, Schedulers</br></br>
### 3-1. Observables
`Observable<T>`는 여러 관찰자들에게 실시간으로 이벤트에 반응하게 한다Observable은 세가지 타입의 이벤트를 방출 할 수 있다
- next event: 가장 최신의 (또는 다음의) 데이터 값을 나른다. 이것은 관찰자들이 값을 받는 방법이다
- completed event: 이 이벤트는 event sequence를 성공적이게 종료 한다. Observable이 그것의 life-cycle를 성공적이게 종료했고, 다른 이벤트를 더 이상 방출 하지 않는 것이다
- error event: Observable이 에러와 함께 종료하고 더 이상 다른 이벤트를 방출하지 않는다</br></br>

<b>Finite observable sequences</b></br>
finite observable sequence는 0, 1 또는 다른 값들을 방출하고 마지막으로 성공적이게 종료하거나 오류와 함께 종료된다</br>

iOS app에서는 인터넷에서 파일을 다운로드하는 코드를 생각하면 된다
1. 너는 다운로드를 시작하고 결과 데이터를 관찰하기 시작한다
2. 그러면 너는 계속 파일 일부분인 데이터 덩어리를 받는다
3. 인터넷 연결이 끊기면, 다운로드는 멈출 것이고 연결은 에러와 함께 중단된다
4. 파일 데이터를 모두 다운로드 받으면, 성공적이게 종료된다

RxSwift 코드는 아래와 같다
```swift
API.download(file: "http://www...")
	.subscribe(onNext: { data in
		... append data to temporary file
	}),
	onError: { error in
		... display error to user
	},
	onCompleted: {
		... use downloaded file
	})
```
- `API.download(file:)`은 `Observable<Data>` 인스턴스를 리턴하고, 네트워크에서 온 데이터 덩어리인 `Data` 값을 방출한다
- `onNext` 클로저를 제공함으로써 next 이벤트를 구독한다. 다운로드 예시에서 디스트에 저장된 일시적인 파일에 데이터를 더한다
- `onError` 클로저를 제공함으로써 error 이벤트를 구독한다. 이 클로저에서 `error.localizedDescription`을 표시할 수 있다
- `completed` 이벤트를 다루기 위해서, `onCompleted` 클로저를 제공한다. 새로운 view contoller에 다운로드 파일이나 앱 로직 데이터를 전달할 수 있다</br></br>

<b>Infinite observable sequences</b>
파일 다운로드나 그와 비슷한 활동와 다르게, UI event는 infinite observable sequence이다<br/>
예로 들어, 너가 디바이스 방향 변화에 반응할 필요가 있다고 가정해보자
1. `UIDeviceOrientationDidChange` observer를 추가한다
2. 방향 변화를 다루기위해 callback 메소드를 제공해야한다
3. `UIDevice`으로부터 현재 방향을 잡고, 알맞게 최신 값에 반응해야한다

RxSwift 코드는 아래와 같다
```swift
UIDevice.rx.orientation
	.subscribe(onNext: { current in
		switch current {
		case .landscape:
			... re-arrange UI for landscape
		case .portrait:
			... re-arrange UI for portrait
		}
	})
```
1. `UIDevice.rx.orientation`는 `Observable<Oritation>`을 제공하는 허구의 컨트롤 프로퍼티이다
2. 이것을 구독하여 현재 방향에 받게 app UI를 갱신할 수 있다
3. 절대 방출하지 않는 `onError`와 `onCompleted` 파라미터를 생략할 수 있다</br></br>

### 3-2. Operators
`ObservableType`과 Observable 구현는 많은 메소드를 가지고 있다.</br>
독립적이고 구성가능하기 때문에 그 메소드들을 operators라고 부른다.</br>
operator는 비동기 입력을 받아 부작용 없는 출력을 하기 때문에, 퍼즐 조각과 같이 쉽게 결합할 수 있다</br>

아래는 Rx operators 예시이다
 ```swift
 UIDevice.rx.orientation
	 .filter { value in
		 return value != .landscape
	 }
	 .map { _ in
		 return "Portrait is the best!"
	 }
	 .subscribe(onNext: { string in
		 showAlert(text: string)
	 })
 ```
</br></br>

### 3-3. Schedulers
Scheduler는 Rx에서 dispatch queues와 같다. 단지 더 강력하고 사용하기 쉽다</br>
RxSwift에는 많은 scheduler 미리 정의 되어 있다</br>
그래서 99%의 상황을 커버하기 때문에, 새로운 scheduler를 생성해서 사용할 경우는 별로 없다</br></br>

## 4. App arhitecture
<img src="https://user-images.githubusercontent.com/43217043/59339373-04882c80-8d3f-11e9-8892-c03cd525d582.png">

RxSwift는 이벤트나 비동기 데이터 시퀀스를 처리하기 때문에 아키텍처에 영향을 받지 않지만, MVVM과 찰떡이다</br>
ViewModel을 사용하면 Observable<T>를 사용하여 ViewController의 UIKit에 직접 바인딩 할 수 있다</br></br>

## 5. RxCocoa
RxCocoa는 다양한 UI 이벤트들을 구독할 수 있도록, UI 구성요소에 reactive extension을 추가했다</br></br>
