## Actor 더 알아보기
- Actor는 프로그램으로부터 독립적인 상태를 가진다.
- 이러한 특징이 프로토콜, 클로저, 클래스 등의 다른 언어적 기능들과 어떻게 상호작용할까?

### Actor와 프로토콜
- 프로토콜의 요구사항을 만족한다면, 다른 타입들처럼 Actor는 프로토콜을 준수할 수 있다.
```swift
actor LibraryAccount {
    let idNumber: Int
    var booksOnLoan: [Book] = []
}

extension LibraryAccount: Equatable {
    static func ==(lhs: LibraryAccount, rhs: LibraryAccount) -> Bool {
        lhs.idNumber == rhs.idNumber
    }
}
```
- 위 코드는 Equatable을 준수하는 LibraryAccount Actor이다.
- 여기서 extension으로 구현한 정적 메소드는 static이기 때문에 self 인스턴스가 없어 actor와 함께 격리되지 않는다.
- 대신 Actor 타입의 프로퍼티 두 개에 접근 가능한데, 해당 정적 메소드는 두 프로퍼티 모두의 외부에 있지만 현재는 let으로 선언된 immutable한 프로퍼티에만 접근하기 때문에 상관없다.

```swift
actor LibraryAccount {
    let idNumber: Int
    var booksOnLoan: [Book] = []
}

extension LibraryAccount: Hashable {
    nonisolated func hash(into hasher: inout Hasher) {
        hasher.combine(idNumber)
    }
}
```
- 위 코드는 Hashable을 준수하는 LibraryAccount Actor이다.
- hash 함수는 actor 내부에서만이 아닌 외부에서도 접근 가능하다.
- 그러나 hash는 async한 함수가 아니기 때문에, Actor isolation을 위배할 수 있다.
    - actor 내부가 아닌 외부에서 접근 가능한데, suspend가 불가능하기 때문에 actor isolation이 불가능하다.
    - ex) 동시에 두 스레드에서 hash 함수를 실행하게 된다면?
    - 그래서 `nonisolated`를 붙이지 않으면 오류 발생 (actor-isolated method hash cannot satisfy synchronous requirement)
- 따라서 `nonisolated` 메소드로 변경해, 해당 메소드가 actor의 외부에서도 호출될 수 있음을 알리고 actor의 mutable state을 참조하지 못하도록 제한해야 한다.
- 이렇게 하면 Hashable 프로토콜의 동기적 요구사항을 충족 가능하다.
- 위 예시는 immutable 프로퍼티인 idNumber을 참조하기 때문에 오류 없음!
- 만약 mutable 프로퍼티인 bookOnLoan을 참조한다면 오류가 발생할 것

### Actor와 클로저
- 클로저는 actor isolated일 수도 있고 nonisolated일 수도 있다.
```swift
extension LibraryAccount {
    func readSome(_ book: Book) -> Int { ... }
    
    func read() -> Int {
        booksOnLoan.reduce(0) { book in
            readSome(book)
        }
    }
}
```
- 위 예시는 actor-isolated 클로저를 보여준다.
- reduce 작업은 동기적으로 실행되고, 해당 클로저를 다른 스레드로 escape시킬 수 없기 때문에 동시적인 접근 발생이 불가능하다.
- 따라서 readSome에는 await이 필오엾고, 클로저는 actor-isolated 하다.

```swift
extension LibraryAccount {
    func readSome(_ book: Book) -> Int { ... }
    func read() -> Int { ... }
    
    func readLater() {
        Task.detached {
            await read()
        }
    }
}
```
- 그렇다면 위 예시는 어떨까?
- detached Task는 actor와 분리된 공간에서 동시적으로 클로저를 실행한다.
- 따라서 actor의 메소드를 호출하기 위해서는 await 키워드를 사용해 비동기적으로 호출해야 한다. 
- 여기까지 actor-isolated와 nonisolated 된 클로저의 동작을 살펴보았다.

### Actor isolation과 데이터
- Actor isolation과 데이터의 관계를 생각해보자. 
- 위의 LibraryAccount 예제에서 Book 타입이 구조체로 이루어져 있다고 가정해보자.
```swift
actor LibraryAccount {
    let idNumber: Int
    var booksOnLoan: [Book] = []
    func selectRandomBook() -> Book? { ... }
}

struct Book {
    var title: String
    var authors: [Author]
}

func visit(_ account: LibraryAccount) async {
    guard var book = await account.selectRandomBook() else {
        return
    }
    book.title = "\(book.title)!!!" // OK: modifying a local copy
}
```
- 이렇게 구현하면 LibraryAccount actor 인스턴스의 모든 상태는 self-contained이기 때문에, selectRandomBook 메소드를 수행하면 Book 인스턴스의 복사본이 주어지게 된다. 
- 따라서 Book의 복사본에 가해지는 변경은 actor에게 영향을 미치지 않는다.
- 그런데 Book 타입이 클래스라면 어떨까?
```swift
actor LibraryAccount {
    let idNumber: Int
    var booksOnLoan: [Book] = []
    func selectRandomBook() -> Book? { ... }
}

class Book {
    var title: String
    var authors: [Author]
}

func visit(_ account: LibraryAccount) async {
    guard var book = await account.selectRandomBook() else {
        return
    }
    book.title = "\(book.title)!!!" // Not OK: potential data race
}
```
- LibraryAccount actor는 Book의 인스턴스에 대한 참조를 가진다.
- 이 자체로는 문제가 되지 않지만, selectRandomBook 메소드를 호출하게 되면 actor의 mutable state에 대한 참조가 actor 외부에 공유되게 된다.
    - actor 외부인 visit 함수의 book에 actor 내부의 변경 가능한 상태가 공유됨.
