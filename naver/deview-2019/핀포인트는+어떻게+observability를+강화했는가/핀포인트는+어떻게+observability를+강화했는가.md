# 핀포인트는 어떻게 observability 를 강화했는가

## About

> 최근 복잡해지는 서비스의 기능을 구현하기 위해서, 서비스를 잘게 분리하는 마이크로서비스 아키텍처를 사용하는 곳이 늘어나고 있습니다. 하지만 마이크로서비스 아키텍처를 적용할 경우 모니터링을 해야 할 지점과 양이 늘어나 기존의 모니터링 방법으로는 어려운 점이 많습니다.
> 
> 이번 세션에서는 갈수록 복잡해지는 서비스에 더 나은 형태의 모니터링을 지원하기 위해서 Pinpoint팀이 해왔던 고민들과 이 고민들을 해결하기 위해서 해왔던 몇몇 작업들을 공유하고자 합니다.

발표자: 구태진 / NAVER / Observability Platform

## Content

### 1. 소개

#### What is Observability?

![1](./1.png)

* Server 환경에서는 Monitoring이라고 부른다.
* Application 환경은 애매하다. (아직 정해진 것이 없고 보는 관점에 따라 얘기하는 것이 미묘하게 다르다.)
* Distributed systems에서는 Observability이다.

#### Pinpoint는 대용량 분산 시스템을 위한 관측 관측 시스템이다.

* 내 애플리케이션에서 단순한 jvm 옵션만 준다면 손쉽게 이용이 가능하다.

  ```shell
  JAVA_OPTS = “$JAVAOPTS
  -javaagent:$PINPOINT_HOME/pinpoint-bootstrap-1.9.0.jar
  -Dpinpoint.agentId=$AGENT_ID -Dpinpoint.applicationName=$APPLICATION_NAME”
  ```

  전체적인 화면

  ![2](./2.png)

* ServerMap

  * 서버를 한눈에 볼 수 있다.

  ![3](./3.png)

  ![4](./4.png)

  ![5](./5.png)

* ScatterChart

  * 닷(.)으로 표시하여 직관적이다.
  * 특정 부분을 드래그하여 자세히 볼 수 있다.

  ![6](./6.png)

  ![7](./7.png)

* Distributed Call tree

  * 내부 코드에 대한 내용을 확인할 수 있다.
  * 굉장히 많은 부분을 볼 수 있게 지원해준다.

  ![8](./8.png)

  ![9](./9.png)

  ![10](./10.png)

* Inspector

  ![11](./11.png)

  * Timeline
  * Memory
  * CPU
  * JVM GC
  * TPS
  * Active Thread
  * Response Time
  * File Descriptor
  * Buffer Usage
  * Data Source

##### Pinpoint는 오픈소스다.

* 매년 빠르게 성장 중

* 현재 9000 스타를 넘었음 (이는 grpc나 zookeeper보다 높고 hadoop이랑 비슷한 정도)

* 중국 한국 미국 등 여러 나라에서 많이 사용 중 (특히 중국은 점유율 73% 실화?)

  ![12](./12.png)

  ![13](./13.png)

  

### 2. 실시간 observability 확보하기

####해결해야 하는 문제

###### 현재 각 에이전트가 처리하고 있는 요청은 얼마나 되는지?

###### 예상하지 못한 문제로 끝나지 않는 요청이 있다면 어떻게 확인하는지?

* 리얼타임 기능으로 해결

* 실시간 스레드덤프 기능 지원

* 구조 변경

  ![14](./14.png)



#### RPC 구조에서는?

![15](./15.png)

##### PinpointPacket 이란?

* 핀포인트내에서 데이터를 주고 받는 최소 단위

* 각 패킷은 역할을 가짐 (Send,Request, ... 등)

* Packet Format (Binary Message)

  * 패킷 앞 부분에 2byte의 PacketType을 가진다.

  ![16](./16.png)

###### 에이전트에서 Packet 송신

![17](./17.png)

###### RequestPacket 구조

![18](./18.png)

###### 콜렉터에서 Packet 수신

