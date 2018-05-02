---
layout: post
comments: true
title:  "EC2 인스턴스 세팅과 Nginx + uWSGI로 Django 앱 배포하기 (2/2)"
subtitle: "EC2 인스턴스의 가상환경을 세팅하고, Nginx와 uWSGI를 이용해 Django 앱을 배포하는 방법에 대해서 다룹니다."
description: "EC2 인스턴스의 가상환경을 세팅하고, Nginx와 uWSGI를 이용해 Django 앱을 배포하는 방법에 대해서 다룹니다."
date:   2018-05-01 18:50:33 -0400
background: '/img/posts/2018-5-1/1.png'
image: '/img/posts/2018-5-1/1.png'
categories: [Deploy]
---


## EC2 인스턴스 세팅과 Nginx + uWSGI로 Django 앱 배포하기(2/2)
`이전 포스트` : [EC2 인스턴스 세팅과 Nginx + uWSGI로 Django 앱 배포하기(1/2)](/deploy/2018/04/25/instance-setting-and-django-deploy-part1.html){:target="_blank"}

1편에 이어서 이어서 인스턴스에 배포하기 위해 Django를 세팅하고, Nginx와 uWSGI를 이용해 배포하는 방법에 대해서 알아보겠습니다.

### 1. 서버 배포 방법
우선 인스턴스에 배포하기 전에 서버를 구성하는 방법의 종류를 살펴보겠습니다.

#### 1.1 Django로만 배포
이 방법은 익숙한 Django의 runserver를 이용하는 방법입니다. Django 프로젝트를 인스턴스에 그대로 옮겨 `python manage.py runserver` 를 켜서 서버를 돌리는 방법입니다.

하지만 Django의 runserver는 개발할 때 쓰는 작은 디버깅용 웹 서버이기 때문에 실제 프로젝트를 이 방법으로 배포하지는 않고, 아래의 `1.2`, `1.3`의 방법을 이용합니다.
<br>

#### 1.2 uWSGI + Django로 배포
Django 프로젝트를 생성하면 `wsgi.py` 파일이 생성되는 것을 많이 보셨을 겁니다. 우선 WSGI의 개념부터 살펴보면, `Web Server Gateway Interface`의 약자로 웹 서버(Apache, Nginx 등)와 웹 애플리케이션(Django, Flask 등) 사이를 이어주는 역할을 합니다. 즉 웹 서버로 들어온 요청을 Python 언어로 만들어진 웹 애플리케이션과 소통할 수 있도록 해주는 역할을 합니다. 웹 서버에 대한 설명은 `1.3`에서 하겠습니다.

uWSGI는 이 WSGI 규격으로 만들어진 서버 중에 하나이며 Django 프로젝트를 배포할 때 가장 일반적으로 사용됩니다.

Django는 수많은 request를 처리할 수 있도록 설계되어 있지 않기 때문에, Django로만 배포하지 않고 수많은 request를 처리하도록 설계된 uWSGI를 함께 사용해서 배포합니다. 그렇기 때문에 물론 uWSGI + Django 만으로도 충분히 배포가 가능하지만 추천하지는 않습니다. 이유는 `1.3`에서 확인하실 수 있습니다.
<br>

#### 1.3 Nginx + uWSGI + Django로 배포
![그림1](/img/posts/2018-5-1/1.png)

{:.image-caption}
*사진출처 : https://www.vndeveloper.com*

위의 그림에서처럼 Nginx는 클라이언트로부터 HTTP 요청을 받아 request에 따라 바뀌지 않는 즉 정적파일(HTML, CSS, JavaScript)을 돌려주고, 동적데이터가 필요한 요청은 웹 애플리케이션으로 보내줍니다. 이게 바로 웹 서버가 하는 일입니다. `1.2`의 방법을 추천하지 않은 이유 역시 이 웹 서버가 정적파일을 미리 클라이언트에게 보여줌으로써 서버 성능을 높여주기 때문입니다.

Nginx에서 받은 요청은 unix socket을 거쳐 uWSGI로 보내지며, uWSGI가 받은 요청을 Python 개발 환경에서 이해(?)할 수 있도록 중계해주면 최종적으로 Django에서 요청을 처리하게 됩니다.

