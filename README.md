## github 개발 블로그입니다.
[https://rainsound-k.github.io](https://rainsound-k.github.io/)

### 배포 순서
#### 1. Build Jekyll
```shell
$ bundle exec jekyll serve
```
#### 2. git add
```shell
$ git add .
```
#### 3. git commit
```shell
$ git commit -m '~~~'
```
#### 4. publish
```shell
$ bash publish.sh
```

**`publish.sh`**

```shell
#!/bin/bash

git checkout source
git branch -D master
git checkout -b master
git filter-branch --subdirectory-filter _site/ -f
git push --all
git checkout source
```
