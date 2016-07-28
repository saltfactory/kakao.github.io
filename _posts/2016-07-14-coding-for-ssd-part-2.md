---
layout: post
title: '개발자를 위한 SSD (Coding for SSD) - Part 2 : SSD의 아키텍처와 벤치마킹'
author: matt.lee
date: 2016-07-14 20:00
tags: [ssd,nand-flash,garbage-collection,LBA,PBA,block,page,clustered-block]
image: /files/covers/solid-state-logic.jpg
cover:
  title: '80-channel Solid State Logic (SSL) XL 9000 K Series Console at Audio Mix House, Studio B'
  link: https://flic.kr/p/j1DcB
  author:
    name: 'Audio Mix House'
    link: https://www.flickr.com/photos/audiomixhouse/
---


이 챕터에서는 NAND 플래시 메모리의 기본적인 내용과 셀 타입 그리고 SSD의 기본적인 내부 아키텍처에 대해서 살펴보고,
추가로 SSD의 벤치마킹 방법과 벤치마킹 결과를 해석하는 부분도 살펴보도록 하겠다.

![](/files/coding_for_ssd_part2_1.jpg)

## 1. SSD의 구조

### 1.1. NAND 플래시 메모리 셀

SSD(Solid-State Drive)는 플래시 메모리를 기반으로한 저장 매체이다. 비트들은 Floating-Gate 트랜지스터로 구성된 셀에 저장된다.
SSD는 모든 컴포넌트가 전기 장치이며, HDD와 같은 기계 장치를 가지고 있지 않다.

Floating-Gate 트랜지스터에 전압이 가해지면서 셀의 비트가 쓰여지거나 읽혀지게 된다.
트랜지스터는 NOR 플래시 메모리와 NAND 플래시 메모리 두 종류가 있는데(여기에서는 NAND의 NOR 플래시 메모리의 차이에 대해서 깊이 있게 살펴보지는 않을 것이다),
이 게시물에서는 많은 제조사들이 채택하고 있는NAND 플래시 메모리에 대해서만 살펴보도록 하겠다. NAND와 NOR 플래시 메모리의 상세한 차이에 대해서 궁금하다면,
Lee Hutchinson[^31]의 문서를 살펴보도록 하자.

NAND 플래시 메모리 모듈의 중요한 속성은 수명이 제한적(Wearing-off)이라는 것이다.
트랜지스터는 셀에 전자를 저장하면서 쓰기를 하게 되는데,
매번 P/E(Program & Erase, Program은 쓰기를 의미) 사이클마다 일부 전자가 오류로 인해서 트랜지스터에 갇히게 된다.
그리고 이렇게 갇힌 전자들이 쌓여서 일정 수준을 넘어서게 되면, 그 셀은 사용 불가능한 상태가 되는 것이다.

> ##### 수명 제한
>
> 각 셀은 최대 P/E 사이클을 가지는데, 이 사이클을 넘어서면 결함 셀로 간주된다.
> NAND 플래시 메모리는 수명 제한을  가지는데, 이 수명 제한은 NAND 플래시 메모리의 타입(SLC, MLC, TLC)에 따라서 조금씩 차이가 있다[^31].

최근 연구에 따르면, NAND 플래시 메모리 칩에 높은 열이 가해지면 각 셀에 갇혀있던(프로그램되어 있던) 전자들이 사라진다는 것이 확인되었다[^14] [^51].
또한 아직 연구중이며 언제 시중에 이런 제품이 출시될지 모르지만, SSD의 수명을 상당히 높일 수 있는 방법도 있다.
현재 시중에 출시되는 SSD의 메모리 셀 타입은:

* SLC (Single level cell), SLC 트랜지스터는 단 하나의 비트만 저장할 수 있지만 상대적으로 긴 수명을 가짐
* MLC (Multiple level cell), MLC 트랜지스터는 2 비트를 저장할 수 있으며, SLC에 비해서 상대적으로 레이턴시(응답 속도)가 높고 짧은 수명을 가짐
* TLC (Triple-level cell), TLC 트랜지스터는 3 비트를 저장할 수 있지만, MLC보다 레이턴시가 높고 수명이 더 짧음

