## Actor 알아보기
### 동시성 프로그램과 Data race
- 동시성 프로그램이란 멀티스레딩을 통해 여러 작업을 동시에 처리하는(것처럼 보이게 만드는) 기술이다.
- 동시성 프로그램은 데이터 경쟁, 데드락 등의 복잡한 문제를 동반한다.
- 데이터 경쟁(Data race)은 아래와 같은 상황에서 발생한다.
    - 두 개의 스레드가 같은 데이터(shared mutable state)에 동시에 접근할 때
    - 둘 중 하나 이상이 write 연산일 때

```swift
class Counter {
    var value = 0

    func increment() -> Int {
        value = value + 1
        return value
    }
}

let counter = Counter()

Task.detached {
    print(counter.increment()) // data race
}

Task.detached {
    print(counter.increment()) // data race
}
```

- 시스템이 어떤 print부터 처리할지 모른다. 1 1 / 2 2 / 1 2 / 2 1 모든 경우의 수가 가능
    ex) 두 증가 연산이 모두 실행된 후 값이 반환되면 2 2를 출력한다
- 여기서 shared mutable state는 변경 가능한 공유 상태를 의미한다.
    - 상태(state)는 변수, 상수, 자료구조, 객체 등 프로그램의 현재 상황을 나타내는 정보의 집합.
    - 여러 스레드가 함께 사용 가능하고, 변경이 가능한 상태는 data race가 발생할 수 있다.

### Data race를 피하는 방법 - shared mutable state 제거하기
- Data race는 왜 일어날까?
- shared mutable state에서 데이터 경쟁이 발생하는 것!
- 값 타입(semantic)을 사용해 shared mutable state를 없앨 수 있다. 
    - 모든 변경이 지역적(local)하기 때문.
    - 변경이 일어나도 다른 인스턴스에 영향을 끼치지 않는다.
    - 값 타입의 let 프로퍼티들은 truly immutable
    - 따라서 concurrent task로부터 안전하게 접근 가능하다.
- Swift가 값 타입을 장려하는 이유 중 하나!

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
    print(counter.increment()) // always prints 1
}

Task.detached {
    var counter = counter
    print(counter.increment()) // always prints 1
}
```
- 그러나 모든 문제를 shared mutable state의 제거만으로 해결할 수 없다.
- 값 타입을 사용한 Counter는 데이터 경쟁은 해결했지만, 원하는 대로 동작하지 않는다.
- 1 2 가 나와야 하는데, 서로 다른 독립적인 인스턴스이기 때문에 항상 1 1이 출력된다.
- 따라서 shared mutable state가 필요한 경우도 있다.

### Data race를 피하는 방법 - 동기화
- 동시성 프로그래밍에서 shared mutable state를 가지려면 데이터 경합을 방지하기 위한 동기화 방법이 필요하다. 
- 저수준부터 고수준까지 다양한 동기화 방법이 존재한다.
    ex) lock, mutex, semaphore, 직렬 dispatch queue 등
- 방법들마다 각자의 장단점이 있지만, 매번 적용해야 하고, 다루기가 어렵고, 조금만 실수해도 데이터 경합이 발생할 수 있다는 단점을 공통적으로 가진다. 

### Data race를 피하는 방법 - Actor
- 이런 문제를 해결하기 위해 Actor를 만들었다!
- Actor는 shared mutable state를 위한 동기화 기법이다.
- Actor는 나머지 프로그램으로부터 분리된 고유한 state를 가진다.
- 해당 state에 접근할 수 있는 유일한 방법은 Actor를 통해서 접근하는 방법 뿐이다.
- Actor를 통해 접근할 때 마다 한 번에 하나의 코드만이 해당 state에 접근하도록 보장하는데, 이것은 lock이나 직렬 dispatch queue 등과 같이 상호배제(mutual exclusion) 프로퍼티를 제공하게 된다. 차이점은 Actor는 해당 동작을 Swift가 보장한다는 것이다.

```swift
actor Counter {
    var value = 0

    func increment() -> Int {
        value = value + 1
        return value
    }
}

let counter = Counter()

Task.detached {
    print(await counter.increment())
}

