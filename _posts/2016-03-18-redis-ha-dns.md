---
layout: post
title: 'DNS 기반의 Redis HA 구현'
author: alden.kang
date: 2016-03-18 13:32
tags: [devops,redis,ha,dns]
image: /files/covers/railway-switch.jpg
---
이번 글에서는 DNS 기반의 Redis HA에 대한 이야기를 해보려고 합니다.
DNS TTL이 무엇인지 그리고 그것을 어떻게 이용해서 Redis HA를 구현했는지 살펴보겠습니다.

## 도메인의 TTL

먼저 도메인의 **TTL**에 대해 이야기해 보겠습니다.
**TTL**은 **Time To Live**의 약자로 도메인을 캐싱하고 있는 시간을 의미합니다.
누군가 `A`라는 도메인에 대해 질의를 했다면 응답을 준 DNS 서버에서 해당 도메인에 대해 TTL 시간 동안 캐싱을 하고 있게 됩니다.
이렇게 동작하는 이유는 **같은 도메인을 여러 사람이 질의할 수 있기 때문에 한 번 질의한 내용을 캐싱하고 있는 게 불필요한 동작을 방지할 수 있기 때문입니다**.

리눅스에서는 간단하게 `dig` 명령으로 `TTL` 시간을 알 수 있습니다.

![kakao.com에 대한 도메인 질의 결과](/files/redis-ha-dig-kakao.png)

위 스크린샷은 `www.kakao.com`에 대한 도메인 질의 결과입니다.

* **1번 항목은 TTL 시간을 의미합니다**.
 질의한 도메인에 대해 DNS 서버가 해당 도메인을 캐싱하는 시간입니다. 531초 동안 캐싱하고 있을 거라는 것을 알 수 있습니다.
* **2번 항목은 `www.kakao.com`에 대한 도메인 질의 결과를 내려준 서버를 의미합니다.**
 모든 DNS 서버가 모든 도메인에 대해 알고 있을 수 없기 때문에 본인이 관리하는 도메인이 아니라면 상위 DNS 서버에 질의하게 됩니다. 이런 방식을 거처 `www.kakao.com` 도메인을 관리하는 서버를 찾게 되고 그 서버가 `ns2.iwilab.com` 임을 알 수 있습니다.
* **3번 항목은 현재 자신이 사용하고 있는 DNS 서버를 의미합니다.**
 이를 통해 도메인 질의 과정을 유추해 보면 `www.kakao.com`에 대한 도메인 질의를 `10.20.30.60`에 먼저 질의하게 됩니다. `10.20.30.60`은 자신이 관리하는 도메인이 아니기 때문에 상위 DNS 서버에 요청해서 해당 도메인을 관리하는 DNS 서버를 찾게 되고 결과적으로 `ns2.iwilab.com` 서버에 질의해서 결과를 받아오게 됩니다.

## TTL이 0이 되면?

그렇다면 한 가지 가정을 해보겠습니다. TTL이 `0`이 되면 어떤 일이 일어날까요? DNS 서버는 해당 도메인에 대한 질의 요청을 사용자에게 해 주었지만 TTL이 0이 되기 때문에 캐싱하지 않고 바로 버립니다. 그럼 다른 사용자가 똑같은 도메인을 요청했을 경우 다시 한 번 상위 DNS 서버를 찾아서 해당 도메인에 대한 질의 결과를 받아오는 작업을 해야 합니다. 이렇게 되면 상위 DNS로의 요청이 많아지고 전체적으로 도메인 질의 요청을 처리하는 데에 많은 시간이 소요되게 됩니다. 하지만 매 요청마다 해당 도메인을 관리하고 있는 DNS 서버를 직접 찾아서 질의하기 때문에 항상 최신의 정확한 정보를 가지고 있게 됩니다.

> 도메인 요청에 걸리는 시간은 늘어나지만 도메인 변경 등에 대한 대응은 정확하고 빠르게 할 수 있다는 장점이 있습니다.

Redis HA에서도 이런 방식을 사용해보려고 합니다.

## 동작 원리

![Redis HA 동작 원리동작](/files/redis-ha-dns.png)