> ##### 메모리 셀 타입
>
> SSD는 플래시 메모리 기반의 저장 매체이다. 비트는 셀에 저장되며, 메모리 셀은 3가지 타입이 있다. SLC는 1비트, MLC는 2비트 그리고 TLC는 3비트를 저장할 수 있다.

표1은 각 NAND 플래시 셀 타입의 상세한 정보를 보여주고 있다. 비교를 위해서 HDD와 메인 메모리(RAM) 그리고 L1/L2 캐시도 같이 포함하였다.

|     | SLC | MLC | TLC | HDD | RAM | L1 cache | L2 cache |
|-----|-----|-----|-----|-----|-----|----------|----------|
| P/E cycles | 100k | 10k | 5k | * | * | * | * | 
| Bits per cell | 1 | 2 | 3 | * | * | * | * |
| Seek latency (μs) | * | * | * | 9000 | * | * | * | 
| Read latency (μs) | 25 | 50 | 100 | 2000-7000 | 0.04-0.1 | 0.001 | 0.004 |
| Write latency (μs) | 250 | 900 | 1500 | 2000-7000 | 0.04-0.1 | 0.001 | 0.004 |
| Erase latency (μs) | 1500 | 3000 | 5000 | * | * | * | * |

표1: 다른 메모리 컴포넌트와 NAND 플래시 메모리의 특성 및 레이턴시 비교

* Notes
    *  `*` metric is not applicable for that type of memory
* Sources
    * P/E cycles [^20]
    * SLC/MLC latencies [^1]
    * TLC latencies [^23]
    * Hard disk drive latencies [^18] [^19] [^25]
    * RAM latencies [^30] [^52]
    * L1 and L2 cache latencies [^52]


동일 량의 트랜지스터로 더 많은 비트들을 저장할 수 있다면, 당연히 제조 단가는 낮아지게 된다.
SLC 타입의 SSD는 MLC 타입보다 신뢰성이 높고 더 오랜 시간 사용할 수 있지만, 제조 비용이 크다.
그래서 대 부분의 SSD는 MLC 또는 TLC 타입이며, 엔터프라이즈급의 SSD만 SLC를 사용한다.
워크로드에 맞게 적절한 메모리 타입을 선정하는 것이 중요한데, 여기서 워크로드라 함은 얼마나 SSD에 데이터가 자주 기록되는지를 의미한다.
예를 들어서 쓰기 빈도가 아주 높다면 SLC 타입의 SSD가 최선이며,
(동영상 스트리밍을 위한 저장소와 같이) 쓰기는 많지 않지만 읽기가 매우 많다면 TLC가 가장 적절한 선택이 될 것이다.
게다가 실제 서비스 수준의 워크로드로 수행된 벤치마크의 결과에 의하면, 실제 TLC 타입의 메모리 수명은 크게 문제되지 않는 것으로 보인다 [^36].

> ##### NAND 플래시 페이지와 블록
>
> 셀들은 Block으로 그룹핑 되어 있으며, Block들은 다시 Plane으로 그룹핑되어 있다.
> SSD를 읽고 쓰기시에 최소 접근 단위는 Page라고 하며, Page는 개별로 삭제(Erase)될 수 없고 반드시 삭제는 Block 단위로만 수행될 수 있다.
> NAND 플래시 메모리의 Page 크기는 2KB, 4KB, 8KB, 16KB로 다양한데, 대 부분 SSD의 Block은 128개 또는 256개의 Page를 가진다.
> 이는 SSD 제조사별로 Block의 크기가 256KB에서 4MB까지 다양하다는 것을 의미한다.
> 예를 들어서 삼성 SSD 840 EVO는 2048KB의 Block 크기를 가지며 각 Block은 256개의 8KB 페이지를 가진다.
> Page와 Block의 접근 방법에 대해서는 섹션 3.1에서 자세히 살펴보도록 하겠다.

### 1.2. SSD의 구성

