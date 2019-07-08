# Chapter 6: Filtering Operators in Practice

## 1. Improving the Combinestagram project

<img width="277" src="https://user-images.githubusercontent.com/43217043/60793959-b3cfec00-a1a3-11e9-862d-2b2c5fc330e3.png">

- 연산자들은 `Observable<E>`클래스의 메소드들이고, 그들 중 몇몇은 `Observalbe<E>`가 채택하는 `ObservableType` 프로토콜에 정의 되어 있다.
- 연산자들은 그들의 `Observable` 클래스의 요소들을 연산하고 새로운 observable sequence를 결과로 생산한다. 그것은 연산자들을 연속으로 연결할 수 있도록 하고, sequence내에서 여러 변환을 수행한다.

</br></br>

### 1-1. Refining the photos sequence

<img width="278" src="https://user-images.githubusercontent.com/43217043/60793961-b4688280-a1a3-11e9-9d14-219ee40b7012.png">

- 너는 사용자가 콜라주에서 사진들을 추가할 때, 매번 미리보기를 갱신하는 것 이상의 것을 하기를 원한다.
-  사진 observable이 완료된후, 사용자는 메인 화면으로 돌아가, on 또는 off 하거나, label을 갱신하는 등의 것들을 할 것이다.
- 같은 `Observable` 인스턴스을 구독하는 것을 공유함으로써 "더 많은 것들을 하는" 방법을 알아 볼 것이다.

</br></br>

## 2. Sharing subscriptions

- 한 observable에 여러번 `subscribe(…)`을 호출하면 무슨 문제가 생길까?
- 이전에 observable은 lazy하고 pull-driven sequence이라고 언급했다. 단순히 Observable의 연산자들을 호출하는 것은 어떠한 동작을 하지 않는다. observable에 바로 `subscribe(…)`을 호출하거나 연산자에 `subscribe(…)`를 추가하면, `Observable`은 활성화되어 요소들을 생산하기 시작한다.
- observable은 구독할때마다 그것의 `create` 클로저을 호출한다. 

</br>

```swift
var start = 0
func getStartNumber() -> Int {
	start += 1
	return start
}

let numbers = Obsrevable<Int>.create { observer in
	let start = getStartNumber()
  	observer.onNext(start)
  	observer.onNext(start+1)
  	observer.onNext(start+2)
  	observer.onCompleted()
  	return Disposables.create()
}

numbers
	.subscribe(onNext: { el in
		print("element [\(el)]")
	}, onCompleted: {
    		print("---------------")
  	})

// element [1]
// element [2]
// element [3]
// ---------------

numbers // copy and paste the exact same subscription code
	.subscribe(onNext: { el in
    		print("element [\(el)]")
  	}, onCompleted: {
    		print("---------------")
  	})

// element [2]
// element [3]
// element [4]
// ---------------

```

- 문제는 매번 `subscribe(…)`을 호출하는 것이다. subscription에 대해 새로운 `Observable`을 생성한다. 복사한 subscription은 이전과 동일한 결과를 보장하지 않는다.
- 심지어 `Observable`이 같은 요소들의 sequence를 생산할때, 각각의 subscription에 대해 똑같은 중복 요소들을 생산하는 것은 비효율적이다.
-  subscription을 공유하기 위해, `share()` 연산자를 사용할 수 있다. Rx 코드의 공통 패턴은 각 결과에서 서로 다른 요소들을 필터링함으로써, 같은 `Observable`에서 여러개의 sequence를 생성하는 것이다.

</br>

- **MainViewController.swift**의 `actionAdd()`에서 `photosViewController.selectedPhotos` 라인을 교체할 것이다.

```swift
let newPhotos = photosViewController.selectedPhotos.share()
// newPhotos [here the existing code continues: .subscribe(...) ]
```

</br>

<img width="649" src="https://user-images.githubusercontent.com/43217043/60793964-b5011900-a1a3-11e9-8a74-75a3bf035de2.png">

<img width="651" src="https://user-images.githubusercontent.com/43217043/60793967-b6324600-a1a3-11e9-9a34-d4e8a3aa5de5.png">

