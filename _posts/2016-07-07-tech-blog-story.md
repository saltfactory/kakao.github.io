---
layout: post
title: 'kakao 기술 블로그가 GitHub Pages로 간 까닭은'
author: iolo.fitzowen
date: 2016-07-07 15:25
tags: [github-pages,jekyll,static-site-generator]
image: /files/covers/blog.jpg
---
kakao 기술 블로그는 올해 초 [Ghost] 블로깅 플랫폼을 사용해서 오픈했으나,
최근 [GitHub Pages]와 [Jekyll]으로 바뀌었습니다.
이 글에서는 kakao 기술 블로그를 [GitHub Pages]로 옮기면서 [Jekyll]에 위해 추가한
기능들 - 태그별 포스트 목록 페이지, 작성자별 포스트 목록 페이지, 사이트맵 - 을 소개합니다.
<!--more-->

## Prologue

<img src="http://item.kakaocdn.net/do/-26p06+UqCd0OAgiRHNZwHaq4FJCveCBKCNZV-bZscw_/89caada5a1ddde2795b892cae089b4da1667fc7b08261b4c493670baa83d5cb9" class="hcenter"/>

오픈 초기에는 kakao는 [브런치], [티스토리], [다음블로그], [플레인]까지
여러가지의 블로그스러운(?) 서비스를 하고 있으면서
정작 기술 블로그는 [Ghost]을 사용하는 것에 대해 사내외에서 많은 의문을 제기했었죠.

<img src="http://item.kakaocdn.net/do/-26p06+UqCd0OAgiRHNZwHaq4FJCveCBKCNZV-bZscw_/326692ab97f80a3e1dd36120231d666b1667fc7b08261b4c493670baa83d5cb9" class="hcenter"/>

[티스토리], [브런치], [설치형 WordPress], 자체 개발까지 다양한 솔루션을 검토했지만,
생뚱맞게도 [Ghost]을 사용하기로 했습니다.
[Markdown] 지원, 작성과 발행 권한 분리, 커스텀 디자인
그리고 기능 추가의 용이성(단순 블로그 외에도 많은 기능이 계획되어 있습니다)을 고려했습니다.
가장 결정적인 이유는 운영자의 **개취**였는다는 설도 있습니다만...

<img src="http://item.kakaocdn.net/do/-26p06+UqCd0OAgiRHNZwHaq4FJCveCBKCNZV-bZscw_/01a3b97731f42b4efc929ebd7fa376431667fc7b08261b4c493670baa83d5cb9" class="hcenter"/>

그러나, 운영 과정에서 블로그 외의 기능 개발이 보류되면서
[Ghost]를 선택한 가장 큰 이유였던 **기능 추가의 용이성**은 의미가 없어졌고,
운영자의 **개취**만이 남았습니다.
무엇보다도(!) 기술 블로그 만의 **Geek스러움**이 부족했습니다.

그래서... 바꿨습니다!

## [GitHub Pages] 와 [Jekyll]

[GitHub Pages]는 아시다시피 GitHub 저장소의 내용을 웹 페이지로 서비스해주는 기능입니다.
공짜로 웹 서버를 구축할 수 있지만, 스태틱 컨텐츠만 서비스할 수 있어서 제한적인 용도(예: 뻔한 오픈소스 소개/문서 사이트)에 주로 사용됩니다.
그러나, 여기에 [정적 사이트 생성기]가 결합되면 활용의 폭이 훨씬 넓어집니다.
그 중에서도 [Jekyll]은 [GitHub Pages]가 기본으로 지원하는 정적 사이트 생성기로,
소스(주로 [Markdown])를 깃헙에 올리면 [Jekyll]을 실행해서 정적 사이트를 생성하는 과정을 깃헙 서버가 자동으로 수행하고,
그것을 웹 페이지로 서비스합니다.

