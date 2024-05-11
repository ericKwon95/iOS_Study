## Instruments란?
- Xcode에서 앱 성능을 측정하기 위한 도구.
- 개발, 테스트, 디버깅 중 어느 때나 사용 가능!
- Xcode에 임베드 되어있고 자주 Xcode와 함께 사용되지만, 사실은 필요한 경우 독립적으로 사용할 수 있는 분리된 앱.

## Instruments를 사용해야 하는 이유
- 좋은 유저 경험을 위해서는 성능이 중요!
- 좋은 반응성은 앱이 원하는 대로 동작할 것이라는 신뢰를 줌
- 앱의 UI가 멋져도 로딩이 길거나 배터리를 많이 잡아먹거나 하면 좋지 못한 사용자 경험을 주게 된다.

## Instruments가 동작하는 방식
- Instruments 앱은 Instrument 라는 에러 프로파일링 도구들을 갖고 있다.
- 이러한 에러 프로파일링 도구들(Instruments)는 앱, 프로세스, 운영체제의 중요한 부분에 삽입되는 시계열 추적 데이터 (time series trace data)를 수집한다.
- 이러한 데이터들은 treat이라고 부르기도 함!

## Instruments 살펴보기
<img width="600" alt="Screenshot 2024-05-11 at 12 41 45 PM" src="https://github.com/ericKwon95/iOS_Study/assets/22342277/fe2ff6e4-699e-421b-a00f-006b2692d222">

- Instruments를 실행하면 위와 같은 화면을 볼 수 있다.
- 위 아이콘들을 프로파일링 탬플릿으로, 미리 구성된 Instrument들의 모음이다. 
    - 탬플릿이 하나의 Instrument 인 것이 아님!
- 예를 들어 Time Profiler 탬플릿은 Timer Profiler와 Point of Interest 등의 Instrument 들을 활용해 성능 문제를 살펴볼 수 있다.

<img width="1000" alt="Screenshot 2024-05-11 at 12 46 45 PM" src="https://github.com/ericKwon95/iOS_Study/assets/22342277/efab1734-950f-4236-8d83-bd091da905a0">

- Timer Profiler을 실행한 화면에서 Timer Profiler / Point of Interest / Thermal State / Hangs 등의 Instrument를 확인 가능. 
- 우측 상단의 + 버튼을 사용해 여러 Instrument 추가 가능!
- 좌측 상단에서 데이터 콜렉션을 기록, 일시정지, 정지할 수 있다.
- 프로파일링 할 타겟 디바이스를 선택할 수 있다.

## Windowed Mode
- Instrument를 사용한 프로파일링에도 시스템 자원이 소모된다.
- 만약 아주 많은 데이터 콜렉션을 추적해야 한다면 상당한 자원이 소모되어 프로파일링 진행에 방해가 될 수 있다.
- 이를 방지하기 위해 기록을 중지하기 전 마지막 몇 초 동안만의 데이터 콜렉션을 추적 및 분석하는 Windowed Mode를 사용할 수 있음.
- 몇몇 탬플릿은 기본적으로 Windowed Mode로 설정되어있기도 하다. 

## Instruments 사용해보기
<img width="1797" alt="Screenshot 2024-05-11 at 1 09 48 PM" src="https://github.com/ericKwon95/iOS_Study/assets/22342277/ca070a10-ab9b-4935-8b02-4677bb7a7a8b">

- Timer Profiler 탬플릿의 녹화를 마친 후 Instruments 창에 많은 추적 데이터들이 채워져 있다.
- 상단에는 Track Viewer가 있는데, Track은 이벤트 소스(프로세스, 스레드, CPU 코어)에 대응하는 시계열 추적 데이터를 나타낸다. 
- 하나의 Instrument가 여러 Track에 추적 데이터를 제공할 수 있다. 

- 특정한 Instrument의 trace는 수십개의 Track을 가질 수 있다. 좌측 상단의 Track filter를 사용해 Instrument만 보여주거나 thread, CPU 기준으로 보여줄 수 있다.
    - 또는 더 구체적으로 이름을 통해 검색할 수도 있음
- 하단의 디테일 뷰는 선택한 Track에 대한 추적 데이터를 살펴볼 수 있다. 위 예시에서는 Time Profiler 선택 후 추적하는 동안 각 스레드에서 호출된 함수들을 살펴볼 수 있다.
- 디테일 뷰의 우측에는 디테일 뷰의 우측에는 Inspector와 함께 확장된 디테일 뷰를 살펴볼 수 있다.
    - 확장된 디테일 뷰에서는 현재 컨텍스트와 선택 항목에 따라 사용하는 instruments로부터 더 풍부한 정보를 살펴볼 수 있다.
    - 예를 들어 현재는 Time Profiler를 활용하고 있기 때문에 가장 무거운 호출 스택을 보여주는 요약이 있다. 