- 첫번째 사진 처럼 새로운 `Observable` 인스턴트를 각각의 subscription 마다 생성하는 것이 아니라, 두번째 사진 처럼 한개의 `Observable`에서 생성한 요소들을 여러개의 subscription들이 소비하도록 하자.
- `newPhotos`에 두번째 subscription을 생성하고 원하지 않은 요소들을 필터링 할 수 있다.
- 다음 과정으로 넘어가기전에 `share`이 어떻게 동작하는 것에 대해 알아보자.
  - `share`은 subscriber의 개수가 0애서 1이 되었을때 subscription을 생성한다.
  - 두번째, 세번째 subscriber들은 sequence을 관찰하기 시작할 때, `share`은 이미 생성된 subscription을 사용하여 추가된 subscriber에게 공유한다.
  - 만약 모든 subscription들이 폐기 되었을때 (= 더이상 subscriber들이 없을 때), `share`은 shared sequence도 폐기할 것이다.
  - 만약 다른 subscriber가 관찰하기 시작하면, `share`은 새로운 subscription을 생성할 것이다.
  - 참고: `share()`은 subscriptuon이 영향을 받기 전에 방출된 값을 subscription에게 제공하지 않는다. 반면에 `shard(replay:scope:)`은 최근 방출했던 값들을 버퍼에 가지고 있다가, 새로운 observer들이 subscription할때 제공한다.
- 완료되지 않거나 완료된 이후 새롭게 subscriptuon을 생성하지 않는 다는 것을 보장하면, `share()`를 사용하는 것은 안전하다.

</br></br>

### 2-1. Ignoring all elements

<img width="493" src="https://user-images.githubusercontent.com/43217043/60793970-b6cadc80-a1a3-11e9-858c-f596038bf3e2.png">

- `newPhotos`는 사용자가 사진을 선택할때마다 `UIImage` 요소를 방출하는 것을 재호출해라. 이번 섹션에서, 메인 화면의 네이베이션 왼쪽에 작은 미리보기 콜라주 아이콘을 추가할 것이다.
-  사용자가 메인 화면에 돌아 왔을때 딱 한번 아이콘을 갱신하고 싶을 것이다. 그러기 위해선 `UImage` 요소들을 모두 무시하다가 `.completed` 이벤트가 호출될때 동작하면 된다.
- 이때 `ignoreElements()`를 사용하면, source sequence의 모든 요소들을 무시하다가 `.completed` 또는 `.error` 이벤트만 통과 시킬 것이다.
- `actionAdd()` 안에, 아래 코드를 추가해라.

```swift
newPhotos
	.ignoreElements()
	.subscribe(onCompleted: { [weak self] in
    		self?.updateNavigationIcon()
  	})
	.disposed(by: photosViewController.bag)
```

- `MainViewController` 클래스안에 아래 메소드를 추가해라

```swift
private func updateNavigationIcon() {
	let icon = imagePrevie.image?
		.scaled(CGSize(width: 22, height: 22))
  		.withRenderingMode(.alwaysOriginal)
  
  navigationItem.leftBarButtonItem = UIBarButtonItem(image: icon, style: .done, target: nil, action: nil)
}
```

</br></br>

### 2-2. Filtering elements you don't need

<img width="453" src="https://user-images.githubusercontent.com/43217043/60793972-b7637300-a1a3-11e9-8611-20a78dd46368.png">

- 때론 전부가 아니라 몇몇의 요소들만 무시하는 방법이 필요가 있을것이다. 이 경우에는 `filter(_:)`를 사용할 것이다.
- 예로 들어, portait 방향의 사진들은 Combinestagram 콜라주에 맞지 않다는 것을 통지 해야한다. 이번 챕터에서는 portrait 사진 대신 landscape 사진으로 바꾸도록 할 예정이다.
- `actionAdd()`에서 첫번째 subscription을 `newPhotos`로 대체해라. 첫번째 operator에 `filter`를 삽입해라.

```swift
newPhotos
	.filter { newImage in
		return newImage.size.width > newImage.size.height
	}
	// [exising code .subscribe(...)]
```

</br></br>

### 2-3. Implementing a basic uniqueness filter

- 현재 Combinestagram은 다른 논쟁 거리가 있다: 같은 사진을 한번 이상 추가할 수 있다. 이번 챕터에서는 같은 사진을 추가하는 것을 방지하기 위한 필터링을 추가할 것이다.
- Observable은 현재 상태 또는 값의 히스토리를 제공하지 않는다. 그러므로 방출되는 요소들이 고유한지 확인하기 위해서, 직접 값을 추적해야한다.
- 같은 이미지를 나타내는 두 `UIImage`는 같지 않기 때문에, 방출된 이미지들의 인덱스를 가지고 있는 것은 도움되지 않는다. 최적의 방법은 해쉬에 이미지 데이터나 asset URL을 저장하는 것이지만, 간단한 연습을 위해 이미지의 바이트 길이를 대신 사용할 것이다. 이미지 인덱스의 고유성을 보장하지만,   구현 세부 사항에 대해 너무 자세히 설명하지 않고 작업 솔루션을 구축하는데 도움이 될 것이다.

</br>