원리는 이렇습니다. master 역할을 하는 Redis 서버에 대표 도메인을 설정합니다. 예를 들어 A라는 서비스에서 사용할 Redis라고 한다면 `A.redis.domain.com`과 같은 방식입니다. 모니터링 서버에서는 master 서버에 대한 connect 및 간단한 `GET/SET` 테스트를 해서 살아 있음을 확인합니다. 이렇게 모니터링하다가 master 서버에 이상이 생기게 된다면 `A.redis.domain.com` 에 바인딩되어 있던 IP를 slave 역할을 하는 Redis 서버로 바꾸게 됩니다. 클라이언트에서는 A.redis.domain.com이라는 도메인을 통해서 Redis 서버에 접속을 하기 때문에 master 서버가 죽는 순간 잠시 단절이 있긴 하겠지만 도메인에 매핑된 IP 주소가 slave 서버로 바뀌기 때문에 금방 다시 커넥션을 맺어서 Redis를 사용할 수 있습니다. 이것도 TTL이 0이기 때문에 클라이언트가 바뀐 IP 주소로 바로 붙을 수 있게 되는 원리입니다.

Redis 명령어까지 포함된 세부 로직은 아래와 같습니다.

1. (slave) `config set slave-read-only no`
2. 도메인 변경
3. (old master) `CLIENT KILL TYPE normal`
4. (new master) `slaveof no one`
5. (new master) `CONFIG REWRITE`

3번의 경우 master 서버가 connect는 되고 `GET/SET` 같은 연산이 안될 경우를 대비해서 필요한 로직 입니다. 이를 통해서 확실하게 연결을 끊어줘야 클라이언트들이 새로운 마스터에 붙을 수 있습니다.

5번의 경우 slave 서버가 master 서버가 되었지만 `redis.conf`에는 여전히 slave 설정이 되어 있는 것이 오해를 불러 일으킬 수 있기 때문에 승격된 후 config 파일을 다시 만들도록 하고 있습니다.

## 주의할 점

하지만 몇 가지 주의할 점이 있습니다.

**첫 번째로는 TTL이 0으로 설정됨으로 인해 발생할 수 있는 DNS 서버의 부하 입니다.** 이 경우 Redis HA를 구현하는 데에 사용되는 도메인을 별도의 전용 DNS 서버로 분리하는 것으로 해결할 수 있습니다. 또한, **실제 도메인 질의는 connect 할 때만 발생하기 때문에** 클라이언트에서 connection pool 방식으로 구현한다면 평상시에 도메인 질의가 많아질 이유는 없습니다.

다만 이 경우도 클라이언트에서 잘못 구현해서 연산 할 때 마다 connection을 맺는다면 문제가 될 소지가 있습니다.

두 번째는 Java 기반으로 개발할 때 JVM이 도메인 캐싱을 하지 않도록 아래와 같이 옵션을 주어야 합니다.
```
Dsun.net.inetaddr.ttl=0
```
이 옵션을 주지 않으면 **JVM이 도메인을 캐싱하기 때문에 (TTL과 상관없이) 도메인이 바뀌어도 여전히 마스터 쪽으로 붙을 수 있습니다.**

**세 번째 경우는 [twemproxy](https://github.com/twitter/twemproxy)를 사용한다면 버전을 [0.4.1](https://github.com/twitter/twemproxy/releases/tag/v0.4.1)로 써야 하는 이슈가 있습니다.** 그 전 버전까지는 twemproxy 가 한번 socket resolve를 하고 나면 해당 IP를 기록해두고 재 접속을 위해서 그걸 이용하기 때문입니다. 이게 0.4.1 에서 기능이 들어갔다고 합니다. ([강대명](https://charsyam.wordpress.com) 님이 소중한 지식을 공유해 주셨습니다. ^^)

## 마치며

도메인의 TTL 값을 활용한 Redis HA에 대해 설명했는데요,

> TTL 값을 잘만 이용하면 Redis 외에 다른 많은 솔루션들에도 HA를 적용할 수 있습니다.

오늘 글이 읽으시는 분들에게 도움이 되었으면 합니다.

> 이 글은 카카오에서 서비스 인프라 시스템을 담당하고 있는 [aden.kang](https://brunch.co.kr/@alden)이 [브런치에 올린 글](https://brunch.co.kr/@alden/23)을 저자의 동의를 얻어 다시 게재한 것입니다. 저자가 브런치에서 운영하는 [All about Linux 매거진](https://brunch.co.kr/magazine/linux)을 통해 리눅스에 대한 풍부한 꿀팁들을 보실 수 있습니다.

* 커버 이미지 출처: [Railway Switch](https://flic.kr/p/nYDooo) &copy; [Rainer Stropek](https://www.flickr.com/photos/rainerstropek/)