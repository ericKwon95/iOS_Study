## String을 합칠 때 +, +=, joined 연산의 성능 차이
Swift는 문자열 병합을 위해 다양한 연산을 지원한다. 그런데 이러한 연산들은 내부적으로 어떤 차이점이 있을까?

Swift는 오픈소스 언어이기 때문에 [GitHub](https://github.com/apple/swift)에서 내부 구현을 살펴볼 수 있다.

  * String에서 `+` 구현 코드
  ```swift
  public static func + (lhs: String, rhs: String) -> String {
    var result = lhs
    result.append(rhs)
    return result
  }
```
* String에서 `+=` 구현 코드
```swift
  public static func += (lhs: inout String, rhs: String) {
    lhs.append(rhs)
  }
```
* String에서 `joined` 구현 코드
```swift
internal func _joined(separator: String) -> String {
  // A likely-under-estimate, but lets us skip some of the growth curve
  // for large Sequences.
  let underestimatedCap =
    (1 &+ separator._guts.count) &* self.underestimatedCount
  var result = ""
  result.reserveCapacity(underestimatedCap)
  if separator.isEmpty {
    for x in self {
      result.append(x._ephemeralString)
    }
    return result
  }

  var iter = makeIterator()
  if let first = iter.next() {
    result.append(first._ephemeralString)
    while let next = iter.next() {
      result.append(separator)
      result.append(next._ephemeralString)
    }
  }
  return results
}
```
### `+`를 사용해 문자열을 합칠 경우
  * String은 기본적으로 immutable 하다, 따라서 새로운 문자열 인스턴스를 생성하기 위해 메모리를 할당받고 다시 해제하는 과정이 필요하다. 
    * copy-on-write로 동작하지만, 기본적으로 변경이 일어나면 새로운 인스턴스가 생성된다. -> 메모리 할당 및 해제 비용 발생 
  * 큰 숫자를 반복할 경우 유의미한 성능 저하를 보인다. 
  * 99999번 `+` 메소드를 사용해 문자열을 합쳤을 때 1.5초 이상의 시간이 걸렸고 88kb 이상의 메모리를 사용하였다.
### `+=` 를 사용해 문자열을 합칠 경우
  * `+=` 메소드의 left hand side는 자기 자신이기 때문에 메모리 최적화가 가능하다. 
  * inout으로 참조를 전달해 새로운 인스턴스를 생성하지 않고 문자열을 변경한다.
  * 따라서 빈 메모리 공간을 찾는 과정이 적어진다. 
    * 아예 없어지는 것이 아닌 이유는, 참조로부터 메모리 공간을 늘려가다 보면 더 큰 메모리 공간이 필요할 때가 있다. 그러면 해당 크기만큼의 공간을 다시 찾아야 한다.
  * 99999번의 테스트에서 0.022초의 시간이 걸렸고 0kb의 메모리를 사용하였다.
### `joined`을 사용해 배열을 문자열로 합칠 경우
  * join 메소드는 Sequence 프로토콜의 메소드이다. StringProtocol을 준수하는 요소를 가지는 Collection에서 사용 가능하다.
  * join 메서드에서는 reserveCapacity 메소드를 사용해 메모리 공간을 한 번에 미리 확보해 놓는다. 최종 크기를 알 수 있기 때문.
  * 그 덕분에 메모리 할당 / 해제의 반복이 없어져 높은 수준의 최적화가 가능하다.
  * 99999번의 테스트에서 0.004초의 시간이 걸렸고 0kb의 메모리를 사용하였다. 
* `+=`는 추가적인 인스턴스 생성 없이 `append`만  사용해서 `join` 보다 훨씬 코드가 간단한데, 왜 `+=`가 더 느릴까? (추정)
  * swift의 Array는 콘텐츠를 저장하기 위한 메모리 용량을 예약해놓는 capacity를 가지는데, 예약된 용량을 초과할 경우 두 배로 큰 용량을 예약하고 그 공간을 찾아 콘텐츠를 재할당한다. <- 이 때 성능적인 비용 발생
  * String도 같은 원리로, `+=`를 사용해 콘텐츠에 새로운 문자열이 더해지며 콘텐츠의 길이가 늘어날수록 capacity를 초과하여 빈 공간을 찾아 재할당하는 과정에서 오버헤드가 발생하는 것으로 추정된다.
  * 반면 `join`을 사용할 경우 `reserveCapacity`를 통해 미리 공간을 확보해둬, 콘텐츠 재할당을 위한 오버헤드가 발생하지 않아 빠르다.

### reserveCapacity
[공식문서](https://developer.apple.com/documentation/swift/array/reservecapacity(_:)-5cknc)

reserveCapacity는 Swift에서 지정된 숫자의 element를 저장할 수 있는 충분한 공간을 확보하는 메소드이다. 

이 메소드를 사용하면 여러 번의 메모리 재할당을 피할 수 있고, 배열이 고유하고 변경 가능한 연속적인 저장 공간을 확보할 수 있다.

위의 join처럼 할당 가능한 크기를 미리 알고 있다면 높은 수준의 성능 향상을 이끌어낼 수 있지만, 동적으로 증가하는 배열에 reserveCapacity를 사용하면 오히려 성능 저하가 일어날 수 있다.

이런 경우엔 일반적인 append만을 사용하는 것이 좋다. append는 동적 증가시 constant-time performance를 달성하기 위한 내부 처리를 하지만 reserveCapacity는 그것이 없기 때문이다.

### 정리
- String 타입에서 병합을 위한 각 연산자는 내부적인 구현 특성으로 인해 성능 차이가 생긴다.
- 배열에 담겨 있는 많은 양의 문자열을 병합하는 경우엔 for문을 통해 + 연산 등을 사용하는 것 보다 joined 메소드를 사용하는 것이 성능적 / 가독성 측면에서 좋다.


## URLSession의 캐싱과 그 한계
### 캐싱이란?
캐싱은 데이터나 값을 미리 복사해 놓는 임시 장소를 이용하여 데이터 접근 속도를 높이는 기술이다. 쉽게 말하면, 자주 사용하는 데이터를 가까운 곳에 미리 복사해 놓아 필요할 때 바로 꺼낼 수 있도록 하는 것이다.

네트워크를 통해 리소스를 요청한다고 생각해보자. 해당 리소스들 중
- 빈번하게 요청되고
- 잘 변경되지 않고
- 용량이 큰

리소스들(ex: 프로필 사진)에 대해 매번 네트워킹을 하면 불필요한 네트워크 비용이 증가하고, 경우에 따라 로딩 속도가 느려 반응성이 떨어지기도 한다. 따라서 해당 리소스들을 메모리 또는 디스크에 저장하여 디바이스에서 빠르게 접근할 수 있도록 캐싱한다.

### URLSession
URLSession은 관련된 네트워크 데이터 전송 task 그룹을 관리하는 객체이다. 해당 객체는 URL로 지정된 엔드포인트간 데이터 다운로드 및 업로드를 수행하는 API를 제공한다.

자세한 사항은 [URL Loading System](https://developer.apple.com/documentation/foundation/url_loading_system)을 참고!

애플은 URLSession을 사용한 HTTP / HTTPS 통신에 대한 기본적인 캐싱을 제공한다. 그렇다면 어떤 구조로, 어디까지 캐싱을 지원하는 것일까?

[이미지]

URLSession은 URLSessionConfiguration을 주입받으며 초기화되고, 해당 configuration에서 캐싱 관련 설정을 한다.
한 번 초기회된 URLSession의 configuration 프로퍼티는 변경 불가능하다. 해당 프로퍼티는 주입받은 URLSessionConfiguration의 복사본이기 때문에, 해당 프로퍼티를 변경해도 주입한 URLSessionConfiguration 설정이 변경되지는 않는다.

URLSessionConfiguration 에서는 URLCache를 사용해 캐시를 관리하고, 그 정책을 NSURLRequest.CachePolicy 타입을 통해 결정 가능하다.

여기서 이야기하는 CachePolicy는 리소스를 캐시에서 로드하는지 또는 원본 소스에서 로드하는지에 대한 정책이고, 캐시 크기 및 삭제 정책과는 관련이 없다.

그렇다면 urlCache 프로퍼티는 어떤 일을 하는 것일까?

이 프로퍼티는 URLCache 타입의 세션 내부의 요청에 대해 캐시된 응답을 제공하기 위한 URL 캐시이다.

URLCache 타입은 URL 요청을 그에 대응하는 캐시된 응답 객체에 매핑하는 객체이다. 메모리와 디스크 캐시를 지원하며, 각 캐시의 크기와 캐시 저장 장소를 지정 가능하다. 

URLSession이 기본적으로 제공하는 shared 인스턴스에 대한 configuration의 urlCache capacity들을 출력해보면 아래와 같은 값을 보여준다.

```swift
print("memory capacity: \(URLSession.shared.configuration.urlCache!.memoryCapacity) bytes")
print("disk capacity: \(URLSession.shared.configuration.urlCache!.diskCapacity) bytes")
```

[이미지]

URLSession이 제공하는 shared 인스턴스의 기본 캐시 용량은 아래와 같다. 
- 메모리 캐시 용량 : 512KB
- 디스크 캐시 용량 : 1MB

생각보다 작다는 것을 알 수 있다! 따라서 만약 이미지 등의 크기가 큰 데이터를 캐싱하고 싶다면 URLSession.shared 인스턴스는 적절하지 않은 선택이다.

이를 개선하기 위해서 싱글턴 인스턴스가 아닌 커스텀 URLSession 인스턴스와 URLSessionConfiguration을 사용, 커스텀 URLCache를 지정해서 캐싱을 따로 지원할수도 있다.
```swift
let configuration = URLSessionConfiguration.default
configuration.urlCache = URLCache(memoryCapacity: /*메모리 캐시 용량*/, diskCapacity: /*디스크 캐시 용량*/, directory: /*디스크 캐시가 저장될 디렉토리*/)
let session = URLSession(configuration: configuration)
```

메모리 및 디스크 캐시 용량과 디스크 캐시가 저장되는 경로를 지정 가능하다.

여기까지 URLsession에서 캐시 요청 정책 및 캐시 크기와 경로를 지정할 수 있다는 사실을 알았다. 그러나 URLSession이 기본 제공하는 캐시에는 한계가 있는데, 그것은 캐시 용량이 가득찰 때 삭제할 데이터를 결정하는 정책에 대한 결정권이 없고, 원하는 데이터만 캐싱하기 어렵다는 것이다.

session에 적용된 캐싱 설정은 session이 수행하는 task 전체에 적용되기 때문에 만약 이미지만을 캐싱하고 싶다면 이미지 데이터에 대해서만 다른 configuration을 가지는 session을 생성하여 적용해야 한다는 불편함이 있다. 

따라서 커스텀이 많이 필요한 경우엔 커스텀 캐시를 만들거나 서드파티 라이브러리를 사용해서 캐싱을 따로 구현하는 것이 좀 더 유연한 캐싱을 지원할 수 있는 방법이다.

### 정리
- URLSession은 기본적으로 지원하는 캐싱 기능이 있다.
- URLSession.shared 인스턴스가 제공하는 캐싱은 그 크기가 작다.
- 커스텀 configuration의 제공으로 캐싱 크기와 위치를 지정할 수 있다.
- 다만 캐싱하고자 하는 대상에 대한 유연성이 떨어지고, LRU 등 캐시 알고리즘을 적용하기 어렵다는 한계점이 있다.