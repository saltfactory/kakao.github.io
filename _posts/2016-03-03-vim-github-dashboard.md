---
layout: post
title: '그래, 가끔 "Vim에서" GitHub을 보자!'
author: jg.choi
date: 2016-03-03 10:45
tags: [vim,vim-plugin,vim-github-dashboard]
image: /files/covers/dashboard.jpg
---
`vimrc` 건드리기 좋은 목요일입니다. ;)

기술 블로그 담당자가 글을 내놓으라고 닥달하니, 예전에 만들었던 플러그인이나 한번 꺼내볼까 합니다:

https://github.com/junegunn/vim-github-dashboard

![vim-github-dashboard 실행 화면](/files/vim-ghd-1.png)

Vim 상에서 [GitHub API](https://developer.github.com/v3/)를 이용해 dashboard 페이지를 보여주는 플러그인입니다. 왜 멀쩡한 브라우저를 놔두고 이런 짓을 한 것이냐 물으신다면 ... 그것 참 좋은 질문이네요.

Vimscript 만 가지고는 API 결과를 받아오는 것이 불가능하므로 [Ruby interface](https://github.com/vim/vim/blob/master/runtime/doc/if_ruby.txt)를 이용합니다만 (`:help ruby`) OS X 의 시스템 디폴트 Vim 에서 기본적으로 지원하기 때문에 사용하시는데 문제는 없을 겁니다.

제공하는 커맨드는 다음과 같습니다.

- `GHDashboard[!] [user]`
    - 특정 유저의 dashboard 화면
- `GHActivity[!] [user|repo]`
    - 특정 유저 혹은 repository 의 활동 내역

Public GitHub 의 경우 조회만 하는 경우는 인증이 필요 없기 때문에 느낌표를 붙여서 실행하시면 되겠습니다.

```vim
:GHD! junegunn
```

![GHD 명령으로 사용자의 대시보드 보기](/files/vim-ghd-2.png)

`CTRL-N` / `CTRL-P` 로 링크 사이를 이동할 수 있고, `Enter` key 나 `o`를 누르면 해당 페이지가 브라우저에서 열립니다.

매번 본인의 ID 를 입력하는 것이 번거롭다면 다음과 같은 설정을 vimrc 에 추가하세요.

```vim
let g:github_dashboard = { 'username': 'junegunn' }
```

GHA 커맨드도 마찬가지 방식으로 사용합니다.

```vim
:GHA! torvalds
```

![GHA 명령으로 torvalds의 활동 보기](/files/vim-ghd-3.png)

[Linus](https://github.com/torvalds) 선생님이 무얼하며 지내시는지 볼 수도 있고요,

```vim
:GHA! torvalds/linux
```

![GHA 명령으로 linux 프로젝트의 활동 보기](/files/vim-ghd-4.png)

[Linux](https://github.com/torvalds/linux)에 무슨 일들이 벌어지고 있는지도 간단히 확인할 수 있습니다.

사내에서 사용하는 [GitHub Enterprise](https://enterprise.github.com)에 접근하려면 프로파일을 지정해야 하는데요. `g:github_dashboard#프로파일명` 의 변수를 선언하면 `GHD -프로파일명` 과 같은 형태로 사용하실 수 있습니다.

다음과 같은 식으로 `foo` 프로파일을 정의하면 됩니다.

```vim
let g:github_dashboard#foo = {
  \ 'username':     'your-github-enterprise-username',
  \ 'password':     ACCESS_TOKEN,
  \ 'api_endpoint': 'https://your-github-enterprise-host-name/api/v3',
  \ 'web_endpoint': 'https://your-github-enterprise-host-name' }
```

(Access token 은 https://your-github-enterprise-host-name/settings/applications 에서 발급 가능)

이제 `foo` 프로파일을 사용하여 `bar` 사용자의 `baz` 프로젝트에서는 무슨 일이 벌어지고 있나 보려면 다음과 같이 하시면 됩니다. 인증이 필요하므로 `!` 이 없는 커맨드를 실행합니다.

```vim
:GHA -foo bar/baz
```

멀쩡한 브라우저 놔두고 왜 vim에서 이런 짓(?)을 하냐는 최초의 질문에 대한 답변은 Emacs 아저씨의 [말씀](https://stallman.org/articles/on-hacking.html)으로 대신하도록 하죠.

>Playfully doing something difficult,
>whether useful or not,
>that is hacking.

그래도 굳이 사용해야 할 이유를 찾아 보자면, 내가 아닌 다른 사람의 대시보드 화면이나 특정 저장소의 활동 정보를 목록으로 볼 수 있는 페이지가 GitHub 에 없기 때문에 그런 용도로 사용해 보실 수는 있겠습니다.

>이 글은 [ranked.in](http://rankedin.kr) 선정 [한국 개발자 인기 1위](http://rankedin.kr/users), [한국 저장소 인기 1위](http://rankedin.kr/repos) 2관왕에 빛나는 "빔신" [jg.choi](http://junegunn.kr)가 사내 게시판에 올린 글을 저자의 동의를 얻어 옮긴 것입니다. [저자의 깃헙](https://github.com/junegunn)을 방문하시면 다양한 vim 플러그인과 유틸리티들, 그리고 [빔신의 vimrc](https://github.com/junegunn/dotfiles/blob/master/vimrc)를 보실 수 있습니다.

* 커버 이미지 출처: [deciphering your dashboard (365-96)](https://flic.kr/p/9wA9A5) © [Robert Couse-Baker](https://www.flickr.com/photos/29233640@N07/)