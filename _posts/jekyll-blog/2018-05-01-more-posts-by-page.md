---
layout: post
comments: true
title:  "Jekyll 블로그에서 포스트 안에 같은 카테고리 글 목록 추가"
subtitle: "Jekyll 블로그에서 같은 카테고리에 있는 다른 글들의 목록을 보여주는 방법에 대해서 다룹니다."
description: "Jekyll 블로그에서 같은 카테고리에 있는 다른 글들의 목록을 보여주는 방법에 대해서 다룹니다."
date:   2018-05-01 20:45:33 -0400
background: '/img/posts/2018-5-1/4.png'
image: '/img/posts/2018-5-1/4.png'
categories: [Jekyll-Blog]
---


## Jekyll 블로그에서 포스트 안에 같은 카테고리 글 목록 추가
Jekyll 블로그의 좋은 점이 많지만, 아쉬운 점 하나는 페이지 하단에 같은 카테고리의 다른 글들의 목록이 아래처럼 나오지 않는 점입니다.
![그림4](/img/posts/2018-5-1/4.png)


물론 테마에 따라 해당 기능을 지원하는 테마도 있겠지만, 제가 사용하는 [clean blog 테마](https://github.com/BlackrockDigital/startbootstrap-clean-blog-jekyll){:target="_blank"}에서는 따로 지원을 안해서 만들어보겠습니다.

여러가지 디자인이 있겠지만 아래처럼 `tistory 블로그`의 디자인이 깔끔해보여서 똑같이 만들어보겠습니다. 참고로 `categories`를 별도로 사용 중일 때 가능합니다. Jekyll에 내장된 categories를 이용하셔도 되고, 저의 경우는 [jekyll-category-pages]( https://github.com/field-theory/jekyll-category-pages){:target="_blank"} 플러그인을 별도로 사용 중입니다. 하지만 Jekyll에서 지원하는 플러그인이 아닌 다른 플러그인을 사용하면, github에 다른 방식으로 push 해야하는데, 방법은 [다른 포스팅](/jekyll-blog/2018/05/02/apply-custom-plugin.html){:target="_blank"}에 적어놓았습니다.

### 1. HTML 작성
우선 HTML 먼저 작성하겠습니다. `_includes` 폴더나 `_layouts` 폴더에 있는 자신의 `post`의 기본 템플릿을 열어 원하는 위치에 아래 내용을 추가합니다.

**'{.'에서 '.'은 빼고 입력해주세요!**
```html
  <div class="more-posts">
    <div class="more-category">
      <!-- 하단의 더보기의 링크는 자신의 사이트에 맞게 수정 -->
      <h4>'{{ page.categories }}' 카테고리의 다른 글</h4>
      <a href="/category/{.{ page.categories }}/index.html" class="more-button">더보기</a>
    </div>
    <table>
      <tbody>
        <!-- if문 도는 횟수 체크하기 위해 변수 선언 -->
        {.% assign count = 0 %}
        {.% for post in site.posts %}
          <!-- 전체 포스트의 카테고리가 현재 들어와 있는 페이지의 카테고리와 같은지 판단-->
          {.% if post.categories == page.categories %}
            {.% assign count = count | plus: 1 %}
            <!-- 글의 목록을 최대 3개만 허용 -->
            {.% if count < 4 %}
              <tr>
                <!-- 각 포스트의 링크도 자신의 사이트에 맞게 수정 -->
                <th class="more-posts-title">
                  <a href="{.{ site.url }}{.{ post.url }}">{{ post.title }}</a>
                </th>
                <td class="more-posts-date">{.{ post.date | date: '%Y. %m. %d' }}</td>
              </tr>
            {.% endif %}
          {.% endif %}
        {.% endfor %}
      </tbody>
    </table>
  </div>
```
<br><br>

### 2. CSS 파일 작성
`assets` 폴더에 있는 css 파일 중 적당한 파일에 붙여넣어줍니다.
```css
  .more-posts {
    border: 1px solid #E5E5E5;
    padding: 10px 10px 5px;
    margin: 10px 0;
    clear: both;
    display: block;
  }
  .more-posts table {
    table-layout: fixed;
    border-collapse: collapse;
    width: 100% !important;
    margin-top: 10px !important;
    display: table;
    border-spacing: 2px;
    border-color: grey;
  }
  .more-category {
    border-bottom: 1px solid #E5E5E5 !important;
  }
  .more-category > h4 {
    line-height: 170%;
    color: #737373 !important;
    font-size: 12px;
    margin: 0;
    padding: 2px 0 6px !important;
    display: inline-block;
  }
  .more-category .more-button {
    float: right;
    color: #909090;
    font-size: 12px !important;
    vertical-align: center;
    border-collapse: collapse;
    padding: 4px 4px 0 0;
  }
  .more-posts-title {
    text-align: left;
    font-size: 12px !important;
    font-weight: normal;
    word-break: break-all;
    overflow: hidden;
    line-height: 1.5;
    padding: 0 0 4px !important;
    display: table-cell;
    vertical-align: inherit;
    border-collapse: collapse;
  }
  .more-posts-title a {
    color: #909090;
  }
  .more-posts-title a:hover {
    color: #34373e;
  }
  .more-posts-date {
    text-align: right;
    width: 80px;
    font-size: 11px;
    padding: 0 0 4px !important;
    color: #909090 !important;
    display: table-cell;
    vertical-align: inherit;
    border-collapse: collapse;
  }
```
HTML이나 CSS는 자신의 취향에 맞게 변경해서 사용하시면 되고, 작성 후 잘 나오는지 확인해보면 됩니다!


<br>

---------------------------------------------------------------------------------------
<br>



> 개인 공부하면서 작성한 글이라 잘못된 내용이 있을 수 있습니다. 잘못된 내용은 편하게 말씀해주시면 수정하겠습니다:)


<br>