- `MainViewController`에 새로운 프로퍼트를 추가해라.

```swift
private var imageCache = [Int]()
```

- 이 배열에서 각각의 이미지의 바이트 길이를 저장하고, 들어오는 각각의 이미지에 대해 검색할 것이다.
- 최근 추가한 `filter` 아래에 다른 `filter`를 추가해라.

```swift
// [existing .filter { newImage in ... ]
.filter { [weak self] newImage in
	let len = UIImagePNGRepresentation(newImage)?.count ?? 0
  	guard self?.imageCache.contains(len) == false else { return false }
  	self?.imageCache.append(len)
  	return true
}
// [existing code .subscribe(...) ]
```

</br>

- 이 기능을 잘 마무리 하려면, `actionClear()`에 아래 코드를 추가해라.

```swift
imageCache = []
```

</br></br>

### 2-4. Kepp taking elements while a condition is met

- Combinestagram의 가장 큰 버그는 여섯개의 사진을 추가했을때 + 버튼을이 비활성화 되는 것이다. 이것은 더 많은 이미지들을 추가하는 것을 막기 위해서 이다. 하지만 photos view controller 안 있다면, 원하는 만큼 사진들을 추가할 수 있다. 그러므로 이것을 제한해야한다.
- `taskWhile(_)` 연산자를 사용해 특정 조건을 충족시킨 후 모든 요소들을 필터링 할 수 있다. Bollean 조건을 제공해서 `takeWhile(_)`은 조건이 `false`일때 모든 요소들을 무시한다.
- `actionAdd()`에서 첫번째 subscription인 `newPhotos`를 찾아 아래 코드를 추가해라.

```swift
newPhotos
	.taskWhile{ [weak self] image in
    		return (self?.images.value.count ?? 0) < 6
  	}
// [existing code: filter { ... }]
```

</br></br>

## 3. Improving the photo selector

- 이번 섹션에서는 `PhotosViewController`로 이동할 것이다. 너는 새 커스텀 `Observable`을 만들고 화면상에 사용자 경험을 향상시키기 위해 다른 방법으로 필터링 할 것이다.

</br></br>

### 3-1. PHPhotoLibrary authorization observable

- 처음 디바이스의 포토 라이브러리에 접근 할때, OS는 비동기적으로 사용자의 접근 권한을 물어볼 것이다. 사용자 접근 권한을 묻는 것은 앱을 실행했을때 단 한번 일어난다. 그러므로 이번 섹션에서 이부분을 위해 시뮬레이터를 초기화해라.
-  새로운 소스 파일을 생성하고 이름을 `PHPhotoLibrary+rx`로 짓고 아래 코드를 적어라.

```swift
import Foundation
import Photos
import RxSwift

extension PHPhotoLibrary {
  static var authorized: Observable<Bool> {
	return Observable.create { observer in
    		return Disposables.create()
    }
  }
}
```

</br>

- 이 observable은 접근 권한 허가 여부에 따라 두 가지 방법으로 진행된다.

<img width="683" src="https://user-images.githubusercontent.com/43217043/60793976-b7fc0980-a1a3-11e9-9ee3-ca8d513d82c2.png">

- `create` 클로저 안에서 `return Dispoable.create()` 위에 아래 코드를 추가해라.

```swift
DispatchQueue.main.async {
	if authorizationStatus() == .authorized {
		observer.onNext(true)
		observer.onCompleted()
	} else {
		observer.onNext(false)
		requestAuthorization { newStatus in
			observer.onNext(newStatus == .authorized)
			observer.onCompleted()
		}
	}
}
```

</br></br>

### 3-2. Reload the photos collection when access is granted

- 사진 라이브러리 접근은 두 가지 시나리오가 있다.
- 처음에 앱을 실행했을때 사용자가 알럿의 OK를 눌렀을때

<img width="433" src="https://user-images.githubusercontent.com/43217043/60794511-e6c6af80-a1a4-11e9-9b3c-501eec3e9cf4.png">

- 이후 앱을 실행했을때, 계속 접근을 허용한 상태일때

<img width="383" src="https://user-images.githubusercontent.com/43217043/60794513-e6c6af80-a1a4-11e9-8626-34808ae34534.png">

- 처음 해야할 것은 `PHPhotoLibrary.authorized.true`를 구독하는 것이다. 이것은 각 sequence의 마지막 요소가 될 수 있으므로, 콜렉션을 다시 로드하고 카메라 롤을 보여준다.

</br>

- `PhotosViewController`의 `viewDidLoad()`안에 아래 코드를 추가해라.

