# Chapter 4: Observables and Subjects in Practice

## 1. Using a variable in a view controller
<img width="362" alt="스크린샷 2019-06-13 오후 6 27 23" src="https://user-images.githubusercontent.com/43217043/59589049-075e9500-9124-11e9-8b30-55a5f4a9d2d4.png">

controller에 `Varialbe<[UIImage]>` 프로퍼티를 추가해서, 선택된 사진들을 그것의 `value`에 저장해라.
MainViewController에 아래 코드를 추가해라.
```swift
private let bag = DisposeBag()
private let images = Variable<[UIImage]>([])
```
- view controller가 dispose bag를 소유하면, view controller가 해제될때 subscription들도 같이 해제된다.
- 하지만 root view controller는 앱이 종료되기 전까지 종료되지 않기 때문에 이런 작용이 일어나지 않는다.
</br>

사용자가 + 버튼을 누를때마다, `images`에 같은 사진을 추가해보자.
```swift
images.value.append(UIImage(named: "IMG_1907.jpg")!)
``` 
- `Variable` 클래스는 자동적으로 각각의 value 마다 observable sequence를 생성한다.
- `+` 버튼을 누를 때마다, `images`에 의해 생성된 observalbe sequence는 새 `.next` 이벤트를 방출한다.
</br>

현재 선택를 초기화 하기위해 `acionClear()`에 아래 코드를 추가해라.
```swift
images.value = []
```
</br></br>

### 1-1. Adding photos to the collage
<img width="410" alt="스크린샷 2019-06-13 오후 6 28 00" src="https://user-images.githubusercontent.com/43217043/59589050-075e9500-9124-11e9-8a73-2cb0ac5e27d4.png">

`images`를 연결 했음으로, 변화를 관찰하여 콜라주를 갱신할 수 있다. `viewDidLoad()`에서 `images`에 subscription을 생성해라.
```swift
images.asObservable()
	.subscribe(onNext: { [weak self] photos in
    	guard let preview = self?.imagePreview else { return }
        preview.image = UIImage.collage(images: photos, size: preview.frame.size)
    })
    .disposed(by: bag)
```
- `images`에 의해 방출된 `.next` 이벤트를 구독한다.
- 각각 이벤트마다 `UIImage.collage(images:size:)` 메소드를 이용하여 콜라주는 생성한다.
- 그리고 subscription에 view controller의 dispose bag을 추가한다.
</br></br>

### 1-2. Driving a complex view controller UI
<img width="341" alt="스크린샷 2019-06-13 오후 6 28 07" src="https://user-images.githubusercontent.com/43217043/59589051-07f72b80-9124-11e9-836d-afc465876f16.png">

사용자의 경험을 향상시키기 위해, UI를 좀 더 스마트하게 만들어보자.
- 아직 선택된 사진이 없거나 사용자가 clear 버튼을 눌렀을때, Clear 버튼을 비활성화 시킨다.
- 선택된 사진이 없으면, Save 버튼도 비활성화 시킨다.
- 콜라주가 비어있는 것을 방지하기 위해서 홀수개의 사진을 선택 했을때, Save 버튼을 비활성화 시킨다.
- 콜라주가 6개 이상의 사진을 선택하지 못하도록 제한한다.
- 선택한 사진을 view controller에 반영한다.
</br>

아래 코드를 `viewDidLoad()`에 추가해라.
```swift
images.asObservable()
	.subscribe(onNext: { [weak self] photos in
    	self?.updateUI(photos: photos)
    })
    .disposed(by: bag)
```
- `images` 값이 바뀔 때마다, `updateUI(photos:)`를 호출한다.
</br>

class안에 아래 코드를 추가해라.
```
private func updateUI(photos: [UIImage]) {
	buttonSave.isEnable = photos.count > - && photos.count % 2 == 0
    buttonSave.isEnable = photos.count > 0
    itemAdd.isEnabled = photos.count < 6
    title = photo.count > 0 ? "\(photos.count) photos" : "Collage"
}
```
</br></br>


