---
layout: post
title: 'kakao의 오픈소스 Ep3 - HBase Region Inspector'
author: jg.choi
date: 2016-03-11 12:55
tags: [opensource,hbase-region-inspector,hri,hbase,clojure]
image: /files/covers/offcuts.jpg
---
<a id="forkme" href="https://github.com/kakao/hbase-region-inspector"></a>

> "카카오의 오픈소스를 소개합니다" 세번째는 [jg.choi](https://github.com/junegunn)와 동료들이 개발한 **HBase Region Inspector**입니다.
>
> [HBase Region Inspector][hri]는 HBase의 여러 리젼에 분산된 데이터를 시각적으로 보여주는 실용적인 도구입니다.
>
> 카카오에서도 대규모 HBase 클러스터 운영에 큰 도움이 되고 있는 유용한 소프트웨어입니다. 특히 [Clojure][clj] 와 [React][react]으로 작성되어 Clojure를 공부하려는 개발자들에게 유용할 것입니다.

### HBase

카카오의 많은 서비스는 대용량의 데이터를 저장하고 서비스하기 위해 [Apache HBase][hbase] 를 사용하고 있습니다.

HBase 는 이미 잘 알려져 있으니 긴 설명이 필요하진 않을 것 같은데요. 간단히 한 문장으로 요약하자면 HBase 는 데이터를 여러 서버에 분산 저장하고 처리하는 Hadoop 기반의 분산 데이터 저장소입니다.

### Region: 데이터 분산의 단위

분산 데이터 저장소이니 당연히 데이터가 여러 서버에 잘 분산 되어 있어야
scalable 한 성능을 얻을 수 있죠. HBase 는 각 테이블 데이터를 여러 구간으로 나누는데 – RDBMS 의 range partitioning 과 동일한 방식 – 이렇게 나눈 각 구간의 데이터를 리젼 (region) 이라 부르고 분산의 기본 단위로 사용합니다. 다른 데이터 스토어에서는 유사한 개념을 partition, chunk 등의 명칭으로 부르기도 합니다.

![리젼의 개념](/files/hri-range-partition.png)

(이미지 출처: [HBase Architecture Analysis Part1(Logical Architecture)][arch])

HBase 클러스터 성능 관리의 핵심은 바로 이런 리젼들의 분산을 모니터링하고 관리하는 일입니다. 리젼들이 적절한 크기로, 또 적절한 범위로 나뉘어져 있는지, 모든 서버에 고르게 분산되어 있는지, 읽기나 쓰기가 몰린 리젼이 있지는 않은지, 특정 리젼 서버에 요청이 많은 리젼들이 다수 배치되어 있지는 않은지, 다양한 관점에서 살펴봐야 하죠.

그런데 안타깝게도 HBase 의 Web UI 를 통해서는 이러한 사항들을 효과적으로 파악하기가 힘듭니다. 각 리젼 서버는 다음과 같은 단순한 화면만을 제공하거든요.

![](/files/hri-metrics.png)

지표들을 visual 하게 보여주지 않고 숫자로만 보여주니 상대적인 크고 작음이 머리 속에 잘 그려지지 않습니다. 읽기/쓰기 요청 수도 누적 횟수만 보여줄 뿐, 실제로 중요한 초당 요청 횟수를 보여주지는 않아요.

가장 큰 문제는 모아서 볼 수 없다는 점입니다. HBase 클러스터를 구성하는 서버의 대수가 수십대, 수백대를 넘어가는 경우에, 그 모든 서버들의 페이지를 일일이 열어 가며 확인 할 수는 없는 노릇이죠. 클러스터 전체 리젼들의 상태를 한 눈에 조망할 수 있는 방법이 없었습니다.

### 오픈소스 툴은 없을까?

분명 비슷한 아쉬움을 느낀 사람들이 있었겠죠. [Hannibal][hannibal] 이라는 오픈소스 모니터링 툴이 있습니다. 다음과 같이 각 리젼 서버에 위치한 리젼들의 크기를 visual 하게 보여줍니다.

![Hannibal 서버 모니터링 화면](/files/hri-hannibal-servers.png)

![[Hannibal 리젼 모니터링 화면](/files/hri-hannibal-regions.png)

좋은 시도인 것은 분명하나 단순히 리젼의 크기 정보만을 보여줄 뿐이라 많은 통찰을 주지는 못합니다. 더 이상 [업데이트 되지 않고][hco] 있기도 합니다.

### 그래서 만든 [hbase-region-inspector][hri]

그래서 결국 [hbase-region-inspector][hri] 라는 (밋밋한 이름의) 툴을 만들기로 했습니다. 목표로 했던 것은 다음과 같습니다.

- 클러스터의 전체적인 상태를 한눈에 조망할 수 있을 것
- 다양한 지표를 확인할 수 있을 것
- 필요 시 리젼을 옮길 수 있을 것
- 설치하고 사용하기 쉬울 것

프로젝트는 빠른 개발을 위해 [Clojure][clj] 와 [React][react] 을 이용해 구현되었습니다. Clojure 에 관심이 많은 분이라면 왜 [Om][om] 이나 [Reagent][reagent] 를 사용하지 않았는지 궁금해 하실 지도 모르겠네요. 그건 단순히 필자가 JavaScript 개발자가 아닌데다 React 도 처음 써보는 입장이라 추상화 계층을 하나 더 두기는 부담스러워서 그랬던 것이죠.

### 사용법

hbase-region-inspector 는 [실행 가능한 바이너리 또는 JAR 의 형태로 배포][hrir]되고 있습니다. 사용하시는 클러스터에 맞는 바이너리 파일을 다운로드 하시고 실행 권한을 부여하신 후 (`chmod +x ...`), 다음과 같이 실행하시면 됩니다.

```sh
./hbase-region-inspector ZOOKEEPER_QUORUM HTTP_PORT [INTERVAL]
```

쉽죠?

### 실행 화면

지정한 포트를 브라우저에서 열어보시면 다음과 같은 화면을 보실 수 있습니다.

![](https://github.com/kakao/hbase-region-inspector/raw/master/screenshot/hbase-region-inspector.png)

각 리젼 서버별 리젼 분포와, 각 리젼의 크기, 누적 요청 수, 초당 요청 수, 스토어파일의 개수, 멤스토어의 크기 등 다양한 정보를 확인 할 수 있습니다.

구체적인 동작은 아래의 영상에서 확인하세요.

### Demo

<video autoplay loop muted controls="true" preload="metadata" crossorigin="anonymous" style="width:100%;">
            <source src="/files/hri.webm" type="video/webm">
            <source src="/files/hri.ogv" type="video/ogg">
            <source src="/files/hri.mp4" type="video/mp4">
</video>

### 활용 사례

사용하기 쉽다 보니 카카오에서는 여러 팀에서 이 툴을 사용해 HBase 상태를 모니터링하고 있습니다. 웹서버의 형태로 실행되므로, 현재 상태를 공유하기에도 편리하죠. 다양한 문제 상황에 대한 진단과 파악도 용이해졌는데요, 실제로 다음과 같은 사례들을 진단하는데 활용했습니다.

- 특정 리젼에 과도하게 많은 요청이 몰리고 있는 상황
- 리젼 개수 기준으로는 고르게 분산되었지만, 요청 횟수는 특정 서버에 쏠린 상황
- Rowkey 를 잘못 만들어 의도와는 다르게 특정 리젼의 사이즈만 커진 상황

### 한계

아직 몇 가지 아쉬운 점들이 있기는 합니다. hbase-region-inspector 는 현재 현황을 파악하는데는 도움이 되지만, 변화 이력을 추적하기에는 적합하지 않습니다 (3차원 그래프가 필요할까요?).

또한 리젼의 개수가 수만개를 넘어가는 대형 클러스터를 대상으로 실행하는 경우, 브라우저가 무척이나 힘들어 하는 모습을 보게 됩니다.

![](/files/hri-chrome.png)

(미안 크롬 ...)

### 오픈소스

자랑할 만한 수준의 코드는 아니지만 – 아니 사실 그 반대이지만 – 다른 HBase 사용자 분들께도 유용하리라는 판단에 오픈소스로 공개하게 되었습니다. 소스코드와 바이너리는 아래 링크에서 보실 수 있습니다.

- https://github.com/kakao/hbase-region-inspector
- https://github.com/kakao/hbase-region-inspector/releases

사용해 보시고 피드백 주세요. 저의 비루한 JavaScript 코드를 다듬어 주실 분도 기다려 봅니다.

[hbase]:    https://hbase.apache.org/
[arch]:     http://www.cyanny.com/2014/03/13/hbase-architecture-analysis-part1-logical-architecture/
[hannibal]: https://github.com/sentric/hannibal
[hco]:      https://github.com/sentric/hannibal/commits/next
[hri]:      https://github.com/kakao/hbase-region-inspector
[hrir]:     https://github.com/kakao/hbase-region-inspector/releases
[clj]:      http://clojure.org/
[cljs]:     https://github.com/clojure/clojurescript
[react]:    https://facebook.github.io/react/
[om]:       https://github.com/omcljs/om
[reagent]:  https://github.com/reagent-project/reagent

* 커버 이미지 출처: [Offcuts](https://flic.kr/p/s3Q7rD) &copy; [Andrew Gustar
](https://www.flickr.com/photos/andrewgustar/)