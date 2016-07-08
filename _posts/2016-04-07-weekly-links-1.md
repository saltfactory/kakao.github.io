---
layout: post
title: 'Weekly Links #1 - 2016년 4월 첫째주'
author: iolo.fitzowen
date: 2016-04-07 16:50
tags: [weekly,links]
image: /files/covers/links.jpg
---
**[Weekly Links](http://tech.kakao.com/tags/weekly/)**에서는 지난 한 주, 카카오의 기술 블로그 담당자가 구독하는 기술 뉴스레터들에서 "인간의 눈"으로 선별한 링크들을 짧은 코멘트와 함께 공유합니다.

> 포함된 뉴스레터 목록은 [awesome-tech-newsletters](https://github.com/kakao/awesome-tech-newletters)에서 확인하실 수 있습니다.

### 2016년 4월 첫째주 추천 링크

* **[Why we chose Akka for our Cloud Device solution](https://techblog.king.com/why-we-choose-akka-for-our-cloud-device-solution/)** - 사탕깨는 모바일 게임으로 유명한 [King](https://king.com)이 "Cloud Device Solution"을 만들기 위해 [Akka](http://akka.io/)를 도입한 과정.
* **[스프링 부트](http://projects.spring.io/spring-boot/)에서 [넷플릭스 오픈소스](http://netflix.github.io) 활용하기**
 - [1부 eureka](https://blog.de-swaef.eu/the-netflix-stack-using-spring-boot/)
 - [2부 hystrix](https://blog.de-swaef.eu/the-netflix-stack-using-spring-boot-part-2-hystrix/)
 - [3부 feign](https://blog.de-swaef.eu/the-netflix-stack-using-spring-boot-part-3-feign/)
 - 분위기로 봐선 몇 편 더 나올 듯~
* **[Building a lexer and parser with Scala's Parser Combinators](https://enear.github.io/2016/03/31/parser-combinators/)** - 스칼라의 표준 라이브러리 [Parser Combinators](https://github.com/scala/scala-parser-combinators)를 활용해서 [DSL](https://en.wikipedia.org/wiki/Domain-specific_language) 인터프리터 만들기. 표준 라이브러리가 이 정도라니... 스칼라 좀 짱인 듯~
* **[The decorator pattern in JavaScript using closures, monkey patching, prototypes, proxies and middleware](http://nickmeldrum.com/blog/decorators-in-javascript-using-monkey-patching-closures-prototypes-proxies-and-middleware)** - 자바스크립트의 클로저를 사용하는 데코레이터 패턴 5가지: 래퍼, 몽키 패치, 프로토타입 상속, 프록시, 미들웨어. 패턴도 공부하고 클로저도 공부하고~ 글이 꽤 깁니다. :S
* **[An Overview of Apache Stream Technologies](https://databaseline.wordpress.com/2016/03/12/an-overview-of-apache-streaming-technologies/)** - 아파치 재단의 다양한 빅데이터 스트리밍(?) 프로젝트들을 그림 한장으로 소개. Flume, NiFi, Apex, Kafka Streams, Spark Streaming, Storm(and Trident), Flink, Samza, Ignite, Beam... 개별 프로젝트 링크는 스킵~ 이름만 읽기도 힘드네요. ㅠㅠ
* **[10 Influential Women in Java, Scala and Everything in Between](http://blog.takipi.com/10-influential-women-in-java-scala-and-everything-in-between/)** - 자바, 스칼라 분야에 주목할만한 여성 개발자 10인. 요즘 [Takipi의 블로그](http://blog.takipi.com/)에 좋은 글을 많이 올라오네요.
* **[A comprehensive react-redux tutorial](http://spapas.github.io/2016/03/02/react-redux-tutorial/)** - 예제를 통해서 배우는 [react-redux](https://github.com/reactjs/react-redux) 완전 정복. 글이 굉장히 깁니다 :S
* **[The Way of the Gopher](https://medium.com/@theflapjack103/the-way-of-the-gopher-6693db15ae1f#.ghzy09pd6)** - [Node.js](https://nodejs.org)에서 [Go](https://golang.org)로 전향하게 된 이야기.
* **[d'Oh My Zsh](https://medium.freecodecamp.com/d-oh-my-zsh-af99ca54212c#.6x2rrlhkg)** - 개발자들이 가장 많이 쓰는 유닉스 셸 중의 하나인 [zsh](http://www.zsh.org), 그 인기의 원동력인 [oh-my-zsh](http://ohmyz.sh)의 개발자가 쓴 oh-my-zsh 연대기 "나는 어쩌다 괴물 OSS 프로젝트를 하게 되었나?". 끝까지 읽어보면, 여러분들도 오픈소스를 하고 싶어지실 겁니다. ;)
* **[A collective list of public JSON APIs for use in web development](https://github.com/toddmotto/public-apis)** - 분류도 잘되어있고, 인증 필요 여부도 표시되어있어서 매시업을 만들거나 학습/데모용으로 유용할 듯 하네요. 물론, 제대로된 오픈API 목록을 원하신다면... [ProgrammableWeb](http://www.programmableweb.com/apis/directory)!
* **[Once Again on TCP vs UDP](http://ithare.com/once-again-on-tcp-vs-udp/)** - 네트웍 개발자들의 오랜 논쟁 거리인 TCP vs UDP. 요약하면, "UDP가 빠르긴하지만 TCP에서 KEEPALIVE, NODELAY, OOB 등을 잘 쓰면 많이 따라잡을 수 있다". 유익한 내용이 참 많습니다. 강추!
* **[Application architectures with persistent storage](http://firstclassthoughts.co.uk/Articles/Design/ApplicationArchitecturesWithPersistentStorage.html)** - DB를 사용하는 애플리케이션의 아키텍쳐 유형. 애플리케이션 간에 DB 인스턴스를 공유하느냐, OLTP/OLAP용 DB 인스턴스을 분리하느냐, ... 등등. 길지 않은 글에 알찬 내용 가득. 잘 아시는 내용이라도 정리삼아 일어볼 만 합니다.
* **[Lwan Web Server](https://lwan.ws)** - 초경량, 비동기, 멀티쓰레드, 이벤트기반 웹 서버~ 흠... 좋은 말은 다 붙였네요. i7 랩탑으로 (I/O없이) 320000 r/s까지 받는다는 군요 @..@ 바닥부터 시작해서 3년 걸려서 만들었다는데...
* **[Why you should use Clojure for your next microservice](https://developer.atlassian.com/blog/2016/03/why-clojure/)** - [JIRA](https://www.atlassian.com/software/jira), [Confluence](https://www.atlassian.com/software/confluence)로 유명한 [아틀라시안](https://www.atlassian.com)에서 [Clojure](https://clojure.org)를 도입한 이야기. 글 내용도 내용이지만, 글에 링크 걸린 동영상들이 재미있네요: [Realtime Collaboration with Clojure](https://www.youtube.com/watch?v=3QR8meTrh5g), [Interactive programming Flappy Bird in ClojureScript](https://www.youtube.com/watch?v=KZjFVdU8VLI)
* **[Five Lesser-Known Ways to Hang Your Main Thread](http://blog.nimbledroid.com/2016/03/21/ways-to-hang-main-thread.html)** - 안드로이드의 메인쓰레드를 먹통으로 만드는 덜 알려진 5가지 방법(?). 글 자체만으로도 유용하지만, 사용된 프로파일링 서비스 [nimbledroid](https://nimbledroid.com)가 눈에 띄네요~ @..@
* **톰캣 성능 문제 완전 분석**
 - [1부 Database, Micro-Services and Frameworks](http://apmblog.dynatrace.com/2016/02/23/top-tomcat-performance-problems-database-micro-services-and-frameworks/)
 - [2부 Bad Coding, Inefficient Logging & Exceptions](http://apmblog.dynatrace.com/2016/03/08/top-tomcat-performance-problems-part-2-bad-coding-inefficient-logging-exceptions/)
 - [3부 Exceptions, Pools, Queues, Threads & Memory Leaks](http://apmblog.dynatrace.com/2016/03/23/top-tomcat-performance-problems-exceptions-pools-queues-threads-memory-leaks)
 - [dynatrace](http://www.dynatrace.com/)를 광고하기 위한 글이지만, 분석 결과 만으로도 유용할 듯.
* **[KakaoTalk speaks volumes about the future of cloud services](http://superuser.openstack.org/articles/kakaotalk-speaks-volumes-about-the-future-of-cloud-services)** - 지난 2월에 있었던 오픈스택데이 2016 Korea에서 [발표](http://www.slideshare.net/openstack_kr/openstack-days-korea-2016-track1-5000vm)했던 내용인데요, 조만간 카카오 기술 블로그를 통해서 자세히 소개할 예정입니다. ;) 외국인들에게 카카오라는 회사를 소개하면서 춤추는 아줌마 이모티콘을 예로 드는게... 재밌네요 ㅎ

### 특집: npm 게이트, 그것이 알고 싶다!

<img src="http://item-kr.talk.kakao.co.kr/do/2FPpx81E0V62RDSr-GVgQXaq4FJCveCBKCNZV-bZscw_/9e7a61ac86c673b1e6a5bbe2cde7ff791667fc7b08261b4c493670baa83d5cb9" class="pull-right" /> 개인적으로 이번 주에 가장 눈에 띄는 소식은 아무래도 npm게이트(a.k.a. leftpad 게이트)가 아닌가 싶습니다. 그래서, 창간 특집(?)으로 npm 게이트의 전말을 파헤쳐 보겠습니다.

다음은 문제의 `left-pad` 코드(전체!)입니다:
```js
module.exports = leftpad;
function leftpad (str, len, ch) {
  str = String(str);
  var i = -1;
  if (!ch && ch !== 0) ch = ' ';
  len = len - str.length;
  while (++i < len) {
    str = ch + str;
  }
  return str;
}
```

메신저 플랫폼 회사인 [kik](https://www.kik.com)이 [azer](https://github.com/azer)가 개발한 npm 모듈 kik의 이름을 바꾸라고 하면서 사건이 시작됩니다.

<img src="http://item-kr.talk.kakao.co.kr/do/2FPpx81E0V62RDSr-GVgQff1+nqjcFZi42Z3wogPJ3I_/117c2d408cabd9d2f8fdacc33a37de341667fc7b08261b4c493670baa83d5cb9" /> <img src="http://item-kr.talk.kakao.co.kr/do/2FPpx81E0V62RDSr-GVgQff1+nqjcFZi42Z3wogPJ3I_/23460135133c815fff292918047c3c871667fc7b08261b4c493670baa83d5cb9" />

* 3/11 10:20 kik -> azer: kik 모듈 이름의 소유권을 주장하며 azer에게 이름을 바꾸라고 요청.
* 3/11 10:50 azer -> kik: 거절.
* 3/11 11:26 kik -> azer: **변호사** 언급.
* 3/11 12:34 azer -> kik: 거절.
* 3/11 12:42 kik -> npm: 문제 해결 요청.
* 3/11 12:44 kik -> azer: 보상을 제안.
* 3/11 12:52 azer -> kik: **$30000** 제시.
* 3/11 12:57 kik -> npm: kik은 등록상표이므로 kik 모듈 이름의 소유권을 주장.
* 3/11 12:59 kik -> npm: azer가 약관을 위반하고 있다고 주장.
* ...
* 3/16 08:42 kik -> npm: **변호사** 언급.
* 3/18 16:39 npm -> kik & azer: kik 모듈의 소유권을 kik으로 넘김.
* 3/18 17:00 kik -> npm: 감솨~
* ...
* 3/20 14:22 azer -> npm & azer: 유감~ 모든 모듈 삭제 의사 표명.

kik의 무례한 요구, npm의 부적절한 중재, 그리고 azer의 성급한 행동(unpublish)이 결합되어 사건이 커집니다.

<img src="http://item-kr.talk.kakao.co.kr/do/2FPpx81E0V62RDSr-GVgQff1+nqjcFZi42Z3wogPJ3I_/5b193322ac18ab63a8d44a71d11576731667fc7b08261b4c493670baa83d5cb9" /> <img src="http://item-kr.talk.kakao.co.kr/do/2FPpx81E0V62RDSr-GVgQff1+nqjcFZi42Z3wogPJ3I_/8b3db1f5a29e64003a8d8534462bdcf01667fc7b08261b4c493670baa83d5cb9" /> <img src="http://item-kr.talk.kakao.co.kr/do/2FPpx81E0V62RDSr-GVgQff1+nqjcFZi42Z3wogPJ3I_/ea9e258469b3741e398b7b6216548e961667fc7b08261b4c493670baa83d5cb9" />

* ...(시끌벅적)
* 3/23 [I’ve Just Liberated My Modules](https://medium.com/@azerbike/i-ve-just-liberated-my-modules-9045c06be67c#.jickcoe6v) - azer가 "본인 소유의 npm 모듈 273개를 내렸다"고 통보. 그 중에 문제의 [left-pad](https://www.npmjs.com/package/left-pad)가 포함되어 있었음.
* 3/23 가장 인기있는 npm 모듈 중의 하나인 [babel](https://babeljs.io)과 [atom](https://atom.io)이 간접적으로 [left-pad](https://www.npmjs.com/package/left-pad)에 의존성이 걸려 있었음. 수많은 ES6 기반 어플리케이션들이 빌드 안됨. **2.5 시간의 재앙!!**
* 3/23 [kik, left-pad, and npm](http://blog.npmjs.org/post/141577284765/kik-left-pad-and-npm) - npm 측의 해명: "우리는 azer의 코드를 훔치지 않았다", "kik은 좋은 회사다"...
* 3/23 [A discussion about the breaking of the Internet](https://medium.com/@mproberts/a-discussion-about-the-breaking-of-the-internet-3d4d2a83aa4d#.7hvnswqn3) - kik 측의 해명: "우리는 잘못한거 없다". 자신만만하게 주고받은 메일을 다 공개했으나...

<img src="http://item-kr.talk.kakao.co.kr/do/2FPpx81E0V62RDSr-GVgQXaq4FJCveCBKCNZV-bZscw_/477c52636630bc15b2890bde099cba0a1667fc7b08261b4c493670baa83d5cb9" /> <img src="http://item-kr.talk.kakao.co.kr/do/2FPpx81E0V62RDSr-GVgQff1+nqjcFZi42Z3wogPJ3I_/d9a75303a2c93be4824eb01e7b52b8ef1667fc7b08261b4c493670baa83d5cb9" /> <img src="http://item-kr.talk.kakao.co.kr/do/2FPpx81E0V62RDSr-GVgQff1+nqjcFZi42Z3wogPJ3I_/30fb689b65179f92d9264471424237511667fc7b08261b4c493670baa83d5cb9" />

* 3/25 [npmGate — Lessons Learned Again](https://dzone.com/articles/npmgate-lessons-learned-again) - 변호사를 통해서 으름장을 놓은 kik도 비난하고, 큰 저장소를 운영하면서도 제대로 된 정책도 없고, 제대로 대응도 못한 npm도 비난하고, 그 와중에 은근슬쩍 [Sonatype](http://www.sonatype.com)의 [Nexus](http://www.sonatype.com/nexus-repository-sonatype) 광고...ㅎ
* 3/29 [changes to npm’s unpublish policy](http://blog.npmjs.org/post/141905368000/changes-to-npms-unpublish-policy): npm이 후속 조치로 unpublish 정책을 변경. 요약하면, 24시간내에는 본인이 unpublish 가능. 그 이후에는 npm에 메일로 연락. 패키지의 모든 버전이 내려가면 땜빵(placeholder) 패키지로 대체. 땜빵 패키지의 소유권을 획득하려면 npm에 메일로 연락.
* 4/4 [11줄의 코드, 인터넷을 패닉에 빠뜨리다](http://www.bloter.net/archives/253447)

> 오픈소스를 생산(기여)하는 측, 사용하는 측, 그리고 이 과정을 중개하는 측이 외부의 도전(?)에 대처하는 방법에 대해서 많은 생각을 하게 만든 사건입니다만... (흠흠) 자세한 설명은 프렌즈의 표정으로 대신합니다.

기술 블로그의 컨텐츠 수급에 어려움 때문에 시작하긴 했는데, 수십개의 뉴스레터에 소개된 수백개의 링크 중에서 몇 개만 선정하는 일이 만만치 않네요. Weekly Links는 쭈욱~~

> 포함된 뉴스레터 목록은 [awesome-tech-newsletters](https://github.com/kakao/awesome-tech-newletters)에서 확인하실 수 있습니다.

* 커버 이미지 출처: [link by link...](https://flic.kr/p/KjJMP) &copy; [Carsten Tolkmit](https://www.flickr.com/photos/laenulfean/)