## 2. Talking to other view controller via subjects
<img width="442" alt="스크린샷 2019-06-13 오후 7 46 21" src="https://user-images.githubusercontent.com/43217043/59589052-07f72b80-9124-11e9-85f4-d12700747670.png">

사용자가 카메라롤에서 사진을 선택할 수 있도록 하기 위해서, `PhotosViewController` 클래스를 main view controller에 연결 할 것이다. `MainViewController`의 `actionAdd()` 메소드에서 `IMG_1907.jpg`를 사용한 코드를 코멘트 처리를 하고, 대신 아래 코드를 추가해라.
```swift
let photosViewController = storyboard!.instantiateViewController(withIdentifier: "PhotosViewController") as! PhotosViewController
navigationController!.pushViewController(photosViewController, animated: true)
```
</br>

Cocoa 패턴으로 만들어진 앱에서는 protocol delegate를 추가하여, photos controller가 main controller에게 알려주도록 하였다. 하지만, RxSwift에서는 `Observable`을 통해 두 class 사이에서 통신한다. `Observable`는 observer들에게 메세지를 전달 할 수 있기 때문이다.
</br></br>

### 2-1. Creating an observable out of the selected photos
<img width="546" alt="스크린샷 2019-06-13 오후 6 50 10" src="https://user-images.githubusercontent.com/43217043/59589067-11809380-9124-11e9-8191-d9dc00a038bc.png">

사용자가 카메라롤에서 사진을 선택할 때마다 `.next` 이벤트를 방출하는 subject를 `PhotosViewController` 에 추가해라. PhotosViewController의 상단에 아래 코드를 추가해라.
```swift
import RxSwift
```
</br>

선택된 사진을 노출하기 위해 `PublishSubject`를 추가할 것이다. 하지만 다른 클래스에서 `onNext()`를 호출하여 subject가 이벤트를 방출하도록, subject가 공개적으로 접근되는 것을 원하지 않을 것이다. `PhotosViewController`에 아래 프로퍼티를 추가해라.
```swift
private let selectedPhotosSubject = PublishSubject<UIImage>()
var selectedPhotos: Observable<UIImage> {
	return selectedPhotosSubject.asObservable()
}
```
- 선택된 사진을 방출할 `PublishSubject`는 private로 선언한다.
- subject의 observable을 노출할 `selectedPhotos`는 public으로 선언한다.
- `selectedPhotos`를 구독하는 것은 방해 없이 main controller가 photo sequence를 관찰 할 수 있는 방법이다.
</br>

`collectionView(:didSelectItemAt:)`안의 코드는 사용자에게 시각적인 피드백을 주기위해, 선택된 사진을 가져온 후에 collection cell을 깜박거리게 하는 코드이다. `imageManager.requestImage(...)`는 사진을 가져와서 completion 클로저 안에 `image`와 `info` 파라미터를 준다. 이 클로저안에 `selectedPhtosSubject`을 통해 `.next` 이벤트를 방출 할 것이다. 클로저 안에 아래 코드를 추가해라.
```swift
if let isThumnail = info[PHImageResultIsDegradedKey as NSString] as? Bool, !isThumbnal {
	self?.selectedPhotosSubject.onNext(image)
}
```
- `info` dictionary를 통해서 원본 사진인지 썸네일 사진인지 확인한다.
- 원본 사진 이면, 너는 subject의 `onNext(_)`를 호출하여 원본 사진을 제공한다.
</br></br>

### 2-2. Observing the sequence of selected photos
`MainViewController`에서 선택한 사진들의 sequence를 관찰할 것이다. `actionAdd()`를 찾아서 navigation에 push하기 전 코드에 아래 코드를 추가해라.
```swift
photosViewController.selectedPhotos
	.subscribe(onNext: { [weak self] newImage in
    
    }, onDisposed: {
    	print ("completed photo selection")
    })
    .dispose(by: bag)
```
- cotroller를 push 하기전에, `selectedPhtos` observable의 이벤트를 구독한다. 
- 사용자가 사진을 누렀다는 것을 의미하는 `.next` 이벤트와 subscription이 해제 될때에 주의 깊게 볼것 이다.
</br>

