---
layout: post
title: '개발자를 위한 SSD (Coding for SSD) - Part 4 : 고급 기능과 내부 병렬 처리'
author: matt.lee
date: 2016-07-16 20:00
tags: [ssd,nand-flash,garbage-collection,LBA,PBA,block,page,clustered-block]
image: /files/covers/solid-state-logic.jpg
cover:
  title: '80-channel Solid State Logic (SSL) XL 9000 K Series Console at Audio Mix House, Studio B'
  link: https://flic.kr/p/j1DcB
  author:
    name: 'Audio Mix House'
    link: https://www.flickr.com/photos/audiomixhouse/
---


이번 챕터에서는 SSD의 주요 기능인 TRIM과 Over-provisioning에 대해서 간단히 살펴보도록 하겠다.
또한 SSD의 내부 병렬 처리와 클러스터링 블록에 대해서도 같이 살펴보도록 하겠다.

![](/files/coding_for_ssd_part4_1.jpg)

## 5. 고급 기능

### 5.1. TRIM

용응 프로그램이 SSD의 모든 논리 블록 주소에 파일을 기록했다고 가정해보자.
그러면 SSD는 풀(full)로 사용되었다고 생각될 수 있다. 이제 이 모든 파일들이 지워졌다고 가정해보자.
파일 시스템은 SSD가 100% 비어 있다고 보지만,
SSD 컨트롤러는 호스트로부터 삭제된 논리 블록의 주소를 알지 못하기 때문에 실제 SSD 드라이브는 여전히 100% 사용중이라고 생각하게 된다.
SSD 컨트롤러는 호스트의 파일 시스템으로부터 덮어 쓰기 명령이 전달될 때에만 그 영역이 빈 공간이라고 판단할 수 있게 되는 것이다.

이때 Garbage-collection 프로세스는 삭제된 파일과 연관된 블록들을 지울(Erase) 것이다.
결과적으로 블록이 “stale” 데이터를 가지고 있다는 것을 알아내는 순간 삭제(Erase)하는 대신 지연 처리되는 것인데, 이는 성능을 심각하게 떨어뜨리게 된다.
또 다른 우려 사항은 삭제된 파일들을 SSD 컨트롤러는 모르기 때문에,
Garbage-collection은 Wear leveling을 위해서 매번 삭제된 파일들의 데이터도 계속 복사해서 새로운 페이지로 저장해야 한다.
이는 SSD의 Write Amplication을 더 높이게 되고, 호스트의 포그라운드 워크로드의 처리를 방해하게 된다.

지연된 삭제(Erase)로 인한 문제점을 해결하기 위한 방법이 TRIM 명령이다.
TRIM 명령은 운영 체제에서 의해서 해당 논리 공간이 더 이상 필요치 않다는 메시지를 SSD 컨트롤러로 전달해 주는 것이다.
이렇게 SSD 컨트롤러가 불필요한 블록들에 대한 정보를 가지고 있다면,
Garbage-collection 프로세스는 더 이상 불필요한 페이지들을 복사해서 이동하지 않아도 되며 언제든지 필요할 때 삭제(Erase)할 수 있게 되는 것이다.
TRIM 명령은 SSD 컨트롤러와 운영 체제 그리고 파일 시스템 3가지 컴포넌트가 모두 TRIM을 지원할 때에만 사용 가능하다.

TRIM 명령의 위키피디아 페이지[^16]에는 TRIM 명령을 지원하는 운영 체제와 파일 시스템의 목록을 나열하고 있다.
ATA TRIM 명령은 리눅스 커널 2.6.33 버전부터 지원되기 시작했다.
하지만 ext2와 ext3 파일 시스템은 여전히 TRIM 명령을 지원하지 않으며, ext4와 XFS 파일 시스템은 TRIM 명령을 지원한다.
Mac OS 10.6.8 그리고 HFS+ 파일 시스템은 TRIM 명령을 지원하며,
윈도우 7에서는 SATA 인터페이스를 사용하는 SSD에서만 TRIM 명령을 지원하며 PCI Express 인터페이스에서는 지원하지 않는다.

