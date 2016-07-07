---
layout: post
title: 'kakao의 오픈소스 Ep1 - 대용량 분산 그래프DB "S2Graph"'
author: shon.0
date: 2016-01-29 13:11
tags: [opensource,s2graph,graphdb,hbase,scala]
image: /files/covers/neuron.jpg
---
<a id="forkme" href="https://github.com/kakao/s2graph"></a>

> "카카오의 오픈소스를 소개합니다" 첫번째는 [shon.0](https://github.com/SteamShon)와 동료들이 개발한 [S2Graph]입니다.
>
> S2Graph는 카카오에서 1년 여의 개발을 거쳐 카카오톡, 카카오스토리, 카카오뮤직, 선물하기, 다음앱, 다음뉴스, 다음쇼핑 등 20여개 이상의 서비스에 적용된 **대용량 분산 그래프 데이터베이스**입니다.
>
> [스칼라] 언어와 [Play 프레임웍]으로 작성된 그래프 API 서버와 [HBase], [Kafka], [Spark] 등 최근 가장 주목받는 기술들로 구성되어, 호기심으로 똘똘뭉친 개발자들에게 많은 도움이 될 것입니다.

<!--more-->

## 그래프 데이터베이스

"그래프"라고 하면, 보고서나 발표자료에 막대 그래프, 파이 챠트,...를 떠올리지만, 이 글에서 언급하는 그래프는 수학자 오일러에서 시작된 [그래프 이론]의 그래프입니다.

![그래프 이론의 시초 - 쾨니히스베르그 다리 문제](/files/s2graph-konigsberg-bridges.png)

그래프 이론은 "유한 개의 점들로 이루어진 집합과 점들 간의 연결(관계)를 다루는" 학문입니다. 예를 들면, "어떤 지역과 지역을 최단거리로 이동하려면 어떻게 해야 되는가? 어떤 지점들이 있는데 이 지점들을 중복으로 지나지 않고 한번에 이동할 수 있는가?" 같은 문제를 연구하는 거죠.

이런 "그래프 구조"를 저장하고 표현하기 위해 만들어진 도구가 [그래프 데이터베이스] (graph database; 이하 그래프DB)입니다.

## 소셜 그래프

카카오의 많은 서비스들은 사용자 사이의 관계(relation)와 사용자 개인의 활동(activity)를 기반으로 합니다.

> 예) 덕선이 선우와 택이는 서로 친구입니다. 덕선이는 카카오 뮤직에서 음악을 듣습니다. 택이는 바둑 게임을 즐깁니다. 보라와 선우는 카카오 스토리를 사용합니다:

이러한 문장들을 단어 그대로 그림으로 표현하면 아래와 같은 "소셜 그래프"가 됩니다:

![소셜 그래프 예](/files/s2graph-sample-1.jpeg)

위의 그림에서 선(edge)으로 표현된 관계와 활동에 속성(property)을 추가하면 더 의미있는 정보를 표현할 수 있습니다.

> 예) 덕선이와 보라는 가족(보라가 덕선이의 언니)입니다. 덕선이는 이승환의 광팬입니다(play count). 택이는 카카오 바둑 만렙입니다(play level). 선우는 보라가 카카오 스토리에 올린 글을 좋아합니다.

실제 카카오의 소셜 그래프도 규모와 속성이 다를 뿐, 아래의 그림과 크게 다르지 않습니다:

![속성이 추가된 소셜 그래프 예](/files/s2graph-sample-2.jpeg)

이런 그래프가 저장되어 있고, 필요할 때 즉시 조회할 수 있다면, 우리는 사용자들을 위해 더 많은 일들을 할 수 있습니다.

> 예) 덕선이가 즐겨듣는 이승환의 음악을 택이에게 추천합니다. 선우의 타임라인에 보라의 글을 바로 보여줍니다. 보라에게 선우를 친구로 추천합니다.

## S2Graph: 기술적 도전

카카오는 S2Graph를 개발하기 전까지 애플리케이션 레벨의 수동 샤딩(sharding)된 MySQL를 사용해 왔습니다. 그러나, 계속 늘어나는 서비스와 데이터량을 코드와 설정과 운영으로 극복하는 것은 말처럼 간단한 일이 아니었습니다. 물리적인 한계로 인해 포기할 수 밖에 없는 데이터들도 계속 늘어났습니다.