![19](./19.png)



##### Handshake

![20](./20.png)

###### HandshakePacket 추가

* ControlHandshakePacket

  ![21](./21.png)

* ControlHandshakeResponsePacket

###### Handshake Data

![22](./22.png)

###### 상태값 생성 및 변경

![23](./23.png)

###### 상태 처리 리스너 생성

* StateChangeEventListener 생성

##### Handshake 결과

![24](./24.png)



##### 에이전트 연결 정보 공유

* Zookeeper 사용

###### 쥬키퍼 이벤트 저장

* Worker Queue 형태

  * 쥬키퍼 부하를 줄이기 위해서 같은 형태의 이벤트는 함께 처리

    ![25](./25.png)

###### 콜렉터 노드 생성

* 연결되어 있는 에이전트의 정보를 저장하기 위한 노드 생성
  * 노드 이름 “${pid}@{HostName}”
  * 쥬키퍼의 ephemeral 노드 생성 
  * 노드에 콜렉터와 연결되어있는 모든 에이전트 정보 저장

#####웹 콜렉터 연동

* 쥬키퍼에 웹 장비의 정보를 저장하여 연결정보 공유

###### 웹 노드 생성

- 웹 모듈의 네트워크 정보 저장을 위한 노드 생성
- 노드 이름 “${대표IP}:{Port}”
- 쥬키퍼의 ephemeral 노드 생성
- 내용으로 웹 장비의 모든 네트웍 주소 정보 저장

###### 웹 콜렉터간 노드 정보 공유

![26](./26.png)

###### 콜렉터에서 웹으로 연결 구조

![27](./27.png)



##### Req/Res 한계점

###### 현재의 구조에서 ActiveThread를 가져오려면?

* 매번 요청을 해야한다.
* 이러한 문제를 해결하기 위해 StreamPacket 추가
  * StreamCreatePacket
  * StreamCreateSuccessPacket
  * StreamCreateFailPacket
  * StreamResponsePacket

###### Stream 동작 구조

![28](./28.png)

* 에이전트의 부담이 줄어든 1Req / Multi Res 



### 3. 효율적으로 observability 정보 저장하기

#### 해결해야 하는 문제

1. 사용자가 늘어난디.
2. 저장공간과 네트워크 대역이 부족해진다.
3. 조회 시간이 증가한다.

개선한 내용은?

1. Inspector 개선

   1. 데이터 전달 구조 변경

      * 기존 데이터 저장 방식
        * 5초 단위로 데이터를 받고 이를 즉시 Hbase에 저장

        ![29](./29.png)

      * 변경 후 데이터 저장 방식

        * 5분 단위 저장

        * 각각의 메트릭 정보를 나누어 저장 (메트릭 6개)

          ![30](./30.png)

      * 개선 후 결과

        * rowkey 갯수 (5분 기준, 메트릭 6개)
          * 60 -> 6
        * 조회 속도
          * parallel 요청 가능으로 성능 개선

   2. 데이터 타입 변경

      1. 데이터를 보니 일정한 규칙을 발견

      2. Dynamic Encoding 지원

         * Delta
         * Delta of Delta
         * Repeat
         * SAME
         * RAW

         ![31](./31.png)

      3. 결과

         1. 저장용량 개선
            1. 294 -> 72
            2. 67% 개선
         2. 조회 시간 개선
            1. 3.5 sec -> 350ms
            2. 90% 개선
         3. 더 긴 조회시간 추가
            1. 기존 2일 조회가 최대였지만 7일 조회 추가
         4. 서버 타임라인 이벤트 추가

   3. 최종 Inspect 개선 결과

      1. DataSource, ResponseTime, OpenFD, ByteBuffer등 추가
      2. 6 Metric 추가