대 부분의 SSD 드라이브는 TRIM 명령을 지원하며, 앞으로의 쓰기 성능 향상을 위해서 Garbage-collection이 최대한 빨리 처리되도록 해줄 것이다.
그래서 가능하다면 TRIM이 지원되는 SSD를 사용할 것을 권장하며, 또한 운영 체제와 파일 시스템 레벨에서 TRIM이 가능하도록 준비하는 것이 좋다.

### 5.2. Over-provisioning

Over-provisioning은 단순히 논리적인 블록보다 물리적인 블록의 수가 더 많도록 해주는 것인데,
일정 비율의 물리 블록을 SSD 컨트롤러는 볼 수 있지만 운영 체제나 파일 시스템은 보지 못하도록 예약해두는 것이다.
전문 SSD 제조사는 이미 7 ~ 25% 정도의 Over-provisioning 공간을 보유하고 있으며[^13],
사용자는 단순히 가능한 물리 공간의 크기보다 작은 크기로 파티셔닝함으로써 더 많은 Over-provisioning 공간을 생성할 수 있다.
예를 들어서, 100GB SSD 드라이브에 90GB 파티션을 생성하면 나머지 10GB는 자동으로 Over-provisioning 공간으로 남겨지는 것이다.
Over-provisioning 공간은 운영 체제 레벨에서는 보이지 않더라도, SSD 컨트롤러는 여전히 그 공간을 볼 수 있고 사용할 수 있다.
SSD 제조사가 Over-provisioning 공간을 제공하는 주된 이유는 NAND 플래시 셀의 제한된 수명 주기를 극복하는 것이다.
보이지 않는 Over-provisioning 블록은 운영 체제에 보여지는 공간의 블록들이 수명이 다 되는 경우에 자동으로 대체되어서 사용된다.

AnandTech는 Over-provisioning 공간이 SSD의 성능과 수명에 미치는 영향을 보여주는 재미있는 게시물[^34]을 공개했다.
그들의 연구에 따르면 드라이브 공간의 25%를 Over-provisioning 공간으로 예약해 두면 엄청난 성능 향상을 얻을 수 있다는 것이다.
또 다른 재미있는 게시물을 Percona에서도 공개했는데,
Intel 320 SSD로 테스트한 결과 SSD의 공간 사용률이 높아질수록 쓰기 스루풋은 점점 떨어진다는 것[^38]이다.

왜 이런 현상이 발생하는 것일까?
Garbage-collection은 “stale” 페이지를 삭제(Erase)하기 위해서 백그라운드로 SSD 드라이브가 한가한 시간을 이용한다.
삭제(Erase) 오퍼레이션은 일반적인 데이터 쓰기보다 느리게 실행되므로 SSD가 지속적으로 과도한 랜덤 쓰기 부하가 있는 시스템에서는
Garbage-collection이 “stale” 페이지를 삭제(Erase)하기도 전에 “free” 페이지를 모두 소진해버리게 되는 것이다.
이때 FTL은 포그라운드의 랜덤 쓰기를 따라가지 못하고, Garbage-collection 프로세스는 호스트로부터 포그라운드 쓰기 요청이 들어오면 그때 동시에 삭제(Erase) 작업을 같이 하게 된다.
아래 그림 7에서 보여지는 바와 같이 벤치마킹의 결과에서 성능이 급격하게 떨어지는 것을 확인할 수 있다.
그래서 Over-provisioning 공간은 많은 쓰기 워크로드를 흡수해주는 버퍼 공간으로 작동하게 되는 것이다.
그리고 이 버퍼가 Garbage-collection이 워크로드를 따라잡을 수 있도록 충분한 시간을 만들어주는 것이다.
충분한 크기의 Over-provisioning 공간은 SSD가 사용되는 시스템의 워크로드에 따라 다른데,
일반적으로 지속적인 랜덤 쓰기 워크로드 환경에서는 25% 정도의 Over-provisioning 공간이 권장[^34]된다.
만약 워크로드가 그렇게 무겁지 않다면, 10 ~ 15% 정도의 Over-provisioning 공간으로도 충분할 수 있다.