카카오도 (페이스북과 트위터가 그랬던 것처럼) 근본적인 해결책으로 2014년부터 그래프 데이터베이스 도입을 검토하기 시작했습니다. 오랜 시간동안 다양한 그래프 데이베이스를 검토하고 고치고 시험했습니다. 그리고, (페이스북과 트위터가 그랬던 것처럼) 기존의 그래프 데이터베이스로 해결할 수 없는 기술적 도전에 직면했습니다.

### 규모: 지속적으로 변하는 대규모 소셜 그래프

- 2억 명의 사용자(vertex; node; object)들이
- 100억 건의 관계(edge; link; relation)를 맺고 있으며
- 매일 5000만 건이 넘는 관계가 변하고
- 매일 30억 건이 넘는 활동(activity)이 추가되고 있습니다

RDB를 사용하면 2억 행을 가진 사용자 테이블과 20억 행을 가진 관계 테이블(m:n)과 매일 1000만 행이 늘어나는 활동 테이블이 필요합니다. 32비트 정수의 최대 범위가 21억이므로, 기본 키(primary key; PK)와 외래 키(foreign key; FK)도 64비트 정수가 되어야겠죠. 이 숫자가 주는 압박감은...

기존의 [그래프 데이터베이스]들은 **미리 확보된 정적인 데이터에 대한 의미 추론**을 목적으로 만들어져서 극단적인 규모의 데이터량과 빈번한 데이터 변경에는 상대적으로 취약합니다. 애초에 그러라고(?) 만든 물건이 아니었던 거죠.

### 성능: 실시간으로 연결된 데이터에 대한 [너비 우선 탐색]

 - 피크 타임에 초당 65,000 쿼리
 - 최대 응답 시간 50ms

RDB로 [너비 우선 탐색]을 구현하려면 대규모의 "JOIN"과 "GROUP BY"가 불가피합니다. 시간 내에 한 개의 응답을 내는 - OLAP에서 OLTP의 성능을 뽑아내는 - 것만 해도 쉽지 않은 일인데, 동시에 2000개의 응답을 내야 합니다.  카카오가 그랬던 것처럼 규모의 한계를 극복하기 위해 수동 샤딩을 했다면 쿼리도 상당 부분 수동으로 해야 합니다.

기존의 그래프DB들은 **전체 데이터를 대상으로 하는 [깊이 우선 탐색] (Depth First Search; DFS)**에 최적화되어 있고, **부분 데이터를 대상으로 하는 [너비 우선 탐색] (breadth First Search; BFS)**에는 상대적으로 취약합니다. 카카오를 비롯한 대부분의 소셜 서비스에게 필요한 것은 몇시간 걸려서 나오는 수학적인 최단 경로(Shortest Path)가 아닙니다.

또한, 서비스 트래픽가 폭증하더라도 서버 규모를 선형 확장(linear scalability)해서 대응할 수 있어야 합니다.

### 바이럴 효과를 위한 실시간 업데이트

소셜과 모바일의 결합이 가져온 가장 큰 변화가 "실시간성"입니다.

매일 혹은 매시간 배치(batch)로 데이터를 분석하고, 이렇게 분석된 정보를 기반으로 추천해서는 사용자들의 소비 속도에 맞출 수 없습니다. 지금 이 시간 사람들이 많이 본 뉴스를 내일 추천하면 "철지난 핫(?) 뉴스"가 됩니다. 그 뉴스를 톡으로 공유하면...

진짜 실시간(hard real-time)은 아니더라도 거의 실시간(soft real-time) 처리가 가능해야 유혈사태(?)를 막을 수 있을 것입니다.

![바이럴 효과를 위한 실시간 업데이트](/files/s2graph-realtime.png)

### 동적 랭킹 로직 지원

 - 푸시(push) 전략: 랭킹 로직을 동적으로 변경하기 어렵습니다.
 - 풀(pull) 전략: 다양한 랭킹 로직을 적용할 수 있습니다.