아래의 그림 1은 SSD의 주요 컴포넌트들을 도식화한 것이다.
이는 이미 다양한 문서([^2] [^3] [^6])들에서 공개된 내용들이며, 간단히 개요만 표시한 것이다.

![그림 1: Solid-State Drive의 아키텍처](/files/coding_for_ssd_part2_2.jpg)

사용자의 요청은 호스트 인터페이스를 통해서 유입되는데,
이 글을 작성하는 시점에는 가장 일반적인 인터페이스 방식은 ATA(SATA)와 PCI Express(PCIe) 타입이다.
SSD 컨트롤러에 장착된 프로세서가 명령을 받아서 플래시 컨트롤러로 전달하게 된다.
SSD는 자체적으로 보드에 내장된 메모리(RAM)을 가지는데, 일반적으로 이 메모리는 맵핑 정보를 저장하거나 캐시 용도로 사용된다.
상세한 맵핑 정책에 대해서는 섹션 4에서는 살펴보도록 하겠다.
SSD는 여러 개의 Channel을 통해서 서로 NAND 플래시 메모리 패키지로 구성된다.
Channel에 대한 상세한 내용은 섹션 6에서 살펴보도록 하겠다.

아래의 그림 2와 3(StorageReview.com의 게시물[^26] [^27]을 참조)은 실생활에서 사용되는 SSD가 어떻게 생겼는지를 보여주고 있다.
그림 2는 2013년 8월에 출시된 512GB 삼성 840 Pro SSD이다. 기판에서 보이듯이, 주요 컴포넌트는:

* 1 SATA 3.0 인터페이스
* 1 SSD 컨트롤러 (삼성 MDX S4LN021X01-8030)
* 1 RAM 모듈 (256 MB DDR2 삼성 K4P4G324EB-FGC2)
* 8 MLC NAND-플래시 모듈, 각 모듈은 64 GB (삼성 K9PHGY8U7A-CCK0)

![그림 2: 삼성 SSD 840 Pro (512 GB)](/files/coding_for_ssd_part2_3.jpg)
![그림 2: 삼성 SSD 840 Pro (512 GB)](/files/coding_for_ssd_part2_4.jpg)
그림 제공: StorageReview.com [^26]

그림 3은 2013년 후반기에 출시된 마이크론(Micron) P420m Enterprise PCIe 이며, 주요 컴포넌트는:

* 8 레인(lane) PCI Express 2.0 인터페이스
* 1 SSD 컨트롤러
* 1 RAM 모듈 (DRAM DDR3)
* 64 MLC NAND-플래시 모듈(32 채널), 각 모듈은 32GB (마이크론 31C12NQ314 25nm)

전체 메모리 공간은 2048GB이지만, 프로비저닝(over-provisioning) 영역을 제외하고 1.4TB 사용 가능.

![그림 3: Micron P420m Enterprise PCIe (1.4 TB)](/files/coding_for_ssd_part2_5.jpg)
![그림 3: Micron P420m Enterprise PCIe (1.4 TB)](/files/coding_for_ssd_part2_6.jpg)
그림 제공: StorageReview.com [^27]

### 1.3. SSD의 생산 공정

많은 SSD 제조사는 SSD 생산을 위해서 SMT(Surface-Mount Technology)를 사용하는데,
SMT 생산 과정에서는 전자 부품들이 회로 기판(PCBs)위에 직접 장착된다.
SMT 생산 라인은 기계들이 줄지어 있고, 각 기계들이 부품을 기판위에 놓거나 놓여진 부품들을 납땜하는 등의 각 작업들을 담당하게 된다.
전체 생산 공정에서 여러번의 품질 체크 과정도 수행된다.
SMT 라인의 사진이나 동영상은 Steve Burke[^67] [^68]의 글(캘리포니아 파운틴 밸리의 Kingston Technologies사를 방문했을 때 사진과
Cameron Wilmot이 작성한 타이완[^69]의 Kingston 공장 사진)에서 참조할 수 있다.