> ##### Over-provisioning은 Wear-leveling과 성능 향상에 도움이 된다
>
> SSD 드라이브는 단순히 물리적인 공간보다 더 적은 크기로 논리적인 공간을 설정(포맷)하면 Over-provisioning 공간을 만들 수 있다.
> 남은 공간은 사용자에게는 보이지 않지만, SSD 컨트롤러는 여전히 그 공간을 사용할 수 있다.
> Over-provisioning은 NAND 플래시 셀의 제한된 수명을 극복하기 위한 Wear-leveling 메커니즘에도 도움이 된다.
> 쓰기 부하가 그렇게 심하지 않은 경우에는 10 ~ 15% 정도의 Over-provisioning 공간으로도 충분하다.
> 하지만 지속적인 랜덤 쓰기가 과도하게 발생하는 경우에는 25% 정도의 Over-provisioning 공간이 성능을 향상시켜 줄 것이다.
> Over-provisioning 공간은 NAND 플래시 블록의 버퍼처럼 작동하기 때문에,
> 쓰기가 포화(Write saturation)되는 시점에도 Garbage-collection이 과도한 쓰기를 견딜 수 있도록 해준다.

Over-provisioning은 TRIM 명령이 지원되지 않는 경우에도 성능 향상을 제공할 것으로 예상(이는 단순히 나의 추측이며, 이를 증명할만한 문서는 찾지 못했다)할 수 있다.
75%의 공간만 운영 체제가 사용하며 나머지 25%의 공간이 Over-provisioning으로 설정된 시스템을 가정해보자.
SSD 컨트롤러는 전체 공간을 모두 사용할 수 있기 때문에 돌아가면서 이 100%의 공간이 사용되었다가 삭제될 것이다.
하지만 어느 한 순간에는 75%의 NAND 플래시 메모리 공간만 사용될 것이다.
이는 나머지 25%의 물리 메모리 공간은 실제 논리 블록 주소와 맵핑되어 있지 않기 때문에,
SSD 컨트롤러는 안전하게 해당 영역이 데이터를 가지고 있지 않다고 판단할 수 있게 되는 것이다.
그래서 Over-provisioning 공간에 대해서는 미리 Garbage-collection을 처리할 수 있으며,
이는 TRIM 명령이 지원되지 않는다 하더라도 TRIM의 효과를 낼 수 있게 되는 것이다.

### 5.3. Secure Erase

일부 SSD 컨트롤러는 ATA Secure Erase 기능을 제공하는데,
이는 사용자가 기록했던 모든 데이터를 삭제하고 FTL 맵핑 테이블을 초기화함으로써 SSD 드라이브를 초기 상태로 만들어서 성능을 회복(초기 상태로)시키는 것이 주요 목적이다.
하지만 ATA Secure Erase 기능이 NAND 플래시 메모리 셀의 P/E Cycle 한계를 초기화 시켜주는 것은 아니다.
이 기능은 스펙상으로는 상당히 기대되는 기능이지만, 이는 제조사의 기능 구현이 얼마나 정확한지에 따라 다르다.
2011년 Wei(외 여러 연구원)은 12개 모델의 SSD에 대한 조사 결과, 단지 8개 모델만 ATA Secure Erase 기능을 제공했으며
이중에서 3개 모델은 잘못된 방식(Buggy implementation)으로 구현되어 있었다[^11].

성능과 관련된 사항은 중요하다. 하지만 보안과 관련된 문제는 더 중요하다. 그렇지만 보안과 관련된 사항은 이 문서의 주제와는 거리가 있다.
SSD의 데이터를 좀 더 신뢰성 있게 삭제(Erase)하는 방법에 대한 Stack Overflow 토의[^48] [^49]들이 있으니, 더 자세한 내용은 이 게시물들을 참조하도록 하자.

### 5.4. Native Command Queueing (NCQ)

NCQ(Native Command Queueing)는 SSD 드라이브가 내부적인 병렬 처리 능력을 이용해서 동시에 여러 호스트 명령을 처리할 수 있도록 해주는 Serial ATA의 기능이다[^3].
SSD 드라이브의 낮은 레이턴시에 더불어 최신의 SSD 드라이브는 호스트로부터의 레이턴시를 극복하기 위해서 NCQ를 사용한다.
예를 들어서 NCQ는 호스트 CPU가 바쁠때에는 드라이버가 항상 명령을 처리할 수 있도록 유입되는 명령의 우선순위를 높여줄 수 있다[^39].
빠른 응답 속도를 위해서 새로운 드라이브는 NCQ를 사용하는데, NCQ는 호스트의 명령을 큐잉하여 우선순위를 재설정하고 때로는 지연처리를 하기도 한다[^39].

