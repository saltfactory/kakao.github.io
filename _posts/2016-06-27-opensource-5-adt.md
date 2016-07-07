---
layout: post
title: 'kakao의 오픈소스 Ep5 - Almighty Data Transmitter'
author: gordon.hahn
date: 2016-06-27 11:19
tags: [opensource,almighty-data-transmitter,adt,mysql,sharding,devops]
image: /files/covers/swiss-army-knife.jpg
---
<a id="forkme" href="https://github.com/kakao/adt"></a>

> "카카오의 오픈소스를 소개합니다" 두번째는 [gordon.hahn]()과 동료들이 개발한 **ADT - Almighty Data Trasmitter**입니다.
>
> [ADT](https://github.com/kakao/adt)는 샤드 구성이나 사딩 규칙이 바뀔 때 샤드를 재분배하는 용도로 만들기 시작했지만, MySQL에서 데이터를 수집하여 다른 MySQL로 데이터를 전송하는 - [CDC](https://en.wikipedia.org/wiki/Change_data_capture)와 [ETL](https://en.wikipedia.org/wiki/Extract,_transform,_load)이 결합된 - 만능 데이터 전송 도구로 변모하고 있습니다.
>
> ADT는 그 자체로도 유용한 소프트웨어 도구지만, MySQL 기반의 CDC/ETL 시스템을 구축하기 위한 좋은 시작점이 될 것 입니다.

## ADT는 무엇을 위한 툴인가요?

<img src="http://item-kr.talk.kakao.co.kr/do/-26p06+UqCd0OAgiRHNZwPf1+nqjcFZi42Z3wogPJ3I_/70d666e5313db413b3539edc1d0fc1ea1667fc7b08261b4c493670baa83d5cb9" class="pull-right" />
ADT는 MySQL의 데이터를 수집하여 사용자가 원하는 형태로 가공하거나 다른 DB에 적재할 수 있는 툴입니다. 크게 나누면 두 가지 용도가 있습니다.

- 1회성 마이그레이션 작업
- 실시간 마이그레이션 작업

각각에 대해 활용 예시는 다음과 같습니다.

* 1회성 마이그레이션
 - 샤드 데이터 재분배 혹은 샤드 룰 변경(Modulus를 Range로, 혹은 반대로 변경)
 - 완전히 새롭게 설계한 스키마로 데이터 복사
 - 1일 1회 MySQL의 데이터를 OLAP DB로 복사

* 실시간 마이그레이션
 - MySQL의 실시간 변경되는 데이터를 NoSQL로 복사하여 read 부하 분산
 - 어떤 특별한 write 이벤트를 감지해서 Push Notification 전송
 - 사용자의 GPS 정보를 기준으로 샤드 재구성 (가까이 있는 다른 사용자들 찾기에 편리하겠죠?)

이 외에도 여러 용도들이 있을 겁니다. 나머지는 여러분들의 상상력에 맡깁니다. 풀리퀘스트의 문은 활짝 열려있습니다 ^^;

## ADT는 어떤 기능들이 있나요?

ADT 자체가 하는 일은 단순합니다.

- MySQL에서 데이터 수집
 - Table Crawler: 각 테이블에 있는 데이터 수집
 - Binary Log Receiver: 실시간으로 변경되는 데이터 수집
- 수집한 데이터를 사용자가 구현한 Custom Handler로 전달

![ADT Overall Architecture](/files/adt-overall-arch.png)

<img src="http://item-kr.talk.kakao.co.kr/do/-26p06+UqCd0OAgiRHNZwPf1+nqjcFZi42Z3wogPJ3I_/553d0111a2757661a4c5bde97bdc88cb1667fc7b08261b4c493670baa83d5cb9" class="pull-right" />
용도에 따라 Custom Handler를 구현하는 게 허들이라면 허들일 수 있지만, [샤드 재분배용 커스텀 핸들러](https://github.com/kakao/adt/tree/master/adt-handler-parent/adt-handler-mysql-shard-rebalancer/src/main/java/com/kakao/adt/handler/msr)의 소스 코드를 참조하면 그렇게 어렵지 않....을 겁니다. 아마도...요.
(이 부분에 대해서는 다른 글을 통해서 더 자세히 알아보겠습니다)

현재는 MySQL에서만 데이터를 수집하지만, 인기가 많으면 더욱 다양한 DB가 추가될 수도 있습니다. 역시나! 풀리퀘스트의 문은 활짝 열려있습니다 ^^;

## 적용 사례

<img src="http://item-kr.talk.kakao.co.kr/do/-26p06+UqCd0OAgiRHNZwPf1+nqjcFZi42Z3wogPJ3I_/14b57ad97df0b8a45c6b521d175994121667fc7b08261b4c493670baa83d5cb9" class="pull-right" />
현재까지는 적용된 사례가 딱 하나 있습니다. 모 서비스의 테이블 스키마가 많이 변경되는 작업이었는데, 도저히 `ALTER TABLE`로 어떻게 할 수 있는 수준이 아니었다고 합니다. 테이블 스키마가 많이 바뀌어서 서비스 무정지 변경은 못했지만, ADT의 초기 버전을 이용해 무사히 마이그레이션 할 수 있었습니다.

ADT를 만들게 된 이유이자 핵심 목표인 **샤드 재분배용 커스텀 핸들러**는 현재 DB 엔지니어들과 안정성/정합성을 검증하는 단계입니다.

## 끝으로...

- ADT 특징에 관한 좀 더 자세한 내용
- 사용 방법
- 요구 사항
- 데이터 정합성에 관한 이야기

등등... 이 글에서 다루지 못한 많은 내용들이 [README](https://github.com/kakao/adt/blob/master/README.md)에 담겨있습니다. 현재까지 구현된 ADT 프레임웍 본체와 검증이 진행 중인 샤드 재분배용 커스텀 핸들러의 소스 코드는 아래 Github 사이트에서 확인할 수 있습니다:

* https://github.com/kakao/adt

[kakao 기술 블로그](http://tech.kakao.com)와 [위키](https://github.com/kakao/adt/wiki)을 통해서 ADT의 활용 사례를 소개할 예정입니다. ADT가 이름처럼 전지전능한 도구가 될 수 있도록 여러분들의 많은 관심과 참여를 기대합니다.

> special thanks to [성동찬](http://gywn.net) (한국카카오 카카오뱅크)

* 커버 이미지 출처: [Swiss Army Knife - 3](https://flic.kr/p/7vQc3w) &copy; [Tom Von Lahndorff](https://www.flickr.com/photos/tomvon/
* 데이터베이스 아이콘 출처: http://www.seaicons.com/database-icon/