- book.title을 변경하게 되면 actor 내부에서 접근 가능한 상태를 변경하게 됨.
- 이는 데이터 경쟁을 일으킬 수 있음! 
- 문제다 문제~~
- 이를 해결하기 위한 방법이 Sendable 타입!

### Sednable 타입
- Sendable은 프로토콜.
- Sendable 타입은 서로 다른 actor들에 의해 공유될 수 있는 값들이다.
- 한 곳에서 다른 곳으로 값을 복사할 수 있고, 두 곳 모두에서 서로 영향을 주지 않고 해당 값의 자체 복사본을 안전하게 수정할 수 있는 경우 해당 타입은 Sendable이 될 수 있다. ex) struct
- 값 타입은 각 복사본이 독립적이기 때문에 Sendable이다.
- actor 타입은 변경 가능한 상태에 대한 접근을 동기화하기 때문에 Sendable하다.
- 클래스는 조심스럽게 구현된 경우 Sendable할 수 있다. 
    - Immutable class / 클래스와 그것의 서브클래스들이 모두 변경 불가능한 데이터만을 가질 때
    - Internally-synchronized class / 클래스가 내부적으로 동기화를 수행할 때 
- 그러나 대부분의 클래스는 이를 만족하지 않기 때문에 Sendable이 될 수 없다.
- Sendable 타입은 데이터 경쟁으로부터 코드를 보호하기 때문에, actor를 포함한 동시성 코드들은 기본적으로 Sendable 타입들로 통신해야 한다.
- Swift는 non-Sendable 타입이 공유되는 상황을 막을 것. 
```swift
struct Book: Sendable {
    var title: String
    var authors: [Author]
}
```
- Sendable한 타입은 해당 프로토콜을 준수시켜서 만들 수 있다.
- Swift가 이 타입이 Sendable을 준수하는지 체크할 것. 
- Book 구조체는 내부 저장 프로퍼티들이 모두 Sendable하다면 Sendable을 준수할 수 있다.
- 만약 Author가 클래스라면 Swift는 Book이 Sendable일 수 없다는 컴파일러 에러를 표시할 것. 
- 제네릭 타입에 대한 Sendable 여부는 제네릭 인자에 따라 달라질 수 있다.
- conditional conformance를 사용하여 적절한 경우 Sendable을 전파(propagate)할 수 있다. 
```swift
struct Pair<T, U> {
    var first: T
    var second: U
}
- Apple은 동시적으로 공유하기 안전한 타입들도 Sendable을 준수하는 것을 권장한다.

extension Pair: Sendable where T: Sendable, U: Sendable {
}
```

### @Sendable 함수 타입
- @Sendable 함수 타입들
    - 함수는 Sendable일 필요가 없어서 함수들이 actor 사이에 안전하게 전달될 수 있도록 하는 새로운 함수 타입이 생김 - @Sendable
- 함수 그 자체는 Sendable할 수 있기 때문에 actor 사이에서 함수를 전달하는 것은 안전하다.
- @Sendable은 데이터 경합을 방지하기 위해 클로저가 수행할 수 있는 작업을 제한한다.
- 예를 들어, Sendable 클로저는 변경 가능한 지역 변수를 캡처할 수 없다. 그것을 캡처하면 지역 변수에 대한 데이터 경쟁을 허용하기 때문이다.
- 클로저가 actor 사이에 non-Sendable 데이터를 전달하는 것을 막기 위해 클로저가 캡처하는 모든 것은 Sendable해야 한다. 
- 동기적인 Sendable 클로저는 actor-isolated될 수 없다. 왜냐하면 그렇게 되었을 때 외부로부터 actor의 코드를 실행할 수 있게 되기 때문이다.
- 실제 사용 예시를 알아보기 위해 아래 코드를 살펴보자.
```swift
struct Counter {
    var value = 0

    mutating func increment() -> Int {
        value = value + 1
        return value
    }
}

let counter = Counter()

Task.detached {
    var counter = counter
    print(counter.increment()) 
}

Task.detached {
    var counter = counter
    print(counter.increment()) 
}
```
```swift
static func detached(operation: @Sendable () async -> Success) -> Task<Success, Never>
```
- Task.detached 메소드는 @Sendable 클로저를 갖고 있다.
- 지역 변수를 캡처할 수 없다는 제약 덕분에, 만약 아래와 같이 코드를 변경한다면 컴파일러는 오류를 발생시키고 데이터 경쟁을 방지할 것이다. 
```swift
let counter = Counter()

Task.detached {
    print(counter.increment()) 
}

Task.detached {
    print(counter.increment())
}
```
- 이전에 나왔던 예시 중 하나를 더 보자.
```swift
extension LibraryAccount {
    func read() -> Int { ... }

    func readLater() {
        Task.detached {
            await self.read()
        }
    }
}
```
- 마찬가지로 detached 클로저의 @Sendable 제약 덕분에 해당 클로저는 actor로부터 독립적이어서(actor-nonisolated), actor와의 상호작용은 비동기적으로 실행되어야 한다는 점을 알 수 있다.
- 그래서 await을 제거하게 되면 오류가 발생한다.
- Sendable 타입과 클로저는 변경 가능한 상태가 actor들 사이에 공유되지 않는지 확인하고, 동시에 변경되지 않도록 확인해서 actor-isolation을 유지한다! Sendable 적극 사용 필수. 