### 5.5. 전력 차단 보호 (Power-loss protection)

SSD 드라이브가 설치된 컴퓨터가 집에 있든지 데이터센터에 있든지 전원 실패(Power Failure, 정전)은 피할 수 없다.
일부 제조사는 SSD 드라이버에 수퍼 커패시터(Super capacity)를 포함해서,
컴퓨터의 전원 공급이 차단되었을 때 요청된 I/O를 처리할 수 있을 정도의 충분한 전원을 내장하여 항상 SSD 드라이버가 일관된 상태를 유지하도록 해준다.
하지만 문제는 모든 SSD 드라이버가 수퍼 커패시터나 전원 차단에 대한 보호 장치를 내장하고 있는 것은 아니며,
SSD 드라이브의 제품 소개서에 항상 명시하는 것도 아니다.
Secure Erase 기능과 같이 전원 보호 장치가 제대로 구현되었는지 명확하지 않다.

2013년 Zheng(외 여러 연구원)은 15개 제품의 SSD 드라이브를 테스트[^72]했다.
다양한 전원 실패 상황으로 테스트를 진행한 결과, 15개 제품 중 13개 제품에서는 데이터 손실이 발생했으며 데이터 손상도 발생했다.
Luke Kenneth Casson Leighton의 전원 실패(Power Fault) 테스트에서는 4개 제품중 3개 SSD 드라이브의 데이터가 손상되었으며,
나머지 1개(Intel SSD 드라이브)만 일관된 상태를 유지했다[^73].

SSD는 아직은 상당히 젊은 기술이기 때문에, 전원 실패에 대한 데이터 손상 문제는 머지않아 해결될 것으로 생각된다.
하지만 현재 시점에서는 무정전 전원 장치(UPS, uninterruptible power supply)에 대한 투자는 충분히 필요할 것으로 보인다.
또한 다른 데이터 저장 솔루션과 마찬가지로 SSD도 정기적인 백업이 필요할 것으로 보인다.

## 6. SSD의 내부 병렬 처리

### 6.1. 제한된 I/O 버스 대역폭

물리적인 한계로 인해서, 비동기 방식의 NAND 플래시 I/O 버스는 32~40MB/s의 대역폭 이상을 서비스할 수 없다[^5].
SSD 제조사의 입장에서 성능을 향상시킬 수 있는 유일한 방법은 다수의 패키지가 병렬로 처리되거나 인터리빙(interleave) 모드로 작동하도록 설계를 변경하는 것이다.
인터리빙 모드의 자세한 설명은 Kim et al. 의 2013년 문서인 “Parameter-Aware I/O Management for Solid State Disks (SSDs)”의 2.2를 참조하도록 하자.

SSD아키텍처에서 여러 레벨의 내부적인 병렬 처리 능력을 묶어서, 여러 칩에 걸쳐서 동시에 2개 이상의 블록을 액세스할 수 있다.
이렇게 동시 접근 가능한 블록을 묶어서 “Clustered block”이라고 한다. SSD 드라이버의 내부적인 병렬 처리에 관련된 상세한 내용은 이 문서의 방향은 아니지만,
Clustered block과 병렬 처리의 레벨에 대해서는 간략하게 살펴보도록 하겠다.
이 주제에 관련된 더 상세한 내용에 대해서 살펴보고자 한다면, 이 두개[^2] [^3]의 문서를 살펴보도록 하자.
더불어 copyback 명령 및 Plane간 데이터 전송등 고급 기능에 대한 내용은 이 문서[5]를 더 참조해보자.

> ##### 내부 병렬 처리 (Internal parallelism)
>
> 내부적으로 여러 레벨의 병렬 처리 능력이 NAND 플래시 칩의 다른 위치에서 2개 이상의 블록을 동시에 읽을 수 있도록 해주는데,
> 이를 “Clustered block”이라고 한다.

### 6.2. 병렬 처리

그림 6은 NAND 플래시 패키지들이 계층형으로 구성되어 있는 SSD 드라이브의 내부를 보여주고 있다.
SSD 드라이브는 채널(Channel), 패키지(Package), 칩(Chip), 플레인(Plane), 그리고 블록(Block)과 페이지(Page)등의 다양한 레벨로 구성되어 있다.
문서 [^3]에 설명된 것처럼, 아래와 같은 병렬 처리 능력을 제공한다.

