---
layout: post
comments: true
title:  "Django ImageField에 이미지 URL로 파일 업로드하기"
subtitle: "Django ImageField에 파일이 아닌 이미지 URL로 파일을 업로드하는 방법을 다룹니다."
description: "Django ImageField에 파일이 아닌 이미지 URL로 파일을 업로드하는 방법을 다룹니다."
date:   2018-06-10 20:50:33 -0400
background: '/img/posts/2018-6-10/1.png'
image: '/img/posts/2018-6-10/1.png'
categories: [Django]
---


## ImageField에 이미지 URL로 파일 업로드하기
Django 프로젝트에서 ImageField를 사용할 때, 파일이 아닌 이미지의 URL로 파일을 자동으로 생성하고 싶은 순간들이 많은데 하는 방법을 알아보겠습니다.

### 1. 모델
먼저 모델은 아래처럼 간단하게 작성하였습니다. 제 경우 `image_file` 필드에 따로 파일을 선택하지 않으면, `purchase_url`의 대표 이미지를 크롤링해서 `image_file` 필드에 넣으려고 합니다.  

`models.py`
```python
class Item(models.Model):
    purchase_url = models.URLField('상품 URL', max_length=400, blank=True)
    image_file = models.ImageField('상품 이미지', upload_to='items', blank=True)
```
<br><br>

### 2. MEDIA_ROOT 설정
이미지가 저장될 위치를 세팅합니다. 저의 경우는 `root` 폴더와 동등한 위치에 `.media` 폴더를 만들고, 안에 저장시키려고 합니다. 모델에서 `image_file` 필드의 `upload_to`를 `items`로 지정해놓았으니 최종적으로 이미지가 저장되는 위치는 `.media/items/~~~~` 가 될 것입니다.

`settings.py`
```python
BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
ROOT_DIR = os.path.dirname(BASE_DIR)
MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(ROOT_DIR, '.media')
...
```
<br><br>

### 3. download 및 확장자 추출 util 작성
이미지의 URL을 임시파일로 다운받는 로직과 파일의 확장자를 추출하는 로직을 작성합니다.  

`utils/file.py`
```python
from io import BytesIO

import magic
import requests

# url로부터 파일을 임시 다운로드
def download(url):
    response = requests.get(url)
    binary_data = response.content
    temp_file = BytesIO()
    temp_file.write(binary_data)
    temp_file.seek(0)
    return temp_file

# 파일 확장자 추출
def get_buffer_ext(buffer):
    buffer.seek(0)
    mime_info = magic.from_buffer(buffer.read(), mime=True)
    buffer.seek(0)
    return mime_info.split('/')[-1]
```
<br><br>

### 4. save 함수 추가
이제 준비는 끝났고, 저장시 실행될 수 있도록 아래의 save 함수를 모델 파일에 추가합니다.  

`models.py`
```python
from urllib.parse import urlparse

from django.core.files import File

from utils.file import download, get_buffer_ext # 위에서 만든 file.py 경로


def save(self, *args, **kwargs):
    # ImageField에 파일이 없고, url이 존재하는 경우에만 실행
    if self.purchase_url and not self.image_file:
        # 우선 purchase_url의 대표 이미지를 크롤링하는 로직은 생략하고, 크롤링 결과 이미지 url을 임의대로 설정  
        item_image_url = 'https://cdn.pixabay.com/photo/2016/08/27/11/17/bag-1623898_960_720.jpg'

        if item_image_url:
            temp_file = download(item_image_url)
            file_name = '{urlparse}.{ext}'.format(
                # url의 마지막 '/' 내용 중 확장자 제거
                # ex) url = 'https://~~~~~~/bag-1623898_960_720.jpg'
                #     -> 'bag-1623898_960_720.jpg'
                #     -> 'bag-1623898_960_720'
                urlparse=urlparse(item_image_url).path.split('/')[-1].split('.')[0],
                ext=get_buffer_ext(temp_file)
            )
            self.image_file.save(file_name, File(temp_file))
            super().save()
        else:
            super().save()

```
<br><br>

### 5. 최종 확인
최종 모델은 아래와 같습니다.  

`models.py`
```python
from urllib.parse import urlparse

from django.core.files import File
from django.db import models

from utils.file import download, get_buffer_ext # 위에서 만든 file.py 경로


class Item(models.Model):
    purchase_url = models.URLField('상품 URL', max_length=400, blank=True)
    image_file = models.ImageField('상품 이미지', upload_to='items', blank=True)

    def save(self, *args, **kwargs):
        # ImageField에 파일이 없고, url이 존재하는 경우에만 실행
        if self.purchase_url and not self.image_file:
            # 우선 purchase_url의 대표 이미지를 크롤링하는 로직은 생략하고, 크롤링 결과 이미지 url을 임의대로 설정  
            item_image_url = 'https://cdn.pixabay.com/photo/2016/08/27/11/17/bag-1623898_960_720.jpg'

            if item_image_url:
                temp_file = download(item_image_url)
                file_name = '{urlparse}.{ext}'.format(
                    # url의 마지막 '/' 내용 중 확장자 제거
                    # ex) url = 'https://~~~~~~/bag-1623898_960_720.jpg'
                    #     -> 'bag-1623898_960_720.jpg'
                    #     -> 'bag-1623898_960_720'
                    urlparse=urlparse(item_image_url).path.split('/')[-1].split('.')[0],
                    ext=get_buffer_ext(temp_file)
                )
                self.image_file.save(file_name, File(temp_file))
                super().save()
            else:
                super().save()
```
실제로 저장을 하게되면 이미지의 주소는 아래처럼 정상적으로 저장되는 것을 확인하실 수 있습니다.
이미지 주소 : `http://localhost:8000/media/items/bag-1623898_960_720.jpeg`
![그림1](/img/posts/2018-6-10/1.png){: width="100%" height="100%"}



<br>

---------------------------------------------------------------------------------------
<br>



> 개인 공부하면서 작성한 글이라 잘못된 내용이 있을 수 있습니다. 잘못된 내용은 편하게 말씀해주시면 수정하겠습니다:)


<br>
