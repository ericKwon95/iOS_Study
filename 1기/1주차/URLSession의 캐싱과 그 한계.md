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

<img src="https://github.com/ericKwon95/iOS_Study/assets/22342277/7122c998-9316-4297-982a-78223b24e2a8" width=900>

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

<img width="266" alt="Screenshot 2024-03-08 at 5 38 26 PM" src="https://github.com/ericKwon95/iOS_Study/assets/22342277/8835fb8a-7891-417b-81c1-5ae2a9434dfd">


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
