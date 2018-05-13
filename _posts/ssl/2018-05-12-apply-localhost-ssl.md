---
layout: post
comments: true
title:  "localhost에서 페이스북 로그인 HTTPS 적용하기"
subtitle: "개발용 localhost 서버에 HTTPS(SSL) 적용시켜 페이스북 로그인 테스트하는 방법에 대해 다룹니다."
description: "개발용 localhost 서버에 HTTPS(SSL) 적용시켜 페이스북 로그인 테스트하는 방법에 대해 다룹니다."
date:   2018-05-12 22:30:33 -0400
background: '/img/posts/2018-5-12/1.png'
image: '/img/posts/2018-5-12/1.png'
categories: [SSL]
---


## localhost에서 페이스북 로그인 HTTPS 적용하기
18년 3월부터 페이스북 로그인과 같은 페이스북 앱을 사용하기 위해선 `HTPPS URL`이 필수이며, 19년 3월부터는 기존에 사용중인 모든 앱을 `HTTPS URL`만 사용하도록 마이그레이션해야 합니다.
![그림1](/img/posts/2018-5-12/1.png)

만약 `HTTP URL`을 그냥 사용한다면 아래와 같은 에러가 발생합니다.
```
Insecure Login Blocked: You can't get an access token or log in to this app from an insecure page. Try re-loading the page as https://
```
![그림2](/img/posts/2018-5-12/2.png)