```swift
let authorized = PHPhotoLibrary.authorized.share()
authorized
	.skipWhile { $0 == false }
	.take(1)
	.subcribe(onNext: { [weak self] _ in
		self?.photos = PhotosViewController.loadPhotos()
		DispatchQueue.main.async {
			self?.collectionView?.reloadData()
		}
  	})
	.disposed(by: bag)
```

- 두 개의 필터링 연산자를 차례로 사용한다.
- 첫번째는 모든 `false` 요소들을 무시하기 위해 `skipWhile(_:)`을 사용한다. 사용자가 접근 권한을 허용하지 않았을때, subscription의 `onNext` 코드는 실행되지 않을 것이다.
- 두번째는 `skipWhile`에서 통과한 true값이 통과했을때,  `task(1)`는 값 하나를 얻고 그 이후 나머지 값들은 무시한다. `true` 값은 항상 가장 마지막 요소이기 때문에 `take(1)`을 사용할 필요 없다. 하지만 `task(1)`을 사용함으로써 너의 의도는 명백히 표현된다. 만약 나중에 접근 허가 메커니즘이 바뀐다고 해도, subscription은 원하는 대로 정확히 동작할 것이다. `true` 요소는 collection view를 리로드하고 이후 오는 것은 모두 무시한다.

</br>

- `subscribe(…)` 클로저 안에 collection view 리로드 이전에 main thread로 바꾼다. 왜 그럴 필요가 있을까? `PHPhotoLibrary.authorized`의 소스 코드를 보아라. 이것은 사용자가 접근 권한을 혀용하기 위해 OK 버튼을 누른 후에 `true` 값이 방출한다.

```swift
requestAuthorization { newStatus in
	observer.onNext(newStatus == .authorized)
}
```

<img width="557" src="https://user-images.githubusercontent.com/43217043/60793977-b894a000-a1a3-11e9-94af-aed3ce30e40f.png">

- `requestAuthorization(_:)`은 어떤 쓰레드에서 completion clocure가 실행될지 보장하지 않는다. 그래서 background thread에서 실행될 수 있다. 만약 background 쓰레드 상태에서 subscription의 `self?.collectionView?.reloadData()`를 호출한다면, UIKit는 크레쉬가 날 것이다. UI 업테이트는 반드시 메인 쓰레드에서 이뤄져야 한다.

</br></br>

### 3-3. Display an error message if the user doesn't grant access

- 접근 허가를 안하지 않을때, 두가지 결과이다.
- 처음 앱을 실행 했을때, 사용자는 알럿에서 Don't Grant를 눌렀을 경우

<img width="495" src="https://user-images.githubusercontent.com/43217043/60793978-b894a000-a1a3-11e9-9ac7-33962ac01c95.png">

- 이후 실행했을때, 계속 접근을 거부한 경우

<img width="461" src="https://user-images.githubusercontent.com/43217043/60793980-b92d3680-a1a3-11e9-960e-cbb557d79257.png">

- 위의 두 가지 경우, 같은 코드에서 실패하기 때문에 sequence 요소들은 모두 같다. 따라서 두 가지 경우 모두 아래 두 가지 로직을 따를 수 있다.
  - sequence에서 방출된 첫 번째 요소를 무시할 수 있다. 왜냐하면 첫 번째 요소는 사용자가 라이브러리 허용 상태가 결과가 아닌 라이브러리 허용 초기값이기 때문이다. 두번째 요소가 사용자가  alert를 통해 사용자가 선택한 값이다.
  - 마지막 요소가 `false`인지 확인해라. 이 경우 에러 메세지를 보여줘라.
- 아래 코드를 `viewDidLoad()`에 아래 코드를 추가해라.

```swift
authorized
	.skip(1)
	.taksLast(1)
	.filter { $0 == false }
	.subscribe(onNext: { [weak self] _ in
		guard let errorMessage = self?.errorMessage else { return }
		DispatchQueue.main.async(execute: errorMessage)
   	})
	.disposed(by: bag)
```

</br>

- `skip`, `tasksLast`, `filter` 조합은 grant-access-alert-box 로직이 절대 변하지 않다는 전체 하에 아래 코드로 리팩토링 할 수 있다.
- 항상 요소가 두개 인것을 알 것이다. 그러므로 첫 번째 요소를 스킵한 후 두번째 요소를 필터 할 수 있다.

```swift
authorized
	.skip(1)
	.filter { $0 == false }
```

- 마지막 요소 이전에 요소들을 무시하다가, 마지막 요소가 `false`인지 확인한다.

```swift
authorized
	.takeLast(1)
	.filter { $0 == false }
```

- `skip`과 `tskeLast`를 `distinctUntilChanged()`로 대체할 수 있다.