아시는 바와 같이 페이스북이나 트위터에서는 사용자마다 컨텐츠의 내용와 순서가 다릅니다. 카카오 스토리도 마찬가지입니다. 과거에는 "시간순"이라는 의미로 "타임라인"이라는 단어를 사용했지만, 최근에는 조작된 타임라인, 즉, 사용자 맞춤형 "피드"라는 의미로 사용됩니다.

트위터는 피드를 구현하기 위해서 푸시 방식을 사용합니다. 이 방식은 사용자가 컨텐츠를 생산할 때(tweet, retweet, follow/unfollow...) 그 컨텐츠를 소비할 사용자(팔로어)들의 피드에 해당 컨텐츠를 추가합니다. 이런 특성 때문에 write fanout 방식이라고도 합니다. 푸시 방식은 컨텐츠 소비(타임라인을 읽을 때) 처리가 간편하고 빠르다는 장점이 있지만, 피드의 내용과 순서를 변경하기 어렵다는 문제점이 있습니다. 불필요한 데이터 추가(트위터를 접은 팔로어의 피드에도 새 트윗을 추가...)로 인한 자원 낭비도 큰 문제입니다.

![기본적인 푸시 방식](/files/s2graph-push.png)

반면, 페이스북은 풀 방식을 사용합니다. 이 방식은 사용자들이 볼 때 컨텐츠의 내용과 순서를 결정합니다. 이런 특성 때문에 read fanout 방식이라고도 합니다. 풀 방식은 컨텐츠 생산(post, like, friend/unfriend...) 처리가 간편하고 빠르다는 장점이 있지만, 컨텐츠 소비 처리가 어렵고 느립니다. 그러나, 피드의 내용과 순서를 변경할 수 있어서 최근에는 더 널리 사용되고 있습니다.

![기본적인 풀 방식](/files/s2graph-pull.png)

가능하다면 풀 방식이 좋겠지만, 서비스의 특성에 따라 푸시 방식이 불가피한 경우도 있습니다. 실제 서비스에서는 풀과 푸시, 둘 다 필요합니다.

### S2Graph: Before & After

S2Graph를 적용하기 전, 수동 샤딩과 상호 연결로 얽히고설켜 확장성 없던 아키텍쳐가:

![S2Graph 적용 전](/files/s2graph-before.png)

S2Graph를 적용한 후, 깔끔하고 무한 확장가능한 아키텍쳐가 되었습니다:

![S2Graph 적용 후](/files/s2graph-after.png)

S2Graph를 처음 공개할 당시만 해도 매일 10억 건 정도의 활동이 추가되었지만, 최근엔 적용한 서비스가 늘면서 매일 30 억 건 정도의 활동이 추가되고 있습니다. 그럼에도 불구하고, 코어의 지속적인 개선을 통해 피크 타임 QPS는 20,000에서 65,000으로, 최대 응답 시간은 100ms에서 50ms이하로 더욱 빨라졌습니다.

## What is S2Graph?

S2Graph를 한 문장으로 표현하면:

> ### Storage-as-a-Service + Graph API = Realtime Breadth First Search

즉, 실시간 [너비 우선 탐색]을 위한 **그래프 데이터 저장소**와 **그래프 API 서버**입니다.

![S2Graph의 아키텍쳐](/files/s2graph-overall-arch.png)

[HBase]를 그래프 데이터 저장소로  사용합니다. HBase/HDFS/Hadoop이 가진 성능, 확장성, 가용성을 그대로 흡수하면서, 실시간 [너비 우선 탐색]에 최적화된 형태로 그래프 데이터를 저장합니다. 저장소 레이어는 물리적 구현체에 독립적으로 설계되어 MySQL을 저장소로 사용할 수도 있습니다.

[스칼라]와 [Play 프레임웍]로 구현된 선형 확장가능한 그래프 API 서버를 포함하고 있습니다. 최근에는 [Netty] 기반의 고성능 REST 서버도 실험적으로 개발되고 있습니다.

그 밖에도 [Kafka]와 [Spark] 기반의 배치 처리, Top카운터, A/B 테스트 등을 포함한 소셜 그래프 기반 서비스에 유용한 여러가지 기능을 포함하고 있습니다.

