---
layout: post
title: '개발자를 위한 SSD (Coding for SSD) - Part 6 : A Summary – What every programmer should know about solid-state drives'
author: matt.lee
date: 2016-07-18 20:00
tags: [ssd,nand-flash,ssd,garbage-collection,LBA,PBA,block,page,clustered-block]
image: /files/covers/solid-state-logic.jpg
cover:
  title: '80-channel Solid State Logic (SSL) XL 9000 K Series Console at Audio Mix House, Studio B'
  link: https://flic.kr/p/j1DcB
  author:
    name: 'Audio Mix House'
    link: https://www.flickr.com/photos/audiomixhouse/
---


이번 챕터에서는 지금까지 언급했던 내용들을 간략히 요약해서 정리해 보았다.
조금 더 상세한 내용을 쉽게 찾아갈 수 있도록, 각 내용들은 다른 파티의 한두개 이상의 섹션을 나열해 두었다.


## SSD 기본

### 1. 메모리 셀 타입

SSD (Solid state drive)는 플래시 메모리를 기반으로 하는 저장 장치이이다.
각 비트들은 셀에 저장되는데, SSD의 셀은 1 비트(SLC), 2 비트(MLC) 그리고 3 비트(TLC) 셀 타입이 있다.

See also: [섹션 1.1](/2016/07/13/coding-for-ssd-part-1/)

### 2. 수명 제한

각 셀은 최대 가능한 P/E(Program/Erase) cycle을 가지며, 최대 가능한 P/E cycle을 초과하면 결함 셀(Defective cell)로 간주된다.
이는 NAND 플래시 메모리는 언젠가는 Wear-off되고 수명이 제한적이라는 것을 의미한다.

See also: [섹션 1.1](/2016/07/13/coding-for-ssd-part-1/)

### 3. 벤치마킹의 어려움

테스트도 결국 사람이 수행하는 것이기 때문에, 모든 벤치마크는 오류나 실수를 담고 있다.
그래서 제조사나 제 삼자에 의한 벤치 마킹 결과를 참조할 때는 주의해야 한다.
SSD를 이용한 벤치마킹을 진행할 경우에는, 실제 사용할 SSD 드라이브와 최대한 응용 프로그램의 워크로드와 비슷한 인하우스 벤치마킹 도구를 이용할 것을 권장한다.
마지막으로 응용 프로그램의 요건(SLA)에 맞는 메트릭에 집중해서 벤치마킹을 할 것을 권장한다.

See also: [섹션 2.2, 2.3](/2016/07/14/coding-for-ssd-part-2/)


## 페이지와 블록

### 4. NAND 플래시 블록과 페이지

SSD의 셀(Cell)은 블록(Block)으로 그룹핑되어 있으며, 블록(Block)은 다시 플레인(Plane)으로 그룹핑된다.
SSD에서 읽고 쓰기의 가장 작은 단위는 페이지(Page)이며, 페이지는 단독으로 삭제(Erase)되지 못하고 블록(Block) 단위로만 삭제될 수 있다.
NAND 플래시 페이지 사이즈는 제품이나 제조사별로 다양한데, 대 부분의 SSD는 2KB와 4KB 그리고 8KB와 16KB를 페이지 사이즈로 사용하고 있다.
또한 대부분의 SSD에서는 하나의 블록은 128개 또는 256개의 페이지를 가지므로, 블록의 사이즈는 256KB에서 4MB까지 다양한 사이즈를 가지게 되는 것이다.
예를 들어서 삼성 SSD 840 EVO는 2048KB의 블록 크기를 가지며, 각 블록은 256개의 8KB 크기 페이지를 가지고 있다.

See also: [섹션 3.2](/2016/07/15/coding-for-ssd-part-3/)

### 5. 읽기는 페이지 단위로 실행(Align)

SSD 드라아브에서는 페이지 하나의 크기보다 작은 사이즈의 데이터를 읽을 수는 없다.
물론 응용 프로그램이나 운영 체제에서는 단 1개의 바이트도 읽을 수는 있지만,
실제 SSD 내부적으로는 하나의 페이지 전체를 읽어서 불필요한 데이터는 모두 버리고 사용자가 원하는 하나의 바이트만 리턴해주는 것일 뿐이다.

See also: [섹션 3.2](/2016/07/15/coding-for-ssd-part-3/)

### 6. 쓰기는 페이지 단위로 실행(Align)