```swift
authorized
	.distinctUntolChanged()
	.takeLast(1)
	.filter { $0 == false }
```

</br>

- **PhotosViewController.swift**에 아래 메소드를 추가해라.

```swift
private func errorMessage() {
	alert(title: "No access to Camera Roll", text: "You can grant access to Combinestagram from the Settings app")
		.subscribe(onDisposed: { [weak self] in
    			self?.dismiss(animated: true, completion: nil)
    			_ = self?.navigationController?.popViewController(animated: true)
  		})
  		.disposed(by: bag)
}
```

</br></br>

## 4. Trying out time based filter operators

- 시간 기반 연산자는 챕터 11 "Time Based Operators"에서 더 자세하게 배울 것이다. 하지만 몇몇 시간 기반 연산자는 필터링 연산자이기도 하다.
- 시간 기반 연산자는 **Scheduler**라고 불리는 것을 사용한다. 아래 예제에서 `MainScheduler.instance`에 대해 배울 것이다.

</br></br>

### 4-1. Completing a subscription after gicen time interval

- 이번 섹션에서 사용자가 `Close` 버튼을 5초내에 누르지 않으면, 자동으로 알럿이 사라지고 subscription이 해제 되게 할 것이다. **PhotosViewController.swift**을 열어 `errorMessage()` 안에 `alert(title:,description:)`안에 아래 코드를 삽입해라.

```swift
.task(5.0, scheduler: MainScheduler.instance)
// [existing code: .subscribe(onDisposed: ...)]
```

- `take(_:scheduler:)`는 `task(1)` 이나 `takeWhile(…)`와 같은 필터링 연산자이다. `task(_:scheduler:)`는 주어진 시간 기간동안 source sequence으로부터 요소들을 얻는다.

</br></br>

### 4-2. Using throttle to reduce work on subscriptions with high load

- **MainViewController.swift**에서 `viewDidLoad()`를 살펴보자.

```swift
images.asObservable()
	.subscribe(onNext: { [unowned self] photos in
    		self.imagePreview.image = UIImage.collage(images: photos, size: self.imagePreview.frame.size)
  	})
	.disposed(by: bag)
```

- 사용자가 사진을 선택 할때마다, subscription은 새로운 사진 콜렉션을 받고 콜라주를 생성한다. 새로운 사진 콜렉션을 받자마자 이전의 사진 콜렉션은 쓸모없다. 하지만, 만약 사용자가 여러 사진들을 빠르게 클릭하더라도, subscription은 들어오는 요소에 대해 일일이 새로운 콜라주를 생성한다. 이것은 헛된 노력이다.
- 위와 같은 상황은 실제로 자주 일어난다. "들어오는 요소가 많다면 마지막 것만 가져와라."는 비동기 프로그래밍에서 흔한 패턴이기 때문에, 이를 위한 Rx 연산자가 있다.
- `viewDidLoad()`의 `images.asObservable()`이후에 아래 코드를 삽입해라.

```swift
.throttle(0.5, scheduler: MainScheduler.instance)
// [existing code: .subscribe(onNext: ...)]
```

- `throttle(_:scheduler:)`는 특정 시간 기간동안내에 뒤따라 오는 요소들을 필터링 한다.
- 그래서 사용자가 사진을 선택하고 0.2초 후에 다른 사진을 선택하면, throttle은 첫 번째 요소를 무시하고 두번째 요소를 통과시킬 것이다. 그러면 첫번째 콜라주를 생성하는 작업을 아낀다.
- `throttle`은 연속적으로 오는 둘 이상의 요소에 대해서도 동작한다. 만약 사용자가 0.5초내에 5장의 사진을 클릭한다면, `throttle`은 첫번째 부터 네번째 사진들을 무시하고, 마지막인 다섯번째 사진만 통과 시킬 것이다.

<img width="531" src="https://user-images.githubusercontent.com/43217043/60793981-ba5e6380-a1a3-11e9-952d-c23b8fe5f261.png">

- `throttle`을 사용할 수 있는 상황은 다음과 같다.
  - 현재 텍스트를 서버 API에 보내는 검색 텍스트 필드 subscription가 있다. `throttle`을 사용하면, 사용자가 타이밍을 끝낸 단어를 서버에 요청 보낼 수 있다.
  - 사용자가 bar 버튼을 눌렀을때, 모달 view controller를 present한다. 모달 view controller가 두번 present 되지 않도록, `throttle`을 사용하여 더블 탭을 방지 할 수 있다.
  - 사용자가 손가락을 이용해 화면을 드래그하고, 드레그를 멈춘 위치에 대해 알고 싶을 때가 있다.