<img width="1420" alt="Screenshot 2024-05-11 at 4 35 52 PM" src="https://github.com/ericKwon95/iOS_Study/assets/22342277/9063f6e7-d294-42b8-9aae-466b53bb5d07">

- 또한 Inspection Head를 통해 특정한 시간 안에 일어난 모든 추적 데이터를 살펴볼 수 있다. 
- Instrument 창에 보이는 모든 것은 추적 문서 (trace document)의 일부분이다. Instrument 앱은 추적 문서를 저장하고 다시 열어 오래된 결과를 살펴보거나 팀원과 공유할 수 있다. 
- Time Profiler을 사용한 실제 프로파일링 에시는 [WWDC19-Getting Started with Instruments](https://developer.apple.com/wwdc19/411)의 9분 7초경부터 참고

## 프로파일링 팁
- Time Profiler는 앱이 어디에서 시간을 많이 보내는지 확인하는 가장 좋은 도구
    - ex) 반응성 문제, 앱 실행에 오래 걸리는 문제 등 살펴보기 좋다
- 반응성 문제가 있다면 Main Thread 체크하기
- 프로파일링은 release build에서 진행하기
    - Xcode에서 컴파일러는 몇 개의 서로 다른 최적화 단계를 지원한다.
        - Xcode에서 Build-Run 사이클을 진행할 땐 빠르게 진행하기 위해 저수준(low-level) 최적화를 사용한다.
        - 그러나 이는 앱스토어에서 실제로 앱을 다운받아 사용자가 사용할 때 사용되는 최적화 단계와 다르다.
        - Xcode의 기본 설정을 사용한다면 프로파일링 할 때는 Release mode로 컴파일된다.
        - 그러나 커스텀 컴파일러 flag를 사용한다면 꼭 확인하기!
- 작업량이 많은 부분을 프로파일링 하거나, 옛날 디바이스에서 프로파일링하기
    - 내 디바이스에서는 잘 되어도, 옛날 디바이스에서는 문제가 발생할 수 있음

## 시뮬레이터 주의사항
- 맥북에서 시뮬레이터를 사용하면 맥북의 CPU, 메모리, 파일시스템, 온도 등의 자원을 사용하기 때문에 실제 디바이스와 완전히 일치하는 분석 결과를 줄 수 없다.
- 실제 사용자에게 릴리즈하기 전에는 무조건 실제 디바이스로 테스트해보기!

## Signpost?
- Signpost를 사용해 수집한 Instrument의 추적 데이터를 보강하고 코드가 어떻게 시스템 리소스를 사용하는지 자세히 이해할 수 있다.
<img width="685" alt="Screenshot 2024-05-11 at 4 39 32 PM" src="https://github.com/ericKwon95/iOS_Study/assets/22342277/289643de-7a9b-434f-8ca5-f5b5105a5cf6">
- Time Profiler는 앱의 모든 스레드를 일정 간격으로 관찰하고 호출 스택과 시간 간의 상관관계를 구축해 코드의 통계적 프로필을 작성한다.
- 그러나 코드가 실행되는 방식과 이유를 알려주는 정확한 측정을 대신할 수 없다.
- 보통 print문을 통해 실행시간을 측정해 함수가 어떤 시간에 어느정도 실행되는지에 관한 측정을 하게 되는데, 이를 Signpost로 해결할 수 있다.

## Signpost의 특징
- printing보다 간단하고 효율적이다
- 시간을 측정하는 기능이 built-in으로 지원된다
- Instruments에 의해 추적된다

## Signpost 예시
```swift
// old
class SceneController {
    //...
    func setupScnene {
        print("setupScene started at: \(mach_absolute_time()))")
        //...
        print("setupScene ended at: \(mach_absolute_time()))")
    }
}

// new (signpost 적용)
class SceneController {
    //...
    static let pointOfInterest = OSLog(subsystem: "com.apple.SolarSystem", category: .pointOfInterest)
    func setupScene {
        os_signpost(.begin, log: SceneController.pointsOfInterest, name: "setupScene")
        defer {
            os_signpost(.end, log: SceneController.pointsOfInterest, name: "setupScene")
        }
    }
}
```
- 위와 같이 signpost를 적용하면 실제 앱 내부에서 코드가 어떻게 호출되고 동작되는지를 시간 측정과 함께 살펴볼 수 있다.
- 관련 예시는 [WWDC19-Getting Started with Instruments](https://developer.apple.com/wwdc19/411)의 22분 54초경부터 확인 가능

## Custom Instruments
- 프레임워크 제작자가 해당 프레임워크의 성능을 측정할 수 있는 Custom Instruments 탬플릿을 제공할 수도 있다.