SSD에 쓰기를 할 때에는 SSD의 페이지 크기의 배수로 처리된다. 그래서 단지 1 바이트만 SSD에 기록하는 경우에는 최소 하나의 페이지 크기의 데이터가 기록되어야 한다. 필요 이상으로 데이터를 기록해야 하는 것을 "Write amplication"이라고 하며, SSD 드라이브에 쓰기를 하는 것을 "program"이라고도 한다.

See also: [섹션 3.2](/2016/07/15/coding-for-ssd-part-3/)

### 7. 페이지는 덮어쓰기 안됨

NAND 플래시 페이지는 "free" 상태일때에만 데이터를 저장(program)할 수 있다.
데이터가 변경되면 이전 버전의 페이지는 SSD의 내부 레지스터로 복사된 후 데이터가 변경되고
그리고 새로운 버전의 데이터는 새로운 "free" 상태의 페이지에 기록되는데 이를 "read-modify-update"라고 한다.
또한 SSD에서 변경된 데이터는 기존 위치에 업데이트되지 못하고, 항상 새로운 "free" 상태의 페이지에만 저장될 수 있다.
일단 데이터가 SSD 드라이브에 영구히 저장되면,
이전 버전의 데이터를 가지고 있는 페이지는 "stale" 상태로 마킹되고 Garbage-collection에 의해서 삭제(Erase)될때까지 그 상태로 남아있게 된다.

See also: [섹션 3.2](/2016/07/15/coding-for-ssd-part-3/)

### 8. 삭제(Erase)는 블록 단위로 실행(Aligned)

SSD의 페이지는 덮어쓰기할 수 없다. 일단 페이지가 "stale" 상태로 바뀌면, 그 페이지를 다시 사용하기 위해서는 반드시 삭제(Erase) 과정을 거쳐야 한다.
그러나 페이지는 단독으로 삭제될 수 없으며, 그 페이지를 포함한 블록이 통째로 삭제(Erase)될때에만 "free" 상태로 바뀔 수 있다.

See also: [섹션 3.2](/2016/07/15/coding-for-ssd-part-3/)


## SSD 컨트롤러와 인터널

### 9. FTL (Flash Translation Layer)

FTL(Flash translation layer)는 호스트의 논리 블록 주소를 SSD의 물리 블록 주소로 맵핑해주는 SSD 컨트롤러의 컴포넌트이다.
최근의 SSD 드라이브는 대 부분 Log structured 파일 시스템과 비슷한 작동 방식을 가진 “hybrid log-block mapping”나 그로부터 파생된 블록 맵핑 알고리즘을 사용하고 있다.
“hybrid log-block mapping” 맵핑 알고리즘은 랜덤 쓰기를 시퀀셜하게 변환해주는 장점을 가지고 있다.

See also: [섹션 4.2](/2016/07/16/coding-for-ssd-part-4/)

### 10. 내부 병렬 처리

SSD 드라이브는 내부적인 병렬 처리 기능을 가지고 있는데, 이 병렬 처리 기능은 NAND 플래시 칩 단위로 여러 블록들에 동시에 쓰기할 수 있도록 해준다.
이렇게 한번에 병렬로 액세스할 수 있는 블록들의 그룹을 "Clustered block"이라고 한다.

See also: [섹션 6](/2016/07/18/coding-for-ssd-part-6/)

### 11. Wear leveling

NAND 플래시 셀은 데이터 쓰기를 할 수 있는 회수 제한이 있어서, 이 제한을 넘어서면 해당 셀은 더 이상 사용할 수 없게 된다.
이런 현상을 "Wear-off"라고 하는데, FTL은 최대한 SSD의 셀들이 최대한 골고루 사용될 수 있도록 쓰기 작업을 분산해준다.
이상적으로는 SSD의 모든 셀들이 동시에 그들의 P/E cycle 한계에 도달하도록 하는 것이 FTL의 중요한 목표인 것이다.

See also: [섹션 3.4](/2016/07/15/coding-for-ssd-part-3/)

### 12. Garbage collection

SSD 컨트롤러의 Garbage-collection은 "stale" 상태의 페이지를 삭제하여 다시 사용 가능한 "free" 상태로 만들어준다.
"stale" 상태의 페이지는 "free" 상태로 전환되지 않고서는 재사용될 수 없다.

See also: [섹션 4.4](/2016/07/16/coding-for-ssd-part-4/)

### 13. 백그라운드 오퍼레이션은 유저 오퍼레이션에 영향을 미침

