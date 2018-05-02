---
layout: post
comments: true
title:  "Jekyll 블로그 plugin 적용하고 page not found(404) 에러 해결하기"
subtitle: "Jekyll 블로그에서 custom plugin을 적용하고 사이트가 제대로 표시되지 않는 에러를 해결하는 방법에 대해 다룹니다."
description: "Jekyll 블로그에서 custom plugin을 적용하고 사이트가 제대로 표시되지 않는 에러를 해결하는 방법에 대해 다룹니다."
date:   2018-05-01 22:00:33 -0400
background: '/img/posts/2018-5-1/5.png'
image: '/img/posts/2018-5-1/5.png'
categories: [Jekyll-Blog]
---


## Jekyll 블로그 plugin 적용하고 page not found(404) 에러 해결하기
Jekyll 블로그에 [jekyll-category-pages](https://github.com/field-theory/jekyll-category-pages){:target="_blank"} 플러그인을 가이드를 보고 설치한 후, 로컬에서 정상 작동되는 것을 확인하고 평소처럼 github에 push를 하였는데 post의 링크가 제대로 연결이 안되서 아래처럼 page not found(404) 에러가 발생하였습니다.
![그림5](/img/posts/2018-5-1/5.png)

로컬에서는 정상 작동되기 때문에 원인을 찾기 힘들었는데, 열심히 구글링 한 결과 Github Pages에서 지원하는 [Jekyll plugin](https://pages.github.com/versions){:target="_blank"}이 따로 있다는 것을 발견했습니다.

일반 post의 글들은 Jekyll에서 알아서 판단해 site를 빌드해서 보여주기 때문에 괜찮지만 custom plugin을 사용한 경우라면 보안상의 이유로 빌드 할때 [liquid tags](https://help.shopify.com/themes/liquid/tags){:target="_blank"}를 인식하지 못하기 떄문에 생기는 이유라고 합니다.

해결책은 로컬에서 빌드하면 생기는 `_site` 폴더를 github에 함께 push를 해줘야 하는데 구체적인 방법은 아래에서 설명하겠습니다.

### 1. 전체 흐름
`_site` 폴더를 포함한 전체 프로젝트는 `source` branch에서 관리.
![그림6](/img/posts/2018-5-1/6.png)
`master` branch에서는 `_site` 안에 파일들만 관리.
![그림7](/img/posts/2018-5-1/7.png)

* `_site` 폴더에 `.nojekyll` 파일 생성
```shell
$  touch .nojekyll
```
* `.gitignore`에서 `_site` 주석 처리
```shell
$  vi .gitignore
```
```
### Jekyll ###
# _site/
```
`esc` 누른 후 `:wq`로 저장
* git add
```shell
$  git add .gitignore _site
```
* git commit
```shell
$  git commit -m 'custom plugin 적용 준비'
```
* `source` branch 생성
```shell
$  git checkout -b source
```
* git push
```shell
$  git push --all
```
* github repository 에서 설정 누른 후 `default branch` 변경
![그림8](/img/posts/2018-5-1/8.png)
* 로컬 remote 저장소의 `default branch` 적용
```shell
$  git remote set-head origin -a
```
* 로컬과 remote의 `master` branch 삭제
```shell
$  git push origin :master
$  git branch -d master
```
<br><br>

### 2. 자동화
위의 수많은 과정을 post를 작성할 때마다 반복한다고 생각하면, 헷갈릴뿐더러 굉장히 번거롭습니다. 언제나 그랬듯이 자동화하는 방법에 대해서 알아보겠습니다.

* Jekyll 폴더에 `publish.sh` 작성
```shell
$  vi publish.sh
```
아래 내용 작성
``` shell
  #!/bin/bash
  git checkout source
  git branch -D master
  git checkout -b master
  git filter-branch --subdirectory-filter _site/ -f
  git push --all
  git checkout source
```
`esc` 누른 후 `:wq`로 저장
* `.zshrc` or `.bash_profile` 설정  
**1. zsh일 경우**
```shell
$  vi ~/.zshrc
```
맨 하단에 아래 내용 작성
```shell
alias run-jekyll="bundle exec jekyll serve"
alias publish-jekyll="bash publish.sh"
```
`esc` 누른 후 `:wq`로 저장
```shell
$  source ~/.zshrc
```
**2. bash일 경우**
```shell
$  vi ~/.bash_profile
```
맨 하단에 아래 내용 작성
```shell
alias run-jekyll="bundle exec jekyll serve"
alias publish-jekyll="bash publish.sh"
```
`esc` 누른 후 `:wq`로 저장
```shell
$  source ~/.bash_profile
```
<br><br>

### 3. 최종 배포 방법
1. 로컬에서 post 작성 및 수정
2. Jekyll 블로그 폴더로 이동
3. Jekyll 실행
```shell
$  run-jekyll
```
4. `control + c` 로 실행되고 있는 Jekyll 종료
5. git add
```shell
$  git add .
```
6. git commit
```shell
$  git commit -m 'commit message'
```
7. 배포
```shell
$  publish-jekyll
```
8. 끝!
<br><br><br><br>


##### 참고 링크  
- [https://stackoverflow.com](https://stackoverflow.com/questions/28249255/how-do-i-configure-github-to-use-non-supported-jekyll-site-plugins/28252200#28252200){:target="_blank"}
- [http://gumpcha.github.io](http://gumpcha.github.io/blog/github-pages-with-jekyll-custom-plugin){:target="_blank"}


<br><br>

---------------------------------------------------------------------------------------
<br>



> 개인 공부하면서 작성한 글이라 잘못된 내용이 있을 수 있습니다. 잘못된 내용은 편하게 말씀해주시면 수정하겠습니다:)


<br>
