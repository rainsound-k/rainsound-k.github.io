---
layout: post
comments: true
title:  "Django 프로젝트에 Travis-CI 연동 및 Secret Key 암호화"
subtitle: "Django 프로젝트에 Travis-CI를 연동하는 방법과 Secret Key를 암호화하는 방법을 다룹니다."
description: "Django 프로젝트에 Travis-CI를 연동하는 방법과 Secret Key를 암호화하는 방법을 다룹니다."
date:   2018-05-13 19:50:33 -0400
background: '/img/posts/2018-5-13/1.png'
image: '/img/posts/2018-5-13/1.png'
categories: [Deploy]
---


## Django 프로젝트에 Travis-CI 연동 및 Secret Key 암호화
Django 프로젝트에 `Travis-CI`를 연동하는 방법과 각종 Secret Key를 Travis-CI에서 인식할 수 있도록 암호화, 복호화하는 방법을 다뤄보겠습니다.

### 1. CI란?
먼저 `CI`가 무엇인지 살펴보면, `Continuous Integration`의 약자로 배포나 빌드 할 때 문제를 미리 파악할 수 있도록 이상이 없는지 자동으로 확인해주는 기술입니다. 즉 github에 push할 때마다 정해놓은 test를 통과하는지 자동으로 검증을 해줍니다.

혹은 오픈소스의 master branch를 관리할 때 이 기술을 이용하면 수많은 `pull request`들에 대해 하나하나 test 검증을 할 필요가 없기 때문에 상당히 편리합니다.