Garbage-collection과 같은 백그라운드 오퍼레이션은 호스트로부터 유입되는 사용자 요청(포그라운드 오퍼레이션)에 좋지 않은 영향을 미치게 된다.
대표적으로 지속적인 작은 랜덤 쓰기가 발생하는 상황에서는 더욱 더 성능상 나쁜 영향을 미치게 된다.

See also: [섹션 4.4](/2016/07/16/coding-for-ssd-part-4/)


## 액세스 패턴

### 14. SSD는 페이지 크기보다 작은 데이터만 쓰기할 수 없음

"Read-modify-write" 오퍼레이션으로 인한 "Write amplication"을 최소화하기 위해서,
가능하다면 NAND 플래시의 페이지 크기로 SSD 드라이브에 쓰기를 하는 것이 좋다.
현재까지 출시된 SSD 제품들중에서 가장 큰 페이지 사이즈는 16KB이므로, 가능하면 16KB 정도를 기본 쓰기 데이터 사이즈로 선택하는 것이 좋다.
SSD의 페이지 사이즈는 SSD 제조사나 모델별로 상이하며, 또한 SSD 기술 발전으로 더 크게 증가할 수도 있다.

See also: [섹션 3.2, 3.3](/2016/07/15/coding-for-ssd-part-3/)

### 15. 쓰기 맞춤 (Align writes)

SSD 드라이브에 쓰기를 할 때에는 페이지 사이즈에 맞춰서(Align) 호출하는 것이 좋으며, 페이지 사이즈의 배수로 데이터를 기록하는 것은 더더욱 권장한다.

See also: [섹션 3.2, 3.3](/2016/07/15/coding-for-ssd-part-3/)

### 16. 작은 데이터 쓰기는 버퍼링후 실행

최대의 스루풋을 위해서, 가능한 작은 사이즈의 데이터 쓰기는 메모리에 모아서 버퍼링했다가 버퍼가 가득 차면 SSD 드라이브로 기록하는 것이 좋다.
작은 데이터 쓰기들을 모아서 배치로 한번의 쓰기 요청으로 처리하는 것이 좋다.

See also: [섹션 3.2, 3.3](/2016/07/15/coding-for-ssd-part-3/)

### 17. 읽기 성능 향상을 위해서, 관련된 데이터는 한번에 같이 저장

읽기 성능은 쓰기 패턴의 결과라고 볼 수 있다.
만약 대용량의 데이터가 한번에 SSD 드라이브에 쓰여진다면, 그 데이터들은 NAND 플래시 칩의 여러 위치로 분산 저장된다.
그래서 연관된 데이터는 동일 페이지와 블록 그리고 Clustered block에 기록하는 것이 좋다.
그래야지만 나중에 (SSD의 내부 병렬 처리의 도움을 받아서) 한번의 I/O 요청으로 데이터를 빠르게 읽을 수 있기 때문이다.

See also: [섹션 5.3](/2016/07/17/coding-for-ssd-part-5/)

### 18. 읽기와 쓰기 요청은 분리 실행

읽기와 쓰기가 동시에 실행되는 워크로드에서는 SSD 내부적인 캐싱과 Readahead 메커니즘의 효과를 얻기 어려우며, 이로 인해서 스루풋이 떨어질 수도 있다.
이렇게 읽고 쓰기가 혼합된 형태의 워크로드는 가능하다면 읽기와 쓰기를 대용량(가능하다면 Clustered block 크기에 맞춰서)으로
순차적으로 실행할 수 있도록 유도하는 것이 좋다.
예를 들어서 1000개의 파일을 읽고 변경해야 한다면, 하나씩 파일을 읽어서 변경하는 형태보다
한번에 1000개의 파일을 읽어서 처리한 후 1000개의 파일을 변경하는 형태가 더 SSD의 처리 효울성에는 도움이 된다.

See also: [섹션 5.4](/2016/07/17/coding-for-ssd-part-5/)

### 19. 불필요한 데이터는 배치로 삭제 (Invalidate obsolete data in batch)

SSD 드라이버의 데이터가 더이상 필요치 않아서 삭제하고자 할 때에는, 삭제 요청을 일정량 모아서 한번의 오퍼레이션으로 삭제하는 것이 좋다.
이는 SSD 컨트롤러의 Garbage-collection이 한번에 더 큰 영역을 처리할 수 있도록 해주며, 그로 인해서 SSD의 내부적인 프레그멘테이션을 최소화할 수 있다.

See also: [섹션 4.4](/2016/07/16/coding-for-ssd-part-4/)

### 20. 랜덤 쓰기가 Random writes are not always slower than sequential writes

