---
layout: post
title: 'Weekly Links #2 - 2016년 4월 넷째주'
author: iolo.fitzowen
date: 2016-04-27 10:04
tags: [weekly,links]
image: /files/covers/links.jpg
---
**[Weekly Links](http://tech.kakao.com/tags/weekly/)**에서는 지난 한 주, 카카오의 기술 블로그 담당자가 구독하는 기술 뉴스레터들에서 "인간의 눈"으로 선별한 링크들을 짧은 코멘트와 함께 공유합니다.

> 포함된 뉴스레터 목록은 [awesome-tech-newsletters](https://github.com/kakao/awesome-tech-newletters)에서 확인하실 수 있습니다.

## 2016년 4월 넷째주 추천 링크

#### [Node.js v6 출시](https://nodejs.org/en/blog/release/v6.0.0/)

[V8](https://developers.google.com/v8/)엔진을 5.0으로 업데이트해서 [ES6 문법의 93%를 지원](http://node.green/)한다는 군요. [v5와 v6로의 변경사항](https://github.com/nodejs/node/wiki/Breaking-changes-between-v5-and-v6)이 꽤 길지만, 눈에 띄는 것은 없습니다. 물론 기존 v4은 2017년 4월까지 계속 지원(LTS; Long Term Support).

<img src="http://item-kr.talk.kakao.co.kr/do/-26p06+UqCd0OAgiRHNZwHaq4FJCveCBKCNZV-bZscw_/89caada5a1ddde2795b892cae089b4da1667fc7b08261b4c493670baa83d5cb9" class="pull-right" />
#### [윈도10 속 우분투 배시](http://www.zdnet.co.kr/news/news_view.asp?artice_id=20160425093205)

링크는 지디넷의 요약 기사구요, 영어 [원문](https://blogs.msdn.microsoft.com/wsl/2016/04/22/windows-subsystem-for-linux-overview/)은 매우 길기 때문에... 꼭 보실 분 만 보세요. 리스닝에 자신 있으시면 [동영상](https://channel9.msdn.com/Blogs/Seth-Juarez/Windows-Subsystem-for-Linux-Architectural-Overview)도 있습니다.

요약하면, 리눅스에서 [WINE](https://www.winehq.org)이 하는 짓(?)과 비슷한 짓(!)을 한다~입니다. WINE은 윈도의 수많은 [Undocumented API ](http://undoc.airesoft.co.uk) 때문에 고통받았었는데 좀 불공평?!

#### MySQL 모니터링 시리즈
- 1부. [성능 메트릭]( https://www.datadoghq.com/blog/monitoring-mysql-performance-metrics/)
- 2부. [수집 & 통계](https://www.datadoghq.com/blog/collecting-mysql-statistics-and-metrics/)
- 3부. [DataLog 광고](https://www.datadoghq.com/blog/mysql-monitoring-with-datadog/)

결론은 광고지만, 거기까지 가는 과정에서 유익한 내용이 많습니다. 이런 광고라면 고맙죠~

#### [개발자의 도서관: 코드를 사랑하는 사람에게 보물같은 책들](https://medium.com/javascript-scene/the-software-developer-s-library-a-treasure-trove-of-books-for-people-who-love-code-f9bc92c7883b#.mv67vc9dq)

[뷰티플코드](http://book.daum.net/detail/book.do?bookid=BOK00000963491AL), [디자인패턴](http://book.daum.net/detail/book.do?bookid=ENG9780201633610), [클린코드](http://book.daum.net/detail/book.do?bookid=ENG6100132350883), [컴파일러스](http://book.daum.net/detail/book.do?bookid=ENG9780321486813)(드래곤북??)... 주옥같은 책들이 많습니다. 안타까운 건... 번역이 안된 책도 많고, 그나마도 절판된 책이 많다는 것...

<img src="http://item-kr.talk.kakao.co.kr/do/-26p06+UqCd0OAgiRHNZwHaq4FJCveCBKCNZV-bZscw_/2a4518420a8d564ceceb58f1ef335add1667fc7b08261b4c493670baa83d5cb9" class="pull-right" />
#### [2016 전세계 소프트웨어 개발자 연봉 설문 조사 by 오라일리](https://www.oreilly.com/ideas/2016-software-development-salary-survey-report)

국가, (미국내)지역, 나이, 업무... 등 다양한 기준으로 결과를 보여주는데요, 미국 평균 10만불, 아시아 평균 3만불... 흠... 차이가 많이 나네... 아프리카 평균 5만불?! OTL

#### [웹사이트가 느려지면 얼마나 많은 사용자가 포기하는가?](https://plumbr.eu/blog/user-experience/performance-causing-users-abandon-site)

요약하면, 2초면 3%, 4초면 6%, 32초면 34%가 포기~한다는 군요. 생각보다 오래 버틴다고 생각하는 건 저 뿐인가요?

같은 블로그의 이전 글 [2016년의 웹 어플리케이션은 얼마나 빨라졌을까?](https://plumbr.eu/blog/user-experience/how-fast-are-web-applications-in-2016)도 재미있네요. 2초 내에 끝내지 못하면... 당신의 웹은 하위 8.5% ㄷㄷㄷ 열심히 해야겠습니다.
#### [알아두면 좋은 22가지 모바일 통계](https://www.soasta.com/blog/22-mobile-web-performance-stats/)

다양한 모바일 관련 통계들을 보여주는데요, 개인적으로 가장 인상적인 통계는: **2015년에 모바일 상거래가 115조 달더인데, 데스크탑 52%, 태블릿 33,% 모바일 15%**!

#### [Visual Studio Code 1.0 출시](http://code.visualstudio.com/blogs/2016/04/14/vscode-1.0)

자사의 제품인 Visual Studio에서 직접 지원하지 못하는 언어/플랫폼을 Code를 통해 지원하려는 전략일까요?  Node.js, Go, C++, PHP, Python... 점점 더 강력해지고 있습니다. 개인적으로는 시간이 지면서 느려지는 문제 때문에 당분간은 젯브레인에 머물러 있을 듯 ;)

#### [MicroPython - 마이크로 컨트롤러 & 전용 파이썬](http://www.micropython.org)

**오픈 하드웨어 + 오픈 소프트웨어** 전용 마이크로 컨트롤러는 20유료~ 정도. 뭔가 재밌는 걸 해볼 수 있을 것 같지만... 지금도 집에서 놀고 있는 아두이노와 라스베리파이와...

<img src="http://item-kr.talk.kakao.co.kr/do/-26p06+UqCd0OAgiRHNZwPf1+nqjcFZi42Z3wogPJ3I_/e72b7ae2f96032e9fd3ff7778cbe169a1667fc7b08261b4c493670baa83d5cb9" class="pull-right" />
#### [swift 안드로이드 포트 풀리케가 보름만에 마스터에 머지](https://github.com/apple/swift/pull/1442)

요즘 한창인 프로그래밍언어 전쟁에서 swift가 거침없이 진격하고 있네요. 굴뚝에 연기가 나는 걸 보니... 누가 불을 때고 있긴 한가 본데...

#### [Kotlin 1.0 이후 계획](http://blog.jetbrains.com/kotlin/2016/04/kotlin-post-1-0-roadmap/)

[Kotlin](https://kotlinlang.org)이 [구글의 평가](http://thenextweb.com/dd/2016/04/07/google-facebook-uber-swift/#gref)에 자극받았는지, [안드로이드 지원 계획](http://blog.jetbrains.com/kotlin/2016/03/kotlins-android-roadmap/)을 발표한데 이어서, 1.0 이후 계획도 발표했습니다. IMHO, 로드맵이 너무 현실적이다 못해 소박하네요.

#### [Rust + nix = 더 쉬운 유닉스 시스템 프로그래밍](http://kamalmarhubi.com/blog/2016/04/13/rust-nix-easier-unix-systems-programming-3/)

[nix](https://github.com/nix-rust/nix)는 다양한 unix의 시스템콜을 래핑한 [rust](https://www.rust-lang.org/) 라이브러리입니다. 개인적으로 C/C++의 대체자로 가장 유력한 후보는 rust가 아닐까 생각하고 있습니다만.... 돌 던지지 마세요!

#### [React Native로 페이스북 F8 앱 만들기](http://makeitopen.com)

페이스북이 이번 F8 컨퍼런스의 공식 앱을 [React Native](http://facebook.github.io/react-native/)로 만들었는데, 이걸 [오픈소스로 공개](https://github.com/fbsamples/f8app/)했고, 만든 과정까지 친절하게 정리했습니다. 예제의 수준이 놀라울 따름~

마이크로소프트도 [윈도에서 구동되는 React Native](https://blogs.windows.com/buildingapps/2016/04/13/react-native-on-the-universal-windows-platform/)를 공개했습니다.

자바가 못 이룬 WORA(Write Once Run Anywhere)의 꿈이 react의 LOWA(Learn Once Write Anywhere)로 이어지는 듯...

## 특집: [NDC 2016 발표 자료 모음](https://t.co/5UFksN5a6v)

넥슨의 개발자 컨퍼런스 [NDC 2016](https://ndc.nexon.com/main)
이 글을 쓰는 지금도 NDC가 한창인데요, [김기웅](https://twitter.com/KayKimTwit)님께서 벌써부터 발표 자료들을 정리해주고 계십니다.
타임라인에 올라오는 발표자료나 후기들 보고 있으니, 어떻게든 표를 구해볼껄~하는 아쉬움이...
마지막 발표자가 무려! Effective C++의 저자 [Scott Meyers](http://www.aristeia.com)~

분야가 분야인지라... 고퀄의 슬라이드들이 넘쳐나고 있는데요, 제 눈에 띄는 짤 두개만 소개하면:

[![프로그래밍 언어들의 '잘못되었을때'에 대한 특성 맵](/files/links-ndc2016-1.jpg)](https://twitter.com/debuggerD/status/724772753325191168)

[![사실은 매일매일이 도전이긴 합니다](/files/links-ndc2016-2.jpg)](https://twitter.com/elore/status/725133232870580224)

BTW, 카카오의 개발자 컨퍼런스는 언제 쯤?

매주 쓸 수 있을까... 걱정은 했었지만... 시작하자마자 펑크라니... ㅠㅠ 핑계를 대자면, 지난 주에는 겨우내 준비했던 카카오의 앱들이 줄줄이 출시되면서 본업(?)으로 정신없이 바빴습니다. 그래도!! Weekly Links는 쭈욱~~ 계속되겠죠?

> 포함된 뉴스레터 목록은 [awesome-tech-newsletters](https://github.com/kakao/awesome-tech-newletters)에서 확인하실 수 있습니다.

* 커버 이미지 출처: [link by link...](https://flic.kr/p/KjJMP) &copy; [Carsten Tolkmit](https://www.flickr.com/photos/laenulfean/)
