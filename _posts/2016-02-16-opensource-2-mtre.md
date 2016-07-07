---
layout: post
title: 'kakao의 오픈소스 Ep2 - MRTE(MySQL Realtime Traffic Emulator)'
author: matt.lee
date: 2016-02-16 13:11
tags: [opensource,mysql-realtime-traffic-emulator,mtre,mysql,go]
image: /files/covers/traffic.jpg
---
<a id="forkme" href="https://github.com/kakao/MRTE-Collector"></a>

> "카카오의 오픈소스를 소개합니다" 두번째는 [matt.lee](https://github.com/SunguckLee)와 동료들이 개발한 **MySQL Realtime Traffic Emulator(MRTE)**입니다.
>
> MRTE는 실서비스용 MySQL 서버의 트래픽을 수집하는 [MRTE-Collector](https://github.com/kakao/MRTE-Collector)와, 수집한 데이터를 테스트용 MySQL 서버에서 재현하는 [MRTE-Player](https://github.com/kakao/MRTE-Player) 두 개의 툴로 구성되어 있습니다.
>
> 카카오에서도 효율적인 MySQL 운영에 큰 도움이 되고 있는 유용한 소프트웨어입니다. 특히 MRTE-Collector는 [Go](https://golang.org)로 작성되어 Go로 네트웍 프로그래밍을 하려는 개발자들에게 유용할 것입니다.

MySQL 서버를 사용하면서, 가끔씩 실 서비스용 MySQL 서버(Production mysql server)로 유입되는 쿼리들을 똑같이 흉내낼 수 없을까 하는 생각들을 많이 했습니다.

실 서비스용 MySQL 서버에서는 MySQL 시스템 변수 하나도 조정해보기 어려운 경우가 많고, 때로는 업그레이드나 통합 또는 하드웨어 테스트를 하는 경우에는 이런 도구들이 절실했죠.

이를 위해서 MRTE (MySQL Real Traffic Emulator) 도구를 생각하기 시작했는데, 조금만 고민해보니 사실 이는 그다지 어려운 일이 아니었다. 여기에서는 MRTE에 대한 간략한 아키텍쳐와 사용법을 간단히 소개하도록 하겠습니다.

MRTE는 크게 유저 트래픽을 수집하는 MRTE-Collector와 수집된 SQL을 재현하는 MRTE-Player로 구성되어 있는데, MRTE-Collector와 MRTE-Player는 Message Queue ([Rabbit MQ](http://www.rabbitmq.com))를 이용해서 통신하도록 설계되었습니다.

![MRTE Collector와 Player 전체적인 아키텍쳐](/files/mrte-overall-arch.png)

* [MRTE-Collector]: Source MySQL 서버에서 네트워크 트래픽을 캡쳐하는 Message Queue로 전달
* [MRTE-Player]: Message Queue의 네트워크 패킷을 가져와서 분석하고 Target MySQL 서버로 전달(실행)

이 도구는 크게 아래와 같은 2가지 제약 사항을 가집니다:

1. [MRTE-Collector]는 네트워크 인터페이스의 패킷을 캡쳐하기 때문에, 반드시 MySQL 서버와 동일 장비에서 실행되어야 합니다.
2. Server-side prepared statement는 외부 툴에서 SQL 문장을 덤프할 수 있는 방법이 없으므로, 현재 버전의 MRTE는 Server side prepared statement는 지원하지 못합니다.

1번 제약 사항을 위해서 MRTE-Collector는 최소한의 자원을 사용하면서도 빠르게 작동할 수 있도록 설계했으며, 이를 위해서 GO 언어를 사용해서 Native code로 컴파일해서 실행할 수 있도록 개발되었습니다.

실제 초당 35000개의 패킷을 캡쳐해서 외부의 Message Queue로 전송하는 경우에도 2~3%의 CPU만 사용하는 것으로 관측되었다. 하지만 일반적인 서비스 환경의 MySQL 서버에서 초당 몇 만정도의 쿼리를 처리하는 경우는 그다지 많지 않다는 것을 감안하면, MRTE-Collector를 MySQL 서버와 동일 장비에 실행한다는 것은 그다지 큰 제약 사항이 아닐 수도 있어 보인다. 또한 MRTE-Collector는 10~15MB 정도의 물리 메모리만 사용했었다. GO 프로그램의 특성상 Virtual memory는 700MB 정도로 꽤 점유하는 편이지만, 사실 Virtual memory 확보는 그다지 시스템의 자원 사용이나 성능에 영향을 미치지 않는다는 것이 Google의 의견입니다.

아래 그래프는 MRTE-Collector가 실행중인 MySQL 서버의 CPU 사용량인데, 잠깐 MRTE-Collector를 멈췄을 때 CPU 사용량이 얼마나 떨어지는지를 보여주고 있습니다:

![MRTE의 CPU 사용량](/files/mtre-cpu.png)

그리고 MRTE-Collector는 `tcpdump`나 `ngrep` 명령과 같이 `pcap` 라이브러리를 이용하기 때문에 매우 안정적으로 패킷을 캡쳐할 수 있다. 실제 `sysbench`로 초당 35000 쿼리가 실행되는 환경에서도 MRTE-Collector 시작 및 종료(패킷 캡쳐 시작 및 종료)시에도 서비스에 특별한 성능 악 영향은 보이지 않았다. 또한 Message Queue나 MRTE-Collector가 문제를 일으켜 제대로 처리하지 못할 때에는, `pcap` 라이브러리는 MRTE-Collector의 처리를 기다리지 않고 수집된 패킷을 버리고 무시하기 때문에 유저의 네트워크 패킷을 블록킹하지는 않습니다.

2번 제약 사항(Server side prepared statement) 제약 사항에 대해서 조금 살펴보겠습니다. MySQL 서버에서는 2종류의 PreparedStatement를 지원하고 있습니다:

* Client side prepred statement: 초기 MySQL 서버가 PreparedStatement를 지원하지 못하던 시절에는 Client JDBC Driver에서 PreparedStatement를 흉내낼 수 있도록(표준 준수) Client side prepared statement를 지원했었는데, 이를 Client side prepared statement라고 합니다.
* Server side prepared statement: 이후 MySQL 서버에서 PreparedStatement를 지원하게 되면서 실제 Client side에서 PreparedStatement emulation이 아닌 타 RDBMS와 동일한 방식의 PreparedStatement 기능을 지원하기 시작했는데, 이를 Server side prepared statement하고 합니다.

초기 MySQL 서버에서는 Text protocol(Text protocol이라고 해서 문자가 전송되는 프로토콜을 의미하는 것이 아니라 SQL 문장이 그대로 전달된다는 의미에서 Text protocol이라 함)만 지원했었는데, Server side prepared statement를 위해서는 새로운 프로토콜(Binary protocol)이 도입되었습니다. 현재 MRTE-Collector에서는 Binary protocol을 사용하는 경우는 지원하지 않고 있는데, 이는 패킷 분석의 어려움이 문제가 아니라 MRTE-Collector에서 PreparedStatement의 Hash Id를 알아낼 수 있는 방법이 없기 때문에 어려움이 있습니다.
만약 MySQL 서버에서 Connection별로 만들어진 PreparedStatement의 dump가 가능하다면 향후 Binary protocol도 지원이 가능할 것으로 보입니다.

하지만 다행스럽게도, JDBC Client에서 server prepared statement의 사용 여부를 명시적으로 활성화하지 않으면 기본적으로는 Client side prepared statement가 사용되고 아직 MySQL 서버에서는 PreparedStatement의 장점이 그다지 크지 않아서 대부분의 경우 Statement 또는 Client side prepared statement를 사용하고 있는 상태입니다.

MRTE 도구의 안정성과 성능 확인을 위해서 서버 4대를 아래와 같이 할당해서 테스트를 수행했습니다. 모두 2 소켓 12 코어 CPU를 사용하는 DELL 장비를 사용했습니다.

* A : Source MySQL Server + MRTE-Collector
* B : Rabbit Message Queue (+sysbench load generator)
* C : Target MySQL Server
* D : MRTE-Player

테스트는 대략 60개 정도의 Connection을 이용해서 초당 30000 QPS(22000 SELECT, 5000 UPDATE, 1600 INSERT, 1600 DELETE) 정도의 SQL을 처리하고 있었으며, MRTE-Collector와 MRTE-Player 모두 Internal queue가 평균 0~1개 정도만 쌓일 정도로 무리 없이 처리하고 있는 상태로 진행되었습니다. 아래 그래프는 테스트 도중 Source와 Target MySQL 서버의 Query activity를 보여주는 그래프입니다. (Source와 Target MySQL 서버 모두 그래프의 스파이크 현상은 MRTE와는 무관한 것임)

![MRTE 환경에서의 QPS](/files/mtre-qps.png)

이 테스트 환경으로 대략 3주 정도 계속 `sysbench` 트래픽을 Target MySQL 서버로 전송하는 테스트중에도 별다른 문제가 발생하지 않았으며, MRTE-Collector를 10분 단위로 종료했다가 재시작하는 테스트도 대략 1주일 정도 진행했었는데 특별히 문제 상황은 발생하지 않았습니다.

Rabbit MQ가 정상적으로 설치(모니터링 플러그인까지)되었다면 `http://rabbtmq_host:15672/` 웹 사이트를 이용해서 MRTE-Collector와 MRTE-Player가 정상적으로 통신을 하고 있는지 그리고 각각의 모듈들이 제대로 작동하고 있는지 바로 확인이 가능합니다.

MRTE-Collector와 MRTE-Player 소스 코드는 아래 Github 사이트에서 참조해볼 수 있습니다:

* [https://github.com/kakao/MRTE-Collector](https://github.com/kakao/MRTE-Collector)
* [https://github.com/kakao/MRTE-Player](https://github.com/kakao/MRTE-Player)

> 이 글은 카카오 DB팀의 기술 블로그 DB Smalltalk에 포스팅한 [MRTE를 이용한 MySQL Real Service 트래픽 테스트 환경 구축](http://small-dbtalk.blogspot.kr/2015/01/mrte-mysql-real-service.html)을 옮긴 것입니다.

* 커버 이미지 출처: [Bangkok Traffic](https://www.flickr.com/photos/fischerfotos/7457906740) &copy; [Mark Fischer](https://www.flickr.com/photos/fischerfotos/)