* Channel-level parallelism. 플래시 컨트롤러는 여러 채널을 통해서 플래시 패키지와 통신한다. 이 채널들은 서로 독립적이며 동시에 액세스 가능하다. 각 개별 채널은 여러 패키지에 의해서 공유된다.
* Package-level parallelism. 채널의 패키지들은 독립적으로 접근 가능하다. 동일 채널을 공유하는 패키지들은 인터리빙 모드로 동시에 명령 실행에 사용될 수 있다.
* Chip-level parallelism. 패키지는 하나 또는 두개의 칩(Chips, 때로는 “dies”라고도 불림)을 가지며, 각 칩들은 독립적으로 병렬 접근이 가능하다.
* Plane-level parallelism. 각 칩은 2개 이상의 플레인을 가지는데, 동일 오퍼레이션(Read, Write, Erase)은 하나의 칩내의 여러 플레인에 대해서 동시 실행이 될 수 있다. 플레인은 블록들을 가지며, 각 블록들은 다시 페이지들을 가진다. 또한 플레인은 플레인 단위의 오퍼레이션에 사용되는 레지스터를 가진다. 

![그림 6: NAND 플래시 패키지](/files/coding_for_ssd_part4_2.jpg)

### 6.3. Clustered blocks

여러 칩에 걸쳐서 한번에 액세스될 수 있는 여러 개의 블록을 “Clustered block”[^2]이라고 하는데,
RAID 시스템[^1] [^5]에서 사용되는 스트라이핑(striping)과 비슷한 아이디어이다.

한번에 접근 가능한 논리 블록 주소들은 개별 플래시 패키지에서 서로 다른 SSD 칩들에 걸쳐서 스트라이핑된다.
이는 FTL의 맵핑 알고리즘 덕분에 가능하며, 이는 Clustered block에 포함되는 블록들이 서로 연속된 주소이든지 관계없이 독립적이다.
스트라이핑 블록들은 여러 채널을 동시에 사용할 수 있으며, 그 채널들의 대역폭을 묶어서 생각할 수 있다.
또한 읽기와 쓰기 그리고 삭제(Erase) 오퍼레이션을 병렬로 처리할 수 있다.
이는 Clustered block 사이즈 또는 그 배수 크기의 I/O 오퍼레이션은 SSD의 다양한 내부 병렬 처리 능력을 최대한 활용할 수 있다는 것을 보장한다.
Clustered block에 대한 자세한 내용은 섹션 8.2와 8.3을 참조하도록 하자.