그럼 `Nginx + uWSGI + Django`의 일련의 과정을 실제로 배포하는 방법을 아래에서 알아보도록 하겠습니다.
<br><br>

### 2. Django 세팅
인스턴스에 배포하기 전에 Django를 세팅합니다.

#### 2.1 Django static, media 파일 처리 설정
먼저 제가 사용 중인 프로젝트의 `tree 구조`는 아래와 같습니다.
```
nginx_test
└── app
    ├── config
    │   ├── __init__.py
    │   ├── __pycache__
    │   ├── settings.py
    │   ├── urls.py
    │   └── wsgi.py
    ├── manage.py
    └── post
        ├── __init__.py
        ├── admin.py
        ├── apps.py
        ├── migrations
        │   └── __init__.py
        ├── models.py
        ├── tests.py
        └── views.py
```

* config에 있는 settings.py 파일에 아래처럼 static과 media 폴더를 추가해줍니다.  
`nginx_test/app/config/settings.py`
```python
  ROOT_DIR = os.path.dirname(BASE_DIR)

  STATIC_URL = '/static/'
  STATIC_ROOT = os.path.join(ROOT_DIR, '.static')

  MEDIA_URL = '/media/'
  MEDIA_ROOT = os.path.join(ROOT_DIR, '.media')
  STATIC_DIR = os.path.join(BASE_DIR, 'static')

  STATICFILES_DIRS = [
      STATIC_DIR,
  ]
```
* 그 후 gitignore 파일을 열어 최상단에도 추가해줍니다.
```shell
# Custom
/.media
/.static
```
<br>

#### 2.2 Django DEBUG 설정 및 ALLOWED_HOST 설정 변경
개발을 완료하고 배포하는 단계이므로 `DEBUG` 모드를 `False`로 바꾸고, `ALLOWED_HOST`에 aws 주소를 허용합니다.

`nginx_test/app/config/settings.py`
```python
DEBUG = False

ALLOWED_HOSTS = [
    '.amazonaws.com',
]
```
<br>

#### 2.3 uWSGI 사이트 파일 작성
`app`폴더와 같은 위치에 `.config`폴더를 만든후, `uwsgi.ini` 파일에 아래와 같이 작성합니다.

**이제부터 custom 해야하는 부분은 <> 안에 작성하겠습니다. <> 부분은 꺽쇠 제거 후 자신의 프로젝트에 맞게 custom 해주세요!**

`nginx_test/.config/uwsgi.ini`
```ini
; Django 프로젝트에 대한 uwsgi 설정파일
[uwsgi]
chdir = /srv/<nginx_test/app>
module = config.wsgi
home = /home/ubuntu/.pyenv/versions/<ec2-deploy>

uid = ubuntu
gid = ubuntu
socket = /tmp/app.sock
chmod-socket = 666
chown-socket = ubuntu:ubuntu

master = true
vacuum = true
logto = /tmp/uwsgi.log
log-reopen = true
```

`<ec2-deploy>` 는 pyenv virtualenv로 설정한 가상환경 이름입니다.

참고로 `pycharm`에서 `ini` 확장자 지원은 `Preferences -> Plugins -> Browse repositories...` 들어간 후 아래의 플러그인을 설치해주시면 됩니다.

![그림2](/img/posts/2018-5-1/2.png)

#### 2.4 uWSGI 서비스 설정 파일 작성
`.config`폴더에 `uwsgi.service` 파일에 아래와 같이 작성합니다.
`nginx_test/.config/uwsgi.service`
```ini
[Unit]
Description=EC2 Deploy uWSGI service
After=syslog.target

[Service]
ExecStart=/home/ubuntu/.pyenv/versions/ipsw/bin/uwsgi -i </srv/nginx_test/app/.config/uwsgi.ini>

Restart=always
KillSignal=SIGQUIT
Type=notify
StandardError=syslog
NotifyAccess=all

[Install]
WantedBy=multi-user.target
```
<br>

