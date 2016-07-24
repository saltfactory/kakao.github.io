---
layout: post
title: '개발자를 위한 SSD (Coding for SSD) - Part 1 : 목차'
author: matt.lee
date: 2016-07-13 20:00
tags: [ssd,nand-flash,garbage-collection,LBA,PBA,block,page,clustered-block]
image: /files/covers/solid-state-logic.jpg
cover:
  title: '80-channel Solid State Logic (SSL) XL 9000 K Series Console at Audio Mix House, Studio B'
  link: https://flic.kr/p/j1DcB
  author:
    name: 'Audio Mix House'
    link: https://www.flickr.com/photos/audiomixhouse/
---

## Introduction

현재 개발중인 [Key/Value Store](http://codecapsule.com/2012/11/07/ikvs-implementing-a-key-value-store-table-of-contents/)가
SSD를 최적으로 사용하도록 하기 위해서는 SSD의 내부적인 특성이나 작동 방식에 대해서 정확한 이해가 필요했다.
인터넷에는 이미 많은 SSD 관련 자료들이 있지만, 대 부분은 부족하거나 잘못된 정보들이 많으며, 제대로 정리된 문서를 찾기는 쉽지 않았다.
결국 내 프로그램이 SSD를 최적으로 사용하도록 하기 위해서는 상당히 많은 문서들과 벤치마크 자료들을 찾고 살펴 봐야 했다.

내가 알게 된 사실들과 결론이, 다른 사람들에게도 많은 도움이 될 것이라는 생각을 하게 되었고 그래서 이미 온라인에 공개된 많은 정보들을 30 페이지 분량의 실용적인 지식을 담은 글을 쓰게 되었다.
단순히 블로그에 올릴 수 있는 분량이 아니어서, 좀 더 압축할 수 있는 수준으로 내용을 분류하여 5개의 챕터로 분류하였다.
전체 목차는 이 글의 아래에서 확인할 수 있다.

“Coding for SSDs”의 요약 정보를 담고 있는 Part 6은 가장 중요한 부분인데,
이는 아마도 급하게 SSD 관련 프로그램을 작성해야 하는 사람에게는 큰 도움이 될 것으로 보인다.
요약 챕터에는 SSD의 기본적인 정보뿐만 아니라, SSD로부터 최적의 방법으로 데이터를 읽고 쓰기 위한 패턴에 대해서도 언급하고 있다.

“Coding for SSDs”의 또다른 중요 포인트는 이 글의 내용이 내가 개발하고 있는 Key-Value 스토어 프로젝트와는 무관하기 때문에,
Key-Value 스토리지에 대한 별도의 선행 지식이 필요치 않다는 것이다.
아직 날짜가 정확히 결정된 것은 아니지만,
Key-Value 스토리지가 SSD의 장점을 최대한 발휘하기 위해서 어떻게 Hash table을 구현해야 하는지에 대한 포스트도 계획하고 있다.

안타깝게도 내가 추천하는 최적의 SSD 액세스 패턴을 증명해줄 별도의 프로그램을 작성하지는 않았다는 것은 후회스러운 부분이긴 하다.
하지만 그런 코드가 있었다면, 나는 시중에 출시된 수 많은 SSD 드라이브들에 대해서 성능 테스트를 진행했어야 할 것이다.
이는 시간적인 부분뿐만 아니라 경제적으로도 많은 어려움이 있었을 것이다. 나는 내가 찾은 자료들을 아주 꼼꼼하고 비판적으로 검토하면서 이 자료를 작성했다. 혹시 내가 추천하는 내용중 잘못되거나 이상한 부분이나 궁금한 부분이 있다면 코멘트를 남겨 주길 바란다.

마지막으로, 우측 상단의 Subscription 패널에 등록하면 Code Capsule에 게재되는 새로운 게시물에 대해서 E-mail을 수신을 할 수 있다.


## Table of Content

###### Part 1: 목차

###### Part 2: [SSD의 아키텍처와 벤치마킹](/2016/07/14/coding-for-ssd-part-2/)

* 1. SSD의 구조
    * 1.1. NAND 플래시 메모리 셀
    * 1.2. SSD의 구성
    * 1.3. SSD의 생산 공정
* 2. 벤치마킹과 성능 메트릭
    * 2.1. 기본 벤치마킹
    * 2.2. 프리 컨디셔닝 (Pre-conditioning)
    * 2.3. 워크로드와 메트릭

###### Part 3: [페이지 & 블록 & FTL(Flash Translation Layer)](/2016/07/15/coding-for-ssd-part-3/)

* 3. 기본 오퍼레이션
    * 3.1. 읽기 & 쓰기 & 삭제
    * 3.2. 쓰기 예제
    * 3.3. Write amplification
    * 3.4. Wear leveling
* 4. FTL (Flash Translation Layer)
    * 4.1. FTL의 필요성
    * 4.2. 논리적 블록 맵핑 (Logical block mapping)
    * 4.3. 업계 상황
    * 4.4. Garbage collection

###### Part 4: [고급 기능과 내부 병렬 처리](/2016/07/16/coding-for-ssd-part-4/)

* 5. 고급 기능
    * 5.1. TRIM
    * 5.2. Over-provisioning
    * 5.3. Secure Erase
    * 5.4. Native Command Queueing (NCQ)
    * 5.1. 전력 차단 보호 (Power-loss protection)
* 6. SSD의 내부 병렬 처리
    * 6.1. 제한된 I/O 버스 대역폭
    * 6.2. 병렬 처리
    * 6.3. Clustered blocks

###### Part 5: [접근 방법과 시스템 최적화](/2016/07/17/coding-for-ssd-part-5/)

* 7. 액세스 패턴
    * 7.1. 시퀀셜과 랜덤 I/O의 정의
    * 7.2. 쓰기
    * 7.3. 읽기
    * 7.4. 동시 읽고 쓰기
* 8. 시스템 최적화
    * 8.1. 파티션 얼라인먼트 (Partition alignment)
    * 8.2. 파일 시스템 파라미터
    * 8.3. 운영 체제의 I/O 스케줄러
    * 8.4. 스왑 (Swap)
    * 8.5. 임시 파일

###### Part 6: [요약 – 개발자가 SSD에 대해서 알아야 할 것들](/2016/07/18/coding-for-ssd-part-6/)


---

**This articles are translated to Korean with original author's([Emmanuel Goossaert](http://www.goossaert.com/)) permission.  
    Really appreciate his effort and sharing.**

**Original articles :**

* [Part 1: Introduction and Table of Contents](http://codecapsule.com/2014/02/12/coding-for-ssds-part-1-introduction-and-table-of-contents/)
* [Part 2: Architecture of an SSD and Benchmarking](http://codecapsule.com/2014/02/12/coding-for-ssds-part-2-architecture-of-an-ssd-and-benchmarking/)
* [Part 3: Pages, Blocks, and the Flash Translation Layer](http://codecapsule.com/2014/02/12/coding-for-ssds-part-3-pages-blocks-and-the-flash-translation-layer/)
* [Part 4: Advanced Functionalities and Internal Parallelism](http://codecapsule.com/2014/02/12/coding-for-ssds-part-4-advanced-functionalities-and-internal-parallelism/)
* [Part 5: Access Patterns and System Optimizations](http://codecapsule.com/2014/02/12/coding-for-ssds-part-5-access-patterns-and-system-optimizations/)
* [Part 6: A Summary – What every programmer should know about solid-state drives](http://codecapsule.com/2014/02/12/coding-for-ssds-part-6-a-summary-what-every-programmer-should-know-about-solid-state-drives/)