`onNext` 클로저 안에 아래 코드를 추가해라.
```
guard let images = self?.images else { return }
images.value.append(newImage)
```
</br></br>

### 2-3. Disposing subscriptions - review
여기에서 하나 간과한 것이 있다. 사진들을 콜라주에 추가하고, 메인 화면에 돌아 왔을때 콘솔을 살펴 보아라. "completed photo selectuin?"이라는 메세지가 보이는가? `onDispose`에 추가 했지만, 메세지는 절대 호출 되지 않다. 즉, 구독이 해제되어 메모리에 해제 되지 않는것이다! 왜냐하면 root viewcontroller의 disposebag를 사용했기 때문이다. `PhotosVeiwController`의 `viewWillDisappear(_:)`에 subject의 `onComplete()`를 호출하는 코드를 추가해라.
```swift
selectedPhotosSubject.onCompleted()
```
</br></br>

## 3. Creating a custom observable
<img width="498" alt="스크린샷 2019-06-17 오후 3 05 11" src="https://user-images.githubusercontent.com/43217043/59589069-11809380-9124-11e9-9bd0-727bcb438db8.png">

기존 API에서 `PHPhotoLibrary`를 extension하여 추가 구현 할 수 있지만, 지금은 커스텀 `Observable`을 생성하여  사용할 것이다. 해당 `Observable`은 disk에 성공적이게 작성하면 asset ID와 `.completed` 이벤트를 방출하고, 그렇지 않으면 .`error` 이벤트를 방출할 것이다.
</br></br>

### 3-1. Wrapping an existing API
`PhotoWriter`를 열어서 상단에 아래 코드를 추가해라.
```swift
import RxSwift
```
</br>

`PhotoWriter`에 새로운 static 메소드를 추가해라. 이것은 저장할 사진들을 줄 observable을 생성할 것이다.
```
// 1
static func save(_ image: UIImage) -> Observable<String> {
	return Observable.create({ observer in
    	
        // 2
        var savedAssetId: String?
        PHPhotoLibary.shard().performChanges({
        	
        }, completionHandler: { success, error in
        
        })
        return Disposable.create()
    })
}
```
- `save(_:)`는 `Observable<String>`을 리턴할 것이다.
- 사진을 저장하기 전에, 생성한 asset의 고유한 로컬 아이덴티를 방출할 것이다.
- `Observable.create(_)`는 새로운 Observable을 생성한다.
- `performChanges(_:completionHandler:)`에서, 제공한 이미지의 photo asset을 생성하고 asset ID 또는 `.error` 이벤트를 방출할 것이다.
</br>

`performChanges(_:completionHandler:)`의 첫번째 클로저 안에 아래 코드를 추가해라.
```swift
let request = PHAssetChangeRequest.creationRequestForAsset(from: image)
savedAssetId = request.placeholderForCreatedAsset?.localIdentifier
```
- `PHAssetChangeRequest.creationRequestForAsset(from:)`을 이용하여 새로운 photo asset을 생성하고 `savedAssetId`에 id을 저장한다.
</br>

`performChanges(_:completionHandler:)`의 `completionHandler` 클로저에 아래 코드를 추가해라.
```swift
DispatchQueue.main.async {
	if success, let id = savedAssetID {
    	observer.onNext(id)
        observer.onComplted()
    } else {
    	observer.onError(error ?? Errors.couldNotSavePhoto)
    }
}
```
- 성공 응답을 받고 `savedAssetId`가 유효한 asset id를 가지고 있다면, `.next`와 `.completed` 이벤트를 방출한다.
- 에러일 경우, 커스텀 또는 기본 에러를 방출한다.
</br>