SSD 드라이브에 기록하고자 하는 데이터가 작다면(SSD의 Clustered block 보다 작은 경우), 랜덤 쓰기는 시퀀셜 쓰기보다 느리게 처리될 것이다.
만약 쓰기가 Clustered block 크기와 일치하거나 Clustered block의 크기의 N배수(정확하게)인 경우에는,
랜덤 쓰기라 하더라도 SSD의 내부 병렬 처리 능력을 활용할 수 있게 된다.
랜덤 쓰기라 하더라도 SSD의 병렬 처리 능력을 활용할 수 있다면 시퀀셜 쓰기만큼의 스루풋을 얻을 수 있다.
대 부분의 SSD 드라이브에서 Clustered block의 사이즈는 16MB에서 32MB 정도이므로,
32MB 정도의 데이터를 한번에 쓰기할 수 있으면 최적의 쓰기 성능을 얻을 수 있다.

See also: [섹션 5.2](/2016/07/17/coding-for-ssd-part-5/)

### 21. 단일 쓰레드로 대량 읽기가 멀티 쓰레드의 소량 데이터 읽기보다 나은 스루풋 보장

동시에 여러 쓰레드로 실행되는 읽기 요청은 SSD 드라이브의 Readahead 메커니즘을 100% 활용하기 어렵다.
게다가 한번에 여러 논리 블록 주소를 액세스하는 것은 결국 하나의 NAND 플래시 칩으로 집중될 수 있고,
이런 경우에는 SSD의 내부 병렬 처리 능력을 활용하지 못하게 된다.
연속된 블록을 대용량으로 읽는 오퍼레이션은 SSD 드라이브의 Readahead 버퍼를 활용(SSD 드라이브가 Readahead cache를 가지고 있는 경우)할 수 있으며,
또한 내부적인 병렬 처리 기능까지 활용할 수 있다.
결과적으로 가능하다면 요청을 버퍼링했다가 한번에 큰 데이터를 읽는 형태가 스루풋 향상에 도움이 된다.

See also: [섹션 5.3](/2016/07/17/coding-for-ssd-part-5/)

### 22. 단일 쓰레드로 대량 쓰기가 멀티 쓰레드의 소량 데이터 쓰기보다 나은 스루풋 보장
단일 쓰레드로 대용량의 데이터를 쓰는 것은 많은 쓰레드로 동시에 작은 데이터를 쓰는 것 만큼의 스루풋을 얻을 수 있다.
하지만 단일 쓰레드로 대용량의 데이터를 기록하는 것이 레이턴시(응답 속도) 측면에서는 여러 쓰레드로 동시에 쓰기를 하는 것보다 빠르게 처리된다.
그래서 가능하다면, 단일 쓰레드로 대량의 데이터를 한꺼번에 기록하는 것이 좋다.

See also: [섹션 5.2](/2016/07/17/coding-for-ssd-part-5/)

### 23. 소량 데이터 쓰기를 버퍼링하거나 그룹핑할 수 없을 때는, 멀티 쓰레드로 실행

멀티 쓰레드로 작은 데이터를 아주 빈번하게 쓰기 요청을 실행하는 것은 단일 쓰레드로 작은 쓰기를 실행하는 것보다는 더 나은 스루풋을 낼 것이다.
그래서 작은 데이터 쓰기가 배치 형태로 묶어서 처리될 수 없다면, 멀티 쓰레드로 쓰기를 실행하는 것이 좋다.

See also: [섹션 5.2](/2016/07/17/coding-for-ssd-part-5/)

### 24. 콜드 데이터와 핫 데이터 분리

핫(Hot) 데이터는 빈번하게 읽고 변경되는 데이터를 의미하며, 반대로 콜드(Cold) 데이터는 그렇지 않은 데이터를 말한다.
만약 핫 데이터가 콜드 데이터와 함께 동일 페이지에 저장되어 있다면,
콜드 데이터는 핫 데이터가 Read-modify-write 형태로 변경될 때마다 항상 같이 복사되어야하며 또한 Wear-leveling을 위해서
Garbage-collection이 해당 페이지를 이동시킬 때에도 동일하게 콜드 데이터가 항상 같이 다른 페이지로 이동되어야 한다.
콜드 데이터와 핫 데이터를 분리하는 것은 Garbage-collection이 좀 더 빠르고 효율적으로 처리될 수 있도록 도와줄 것이다.

See also: [섹션 4.4](/2016/07/16/coding-for-ssd-part-4/)

### 25. 핫 데이터는 버퍼링해서 배치로 업데이트