#### 2.5 Nginx 설정 파일 작성
`.config` 폴더에 `nginx-app.conf`라는 Nginx 설정 파일을 만들고 아래 내용을 작성합니다.
`nginx_test/.config/nigx-app.conf`
```nginx
server {
    listen 80;
    server_name *.amazonaws.com;
    charset utf-8;
    client_max_body_size 128M;

    location / {
        uwsgi_pass      unix:///tmp/app.sock;
        include         uwsgi_params;
    }

    location /static/ {
        alias /srv/app/.static/;
    }

    location /media/ {
        alias /srv/app/.media/;
    }
}
```
<br>

#### 2.6 requirements.txt 파일 업데이트
uWSGI를 설치 후, Django 프로젝트에 사용된 패키지들을 서버에 올리기 전에 최종적으로 업데이트 해줍니다.
```shell
$  pip install uwsgi
```
```shell
$  pip freeze > requirements.txt
```
<br><br>

### 3. 인스턴스에 로컬 프로젝트 복사
이제 로컬의 Django 프로젝트를 EC2 인스턴스에 복사할 차례입니다. 복사할 땐 `scp` 명령어를 사용하며, 아래와 같습니다.

```shell
$  scp -i <pem경로> -r <복사할 로컬 프로젝트 위치> <username>@<public dns name>:<복사될 인스턴스 위치>
```
제 경우는 아래와 같습니다.
```shell
$  scp -i ~/.ssh/examplekey.pem -r ~/Desktop/Backend/Study/django/nginx_test ubuntu@ec2-52-78-234-247.ap-northeast-2.compute.amazonaws.com:/srv/nginx_test
```
<br><br>

### 4. 인스턴스 세팅
인스턴스에 복사한 파일들로 인스턴스에도 세팅을 해줍니다.

#### 4.1 pip list 설치
* 먼저 인스턴스에 접속합니다.
```shell
$  ssh -i <pem경로> <user name>@<public dns name>
```
* 인스턴스에서 복사한 프로젝트 폴더로 들어가 로컬 프로젝트와 동일한 가상환경명으로 만들어줍니다.
```shell
$  cd /srv/nginx_test
```
```shell
$  pyenv install 3.6.4
```
```shell
$  pyenv virtualenv 3.6.4 ec2-deploy
```
* `requirements.txt`로 `pip list` 설치
```shell
$  pip install requirements.txt
```
<br>

#### 4.2 인스턴스에 Nginx 및 uWSGI 설정
아래 내용은 모두 로컬이 아니라 `인스턴스`에서 진행해야 합니다.

* Nginx 설치
```shell
$  sudo add-apt-repository ppa:nginx/stable
$  sudo apt-get update
$  sudo apt-get install nginx
```
* `nginx-app.conf` 파일 `sites-available` 폴더에 복사
```shell
$  sudo cp -f </srv/nginx_test/app/.config/nginx-app.conf> /etc/nginx/sites-available/nginx-app.conf
```
* `site-enabled`에 링크 걸어주기
```shell
$  sudo ln -sf /etc/nginx/sites-available/nginx-app.conf /etc/nginx/sites-enabled/nginx-app.conf
```
* `uwsgi.sevice` 파일 `system` 폴더에 복사
```shell
$  sudo cp -f /srv/nginx_test/app/.config/uwsgi.service /etc/systemd/system/uwsgi.service
```
* `manage.py`가 복사된 위치로 이동 후, `manage.py collectstatic` 실행
```shell
$  python manage.py collectstatic --noinput
```
* uwsgi 사용
```shell
$  sudo systemctl enable uwsgi
```
* daemon-reload
```shell
$  sudo systemctl daemon-reload
```
* uwsgi 재부팅
```shell
$  sudo systemctl restart uwsgi nginx
```
<br><br>

### 5. 세팅 완료 및 자동화
위의 내용들을 완료 후, 자신의 `ec2-`로 시작하는 도메인으로 접속하면 사이트가 아래처럼 제대로 나오는 것을 확일할 수 있습니다!
![그림3](/img/posts/2018-5-1/3.png)
참고로 페이지에 내용을 따로 작성하지 않았으면 아래처럼 `Not found`가 뜨는게 정상입니다. 제대로 확인해보고 싶으시면 `/admin`으로 접속해보시면 됩니다.

정상적으로 AWS EC2 인스턴스에 배포하였지만, 코드를 수정한다면 위의 복잡한 과정을 다시 해야하는 번거로움이 있습니다. 이를 해결하기 위해 `.zshrc` 와 `deploy.sh` 파일을 작성해보겠습니다.

