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