> **This articles are translated to Korean with original author's([Emmanuel Goossaert](http://www.goossaert.com/)) permission. Really appreciate his effort and sharing.**
>
> **Original articles :**
>
> * [Part 1: Introduction and Table of Contents](http://codecapsule.com/2014/02/12/coding-for-ssds-part-1-introduction-and-table-of-contents/)
> * [Part 2: Architecture of an SSD and Benchmarking](http://codecapsule.com/2014/02/12/coding-for-ssds-part-2-architecture-of-an-ssd-and-benchmarking/)
> * [Part 3: Pages, Blocks, and the Flash Translation Layer](http://codecapsule.com/2014/02/12/coding-for-ssds-part-3-pages-blocks-and-the-flash-translation-layer/)
> * [Part 4: Advanced Functionalities and Internal Parallelism](http://codecapsule.com/2014/02/12/coding-for-ssds-part-4-advanced-functionalities-and-internal-parallelism/)
> * [Part 5: Access Patterns and System Optimizations](http://codecapsule.com/2014/02/12/coding-for-ssds-part-5-access-patterns-and-system-optimizations/)
> * [Part 6: A Summary – What every programmer should know about solid-state drives](http://codecapsule.com/2014/02/12/coding-for-ssds-part-6-a-summary-what-every-programmer-should-know-about-solid-state-drives/)

### References

[^1]: [Understanding Intrinsic Characteristics and System Implications of Flash Memory based Solid State Drives, Chen et al., 2009](http://www.cse.ohio-state.edu/hpcs/WWW/HTML/publications/papers/TR-09-2.pdf)  
[^2]: [Parameter-Aware I/O Management for Solid State Disks (SSDs), Kim et al., 2012](http://csl.skku.edu/papers/CS-TR-2010-329.pdf)  
[^3]: [Essential roles of exploiting internal parallelism of flash memory based solid state drives in high-speed data processing, Chen et al, 2011](http://bit.csc.lsu.edu/~fchen/paper/papers/hpca11.pdf)  
[^4]: [Exploring and Exploiting the Multilevel Parallelism Inside SSDs for Improved Performance and Endurance, Hu et al., 2013](http://ieeexplore.ieee.org/xpls/abs_all.jsp?arnumber=6165265)  
[^5]: [Design Tradeoffs for SSD Performance, Agrawal et al., 2008](http://research.microsoft.com/pubs/63596/usenix-08-ssd.pdf)  
[^6]: [Design Patterns for Tunable and Efficient SSD-based Indexes, Anand et al., 2012](http://instartlogic.com/resources/research_papers/aanand-tunable_efficient_ssd_indexes.pdf)  
[^7]: [BPLRU: A Buffer Management Scheme for Improving Random Writes in Flash Storage, Kim et al., 2008](https://www.usenix.org/legacy/events/fast08/tech/full_papers/kim/kim.pdf)  
[^8]: [SFS: Random Write Considered Harmful in Solid State Drives, Min et al., 2012](https://www.usenix.org/legacy/event/fast12/tech/full_papers/Min.pdf)  
[^9]: [A Survey of Flash Translation Layer, Chung et al., 2009](http://idke.ruc.edu.cn/people/dazhou/Papers/AsurveyFlash-JSA.pdf)  
[^10]: [A Reconfigurable FTL (Flash Translation Layer) Architecture for NAND Flash-Based Applications, Park et al., 2008](http://idke.ruc.edu.cn/people/dazhou/Papers/a38-park.pdf)  
[^11]: [Reliably Erasing Data From Flash-Based Solid State Drives, Wei et al., 2011](https://www.usenix.org/legacy/event/fast11/tech/full_papers/Wei.pdf)  
[^12]: <http://en.wikipedia.org/wiki/Solid-state_drive>  
[^13]: <http://en.wikipedia.org/wiki/Write_amplification>  
[^14]: <http://en.wikipedia.org/wiki/Flash_memory>  
[^15]: <http://en.wikipedia.org/wiki/Serial_ATA>  
[^16]: <http://en.wikipedia.org/wiki/Trim_(computing)>  
[^17]: <http://en.wikipedia.org/wiki/IOPS>  
[^18]: <http://en.wikipedia.org/wiki/Hard_disk_drive>  
[^19]: <http://en.wikipedia.org/wiki/Hard_disk_drive_performance_characteristics>  
[^20]: <http://centon.com/flash-products/chiptype>  
[^21]: <http://www.thessdreview.com/our-reviews/samsung-64gb-mlc-ssd/>  
[^22]: <http://www.anandtech.com/show/7594/samsung-ssd-840-evo-msata-120gb-250gb-500gb-1tb-review>  
[^23]: <http://www.anandtech.com/show/6337/samsung-ssd-840-250gb-review/2>  
[^24]: <http://www.storagereview.com/ssd_vs_hdd>  
[^25]: <http://www.storagereview.com/wd_black_4tb_desktop_hard_drive_review_wd4003fzex>  
[^26]: <http://www.storagereview.com/samsung_ssd_840_pro_review>  
[^27]: <http://www.storagereview.com/micron_p420m_enterprise_pcie_ssd_review>  
[^28]: <http://www.storagereview.com/intel_x25-m_ssd_review>  
[^29]: <http://www.storagereview.com/seagate_momentus_xt_750gb_review>  
[^30]: <http://www.storagereview.com/corsair_vengeance_ddr3_ram_disk_review>  
[^31]: <http://arstechnica.com/information-technology/2012/06/inside-the-ssd-revolution-how-solid-state-disks-really-work/>  
[^32]: <http://www.anandtech.com/show/2738>  
[^33]: <http://www.anandtech.com/show/2829>  
[^34]: <http://www.anandtech.com/show/6489>  
[^35]: <http://lwn.net/Articles/353411/>  
[^36]: <http://us.hardware.info/reviews/4178/10/hardwareinfo-tests-lifespan-of-samsung-ssd-840-250gb-tlc-ssd-updated-with-final-conclusion-final-update-20-6-2013>  
[^37]: <http://www.anandtech.com/show/6489/playing-with-op>  
[^38]: <http://www.ssdperformanceblog.com/2011/06/intel-320-ssd-random-write-performance/>  
[^39]: <http://en.wikipedia.org/wiki/Native_Command_Queuing>  
[^40]: <http://superuser.com/questions/228657/which-linux-filesystem-works-best-with-ssd/>  
[^41]: <http://blog.superuser.com/2011/05/10/maximizing-the-lifetime-of-your-ssd/>  
[^42]: <http://serverfault.com/questions/356534/ssd-erase-block-size-lvm-pv-on-raw-device-alignment>  
[^43]: <http://rethinkdb.com/blog/page-alignment-on-ssds/>  
[^44]: <http://rethinkdb.com/blog/more-on-alignment-ext2-and-partitioning-on-ssds/>  
[^45]: <http://rickardnobel.se/storage-performance-iops-latency-throughput/>  
[^46]: <http://www.brentozar.com/archive/2013/09/iops-are-a-scam/>  
[^47]: <http://www.acunu.com/2/post/2011/08/why-theory-fails-for-ssds.html>  
[^48]: <http://security.stackexchange.com/questions/12503/can-wiped-ssd-data-be-recovered>  
[^49]: <http://security.stackexchange.com/questions/5662/is-it-enough-to-only-wipe-a-flash-drive-once>  
[^50]: <http://searchsolidstatestorage.techtarget.com/feature/The-truth-about-SSD-performance-benchmarks>  
[^51]: <http://www.theregister.co.uk/2012/12/03/macronix_thermal_annealing_extends_life_of_flash_memory/>  
[^52]: <http://www.eecs.berkeley.edu/~rcs/research/interactive_latency.html>  
[^53]: <http://blog.nuclex-games.com/2009/12/aligning-an-ssd-on-linux/>  
[^54]: <http://www.linux-mag.com/id/8397/>  
[^55]: <http://tytso.livejournal.com/2009/02/20/>  
[^56]: <https://wiki.debian.org/SSDOptimization>  
[^57]: <http://wiki.gentoo.org/wiki/SSD>  
[^58]: <https://wiki.archlinux.org/index.php/Solid_State_Drives>  
[^59]: <https://www.kernel.org/doc/Documentation/block/cfq-iosched.txt>  
[^60]: <http://www.danielscottlawrence.com/blog/should_i_change_my_disk_scheduler_to_use_NOOP.html>  
[^61]: <http://www.phoronix.com/scan.php?page=article&item=linux_iosched_2012>  
[^62]: <http://www.velobit.com/storage-performance-blog/bid/126135/Effects-Of-Linux-IO-Scheduler-On-SSD-Performance>  
[^63]: <http://www.axpad.com/blog/301>  
[^64]: <http://en.wikipedia.org/wiki/List_of_solid-state_drive_manufacturers>  
[^65]: <http://en.wikipedia.org/wiki/List_of_flash_memory_controller_manufacturers>  
[^66]: <http://blog.zorinaq.com/?e=29>  
[^67]: <http://www.gamersnexus.net/guides/956-how-ssds-are-made>  
[^68]: <http://www.gamersnexus.net/guides/1148-how-ram-and-ssds-are-made-smt-lines>  
[^69]: <http://www.tweaktown.com/articles/4655/kingston_factory_tour_making_of_an_ssd_from_start_to_finish/index.html>  
[^70]: <http://www.youtube.com/watch?v=DvA9koAMXR8>  
[^71]: <http://www.youtube.com/watch?v=3s7KG6QwUeQ>  
[^72]: [Understanding the Robustness of SSDs under Power Fault, Zheng et al., 2013](https://www.usenix.org/conference/fast13/technical-sessions/presentation/zheng) — [discussion on HN](https://news.ycombinator.com/item?id=7047118)  
[^73]: <http://lkcl.net/reports/ssd_analysis.html> - [discussion on HN](https://news.ycombinator.com/item?id=6973179)  