Task.detached {
    print(await counter.increment())
}
```

- 따라서 위 코드는 1 2 또는 2 1 만 출력하게 된다.
- Actor의 내부 동기화 메커니즘이 한 번에 여러 스레드가 공유 자원에 접근하는 것을 제한하기 때문이다. 

### Actor의 특징
- Actor는 Swift의 새로운 타입이다.
- Swift의 named type이 할 수 있는 것을 모두 할 수 있다.
    - 프로퍼티, 메소드, 초기자, 서브스크립트, 프로토콜 준수, extension 등
- Actor의 목적은 shared mutable state를 표현하는 것이기 때문에 actor는 클래스와 같은 참조 타입이다.

### Actor에서 동기화가 일어나는 과정 
- `actor Counter` 예제를 살펴보자.
- 두 task 중 하나가 먼저 actor에 접근하고, 나머지 하나가 차례를 기다릴 것이다.
- 그런데 어떻게 두 번째 task가 actor를 사용하기 위한 순서를 잘 기다릴 것이라는 것을 보장할 수 있을까?
- 외부에서는 비동기적으로 actor와 상호작용한다. 
- 만약 actor가 다른 task에 의해 이미 사용되고 있다면(busy), actor에 접근하려는 코드는 CPU가 다른 작업을 실행할 수 있게 suspend 상태가 된다.
- 다른 task에 의한 actor의 사용이 끝나면(free), suspend되었던 코드를 깨우고 실행을 이어간다.
- 따라서 actor에 접근할 땐 `await` 키워드가 해당 지점에서 **suspend**가 일어날 수 있음을 알려준다.

### Actor 내부에서 동기적 코드의 실행
- 아래 extension의 추가를 생각해보자.
```swift
extension Counter {
    func resetSlowly(to newValue: Int) {
        value = 0
        for _ in 0..<newValue {
            increment()
        }
        assert(value == newValue)
    }
}
```
- resetSlowly 함수는 actor 스스로의 상태에 직접적으로 접근 가능하다. ex) `value = 0`
- 또한 actor의 다른 메소드들을 동기적으로 호출할 수 있다. ex) `increment()`
- actor 내부에서는 스스로의 프로퍼티에 접근하거나 메소드를 호출할 때 `await` 키워드가 필요 없다.
- 왜냐하면 actor 내부의 동기적인 코드는 언제나 방해받지 않고 끝까지 수행되기 때문!
- 따라서 동기적인 코드가 순차적으로 실행됨을 추론할 수 있고, actor 상태의 동시성이 불러올 수 있는 부작용들을 고려하지 않아도 된다.
- 그러나 acotr는 시스템의 다른 actor나 비동기적 코드와 상호작용해야 할 수도 있다...!

### Actor와 비동기적 코드 (Actor 재진입)
- actor와 비동기적 코드가 상호작용하며 문제를 일으키는 예시를 살펴보자

```swift
actor ImageDownloader {
    private var cache: [URL: Image] = [:]

    func image(from url: URL) async throws -> Image? {
        if let cached = cache[url] {
            return cached
        }

        let image = try await downloadImage(from: url)

        // Potential bug: `cache` may have changed.
        cache[url] = image
        return image
    }
}
```
- 이미지를 다운받아 캐시에 저장하는 코드이다.
- actor로 보호되기 때문에, cache 프로퍼티에 한 번에 여러 스레드가 접근할 수는 없다.
- 그러나 문제는 `await downloadImage(from: url)`에서 suspend가 일어날 수 있다는 것이다.
- suspend 상태에서는 프로그램의 상태 변화가 일어날 수 있기 때문에 resume 이후에 이전과 같은 상태임을 기대할 수 없다.
- 아래와 같은 상황을 가정해보자.
    1. image(from: URL(string: "https://example.com")!) 호출
    2. 캐시 체크하여 비어있음을 확인
    3. await downloadImage에서 suspend (이 시점에서는 😺 다운로드중)
    4. image(from:URL(string: "https://example.com")!) 다시 호출
    5. 서버에서 이미지 교체 (😺 -> 😿)
    6. 첫 번째 다운로드가 끝나지 않았기 때문에 캐시가 비어있음을 확인 
    7. await downloadImage에서 suspend (이 시점에서는 😿 다운로드중)
    8. 3번 상황의 downloadImage가 끝나고 캐시에 이미지 등록 (😺)
    9. 😺 반환
    10. 7번 상황의 downloadImage가 끝나고 캐시에 이미지 등록 (😿)
    11. 😿 반환 
- 캐시에 이미지가 이미 등록되어 있음에도, 같은 URL로부터 다른 이미지를 반환받게 된다. 
- 본래 의도는 캐시에 이미지가 있다면, 캐시를 지우기 전에는 같은 URL에 대해서는 항상 같은 이미지를 반환하는 것. 
- 이렇게 의도하지 않은 결과를 얻은 이유는 데이터 경합 때문이 아닌, await 전후 상태가 같을 것이라고 가정했기 때문에 잠재적인 버그가 발생했기 때문이다. 

### 간단하게 버그 수정하기
- 간단한 수정법은 await 이후의 상태를 체크하는 것이다.
```swift
actor ImageDownloader {
    private var cache: [URL: Image] = [:]

    func image(from url: URL) async throws -> Image? {
        if let cached = cache[url] {
            return cached
        }

        let image = try await downloadImage(from: url)

        // Replace the image only if it is still missing from the cache.
        cache[url] = cache[url, default: image]
        return cache[url]
    }
}
```
- 캐시가 존재하는지 여부를 한 번 더 체크하여 캐시가 없을 경우에만 이미지를 삽입한다.

### 더 좋은 방법으로 버그 수정하기
- 더 좋은 방법은 중복 다운로드를 피하는 것이다.
```swift
actor ImageDownloader {

    private enum CacheEntry {
        case inProgress(Task<Image, Error>)
        case ready(Image)
    }

    private var cache: [URL: CacheEntry] = [:]

    func image(from url: URL) async throws -> Image? {
        if let cached = cache[url] {
            switch cached {
            case .ready(let image):
                return image
            case .inProgress(let task):
                return try await task.value
            }
        }

        let task = Task {
            try await downloadImage(from: url)
        }

        cache[url] = .inProgress(task)

        do {
            let image = try await task.value
            cache[url] = .ready(image)
            return image
        } catch {
            cache[url] = nil
            throw error
        }
    }
}
```

### Actor 재진입 정리
- Actor 재진입(reentrancy)는 데드락을 방지하고 코드의 진행을 보장하지만, 각 await에서 상태에 대한 추측을 확인해야 한다. 
- 따라서 Actor 상태의 변경은 동기적 코드 내에서 수행하는 것이 좋다.
- 이상적으로는 동기적 함수 내부에서 처리하여 모든 상태 변경을 잘 캡슐화 하는 것이 좋다.
- suspend 된 상태에선 프로그램의 다른 상태들이 모두 변경될 수 있기 때문에, 사용하고자 하는 상태에 대한 확인을 수행하는 것이 좋다.