### MainActor
- 메인 스레드는 UI 렌더링과 유저 상호작용 이벤트를 처리(Main run loop)하는 중요한 요소이다. 
- 메인 스레드에 너무 많은 작업을 할당하면 UI가 멈출 수 있기 때문에 메인 스레드에 작업을 할당할 땐 조심성있게 할당해야 하고, 연산비용이 비싸거나 오래 기다리는 작업은 다른 스레드에서 수행하는 것이 좋다.
- 그래서 보통 메인 스레드가 아닌 곳에서 작업하고, 필요한 경우에만 `DispatchQueue.main.async`를 호출해 메인 스레드에서 작동시킨다.
```swift
func checkedOut(_ booksOnLoan: [Book]) {
    booksView.checkedOutBooks = booksOnLoan
}

// Dispatching to the main queue is your responsibility.
DispatchQueue.main.async {
    checkedOut(booksOnLoan)
}
```
- 그런데 이 구조는 Actor와 비슷하다. 
- 이미 메인 스레드에서 실행 중이라면, 안전하게 접근해서 UI 상태를 업데이트할 수 있다.
- 메인 스레드에서 실행중이 아니라면, 비동기적으로 메인 스레드와 상호작용하면 된다. 
- Actor의 작동방식과 정확히 일치함!
- 따라서 메인 스레드를 describe하기 위한 특별한 actor인 MainActor 탄생!
- MainActor는 메인 스레드를 표현하는 actor이다.
- 두 가지 측면에서 일반적인 Actor와 다름. 
    - MainActor는 항상 main dispatch queue에서 동기화를 수행한다. 따라서 런타임 관점으로 보면 DispatchQueue.main을 대체할 수 있다.
    - 메인 스레드에서 실행되어야 하는 코드와 데이터는 여러 곳에 분산되어 있다. ex) SwiftUI, AppKit, UIKit, 등 여러 시스템 프레임워크에도 존재함. 뷰, 뷰컨 등 여러 곳에 존재
```swift
@MainActor func checkedOut(_ booksOnLoan: [Book]) {
    booksView.checkedOutBooks = booksOnLoan
}

// Swift ensures that this code is always run on the main thread.
await checkedOut(booksOnLoan)
```
- @MainActor attribute로 해당 함수가 MainActor에서 실행된다고 작성할 수 있다.
- 해당 함수를 MainActor의 바깥에서 호출하려면 `await` 키워드를 사용해서 메인 스레드에서 비동기적으로 수행된다는 것을 알려줘야 한다.
- 이러한 MainActor사용의 장점은 DispatchQueue.main에 비해 추론작업이 없다는 것이다. 
    - 이 코드가 항상 메인 스레드에서 실행된다는 것을 Swift가 보장한다.


```swift
@MainActor class MyViewController: UIViewController {
    func onPress(...) { ... } // implicitly @MainActor

    nonisolated func fetchLatestAndDisplay() async { ... } 
}
```
- 타입도 MainActor가 될 수 있고, 모든 멤버와 서브클래스도 MainActor로 만든다. 
- 이는 UI와 상호작용해야만 하는 코드베이스를 작성하는 데 유용하다. (메인 스레드를 주로 사용해야 하기 때문)
- MainActor에서 수행되지 않을 메소드는 `nonisolated` 키워드를 사용해 제외할 수 있다.
- UI와 밀접한 타입과 작업에 대한 MainActor의 사용과 다른 프로그램 상태 관리를 위한 actor들의 사용으로 안전하고 올바르게 동시성을 사용할 수 있도록 앱을 설계할 수 있다.


### 총정리
- Actor를 사용해서 변경 가능한 상태에 대한 접근을 동기화하자.
- Actor 재진입을 고려해서 설계하자.
    - 코드에서 await을 만나면 나머지 상태가 변경될 수 있음을 생각하기.
- 데이터 경쟁을 제거하기 위해 값 타입과 Actor를 사용하자.
    - non-sendable 타입들에 주의하자.
- UI 상호작용을 보호하기 위해 MainActor를 활용하자.