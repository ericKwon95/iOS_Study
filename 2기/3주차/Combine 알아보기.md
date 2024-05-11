## Combine이란?
- 시간 경과에 따른 값 처리를 위한 통합된 선언적 API
- Cocoa SDK는 아래와 같은 비동기적 인터페이스를 가진다.
    - Traget / Action
    - URLSession
    - Key-value Observing (KVO)
    - Ad-hoc callbacks
- 이 API들은 다양한 사용 사례를 가지며 함께 사용할 때 조금 어렵다. (구성하기도 어렵고 코드를 읽기도 어렵다)
- 이러한 API들의 공통점을 모아 Combine을 만듦!

## Combine의 특징
- Generic
    - Combine은 Swift로 작성되었기 때문에 Generic 등의 Swift 기능 사용 가능
    - 비동기적인 작업에 대한 제네릭 알고리즘을 작성해 다양한 비동기적 인터페이스에 적용 가능
- Type Safe
    - Combine은 타입 안전하기 때문에 런타임이 아닌 컴파일 타임에 에러 감지 가능
- Composition First
    - Combine의 주요 디자인 포인트는 composition first. 
    - 핵심 개념은 간단하고 이해하기 쉽지만, 이를 조합해 부분의 합보다 더 큰 무언가를 만들 수 있음
- Request Driven
    - Combine은 요청 기반이기 때문에 앱의 메모리 사용량과 성능을 더욱 세심하게 관리 가능하다.
    - 실제로 필요로 하는 경우에만 시스템 리소스 사용 / 해제 가능

## Combine의 주요 개념
- Publishers
- Subscribers
- Operators

## Publisher
```swift
protocol Publisher {
	associatedType Output
	associatedType Failure: Error

	func subscribe<S: Subscriber>(_ subscriber: S)
		where S.Input == Output, S.Failure == Failure
}
```
- 값 타입
- 값과 에러가 어떻게 생산되는지를 정의하는 역할
- 시간이 지남에 따라 이러한 값들을 수신하는 Subscriber의 등록을 허락함
- Publisher가 에러를 방출하게 하고 싶지 않다면 Failure 타입에 Never 사용

## Publisher 예시 - NotificationCenter
```swift
extension NotificationCenter {
	struct Publisher: Combine.Publisher {
		typealias Output = Notification
		typealias Failure = Never
		init(center: NotificationCenter, name: Notification.Name, object: Any? = nil)
	}
}
```
- 위 예시는 Notification을 위한 새로운 Publisher
- 값 타입임을 확인할 수 있고, Notification을 대체하는 것이 아닌 새로운 기능을 추가하는 것임을 확인 가능

## Subscriber
``` swift
protocol Subscriber {
	associatedType Input
	associatedType Failure: Error

	func receive(subscription: Subscription)
	func receive(_ input: Input) -> Subscribers.Demand
	func receive(completion: Subscribers.Completion<Failure>)
}
```
- 참조 타입
- 값과 (Publisher가 유한한 경우) completion을 수신함
- Publisher로부터 Subscriber로의 데이터 흐름을 제어하는 방식인 subscription을 수신하는 함수
- Input을 수신하는 함수
- Completion을 전달받는 함수 등 세 가지 receive 함수가 있다.

## Subscriber 예시 - KVO
```swift
extension Subscribers {
	class Assign<Root, Input>: Subscriber, Cancellable {
		typealias Failure = Never
		init(object: Root, keyPath: ReferenceWritableKeyPath<Root, Input>)
	}
}
```
- Input을 수신하면 object의 해당 프로퍼티에 값 작성 
- 프로퍼티 값을 할당할 땐 에러 처리가 필요하지 않기 때문에 Failure는 Never로 설정

## Publisher - Subscriber 동작 순서


## 사용 예시 및 Operator의 필요성
``` swift
// Using Publishers and Subscribers
class Wizard {
	var grade: Int
}
let merlin = Wizard(grade: 5)
let graduationPublisher = NotificationCenter.Publisher(center: .default, name: .graduated, object: merlin)
let gradeSubscriber = Subscribers.Assign(object: merlin, keyPath: \.grade)

graduationPublisher.subscribe(gradeSubscriber)
```
- NotificationCenter의 Publisher의 값이 발행되면 Assign Subscriber에게 전달, 값을 저장
- 그러나 이 코드는 Publisher의 Output 타입과 Subscriber의 Input 타입이 일치하지 않기 때문에 오류가 발생
- 타입을 변환해줄 무언가가 Publisher와 Subscriber의 사이에 필요함
- 그것이 Operator

## Operator
- 선언적이기 때문에 값 타입 
- Publisher 프로토콜 준수
- 값을 변경하고, 추가하고, 제거하는 등의 작업들 표현
- Publisher("upstream")를 구독하고 
- Subscriber("downstream")에게 결과를 전송한다

## Operator 예시
```swift
extension Publishers {
	struct Map<Upstream: Publisher, Output>: Publisher {
		typealias Failure = Upstream.Failure

		let upstream: Upstream
		let transform: (Upstream.Output) -> Output
	}
}

// Using Operators
let graduationPublisher = NotificationCenter.Publisher(center: .default, name: .graduated, object: merlin)
let gradeSubscriber = Subscribers.Assign(object: merlin, keyPath: \.grade)

let converter = Publishers.Map(upstream: graduationPublisher) { note in
	return note.userInfo?["NewGrade"] as? Int ?? 0
}

converter.subscribe(gradeSubscriber)
```
- Output과 Input이 일치해 오류가 사라졌다.
- 구문이 장황한 것을 아래와 같이 고쳐 간단하게 사용할 수도 있다.
```swift
extension Publisher {
	func map<T>(_ transform: @escaping (Output) -> T) -> Publishers.Map<Self, T> {
		return Publishers.Map(upstream: self, transform: transform)
	}
}
```

## Combine의 의의
- 매우 사소한 편의 기능처럼 보일 수 있지만, Combine은 앱에서 비동기 프로그래밍에 대해 생각하는 방식을 바꿀 수 있는 기능이다.
```swift
let cancellable = 
	NotificationCenter.default.publisher(for: .graduated, object: merlin)
		.map { note in
			return note.userInfo?["NewGrade"] as? Int ?? 0
		}
		.assign(to: \.grade, on: merlin)
```
- Notification을 수신하면 map 클로저를 적용해 타입을 변경하고, merlin의 grade 프로퍼티에 값을 할당한다.
- 위와 같은 문법은 단계별로 어떤 일이 일어나는지에 대한 선형적이고 이해하기 쉬운 흐름을 제공한다.

## 선언적인 Operator API
- 위와 같은 단계별 구문이 Combine 사용법의 핵심
- 각 단계는 다음 내용을 연쇄적으로 표현하고, 첫 Publisher로부터 여러 Operator를 통과해 Subscriber까지 값을 변경시킨다.
- 이러한 Operator들이 선언적인 Operator API이다.
- Swift는 기본적으로 아래와 같은 Operator들을 제공한다.
    - Functional transformations
    - List operators
    - Error handling
    - Thread or queue movement
    - Scheduling and time
- 어디서 무슨 Operator를 사용해야 하는지 고민될 수 있다.
- 한 번에 많은 기능을 담은 Operator를 찾기 보다는, composition을 활용해 원하는 결과값을 만들어 보는 것에서 시작하자!