2. Trace 개선

   1. 기존 SpanEvent 단위를 변경
      1. SpanChunk(임의의 SpanEvent 묶음) 단위로 저장
         1. rowkey 갯수: 10 -> 1
   2. 가까운 SpanEvent 뭉쳐서 필드벽 자주 사용하는 규칙 적용
      1. 시간이 같거나 비슷한 데이터
         1. Equal or Delta
      2. API 타입이 앞에 것과 같은 경우
         1. Equal or Raw
      3. Argument가 없는 경우
         1. Non exist or Raw
      4. depth가 같은 경우
         1. Equal or Raw
   3. 규칙 + 비트패킹 적용
      1. 기존에 모든 필드 정보를 기록하던 것을 미리 정해진 규칙으로 저장되는 경우 1bit만으로 기록
   4. 최종 Trace 개선 결과
      1. 63% 개선

3. Hbase Parallel Scanner 개발

   1. 기존에 Agent 이름으로 Salting
   2. Hotspotting은 피하고, 연관성 있는 데이터를 모았음.
   3. 하지만 데이터가 많아지면 문제 발생
      * 사내 87만 TPS가 감당 안됐음
   4. 정말 많은 데이터를 1Region에 모으고 순차적으로 조회 -> 정말 많은 데이터를 여러 Region 넣고 병렬적으로 조회
      1. 무작위 Salt 
         1. 병력적인 호출 가능
         2. 62% 개선
         3. CPU만 고생하면 된다는 결론 얻음

4. 회고

   1. Salting Key를 이 처럼 쓰는건 매우 데이터가 많이 들어오는 경우에만 권장합니다.
   2. Hbase client 2.0 부터는 유사한 기능을 지원하고 있습니다.

   

### 4. 다양한 컴포넌트에 대한 observability 확보하기

#### 해결해야 하는 문제

###### 많고 다양한 플러그인을 지원하려면

###### 지속적으로 릴리즈 되는 라이브러리의 플러그인을 지원하려면

1. 플러그인 테스트는 어떻게 할까?

   1. 고민
      * 플러그인 테스트는 최대한 실상황에서 해야함
      * 그럴려면 에이전트를 활성화 해야함
      * 그럴려면 프로세스를 실행 해야함
   2. 고민 해결
      * Custom TestRunner 생성
      * 테스트에서 외부 프로세스를 실행
        * 알고보니 IDE도 이런식으로 구현
      * 자바 프로세스는 main부터
        * ${runClass} // main에서 테스트를 실행하는 클래스 생성

2. 플러그인 테스트 구조

   ![32](./32.png)

3. 다양한 버전 테스트

   1. 네티의 경우 버전이 129개나 된다.

      * netty 버전 테스트를 하려면 pom.xml을 129번 바꾸면 되긴하는데...
      * Eclipse Aether가 이러한 기능 지원

   2. 지원 라이브러리 업데이트 구조

      ![33](./33.png)

   3. 핀포인트 테스트를 넘어서 다른 라이브러리의 버그도 찾을 수 있었음

   

### 5. What’s next 

##### Multi language support

1. 현재는 Java와 PHP만 지원

2. 앞으로 예정

   * Node.js 지원

   * gRpc으로 통신

   * GraalVM 지원

     * GraalVM이 가능하면 굉장히 많은 언어들에서 사용 가능해진다.

       ![34](./34.png)

   * Integration with k8s

     * k8s에서 사용하기 좋게 최적화 예정

   * New UI

3. 커뮤니티

   * 세계 여기저기 오픈소스 컨퍼런스 참가 및 세션 진행 중
   * 글로벌 APM 리더 초대 및 세미나도 진행 중
   * APM 주요 이슈 공유 및 토론 진행 중

   

## Review

요즘 APM에 대한 얘기를 많이 듣고 있어서 관심이 있다.

대부분 유료인데 네이버에서는 오픈소스로 공유해둬서 주변에서 많이 사용하는 것 같다.

다음에 기회가 된다면 적용 해보고 싶다.

조금 아쉬운 것은 너무 문제 해결 위주로 흘러간게 아닌가 싶다. 

사실 문제 해결을 어떻게 했는지가 중요할까..? 싶었는데 지금 생각해보니 기술 컨퍼런스니까 중요할 것 같다.

근데 PPT 장수가 너무 많아서 급하게 넘어가는 감이 있어서 그런지 머리에 안들어왔...