#### 5.1 deploy.sh 작성
위에서 입력했던 shell 스크립트 명령어들을 한 곳에 작성 후 나중에 호출하는 형식으로 진행할 예정입니다. `로컬`로 돌아와서 아래 경로에 `deploy.sh` 파일을 작성해줍니다.
`nginx_test/.config/deploy.sh`
```shell
#!/usr/bin/env bash
sudo rm -rf /etc/nginx/sites-enabled/*
sudo cp -f </srv/nginx_test/app/.config/nginx-app.conf> /etc/nginx/sites-available/nginx-app.conf
sudo ln -sf /etc/nginx/sites-available/nginx-app.conf /etc/nginx/sites-enabled/nginx-app.conf
sudo cp -f </srv/nginx_test/app/.config/uwsgi.service> /etc/systemd/system/uwsgi.service

cd /srv/app
/bin/bash -c \
'/home/ubuntu/.pyenv/versions/<ec2-deploy>/bin/python \
</srv/ngingx_test/app/manage.py> collectstatic --noinput' ubuntu

sudo systemctl enable uwsgi
sudo systemctl daemon-reload
sudo systemctl restart uwsgi nginx
```

**작성시 < > 안에 내용 변경해주세요!**
<br>

#### 5.2 .zshrc 파일에 명령어 추가
* `로컬`에서 아래 명령어로 `.zshrc` 파일을 열어줍니다.
```shell
$  vi ~/.zshrc
```
참고로 `zsh`가 아닌 `bash`를 사용중이시라면 `.bash_profile` 을 열어줍니다.
```shell
$  vi ~/.bash_profile
```
* 맨 하단에 아래 내용 작성
```shell
  # 인스턴스 username
  EC2_USER="ubuntu"

  # AWS 도메인 주소
  EC2_DOMAIN="<ec2-52-78-234-247.ap-northeast-2.compute.amazonaws.com>"

  # pem 경로
  EC2_PEM="<~/.ssh/examplekey.pem>"

  # 복사할 로컬 프로젝트 위치
  EC2_DIR="<~/Desktop/Backend/Study/django/nginx_test>"

  # 인스턴스 접속 명령어 세팅
  alias con-ec2="ssh -i $EC2_PEM $EC2_USER@$EC2_DOMAIN"

  # 인스턴스에 복사하는 명령어 세팅
  alias copy-ec2="scp -i $EC2_PEM -r $EC2_DIR $EC2_USER@$EC2_DOMAIN:</srv/nginx_test>"

  # 인스턴스에 기존에 존재하던 파일 삭제 명령어 세팅
  alias delete-ec2-command="con-ec2 sudo rm -rf </srv/nginx_test>"

  # 인스턴스에 접속해서 작성한 shell 스크립트 명령어 실행
  alias deploy-ec2-command="con-ec2 bash </srv/nginx_test/app/.config/deploy.sh>"

  # 인스턴스에 접속해 pip list 설치 명령어 세팅
  alias pip-ec2-command="con-ec2 /home/ubuntu/.pyenv/versions/<ec2-deploy>/bin/pip install -r </srv/nginx_test/requirements.txt>"

  # 명령어들을 조합해 배포 명령어 세팅
  alias deploy-ec2="delete-ec2-command; copy-ec2; pip-ec2-command; deploy-ec2-command"
```
* `esc`를 누른 후 `:wq` 로 저장
* `.zshrc` 새로고침
```shell
$  source ~/.zshrc
```
<br>

#### 5.3 배포 준비 완료
이제 모든 준비는 끝났습니다. 복잡한 과정 없이 아래의 짧은 명령어들로 인스턴스에 접속 및 배포 가능합니다!

* AWS EC2 인스턴스에 접속
```shell
$  con-ec2
```
* AWS EC2 인스턴스에 배포
```shell
$  deploy-ec2
```
<br>

---------------------------------------------------------------------------------------
<br>



> 개인 공부하면서 작성한 글이라 잘못된 내용이 있을 수 있습니다. 잘못된 내용은 편하게 말씀해주시면 수정하겠습니다:)


<br>
