---
layout: post
comments: true
title:  "EC2 인스턴스 세팅과 Nginx + uWSGI로 Django 앱 배포하기 (1/2)"
subtitle: "EC2 인스턴스의 가상환경을 세팅하고, Nginx와 uWSGI를 이용해 Django 앱을 배포하는 방법에 대해서 다룹니다."
description: "EC2 인스턴스의 가상환경을 세팅하고, Nginx와 uWSGI를 이용해 Django 앱을 배포하는 방법에 대해서 다룹니다."
date:   2018-04-24 13:30:26 -0400
background: '/img/posts/2018-4-24/1.png'
image: '/img/posts/2018-4-24/1.png'
categories: [Deploy]
---


## EC2 인스턴스 세팅과 Nginx + uWSGI로 Django 앱 배포하기(1/2)
`이전 포스트` : [AWS 가입 및 EC2 인스턴스 생성](/deploy/2018/04/24/aws-signup-and-create-ec2.html){:target="_blank"}

위에서 만든 EC2 인스턴스의 가상환경을 세팅하고, Nginx와 uWSGI를 이용해 Django 앱을 배포하는 방법에 대해서 다룰 예정입니다.

### 1. EC2 인스턴스 가상환경 세팅
인스턴스 환경을 현재 사용중인 로컬 컴퓨터의 환경과 같게 만들어주기 위해서 세팅해줍니다.

#### 1.1 pyenv 설치
터미널로 인스턴스에 접속합니다.   **로컬이 아니라 `가상환경`입니다!**

* 참고로 인스턴스 접속은 아래 명령어로 가능합니다.
```shell
$  ssh -i <pem경로> <user name>@<public dns name>
```
* locale 오류(한글 깨짐 현상) fix
```shell
$  sudo vi /etc/default/locale
```
  * 맨 아랫줄에 아래 내용 입력 후 esc를 누르고 `:wq` 로 저장
  ```shell
  LC_CTYPE="en_US.UTF-8"
  LC_ALL="en_US.UTF-8"
  LANG="en_US.UTF-8"
  ```
  ![그림2](/img/posts/2018-4-24/2.png){: width="100%" height="100%"}
* 인스턴스 재접속
```shell
$  exit
```
```shell
$  ssh -i <pem경로> <user name>@<public dns name>
```
* apt-get 업데이트
```shell
$  sudo apt-get update
```
* apt-get 업그레이드
```shell
$  sudo apt-get dist-upgrade
```
* [Common build problems](https://github.com/pyenv/pyenv/wiki/Common-build-problems){:target="_blank"}에 나온 내용부터 설치
```shell
$  sudo apt-get install -y make build-essential libssl-dev zlib1g-dev libbz2-dev \
libreadline-dev libsqlite3-dev wget curl llvm libncurses5-dev libncursesw5-dev \
xz-utils tk-dev
```
* pyenv 설치
```shell
$  curl -L https://github.com/pyenv/pyenv-installer/raw/master/bin/pyenv-installer | bash
```
<br>

#### 1.2 zsh 설치
좀 더 사용성을 좋게 하기 위해 기본 `bash`을 `zsh`로 변경합니다.

* zsh 설치
```shell
$  sudo apt-get install zsh
```
* oh-my-zsh 설치
```shell
$  curl -L http://install.ohmyz.sh | sh
```
* 기본 쉘을 `bash`에서 `zsh`로 변경
```shell
$  sudo chsh ubuntu -s `which zsh`
```
* 종료 후 재접속
```shell
$  exit
```
```shell
$  ssh -i <pem경로> <user name>@<public dns name>
```
* zshrc에 pyenv 관련 내용 추가
```shell
$  vi ~/.zshrc
```
  * 2번째 줄 `export PATH ~` 부분 주석 해제
  ![그림3](/img/posts/2018-4-24/3.png){: width="100%" height="100%"}
  * 맨 아래 부분에 아래 내용 추가
  ```shell
  export PATH="/home/ubuntu/.pyenv/bin:$PATH"
  eval "$(pyenv init -)"
  eval "$(pyenv virtualenv-init -)"
  ```
  ![그림4](/img/posts/2018-4-24/4.png){: width="100%" height="100%"}
  * esc 키를 누른 후 `:wq` 를 입력해 저장
  * zshrc 새로고침
  ```shell
  $  source ~/.zshrc
  ```  
<br>

#### 1.3 /srv 폴더 권한 수정
로컬의 파일을 이제 인스턴스에 옮길 차례입니다. 파일은 인스턴스의 `/srv` 폴더로 옮겨야하는데, 권한이 필요합니다. 옮길 수 있도록 아래 명령어로 권한을 `ubuntu`로 변경합니다. (참고 : [리눅스 폴더 구조](https://ko.wikipedia.org/wiki/%ED%8C%8C%EC%9D%BC%EC%8B%9C%EC%8A%A4%ED%85%9C_%EA%B3%84%EC%B8%B5%EA%B5%AC%EC%A1%B0_%ED%91%9C%EC%A4%80){:target="_blank"})
```shell
$  sudo chown -R ubuntu:ubuntu /srv
```
<br>

---------------------------------------------------------------------------------------
<br>

이어서 인스턴스에 배포하기 위해 Django를 세팅하고, Nginx와 uWSGI를 이용해 배포하는 방법은 다음 포스트에서 확인 가능합니다.

`다음 포스트` : [EC2 인스턴스 세팅과 Nginx + uWSGI로 Django 앱 배포하기 (2/2)](/deploy/2018/05/02/instance-setting-and-django-deploy-part2.html){:target="_blank"}


> 개인 공부하면서 작성한 글이라 잘못된 내용이 있을 수 있습니다. 잘못된 내용은 편하게 말씀해주시면 수정하겠습니다:)

<br>