매우 빈번하게 변경되는 핫 데이터나 메타 데이터는 최대한 시스템의 메모리에 캐시되거나 버퍼링될 수 있도록 유지하고,
SSD 드라이브에 변경되는 빈도를 최소화해주는 것이 좋다.

See also: [섹션 4.4](/2016/07/16/coding-for-ssd-part-4/)


## 시스템 최적화

### 26. PCI Express 와 SAS 인터페이스는 SATA 인터페이스보다 빠르다

최근의 SSD 드라이버는 일반적으로  SATA 3.0(550MB/s)과 PCI Express 3.0(채널당 1GB/s, 모델별 차이는 있지만 일반적으로는 다중 채널 채택)을 지원하고 있다.
SAS(Serial Attached SCSI) 인터페이스는 일부 엔터프라이즈 SSD 모델에 장착되긴 하지만, 일반적이지는 않다.
최근 버전에서는 PCI Express와 SAS 인터페이스가 SATA 보다는 훨씬 뛰어난 전송 속도를 보이지만, 조금 비싼 편이다.

See also: [섹션 2.1](/2016/07/14/coding-for-ssd-part-2/)

### 27. Over-provisioning은 Wear-leveling과 성능 향상에 많은 도움이 된다.

SSD 드라이브는 최대 물리 용량보다 더 작은 용량으로 파티션을 생성함으로써 Over-provisioning 공간을 할당할 수 있다.
남은 공간은 사용자나 호스트에게는 보이지 않지만 SSD 컨트롤러는 여전히 남은 물리 공간을 활용할 수 있다.
Over-provisioning 공간은 NAND 플래시 셀의 제한된 수명을 극복하기 위한 Wear-leveling이 좀 더 원활하게 처리될 수 있도록 도와준다.
데이터 쓰기가 아주 빈번한 경우에는 성능 향상을 위해서 전체 용량의 25% 정도의 공간을 Over-provisioning 공간을 할당하는 것이 좋으며,
그렇게 쓰기 부하가 심하지 않은 경우에는 10%~15% 정도의 공간을 Over-provisioning 공간으로 할당해주는 것이 좋다.
Over-provisioning 공간은 NAND 플래시 블록의 버퍼처럼 작동하기도 하는데,
이는 일시적으로 평상시보다 높은 쓰기 부하가 발생하더라도 Garbage-collection이 충분한 "free" 상태 페이지를 만들어 낼수 있도록 완충 역할을 해준다.

See also: [섹션 5.2](/2016/07/17/coding-for-ssd-part-5/)

### 28. TRIM 명령의 활성화

운영 체제의 커널과 파일 시스템이 TRIM 명령을 지원하는지 확인하도록 하자.
TRIM 명령은 운영 체제나 파일 시스템 레벨에서 삭제된 블록의 정보를 SSD 컨트롤러에게 알려주는 역할을 한다.
SSD 컨트롤러는 새로운 쓰기 요청들을 위해서, 시스템이 한가한 시간에 Garbage-collection을 실행하는데,
TRIM 명령으로 삭제된 블록 정보를 SSD 컨트롤러가 받을 수 있으면 Garbage-collection의 삭제(Erase) 작업이 훨씬 효율적으로 처리될 수 있다.

See also: [섹션 5.1](/2016/07/17/coding-for-ssd-part-5/)

### 29. 파티션 얼라인 (Align the partition)

쓰기 요청이 NAND 플래시의 물리 메모리와 맞춰지려면(Alignment),
SSD 드라이브를 파티셔닝하고 포맷할때 NAND 플래시의 페이지 크기와 일치(Align)시켜야 한다.

See also: [섹션 5.1](/2016/07/17/coding-for-ssd-part-5/)


## 결론

지금까지 "Coding for SSDs" 시리즈의 내용을 간단히 요약했다.
SSD에 대한 개인적인 연구를 통해 배운 것들을 누구나 쉽게 이해할 수 있도록 전달되었기를 희망한다.
이 시리즈를 읽고 더 상세히 SSD에 대해서 공부하기를 원한다면,
챕터 2부터 5까지의 게시물 하단에 냐열된 레퍼런스 문서들은 아마도 훌륭한 가이드가 될 수 있을 것으로 생각한다.
USENIX에서도 매년 SSD에 관련된 놀라운 연구들이 진행되고 있으므로,
USENIX FAST 컨퍼런스(파일시스템과 스토리지 기술에 관련된 USENIX 컨퍼런스) 웹 사이트도 많은 도움이 될 것을 생각된다.


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
