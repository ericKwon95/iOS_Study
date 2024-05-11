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
- Instruments를 실행하면 위와 같은 화면을 볼 수 있다.
- 위 아이콘들을 프로파일링 탬플릿으로, 미리 구성된 Instrument들의 모음이다. 
    - 탬플릿이 하나의 Instrument 인 것이 아님!


- 예를 들어 Time Profiler 탬플릿은 Timer Profiler와 Point of Interest 등의 Instrument 들을 활용해 성능 문제를 살펴볼 수 있다.

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
- Timer Profiler 탬플릿의 녹화를 마친 후 Instruments 창에 많은 추적 데이터들이 채워져 있다.
- 상단에는 Track Viewer가 있는데, Track은 이벤트 소스(프로세스, 스레드, CPU 코어)에 대응하는 시계열 추적 데이터를 나타낸다. 
- 하나의 Instrument가 여러 Track에 추적 데이터를 제공할 수 있다. 


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
- printing보다 간단하고 효율적이다
- 시간을 측정하는 기능이 built-in으로 지원된다
- Instruments에 의해 추적된다

## Custom Instruments
- 프레임워크 제작자가 해당 프레임워크의 성능을 측정할 수 있는 Custom Instruments 탬플릿을 제공할 수도 있다.