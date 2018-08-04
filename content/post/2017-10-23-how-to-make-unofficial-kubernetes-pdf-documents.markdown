---
layout: post
title: "how to make unofficial kubernetes pdf documents"
date: 2017-10-23 21:21:47 +0900
comments: true
categories: 
- kubernetes
- k8s
- document
- 1.8
- pdfmerge
- jekyll
- chrome
- headless
- pdf
- unofficial
---

![kubernetes](/images/kubernetes-production-grade-container-orchestration.png)

### [download pdf](/images/kubernetes-documents.pdf)

---

### 왜?

밖에서 심심할때 종이로 출력해서 문서를 좀 더 봐야겠다는 생각에 `pdf` 버전을 구해보려고 했으나. 우선 k8s webpage를 관리하는 [website](https://github.com/kubernetes/website)프로젝트에서 아래와 같이 [지원을 안한다고 딱잘라서 이야기](https://github.com/kubernetes/website/issues/666#issuecomment-271741289) 하는것을 찾았다.

> ... we don't plan to make PDFs available.

그리고 마땅히 검색했을때 방법이 없기에 만들어야 겠다고 생각했다.

### 어떻게 할까?

외부의 웹사이트를 pdf 로 출력하는 방법들은 여러가지가 있다 그냥 웹 주소를 넣으면 바로 pdf 로 변환해 주는 솔루션이나 사이트들도 많다.(물론 단일 페이지만 렌더링 한다.)

하지만 이럴 경우 footer 와 같이 pdf 로 볼때 불 필요한 부분들이 너무 많아서 좋지 않다. 그래서 이번에 진행할때는 그냥 실제 코드에서 필요없는 부분들을 제거해서 렌더링 시키는 방법으로 하기로 했다.

*물론 web client를 패치하는 방법이나 adblock 등으로 특정 구문들을 제거하는 방법들도 있으나 지금 방법보다 더 손이 많이 갈것 같았다..*

### 준비 환경

우선 레포지토리를 받아서 stable중 가장 최신인 1.8버전으로 체크아웃 한다.

```
$ git clone https://github.com/kubernetes/website
$ cd website
$ git checkout release-1.8
```

jekyll project이니 Gemfile로 디펜던시를 인스톨한다.

```
$ bundle install
```

이후 build 해서 md 파일들을 html 로 빌드 한다.

```
$ make build
```

### 어떤 페이지들을 pdf로 출력할까?

아래와 같이 왼쪽 도큐먼트 리스트들 기준으로 출력하고자 해당 페이지만 아래와 같은 스크립트로 추출했다.
여기에 아무래도 api spec과 같은 양이 많은 reference 를 제외하고, 또 포맷이 안맞는 외부링크 제외한 페이지를 필터링 했다.

![image](/images/k8sdocstoc.png)

```
$ cd _site/docs/
$ cat home/index.html setup/index.html concepts/index.html \
tasks/index.html tutorials/index.html | \
grep 'a class="item"' | grep 'href="/docs' | \
uniq | cut -d'"' -f6
/docs/home/
/docs/tasks/debug-application-cluster/troubleshooting/
/docs/home/contribute/create-pull-request/
/docs/home/contribute/write-new-topic/
/docs/home/contribute/stage-documentation-changes/
/docs/home/contribute/page-templates/
/docs/home/contribute/review-issues/
/docs/home/contribute/style-guide/
/docs/setup/
/docs/setup/pick-right-solution/
/docs/getting-started-guides/minikube/
/docs/setup/independent/install-kubeadm/
/docs/setup/independent/create-cluster-kubeadm/
/docs/getting-started-guides/scratch/
...
```

### 출력시키지 않을 부분들을 지우기

아래와 같이 layout에서 출력시키기 싫은 부분들을 다 지웠다. 기존코드와 비교해 보면 한눈에 알 수 있을 것이다.

[변경된 코드](https://github.com/leoh0/website/commit/6bde83fce7174f106eb63bbf98af6aacf2a2b0c4)

물론 이후에 다시 빌드 한다.

```
$ make build
```

### 여기에 웹페이지 들을 pdf 화 하기

우선 출력을 위해서 서버를 띄운다.

```
$ make serve
bundle exec jekyll serve
Configuration file: /Users/al/Projects/k8s/website/_config.yml
Configuration file: /Users/al/Projects/k8s/website/_config.yml
            Source: /Users/al/Projects/k8s/website
       Destination: /Users/al/Projects/k8s/website/_site
 Incremental build: enabled
      Generating...
                    done in 7.797 seconds.
 Auto-regeneration: enabled for '/Users/al/Projects/k8s/website'
Configuration file: /Users/al/Projects/k8s/website/_config.yml
    Server address: http://127.0.0.1:4000/
  Server running... press ctrl-c to stop.
...
```

그리고 여러가지 방법이 있겠지만 chrome이 깔려있으면 가장 간단할 headless로 pdf 로 출력 시킨다. 아래의 list는 위의 url들 리스트 이다.

```
$ alias chrome="/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome"

$ for url in $(cat list); do
  echo $url
  # url로 파일이름을 만듬
  file=$(echo $url.pdf | sed 's|/docs/||g;s|/|-|g')
  chrome --headless --disable-gpu --print-to-pdf=$file http://localhost:4000$url
done

```

위와 같이하면 각 항목들의 url들이 pdf 화 된다.

### pdf 는 어떻게 합치지?

py27 기준으로 pdfmerge 를 쓰면 간편하다.
아래와 같이 커맨드로 합치면 된다.

```
$ pip install pdfmerge
$ pdfmerge -o output.pdf home-.pdf \
tasks-debug-application-cluster-troubleshooting-.pdf \
home-contribute-create-pull-request-.pdf \
...
```

### 결론

이 과정을 거쳐서 pdf를 만들어 보면 왜 pdf를 지원하지 않는지 알게된다.

왜냐하면 나온결과가 1818 페이지의 pdf 이기 때문이다.

그래서 결론적으로 그냥 website에서 보는게 나은 것 같다.