"왜 `.next` 이벤트를 방출하는 `Observable`이 필요하지?"라는 의문이 들 수 있다. 물론 다음에 나오는 `Observable`도 사용할 수 있다.
- `Observable.never()`: 어떠한 이벤트도 방출하지 않는 observable sequence를 생성
- `Observable.just(_:)`: 값 하나와 `.completed` 이벤트 방출
- `Observable.empty()`: `.completed` 이벤트만 방출
- `Observable.error(_)`: `.error` 이벤트만 방출</br>

그럼 `Single`으로 사용할 수 있을까?

</br></br>

## 4. RxSwift traits int practice

### 4-1. Single
<img width="482" alt="스크린샷 2019-06-17 오후 4 22 08" src="https://user-images.githubusercontent.com/43217043/59589071-12192a00-9124-11e9-8521-f067f2fd0d2e.png">

- `Single`은 `Observable`에 특화 되었다.
- `Single`의 sequence에서 `.success(Value)` 또는 `.error` 이벤트를 방출할 것이다.
- `.success`는 `.next`와 `.completed`를 합한 것과 같다.
- 파일 저장, 파일 다운로드, 디스트에서 데이터 로딩 또는 값을 산출하는 비동기적인 연산 등에 사용할 수 있다.
- 그리고 `Single`을 두가지 방법으로 사용할 수 있다.
	1. `PhotoWriter.save(_)` 처럼 성공 했을때 한개의 값을 방출하는 연산자를 랩핑할 때: `PhotoWriter`의 `save(_)` 메소드에 `Observable`을 대신하여 `Single`로 수정할 수 있다.
	2. `.error` 이벤트를 구독: 어떤 observable이던 `.asSingle()`을 통해 Single으로 전환 가능할 수 있다.
</br></br>

### 4-2. Maybe
<img width="492" alt="스크린샷 2019-06-17 오후 4 22 15" src="https://user-images.githubusercontent.com/43217043/59589072-12b1c080-9124-11e9-935a-992b7cbf0bb0.png">

- 성공적이게 완료되었을때 값을 방출하지 않는 것을 빼고, `Maybe`도 `Single`과 비슷하게 동작한다.
- Maybe의 사용 예시를 생각 해보자.
	- 앱은 커스텀 사진 앨범에서 사진들을 저장하는 것이다.
	- 앨범의 identifier를 UserDefaults에 저장하고 앨범을 열때 각각의 id를 사용할 것이다.
	- `open(alumId:) -> Maybe<String>` 메소드를 디자인 해보자.
		1. 앨범에 id가 존재하면, `.completed` 이벤트 방출
		2. 사용자가 앨범을 삭제한 경우, 새로운 앨범을 생성하고 `UserDefault`에 저장할 수 있는 새로운 id와 함께 `.next` 이벤트를 방출한
		3. 뭔가 잘못 되었거나 사진 앨범을 접근할 수 없을 때, `.error` 이벤트 방출
- `Maybe.create({ ... })` 이나 obvservable을 `.asMaybe()`로 전환하여 사용 가능하다.
</br></br>

### 4-3. Completable
<img width="466" alt="스크린샷 2019-06-17 오후 4 22 21" src="https://user-images.githubusercontent.com/43217043/59589073-12b1c080-9124-11e9-950f-19c344e42d85.png">

- 해당 `Observable`는 `.completed`나 `.error` 이벤트를 방출한다.
- observable sequence를 completable으로 바꿀 수 없다.
- observable가 값을 방출하면, completable로 전환하지 못한다.
- 그래서 다른 Observable들을 사용한 것 처럼, `Completable.create({ ... })`을 사용해서만 completable sequence을 생성할 수 있다.
- 비동기 연산이 성공 여부를 알기 위해 사용한다.
</br>