실제 서비스로 배포할 땐 [Let's Encrypt](https://letsencrypt.org){:target="_blank"}를 통해 HTTPS URL을 적용하겠지만, 개발 테스트를 할 때조차 적용하기는 여간 귀찮은 일이 아닙니다.

그렇기 때문에 HTTP 기반의 `localhost` 서버에서 임시로 간단하게 SSL을 적용하는 방법을 알아보겠습니다.

### 1. Django 기반 Test Server
먼저 Django 기반 localhost에 SSL을 적용하는 방법은 [Django SSL Server](https://github.com/teddziuba/django-sslserver){:target="_blank"}라는 오픈소스를 이용하면 간단합니다.

#### 1.1 설치
아래 명령어로 설치해줍니다.
```shell
$  pip install django-sslserver
```
<br>

#### 1.2 INSTALLED_APPS에 추가
`settings.py` INSTALLED_APPS에 `sslserver`를 추가해줍니다.
```python
INSTALLED_APPS = [
    ...
    'sslserver',
    ...
]
```
<br>

#### 1.3 runsslserver 실행
기존 `runserver`대신 `runsslserver`를 이용해서 서버를 작동시킵니다.
```shell
$  python manage.py runsslserver
```
<br>

#### 1.4 개인정보 보호 오류 해결
* `https://localhost:8000` 으로 접속하면 공인된 곳에서 인증서를 발급받지 않아 아래와 같은 `개인정보 보호 오류`가 발생합니다. 단순 테스트용 서버이므로 우선 `고급`을 눌러줍니다.
![그림3](/img/posts/2018-5-12/3.png)

* `localhost(안전하지 않음)(으)로 이동` 을 눌러서 접속합니다.
![그림4](/img/posts/2018-5-12/4.png)

* HTTPS URL로 접속된 것을 확인합니다.
![그림5](/img/posts/2018-5-12/5.png)
<br>

#### 1.5 페이스북 로그인 유효한 OAuth 리디렉션 URI에 주소 추가
아래와 같이 HTTPS URL을 `유효한 OAuth 리디렉션 URI`에 추가해주면 끝납니다!
![그림6](/img/posts/2018-5-12/6.png)
<br><br>

### 2. 프론트엔드 기반 Test Server
프론트엔드 기반에서 SSL 적용시키기는 Django보다는 까다롭지만 크게 어렵지 않아 금방 하실 수 있습니다.

#### 2.1 SSL 인증서와 key 파일 발급
`openssl`을 이용해 임의의 인증서와 key 파일을 발급 받아 접속하는 방식을 사용합니다.

* openssl 설치
  * Mac OS 기준
  ```shell
    $  brew install openssl
  ```
* private key 생성
```shell
$  openssl genrsa -des3 -out server.key 1024
```
이후 `Enter pass phrase for server.key:` 에 key 파일에 암호를 임의로 입력해줍니다.
* CSR 파일 생성
```shell
$  openssl req -new -key server.key -out server.csr
```
```
  // 국가 코드 (ex KR)
  Country Name (2 letter code) []:

  // 도 이름. 없으면 공백 (ex Gyeonggi)
  State or Province Name (full name) []:

  // 도시 이름 (ex Seoul)
  Locality Name (eg, city) []:

  // 회사명
  Organization Name (eg, company) []:

  // 부서명
  Organizational Unit Name (eg, section) []:

  // 도메인명
  Common Name (eg, YOUR name) []:

  // E-mail 주소
  Email Address []:
```
![그림7](/img/posts/2018-5-12/7.png)
* 시작시마다 암호를 묻는 것을 방지하기 위해 key의 암호를 제거해줍니다.
``` shell
$  cp server.key server.key.org
$  openssl rsa -in server.key.org -out server.key
```
* 자체 서명된 인증서 생성
```shell
$  openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt
```
* server.key.org, server.csr 파일 삭제 및 파일 이동  
  * 위의 절차를 완료했다면 총 4개의 파일(`server.crt`, `server.key`, `server.key.org`, `server.csr`)이 생성된 것을 확인하실 수 있습니다.   
  * 그 중 `server.key.org`, `server.csr` 파일을 삭제하고 나머지 2개의 파일을 `index.html`이 있는 프로젝트 루트 폴더로 이동시켜줍니다.
<br>

#### 2.2 python 파일 생성
위에서 생성한 파일을 토대로 SSL Server를 만들어주는 python 파일을 프로젝트 루트 폴더에 작성해줍니다.
`local-ssl-server.py`
```python
import http.server

import ssl

httpd = http.server.HTTPServer(('0.0.0.0', 3004), http.server.SimpleHTTPRequestHandler)

httpd.socket = ssl.wrap_socket(httpd.socket, certfile='server.crt', keyfile='server.key')

httpd.serve_forever()
```
위의 코드는 `server.crt`, `server.key` 파일과 python 파일이 동등한 위치에 있다고 가정하고 작성한 것이므로, 인증서를 다른 폴더에 넣었다면 경로를 커스텀 해주시면 됩니다.
<br>

#### 2.3 python 파일로 runserver
기존에는 `python -m http.server 3000`으로 HTTP 기반의 3000번 포트를 열어줬다면, 지금 부터는 위의 작성한 python 파일로 HTTPS 기반의 3004번 포트를 열어줄 차례입니다.

* runserver 실행
```shell
$  python local-ssl-server.py
```
* `https://localhost:3004` 으로 접속하면 공인된 곳에서 인증서를 발급받지 않아 아래와 같은 `개인정보 보호 오류`가 발생합니다. 단순 테스트용 서버이므로 우선 `고급`을 눌러줍니다.
![그림3](/img/posts/2018-5-12/3.png)
* `localhost(안전하지 않음)(으)로 이동` 을 눌러서 접속합니다.
![그림4](/img/posts/2018-5-12/4.png)
* HTTPS URL로 접속된 것을 확인합니다.
![그림8](/img/posts/2018-5-12/8.png)
<br>

#### 2.4 페이스북 로그인 유효한 OAuth 리디렉션 URI에 주소 추가
아래와 같이 HTTPS URL을 `유효한 OAuth 리디렉션 URI`에 추가해줍니다.
![그림9](/img/posts/2018-5-12/9.png)
<br>

#### 2.5 페이스북 로그인 Test
페이스북 로그인 버튼을 눌러보면 팝업창이 잘 뜨는 것을 확인하실 수 있습니다!
![그림8](/img/posts/2018-5-12/8.png)
![그림10](/img/posts/2018-5-12/10.png)
<br><br>

### 3. .gitignore에 추가
인증서 파일과 python 파일을 `.gitignore`에 작성해서 github에 함께 올라가는 것을 방지합니다.
```
### Custom ###
server.crt
server.key
local-ssl-server.py
```
<br><br><br><br>


##### 참고 링크  
- [https://www.akadia.com](https://www.akadia.com/services/ssh_test_certificate.html){:target="_blank"}
- [http://www.hanbit.co.kr](http://www.hanbit.co.kr/channel/category/category_view.html?cms_code=CMS6163871474){:target="_blank"}


<br><br>

---------------------------------------------------------------------------------------
<br>



> 개인 공부하면서 작성한 글이라 잘못된 내용이 있을 수 있습니다. 잘못된 내용은 편하게 말씀해주시면 수정하겠습니다:)


<br>