그 이외에도 2개의 참조할 만한 영상이 있는데, 첫번째 동영상은 Micron사의 "Crucial SSDs"[^70]이며 두번째는 Kingston[^71]에 관련된 것이다.
두번째 동영상은 "Steve Burke"가 작성한 글의 일부인데, 이 글의 하단에 포함시켜 두었다.
Kingston사의 Mark Tekunoff는 SMT 라인 투어를 시켜 주었으며,
동영상의 모든 사람들은 즐겁게 정전기 방지용 파자마를 입고 있는 모습을 볼 수 있다.

[![RAM & SSD & 메인보드는 어떻게 만들어지는가?](http://img.youtube.com/vi/3s7KG6QwUeQ/0.jpg)](http://www.youtube.com/watch?v=3s7KG6QwUeQ)

## 2. 벤치마킹과 성능 메트릭

### 2.1. 기본 벤치마킹

아래의 표2는 여러 SSD 드라이브들의 랜덤과 시퀀셜 워크로드에서 스루풋을 보여주고 있다.
비교를 위해서 2008년과 2013년에 출시된 SSD를 HDD와 메모리(RAM)과 함께 나열해 보았다.

|      | Samsung 64 GB | Intel X25-M | Samsung 840 EVO | Micron P420m | HDD | RAM |
|------|---------------|-------------|-----------------|--------------|-----|-----|
| Brand/Model | Samsung (MCCDE64G5MPP-OVA) | Intel X25-M (SSDSA2MH080G1GC) | Samsung (SSD 840 EVO mSATA) | Micron P420m | Western Digital Black 7200 rpm | Corsair Vengeance DDR3 |
| Memory cell type | MLC | MLC | TLC | MLC | * | * |
| Release year | 2008 | 2008 | 2013 | 2013 | 2013 | 2012 |
| Interface | SATA 2.0 | SATA 2.0 | SATA 3.0 | PCIe 2.0 | SATA 3.0 | * |
| Total capacity | 64 GB | 80 GB | 1 TB | 1.4 TB | 4 TB | 4 x 4 GB |
| Pages per block | 128 | 128 | 256 | 512 | * | * |
| Page size | 4 KB | 4 KB | 8 KB | 16 KB | * | * |
| Block size | 512 KB | 512 KB | 2048 KB | 8196 KB | * | * |
| Sequential reads (MB/s) | 100 | 254 | 540 | 3300 | 185 | 7233 |
| Sequential writes (MB/s) | 92 | 78 | 520 | 630 | 185 | 5872 |
| 4KB random reads (MB/s) | 17 | 23.6 | 383 | 2292 | 0.54 | 5319 ** |
| 4KB random writes (MB/s) | 5.5 | 11.2 | 352 | 390 | 0.85 | 5729 ** |
| 4KB Random reads (KIOPS) | 4 | 6 | 98 | 587 | 0.14 | 105 |
| 4KB Random writes (KIOPS) | 1.5 | 2.8 | 90 | 100 | 0.22 | 102 |

표2: 다른 저장 매체와 SSD 드라이브의 스루풋 및 특성 비교

Notes

* `*` metric is not applicable for that storage solution
* `**` measured with 2 MB chunks, not 4 KB

Metrics

* MB/s: Megabytes per Second
* KIOPS: Kilo IOPS, i.e 1000 Input/Output Operations Per Second

Sources

* Samsung 64 GB [^21]
* Intel X25-M [^2][^28]
* Samsung SSD 840 EVO [^22]
* Micron P420M [^27]
* Western Digital Black 4 TB [^25]
* Corsair Vengeance DDR3 RAM [^30]

호스트 인터페이스는 성능을 결정하는 중요한 요소중 하나이다.
최근에 출시되는 SSD의 일반적인 인터페이스는 SATA 3.0 또는 PCI Express 3.0이다.
SATA 3.0 인터페이스는 6 Gbit/s 정도의 데이터 전송량을 낼 수 있는데, 이는 초당 550MB 정도의 성능이다.
그리고 PCIe 3.0 인터페이스에서는 레인(lane)당 8 GT/s(GT/s 는 초당 전송가능한 Giga를 의미, Gigatransfers/second) 데이터 전송이 가능한데,
이는 대략 1 GB/s 정도이다.
PCIe 3.0 인터페이스는 최소 1개 이상의 레인(lane)을 가지고 있는데,
4개 레인을 가진 SSD는 SATA 3.0 인터페이스보다 8배나 빠른 4 GB/s 전송 속도를 낼 수 있다.
일부 엔터프라이즈 SSD중에는 SAS (Serial Attached SCSI) 인터페이스를 장착한 제품들도 있는데,
이들은 12 GBit/s정도의 전송 속도를 제공하지만 실제 SAS 인터페이스를 장착한 제품은 많지 않다.

최근 출시되는 대 부분의 SSD들은 SATA 3.0의 최대 전송 속도인 550MB/s를 초과할 정도의 성능을 가지고 있으므로,
현재는 대부분 인터페이스가 병목 지점인 경우가 많다.
PCI Express 3.0이나 SAS 인터페이스를 장착한 SSD는 엄청난 성능 향상을 제공할 것이다.[^15]
SATA보다 빠른 PCI Express 와 SAS 인터페이스 제조사에서 주로 사용되는 호스트 인터페이스는
SATA 3.0 (550 MB/s)과 PCI Express 3.0 (레인당 1 GB/s, 다중 채널 사용)이다.
SAS(Serial Attached SCSI) 또한 엔터프라이즈 SSD에 사용되는데,
PCIe와 SAS 인터페이스는 SATA보다 훨씬 빠른 성능을 제공하지만, 그만큼 가격도 비싼 편이다.

### 2.2. 프리 컨디셔닝 (Pre-conditioning)

> If you torture the data long enough, it will confess.  
>
> — Ronald Coase

SSD 제조사가 제공하는 제품의 데이터 시트에는 놀라울 정도의 성능 메트릭들로 채워져 있다.
마케팅 효과를 위해서 제조사들은 항상 반짝이는 수치들을 보여주기 위해서 방법을 찾아내는 것처럼 보인다.
그 수치들이 진정으로 뭔가를 의미하는지 모르겠지만, 실제 데이터 시트의 수치와 운영 시스템에서 사용되는 제품의 성능 예측은 다른 문제이다.

Marc Bevand가 작성한 문서[^66]는 일반적인 SSD 벤치마크의 결함에 대해서 언급하고 있는데,
이 문서에서 Marc Bevand는 SSD의 LBA(Logical Block Addressing) 크기 설정에 대한 언급없이 랜덤 쓰기 성능을 명시하거나
Queue depth를 1로 설정하고 테스트한 결과를 레포팅하는 것은 적절하지 못하다고 이야기하고 있다.
또한 이 이외에도 벤치마크 도구의 버그나 잘못된 사용으로 인한 케이스들도 다양하다.

SSD의 성능을 정확하게 예측하는 것은 어려운 부분이다.
많은 하드웨어 리뷰 블로그들이 10여분 정도의 테스트 실행 후, 그 결과를 두고 충분히 신뢰성 있다라고 이야기하고 있다.
그러나 SSD는 지속적인 랜덤 쓰기(SSD의 전체 공간이 30분에서 3시간 정도 저장할 수 있는 수준의 워크로드)가 발생하는 테스트 환경에서만 성능 저하를 보여준다.
그래서 어느정도의 쓰기 부하를 발생(이를 “pre-conditioning” [^50]라고 함)시킨 후, 벤치마크가 좀 더 신뢰성 있는 테스트라고 볼 수 있는 것이다.
아래의 그림 7은 StorageReview.com [^26]으로부터 가져온 것인데, 다양한 SSD에서 "pre-conditioning"의 효과를 잘 보여주고 있다.
좀더 명확한 성능 저하는 30분 정도 지난 이후 시점부터 나타나는데,
이때부터 모든 SSD 드라이브에 대해서 스루풋은 떨어지고 레이턴시(응답 속도)는 커지는 것을 확인할 수 있다.
그리고 4시간정도 경과한 후에는 성능이 더 떨어져서 최하 수준으로 수렴하는 것을 확인할 수 있다.


![그림 7: 여러 SSD에서 “pre-conditioning”의 효과](/files/coding_for_ssd_part2_7.jpg)
그림 제공: StorageReview.com [^26]
 
그림 7에서 나타나는 현상은 섹션 5.2에서 설명된 것인데, 랜덤 쓰기가 많이 발생해서 SSD가 서스테이닝 모드(Sustaing mode)로 들어가면,
SSD의 Garbage-collection이 사용자의 요청을 따라가지 못하게 된다.
사용자의 요청이 들어올때마다 Garbage-collection이 먼저 블록을 지워야(Erase)하기 때문에,
호스트로부터 오는 사용자 요청과 백그라운드로 실행되는 Garbage-collection이 서로 경합하게 된다.
모든 종류의 워크로드에서 SSD 드라이브가 어떻게 반응하고 어떤 성능을 보여주는지를 확인하는 적절한 모델인지 아닌지는 명확치 않지만,
사람들은 SSD가 보여줄 수 있는 최악의 상황을 만들어 내기 위해서 "Pre-conditioning"을 자주 사용한다.

여러 제조사들의 다양한 모델들을 비교하기 위한 공통적인 방법과 SSD에서 가능한 최악의 상태 확인은 필요하다.
하지만 최악의 상황에서 가장 좋은 성능을 내는 SSD가 실제 운영 서버용으로 최적이라는 것을 보장하지는 않는다.
많은 운영 환경에서 SSD는 하나의 시스템에서만 사용된다. 그 시스템은 그 시스템만의 특정한 워크로드를 가지기 때문에,
여러 SSD 드라이브에 대해서 조금 더 정확한 비교를 위해서는 동일한 워크로드로 모두 테스트를 수행해야 한다.
지속적인 랜덤 쓰기를 이용한 "Pre-conditioning"이 여러 종류의 SSD에 대해서 적절한 비교 방법이라 할지라도,
가능하다면 대상 서비스의 워크로드를 직접 구현한 벤치마크로 주의해서 진행해야 하는 것이다.
실제 대상 서비스의 워크로드에 따라서 가장 좋은 성능의 SSD가 최적의 선택이 아닐 수도 있는데,
인하우스(in-house)로 개발된 벤치마크 도구는 이러한 오버 스펙을 방지하여 경제적인 절약 효과를 유도하기도 한다.

> ##### 어려운 SSD 벤치마킹
>
> 결국 테스트도 사람이 수행하는 것이므로, 모든 벤치마킹이 에러를 회피할 수 있도록 해주는 것은 아니다.
> 그러므로 제조사나 써드파티(Third-party)에서 제공하는 벤치마크 결과를 참조할 때는 주의해야 하며,
> 여러 곳으로부터 테스트 결과를 참조하는 것이 좋다.
> 가능하다면 당신의 서비스 워크로드를 잘 표현할 수 있는 인 하우스 벤치마킹 도구로 사용하고자 하는 SSD를 테스트하도록 하자.
> 마지막으로 당신의 시스템이나 서비스에서 필요로 하는 성능 메트릭에 집중하도록 하자.

### 2.3. 워크로드와 메트릭

성능 벤치마크들은 모두 다양하지만 동일한 파라미터를 사용하며, 결과 또한 동일한 메트릭으로 성능 정보를 제공한다.
이번 섹션을 통해서 이러한 파라미터와 결과 메트릭을 해석하는 데 있어서 인사이트를 제공해줄 수 있기를 바란다.

일반적으로 사용되는 파라미터는 다음과 같다:

* 워크로드 타입: 사용자로부터 수집되는 데이터에 기반한 특정 패턴의 벤치마크 또는 동일한 시퀀셜 또는 랜덤 데이터 접근의 일반적인 타입 (예: 랜덤 쓰기만 테스트)
* 읽기와 쓰기를 적정 퍼센트로 배분해서 동시에 읽기 쓰기 부하 발생 (예: 30% 읽기와 70% 쓰기)
* 큐 길이(Queue Length): SSD 드라이브로 읽고 쓰기 명령을 전송하는 동시 쓰레드의 개수를 의미
* 접근(읽고 쓰기)하는 데이터 청크의 크기 (4 KB, 8 KB, 기타..)

벤치마크의 결과는 다른 메트릭으로 표시되는데, 가장 일반적인 메트릭은:

* 스루풋(Throughput): 전송 속도를 나타내며, 일반적으로 KB/s 또는 MB/s를 사용한다. 이 메트릭은 시퀀셜 읽고 쓰기용 벤치마크 결과에 사용된다.
* IOPS: 초당 읽고 쓰기(Input & Ouput) 회수이며, 각 읽고 쓰기 오퍼레이션은 동일한 데이터 청크 크기가 사용(일반적으로는 4KB 청크가 사용됨)된다. 이 메트릭은 랜덤 읽고 쓰기 벤치마크에서 사용된다. [^17]
* 레이턴시(Latency): 입출력 명령이 전달된 후 응답을 받기까지의 시간을 의미하며, 일반적으로는 μs(마이크로 초) 또는 ms(밀리 초)가 사용된다.

스루풋(throughput)은 쉽게 이해되는 반면, IOPS는 조금 감을 잡기가 어려울 수도 있다.
예를 들어서, 디스크가 4KB 청크에 대해서 초당 1000 IOPS를 처리한다면 이를 스루풋(throughput)으로 계산하면 1000 x 4096 = 4 MB/s가 되는 것이다.
결과적으로 가능하면 청크 사이즈가 크면 클수록, 높은 IOPS는 높은 스루풋으로 환산될 수 있는 것이다.

조금 더 이해를 돕기 위해서, 몇 천개의 파일에 아주 조금씩 업데이트하는 로깅 시스템에서 10k IOPS를 처리한다고 가정해보자.
업데이트는 매우 많은 파일에 걸쳐져 있기 때문에 스루풋은 20MB/s정도밖에 안될 것이다.
그런데 만약 이 로깅 시스템이 하나의 파일에만 시퀀셜하게 기록한다고 가정하면, 200MB/s 정도의 향상된 스루풋을 보이게 될 것이다.
이 예제는 랜덤과 시퀀셜 입출력을 비교 설명하기 위해서 만들어낸 수치이긴 하지만, 실제 서비스 환경에서 본인이 경험했던 것이기도 하다.

스루풋과 IOPS의 차이를 이해하기 위한 또 하나의 예는, 높은 스루풋이 반드시 빠른 시스템을 의미하는 것은 아니다.
만약 레이턴시가 높다면, 아무리 스루풋이 높다고 하더라도 전체적인 시스템의 처리는 느린 것일 수 있다.
예를 들어서 25개 데이터베이스 접속해야 하는 단일 쓰레드 프로세스를 생각해보자.
각 컨넥션은 20ms의 레이턴시를 가질 때, 매번 컨넥션을 생성하는 데에 20ms가 소요되므로,
전체 25개의 컨넥션을 생성하는 데에는 25 x 20ms = 500ms가 소요될 것이다.
이때 우리 서버가 아무리 대역폭이 넓은 좋은 네트워크 카드를 장착하고 있다 하더라도,
이 가상의 프로세스는 레이턴시로 인해서 여전히 느리게 처리될 것이다.

적어도 이 섹션에서 기억해야 할 중요한 부분은, 각각의 벤치마킹 메트릭은 시스템의 다른 면을 보여주기 때문에 모든 메트릭에 집중해야 한다는 것이다.
또한 이런 메트릭을 제대로 이해하고 있어야 시스템의 병목 현상이 발생했을 때, 정확한 원인을 찾을 수 있을 것이기 때문이다.
벤치마킹 결과를 분석하고 특정 SSD 모델을 선택할 때, SSD를 장착하고자 하는 시스템에서 어떤 메트릭이 가장 크리티컬한 요소인지를 파악해야 한다.
물론 섹션 2.2에서 소개된 바와 같이 인하우스로 개발된 벤치마킹 도구를 대체할 만한 테스트 도구는 없을 것이다.
Jeremiah Peschka[^46] 가 작성한 “IOPS are a scam”라는 문서도 많은 도움이 될 것이다.


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