`Travis-CI`는 `CI` 서비스 중 하나입니다. 기본적으로 github 공개 Repository와 연동하는 것은 무료, 비공개 Repository와 연동하는 것은 유료이며 사이트 주소도 다릅니다.
- [https://travis-ci.org](https://travis-ci.org){:target="_blank"} **공개 Repository(무료)**
- [https://travis-ci.com](https://travis-ci.com){:target="_blank"} **비공개 Repository(유료)**

우선 test 해보는 과정이므로 무료로 이용해보겠습니다.
<br><br>

### 2. Travis-CI 가입
먼저 [https://travis-ci.org](https://travis-ci.org){:target="_blank"} 사이트에 접속해서 가입한 후, github과 연동해줍니다.
<br><br>

### 3. Github Repository 생성
기존 프로젝트가 있다면 따로 Repository를 만들 필요없이 그대로 사용하시면 되며, 없는 경우엔 만들어줍니다.
저는 아래와 같이 `test`라는 Repository를 만들었고, 프로젝트 폴더와 연결했습니다.   
**Secret Key를 숨겨야하기 때문에 아직 push는 잠시 보류해주세요!**
![그림1](/img/posts/2018-5-13/1.png)
<br><br>

### 4. Django 프로젝트의 Secret Key 분리
Github에 올라가선 안되는 각종 Secret Key들을 따로 폴더를 만들어 관리해줍니다.
아직까진 Secret Key가 Django 기본 `SECRET_KEY`만 있는 상태이므로, 이것만 다른 폴더로 빼서 관리해보겠습니다.

#### 4.1 .secrets 폴더 생성 및 base.json 파일 생성
먼저 루트 폴더와 동일한 위치에 `.secrets` 폴더를 생성해주고, 안에 `base.json` 파일을 만들어줍니다.
![그림2](/img/posts/2018-5-13/2.png)
<br>

#### 4.2 base.json 파일에 Django SECRET_KEY 입력
`settings.py` 파일에 있는 Django `SECRET_KEY`를 `base.json` 파일에 옮겨줍니다.  

`.secrets/base.json`
```json
{
  "SECRET_KEY": "각자의 SECRET_KEY 입력해주세요"
}
```
<br>

#### 4.3 settings.py 파일 수정
외부 폴더에 있는 `SECRET_KEY`를 불러올 수 있도록 `settings.py`을 수정해줍니다.  

`app/config/settings.py`
```python
import json
import os

# Build paths inside the project like this: os.path.join(BASE_DIR, ...)
BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
ROOT_DIR = os.path.dirname(BASE_DIR)
SECRETS_DIR = os.path.join(ROOT_DIR, '.secrets')
SECRETS_BASE = os.path.join(SECRETS_DIR, 'base.json')
secrets_base = json.loads(open(SECRETS_BASE, 'rt').read())

# SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = secrets_base['SECRET_KEY']
...

```
<br>

#### 4.4 gitignore에 .secrets 폴더 추가
Github에 `.secrets` 폴더가 올라가는 것을 방지하기 위해 gitignore 파일에 추가해줍니다.
```shell
# Custom
/.secrets
...

```
<br>

#### 4.5 requirements.txt 파일 생성
설치된 pip list를 Travis-CI에서 확인할 수 있도록 `requirements.txt` 파일을 만들어줍니다.
```shell
$  pip freeze > requirements.txt
```
<br><br>

### 5. Travis-CI와 연동
다시 Travis-CI 사이트로 돌아와 `my profile`에서  `Sync account` 누른 후 새로고침을 하면 아까 생성했던 Repository가 보일 것입니다. 그럼 아래처럼 버튼을 눌러 연결해줍니다.
![그림3](/img/posts/2018-5-13/3.png)

#### 5.1 .travis.yml 파일 작성
Django 프로젝트 루트 폴더에 `.travis.yml` 파일을 작성해줍니다.  
*(python 버전은 프로젝트에 맞게 바꿔주세요!)*
```yml
language: python
python:
  - "3.6"
install:
  - pip install -r requirements.txt
script:
  - python app/manage.py test
```
최종 `tree 구조`는 아래와 같습니다.
```
travis-ci-test
	├── .git
	├── .gitignore
	├── .python-version
	├── .travis.yml
	├── .secrets
	│   └── base.json
	└── app
	    ├── config
	    │   ├── __init__.py
	    │   ├── settings.py
	    │   ├── urls.py
	    │   └── wsgi.py
	    └── manage.py
```
<br>

#### 5.2 git push
이제 준비가 끝났으니 프로젝트를 git에 push 해줍니다.
```shell
$  git add .
$  git commit -m 'first commit'
$  git push -u origin master
```
<br>

#### 5.3 에러 확인
Travis-CI 사이트로 넘어와서 프로젝트를 눌러보면 아래와 같이 에러가 발생한 것을 확인 할 수 있습니다.
![그림4](/img/posts/2018-5-13/4.png)
에러의 내용은 다음과 같습니다.
```
FileNotFoundError: [Errno 2] No such file or directory: '/home/travis/build/rainsound-k/test/.secrets/base.json'
```
즉 gitignore에 `.secrets` 폴더를 넣어놨기 때문에 `settings.py`에서 `SECRET_KEY` 값을 참조할 수 없다는 내용입니다. 이 에러를 아래서 해결해보겠습니다.
<br><br>

### 6. SECRET_KEY 에러 해결
`SECRET_KEY` 값을 참조할 수 있도록 `.secrets` 폴더를 `압축 -> 암호화 -> 업로드 -> 복호화 -> 압축해제` 하는 방법으로 에러를 해결해보겠습니다.

#### 6.1 .secrets 폴더 압축
아래 명령어로 `.secrets` 폴더를 `.secrets.tar` 파일로 압축해줍니다.
```shell
$  tar -cvf secrets.tar .secrets
```
<br>

#### 6.2 .secrets.tar 파일 암호화
* 터미널에 travis를 설치
```shell
$  gem install travis
```
*(위의 gem 명령어가 안 먹힌다면 ruby를 설치해주세요!)*
* 터미널에 travis 로그인
```shell
$  travis login
```
*(Username은 github의 Username 입니다.)*
* 암호화
```shell
$  travis encrypt-file secrets.tar --add
```
<br>

#### 6.3 gitignore에 secrets.tar 파일 추가
위의 과정을 마치면 `secrets.tar`, `secrets.tar.enc` 파일이 프로젝트 폴더에 생긴 것을 확인할 수 있습니다.   
**`secrets.tar` 파일은 github에 올라가면 안되므로 gitignore에 추가해줍니다.**
```shell
# Custom
/.secrets
secrets.tar
...

```
<br>

#### 6.4 .travis.yml에 압축해제 명령어 추가
`.travis.yml` 파일을 다시 열어보면, 아래처럼 `before_install`에 명령어가 자동으로 추가된 것을 확인할 수 있습니다. 이는 travis에 코드를 올리면 `secrets.tar.enc` 파일을 `secrets.tar` 파일로 복호화 하라는 명령어입니다.
![그림5](/img/posts/2018-5-13/5.png)
하지만 파일을 복호화한다해도 여전히 압축파일로 남아있기 때문에, 압축해제 명령어를 `before_install`에 추가해줍니다.
```yml
before_install:
...
- tar xvf secrets.tar
```
<br><br>

### 7. 최종 확인
모든 준비는 끝났습니다. 이제 github에 push 했을때 에러가 발생 안하는지 확인하면 됩니다. 아래 명령어로 다시 push 해주고 에러 없는 것을 확인하면 끝입니다!
```shell
$  git add .
$  git commit -m '~~~'
$  git push -u origin master
```
![그림6](/img/posts/2018-5-13/6.png)

<br>

---------------------------------------------------------------------------------------
<br>



> 개인 공부하면서 작성한 글이라 잘못된 내용이 있을 수 있습니다. 잘못된 내용은 편하게 말씀해주시면 수정하겠습니다:)


<br>