단일 머신에 설치하고 실행할 수 있으므로 개발과 테스트가 용이합니다. [Vagrant]와 [VirtualBox] 등이 설치되어 있다면, 아래의 간단한 명령 몇 개 만으로 S2Graph를 체험할 수 있습니다:

```
$ git clone https://github.com/kakao/s2graph.git
$ cd s2graph
$ vagrant up
$ vagrant ssh
# 가상 머신에서...
$ cd s2graph
$ activator run
```

## S2Graph의 새 이름: Apache S2Graph!

지난해 11월 [아파치 재단의 인큐베이터 프로젝트](http://incubator.apache.org/projects/s2graph.html)로 선정되었습니다([관련 소식](http://www.kakaocorp.com/pr/pressRelease_view?page=5&group=1&idx=8425)).

아파치 인큐베이터 프로그램에 맞춰 현재 소스 코드와 이슈 트래커의 이전, 빌드, 문서화 등의 작업이 한창 진행 중입니다. 카카오에서도 다수의 개발자들이 풀타임으로 S2Graph 개발에 전념하고 있지만, 이 글을 읽는 분들도 언제든지 참여할 수 있습니다.

## 나가는 글

이 글은 그래프 데이터베이스가 생소한 분들에게 S2Graph를 소개하기 위해 의도적으로 기술적인 세부사항을 생략했지만, 앞으로는 [이 곳, kakao 기술 블로그](http://tech.kakao.com)와 [브런치의 Apache S2Graph 매거진](https://brunch.co.kr/magazine/apaches2graph) 등을 통해 **S2Graph의 활용과 내부 구조**를 포함한 다양한 기술 자료를 지속적으로 공유하겠습니다. S2Graph가 더 좋은 오픈소스 소프트웨어가 될 수 있도록 여러분들의 지속적인 관심과 참여를 기대합니다.

## 참고자료

* S2Graph [공식 소스 저장소 & 이슈 트래커](https://github.com/kakao/s2graph)
* S2Graph [공식 문서](https://steamshon.gitbooks.io/s2graph-book/content/)
* S2Graph [공식 사용자 포럼(메일링)](https://groups.google.com/forum/#!forum/s2graph)
* S2Graph [아파치 인큐베이터 프로젝트 홈페이지](http://incubator.apache.org/projects/s2graph.html)
* S2Graph 개발팀의 [Apache S2Graph 블로그/웹진](https://brunch.co.kr/magazine/apaches2graph)
* Apache Big Data 2015 [발표 자료](http://apachebigdata2015.sched.org/event/de6abfbd8f0b9e66b1c03feb2b9e2078)
* DEVIEW 2015 [발표 자료](http://www.slideshare.net/deview/263-s2graph-largescalegraphdatabasewithhbase2) & [발표 동영상](http://serviceapi.rmcnmv.naver.com/flash/outKeyPlayer.nhn?vid=476077CFB274F31E3A3C9017F65C3DC26D30&outKey=V125fb0a32f63f576621e1b6811099841c5e088c1a1d90e05bef21b6811099841c5e0&controlBarMovable=true&jsCallable=true&skinName=tvcast_white)
* HBaseCon 2015 [발표 자료](http://www.slideshare.net/HBaseCon/use-cases-session-5) & [발표 동영상](https://vimeo.com/128203919)

* 커버 이미지 출처: [Articles A Inteligência Humana, por Ozires Silva](http://www.blogdoozires.com.br/ozires/tag/inteligencia/) &copy; Divulgação

[S2Graph]:https://github.com/kakao/s2graph
[그래프 이론]:https://en.wikipedia.org/wiki/Graph_theory
[그래프 데이터베이스]:https://en.wikipedia.org/wiki/Graph_database
[깊이 우선 탐색]:https://en.wikipedia.org/wiki/Depth-first_search
[너비 우선 탐색]:https://en.wikipedia.org/wiki/Breadth-first_search
[Scala]:http://www.scala-lang.org
[Play 프레임웍]:https://www.playframework.com
[HBase]:https://hbase.apache.org
[Kafka]:http://kafka.apache.org
[Spark]:http://spark.apache.org
[Netty]:http://netty.io
[Vagrant]:https://www.vagrantup.com
[VirtualBox]:https://www.virtualbox.org