자세한 내용은 [GitHub Pages에서 Jekyll을 정적 사이트 생성기로 사용하기](https://help.github.com/articles/using-jekyll-as-a-static-site-generator-with-github-pages)에게 맡기고,
이 글에서는 kakao 기술 블로그를 만들면서 추가한 기능들만 소개하겠습니다.

### 태그별 포스트 목록 페이지

[Jekyll]은 직접적으로 태깅(tagging)을 지원하지 않습니다.
이를 지원하는 여러가지 플러그인들을 테스트 해보았지만, 아쉽게도 [GitHub Pages]와 함께 쓸 수 없는 경우가 많았습니다.
그래서 [Jekyll의 Collections 기능](https://jekyllrb.com/docs/collections/)을 활용해서 간단하게 만들었습니다.

1. `_config.yml` 파일에 다음과 같이 `collections`와 `defaults` 설정을 추가합니다:

```yaml
...
collections:
  tags:
    output: true
    permalink: /tags/:path/
...
defaults:
  - scope:
      path: ''
      type: tags
    values:
      layout: tag
...
```

`_tags` 디렉토리 안에 있는 파일 목록을 수집해서 `/tags/...`로 서비스되도록 HTML 파일을 생성하라는 설정입니다.
(디렉토리 이름 앞의 `_`는 오타가 아닙니다. 설정에는 `_`가 없고, 디렉토리 이름에는 `_`가 있습니다.)

2. `_layouts` 디렉토리에 **태그별 포스트 목록**을 보여줄 템플릿 파일
[`tag.html`](https://raw.githubusercontent.com/kakao/kakao.github.io/master/_layouts/tag.html)을 만듭니다.
참고로, [Jekyll]은 [Liquid] 템플릿을 사용합니다.

{% raw %}
```html
---
layout: default
---
<div id="navbar" class="container">
    <h5><a id="link-back" href="/">Main</a></h5>
    {% include share.html %}
</div>

<div id="cover" class="container"
     {% if page.image %}style="background-image: url({{ page.image }});"{% endif %}>
    <div>
        <h1>{{ page.title | default: page.name }}</h1>
        <p>All Posts in <strong>{{ page.name }}</strong></p>
    </div>
</div>

<div id="content" class="container">
    <div id="tag-content" role="main">
        <ul id="post-list">
            {% for post in site.posts %}
                {% if post.tags contains page.name %}
                    {% include item.html %}
                {% endif %}
            {% endfor %}
        </ul>
        {% include pagination.html %}
    </div>
</div>
```
{% endraw %}

3. `_tags` 디렉토리를 만들고, 그 아래에 각 태그마다
하나의 파일(예: [`opensource.md`](https://raw.githubusercontent.com/kakao/kakao.github.io/master/_tags/opensource.md))을 만듭니다.

```markdown
---
name: opensource
title: 오픈소스
image: /files/covers/opensource.jpg
---
```

[Front Matter]의 내용을 살펴보면:

* `name`: **매칭을 위한 문자열**(일종의 PK)입니다.
* `title`: 태그별 포스트 목록 페이지 상단에 표시되는 것입니다.
* `image`: 태그별 포스트 목록 페이지 상단에 표시되는 것으로 없어도 됩니다.
* `layout`은 생략했으므로, 위에서 `_config.yml`에 설정한 `tag`가 됩니다.
* `permalink`는 생략했으므로, 위에서 `_config.yml`에 설정한 `/tags/:path/`가 됩니다.

> 주의: 새로운 태그가 추가되면 새로운 파일을 만들어줘야 합니다.
>
> 이 부분이 이 글에서 소개하는 태깅 방식의 가장 큰 문제점인데...
> 플러그인 없이 해결하는 방법을 아직 찾지 못했습니다.

**태그별 포스트 목록 페이지**를 생성하는 과정은 다음과 같습니다:

1. `_tags` 디렉토리 아래의 파일목록을 수집해서 템플릿 변수 `site.tags` 에 담아둔다.(`_config.yml`의 `collections` 설정에 `tags`가 있으므로)
2. `_tags` 디렉토리 아래의 파일들(예: `_tags/opensource.md` 등)을 처리해서 HTML 파일을 생성한다.(`_config.yml`의 `collections` 설정에 `output: true` 이므로)
  1. 각 태그 파일의 데이터를 `_layouts/tag.html` 레이아웃 템플릿을 결합해서(`_config.yml`의 `defaults` 설정에 `layout: tag` 이므로)
  2. `tags/opensource/index.html` 등의 파일에 저장한다.(`_config.yml`의 `collections` 설정에 `permalink: /tags/:path/` 이므로)

레이아웃 템플릿(`_layouts/tag.html`)에서 핵심적인 [Liquid] 템플릿 코드를 살펴보면:

{% raw %}
```html
<ul id="post-list">
    {% for post in site.posts %}
        {% if post.tags contains page.name %}
            {% include item.html %}
        {% endif %}
    {% endfor %}
</ul>
```
{% endraw %}

> 모든 포스트 목록(`site.posts`)을 순회하면서,
> 포스트의 태그목록 배열(`post.tags`)에 지정한 태그를(`page.name`)가 포함되어 있으면
> 해당 포스트를 출력

하는 코드가 눈에 들어올 겁니다.
(주의: 여기서 `page` 변수에는 태그 파일의 데이터가 담겨있습니다.)

단순무식한 `O(포스트개수 x 태그개수)` 풀스캔이지만,
이 과정은 정적 사이트 생성 과정에만 이루어지므로 실제 서비스에는 영향이 없습니다.

[Jekyll의 Data Files 기능](https://jekyllrb.com/docs/datafiles/)을 이용하면
좀 더 쉽고 효율적으로 만들 수 있을것 같지만, 플러그인을 만들어야 해서 pass!
(포스트가 수만개, 태그가 수만개라면, 그냥 데이터베이스 기반 블로그를 쓰시는 것이 정신 건강에 좋습니다.)

### 작성자별 포스트 목록 페이지

[Jekyll]은 작성자라는 개념(author attribution)을 지원하지 않습니다.
애초에 흔한 웹UI도 없으니, 로그인도 없고, 작성자라는 개념도 없죠.
그래서 태그 분류하기에서 사용했던 방법을 조금 변형해서 작성자로 분류하기를 만들었습니다.
과정은 태깅과 거의 같습니다.

1. `_config.yml` 파일에 다음과 같이 `collections`와 `defaults` 설정을 추가합니다:

```yaml
...
collections:
  ...
  authors:
    output: true
    permalink: /authors/:path/
...
defaults:
  - scope:
      path: ''
      type: authors
    values:
      layout: author
...
...
```

설명이 필요없겠죠? 태깅과 거의 같습니다. ;)

2. `_layouts` 디렉토리에 **작성자별 포스트 목록**을 보여줄 템플릿 파일
[`author.html`](https://raw.githubusercontent.com/kakao/kakao.github.io/master/_layouts/author.html)을 만듭니다:

{% raw %}
```html
---
layout: default
---
<div id="navbar" class="container">
    <h5><a id="link-back" href="/">Main</a></h5>
    {% include share.html %}
</div>

<div id="cover" class="container"
     {% if page.cover %}style="background-image: url({{ page.cover }});"{% endif %}>
    <div>
        <div id="cover-author-image"
             {% if page.image %}style="background-image:url({{ page.image }});"{% endif %}>
            <span class="sr-only">{{ page.name }}'s profile image</span>
        </div>
        <h1>{{ page.title }}</h1>
        <p>All Posts by <em>{{ page.name }}</em></p>
    </div>
</div>

<div id="content" class="container">
    <div id="author-content" role="main">
        <ul id="post-list">
            {% for post in site.posts %}
                {% if post.author == page.name %}
                    {% include item.html %}
                {% endif %}
            {% endfor %}
        </ul>
        {% include pagination.html %}
    </div>
</div>
```
{% endraw %}

역시나 설명이 필요없겠죠? 태깅과 거의 같습니다. ;)

3. `_authors` 디렉토리를 만들고, 그 아래에 각 사용자마다
하나의 파일(예: [`ryan.md`](https://raw.githubusercontent.com/kakao/kakao.github.io/master/_authors/ryan.md))을 만듭니다.

```markdown
---
name: ryan
title: 라이언
image: /files/authors/ryan.jpg
---
```

**작성자별 포스트 목록 페이지**를 생성하는 과정은 다음과 같습니다:

1. `_author` 디렉토리 아래의 파일목록을 수집해서 템플릿 변수 `site.authors` 에 담아둔다.(`_config.yml`의 collections 설정에 `authors`가 있으므로)
2. `_authors` 디렉토리 아래의 파일들(예: `_authors/ryan.md` 등...)을 처리해서 HTML 파일을 생성한다.(`_config.yml`의 `collections` 설정에 `output: true` 이므로)
  1. 각 태그 파일의 데이터를 `_layouts/author.html` 레이아웃 템플릿을 결합해서(`_config.yml`의 `defaults` 설정에 `layout: author` 이므로)
  2. `authors/ryan/index.html` 등의 파일에 저장한다.(`_config.yml`의 `collections` 설정에 `permalink: /authors/:path/` 이므로)

<img src="/files/blog-copy-paste.jpg" class="hcenter" />

이하 동문 생략...

### 사이트맵

[Ghost]를 선택한 이유 중의 하나가 [검색 엔진 최적화]였습니다.
별다른 설정없이도 페이지에 meta 태그들도 주렁주렁 많이 달아주고,
[사이트맵]도 자동으로 만들어 줍니다.

물론 [Jekyll]에도 [사이트맵]을 지원하는 플러그인들이 많습니다만,
역시 [GitHub Pages]가 발목을 잡습니다.
그래서... 기본 테마에 포함된 `feed.xml`를 참조해서 간단하게 만들었습니다.

{% raw %}
```xml
---
layout: null
---
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="/sitemap.xsl"?>
<urlset xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.sitemaps.org/schemas/sitemap/0.9 http://www.sitemaps.org/schemas/sitemap/0.9/sitemap.xsd"
        xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  {% for post in site.posts %}
  <url>
    <loc>{{ site.url }}{{ site.baseurl }}{{ post.url }}</loc>
    {% if post.date %}
    <lastmod>{{ post.date | date_to_xmlschema }}</lastmod>
    {% endif %}
    <changefreq>weekly</changefreq>
    <priority>1.0</priority>
  </url>
  {% endfor %}
  {% for page in site.pages %}
  {% if page.sitemap %}
  <url>
    <loc>{{ site.url }}{{ site.baseurl }}{{ page.url }}</loc>
    {% if page.date %}
    <lastmod>{{ page.date | date_to_xmlschema }}</lastmod>
    {% endif %}
    <changefreq>{{ sitemap.freq | default: 'monthly' }}</changefreq>
    <priority>{{ sitemap.priority | default: '1.0' }}</priority>
  </url>
  {% endif %}
  {% endfor %}
  {% for author in site.authors %}
  <url>
    <loc>{{ site.url }}{{ site.baseurl }}/authors/{{ author.name }}</loc>
    <changefreq>monthly</changefreq>
    <priority>0.8</priority>
  </url>
  {% endfor %}
  {% for tag in site.tags %}
  <url>
    <loc>{{ site.url }}{{ site.baseurl }}/tags/{{ tag.name }}</loc>
    <changefreq>monthly</changefreq>
    <priority>0.8</priority>
  </url>
  {% endfor %}
</urlset>
```
{% endraw %}

파일 맨 앞에 [Front Matter]는 없어도 있어야 합니다(응?)
포스트(`site.posts`)는 모두 포함되고,
페이지(`site.pages`)중에서 `sitemap`속성이 있는 페이지만 포함됩니다.
그리고, 위에서 만든 태그별 포스트 목록과 작성자별 포스트 목록도 포함됩니다.
차~ㅁ 쉽죠?

[sitemap.xml](https://raw.githubusercontent.com/kakao/kakao.github.io/master/sitemap.xml) 파일은 검색 엔진을 위한 것입니다만,
휴먼을 위한 최소한의 배려로, [Ghost]를 참고해서
[sitemap.xsl](https://raw.githubusercontent.com/kakao/kakao.github.io/master/sitemap.xsl)도 추가했습니다.

그 결과는 [http://tech.kakao.com/sitemap.xml](http://tech.kakao.com/sitemap.xml)에서 확인하실 수 있습니다.

## Epilogue

기업의 기술 블로그에 이런 가벼운 내용이 올라가도 될까...라는 고민을 잠시했지만...
[우아한형제들의 기술 블로그에 김범준 님이 쓰신 글](http://woowabros.github.io/woowabros/2016/06/30/woowabros_cto.html)을 보고 용기내어 올려봅니다.

앞으로도 kakao 기술 블로그 많이 사랑해 주세요. 꾸벅~

<img src="http://item.kakaocdn.net/do/-26p06+UqCd0OAgiRHNZwHaq4FJCveCBKCNZV-bZscw_/46d01f5ff600e23a970e9ce86c02eff81667fc7b08261b4c493670baa83d5cb9" class="hcenter"/>

> 전하는 말씀(or 광고)
>
> kakao의 오픈소스 업무를 담당할 인재를 모십니다.
> [지원하기](http://www.kakaocorp.com/recruit/progressRecruit?uid=9789)

[Ghost]:https://ghost.org
[GitHub Pages]:https://pages.github.com
[Jekyll]:https://jekyllrb.com
[Markdown]:https://guides.github.com/features/mastering-markdown/
[브런치]:https://brunch.co.kr
[티스토리]:http://www.tistory.com
[다음블로그]:http://blog.daum.net
[플레인]:https://plain.is
[설치형 WordPress]:https://wordpress.org
[정적 사이트 생성기]:https://staticsitegenerators.net
[Liquid]:https://shopify.github.io/liquid/
[Front Matter]:https://jekyllrb.com/docs/frontmatter/
[사이트맵]:http://www.sitemaps.org
[검색 엔진 최적화]:https://en.wikipedia.org/wiki/Search_engine_optimization