`Combinestagram`을 예시로 들어보자
```swift
    saveDocument()
    .andThen(Observable.from(createMessage))
    .subscribe(onNext: { message in
    	message.display()
    }, onError: { error in
    	alert(error.localizedDescription)
    })
```
- 사용자가 작업 하는 동안, 문서를 자동 저장하도록 만든다.
- background queue에서 비동기적으로 문서를 저장하고, 성공하면 작은 notification을 띄우고 실패하면 alert를 띄운다.
- 저장 로직을 `saveDocument() -> Completable`으로 랩핑한다.
- `andThen` 연산자는 성공 이벤트에 따라 completable이나 observable로 전환할 수 있도록 한다.
</br></br>

### 4-4. Subscribing to the custom observable
`PhotoWriter.save(_)` observable은 성공시 asset id를 방출하거나 에러를 방출한다. 그러므로 `Single`은 좋은 케이스이다. `MainViewController`의 `actionSave()` 메소드에서 아래 코드를 추가해라.
```swift
guard let image = imagePreview.image else { return }
PhotoWriter.save(image)
	.asSingle()
    .subscribe(onSuccess: { [weak self] id in
    	self?.showMessage("Saved with id: \(id)")
        self?.actionClear()
    }, onError: { [weak self] in
    	self?.showMessage("Error", description: error.localizedDescription)
    })
    .disposed(by: bag)
```
</br></br>

## Challenges

### Challenge 1: It's only logical to use a Single
- `PhotoWriter`의 `save(_)` 리턴 타입을 `Single<String>`으로 바꾼다.
- `Observable.create`를 `Single.create`로 바꾼다.
- 주의할 점은 `Single.create`는 observer가 아닌 클로저를 파라미터로 받는다.
- `Observable.create`는 observer를 파라미터로 받아 여러개의 값을 방출하고 이벤트를 종료할 수 있지만, `Single.create`는 `.success(T)` 또는 `.error(E)` 값을 출력할 수 있는 클로저를 파라미터로 받는다.
- 그러므로 `single(.success(id))`와 같은 방식으로 호출한다.
```swift
static func save(_ image: UIImage) -> Single<String> {
	return Single.create(subscribe: { observer in // 1) `Single.create`로 바뀜
    	var savedAssetId: String?
        PHPhotoLibary.shard().performChanges({ [weak self] in
        	let request = PHAssetChangeRequest.creationRequestForAsset(from: image)
            self?.savedAssetId = request.placeholderForCreatedAsset?.localIdentifier
        }, completionHandler: { success, error in
        	DispatchQueue.main.async {
            	if success, let id = savedAssetId {
                	observer(.success(id)) // 2) 성공시 값 방출
                } else {
                	observer(.error(error ?? Errors.couldNotSavePhoto)) // 2) 실패시 에러 방출
                }
            }
        })
        return Disposables.create()
    })
}
```
</br></br>

### Challenge 2: Add custom observable to present alerts
- `MainViewController`에서 `showMessage(:description:)` 메소드는 alert의 Close 버튼을 눌렀을때 실행된다.
- 이는 `PHPhotoLibarary.performChanges(_)`를 구현했던 것과 비슷해 보인다.
- `UIViewController`에 extention을 추가해서, 제목과 메세지를 포함한 alert를 띄우고 completable을 리턴하는 메소드를 만들어보자.
- subscribtion이 종료되었을 때, alert도 종료해야한다.
```swift
import UIKit
import RxSwift

extension UIViewController {
	func alert(title: String, text: String?) -> Completable {
    	return Completable.create({ subscribe: [weak self] completable in
        	let alertVC = UIAlertController(title: title, message: text, preferredStyle: .alert)
            let closeAction = UIAlertAction(title: "Close", style: .default) { _ in
            	completable(.completed)
            }
            alertVC.addAction(closeAction)
            self?.present(alertVC, animated: true, completion: nil)
            return Disposable.create {
            	self?.dismiss(animated: true, completion: nil)
            }
        })
    }
}

```
</br>

- 새로운 completable을 이용하여 `showMessage(_:description:)`내에 alert를 띄울 수 있도록 한다.
- MainViewController의 showMessage(_:decription)`에 구현하자
```
alert(title: title, text: description)
	.subscribe()
    .disposed(by: bag)
```
</br></br>























