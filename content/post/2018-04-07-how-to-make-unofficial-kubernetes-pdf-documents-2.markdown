---
layout: post
title: "how to make unofficial kubernetes pdf documents 2"
date: 2018-04-07 01:15:37 +0900
comments: true
categories: 
- kubernetes
- k8s
- document
- 1.10
- docker
- jekyll
- wkhtmltopdf
- ghostscript
- pdf
- unofficial
---

![kubernetes](/images/kubernetes-production-grade-container-orchestration.png)

### [download k8s 1.10 pdf](/images/kubernetes-documents-1.10.pdf)

---

### 이전 포스팅과 뭐가 다른가?

이전에 해보고 더이상 안할거라 생각했다가 우연히 다시 포스팅을 읽었다가 너무 복잡한거 같아서 docker로 제작했다.

아래와 같이 사용할시 pdf를 뽑아 낼 수 있다.

```
docker run -ti -v $PWD:/out3 leoh0/k8s-website-to-pdf
```

아니면 아래 dockerfile로 새로 제작해서 뽑아내면 된다. [참고](https://github.com/leoh0/k8s-website-to-pdf)

```
docker build -t k8s-website-to-pdf .
docker run -ti -v $PWD:/out3 k8s-website-to-pdf
```

우선 전체 dockerfile은 아래와 같다.

```
# 참고 https://github.com/leoh0/k8s-website-to-pdf/blob/master/Dockerfile
FROM alpine/git as source

ARG BRANCH=master
WORKDIR /app

RUN git clone https://github.com/kubernetes/website && cd website && git checkout ${BRANCH}

RUN {% raw %}sed -i '/{% include header.html %}/d;/{% include_cached footer.html %}/d;/{% include footer-scripts.html %}/d;/^<!--  HERO  -->/,/^<\/section>/d;s/<div id="docsToc">/<div id="docsToc" style="display: none;">/g;/editPageButton/d;s/<div id="docsContent">/<div id="docsContent" style="width: 100%;">/g;/<p><a href=""><img src="https:\/\/kubernetes-site/,/{% endif %}/d' /app/website/_layouts/docwithnav.html{% endraw %}

FROM jekyll/jekyll as build

COPY --from=source /app/website /srv/jekyll

ARG TARGET=/build

RUN mkdir -p ${TARGET} && chown jekyll.jekyll ${TARGET}

RUN jekyll build --destination ${TARGET}/_site && cat ${TARGET}/_site/docs/home/index.html ${TARGET}/_site/docs/setup/index.html ${TARGET}/_site/docs/concepts/index.html \
  ${TARGET}/_site/docs/tasks/index.html ${TARGET}/_site/docs/tutorials/index.html | \
  grep 'a class="item"' | grep 'href="/docs' | \
  uniq | cut -d'"' -f6 > ${TARGET}/_site/list

FROM madnight/docker-alpine-wkhtmltopdf as pdfs

ARG TARGET=/build

COPY --from=build ${TARGET}/_site /_site

WORKDIR /_site

RUN mkdir -p /out /out2 && apk add --no-cache ghostscript

RUN count=1 ; for l in $(cat list); do sed -i 's|/css/|/_site/css/|g;s|/js/|/_site/js/|g;s|/images/|/_site/images/|g' /_site${l}index.html || : ; wkhtmltopdf /_site${l}index.html /out/$(printf "%03d" $count)-$(echo $l | sed 's/^.\(.*\).$/\1/;s|/|-|g').pdf || : ; count=$((count+1)) ; done

WORKDIR /out

RUN gs -q -sPAPERSIZE=letter -dNOPAUSE -dBATCH -sDEVICE=pdfwrite -sOutputFile=/out2/out.pdf $(ls /out)

VOLUME /out3

ENTRYPOINT ["sh"]

CMD ["-c", "cp /out2/out.pdf /out3/"]
```

위의 파일을 크게 2가지로 분류해서 보면 다음과 같다.

우선 아래까지는 이전에도 설명한것과 비슷하게 website repository를 가져와서 필요없는 부분을 적당히 제거하고 jekyll로 빌드하고 빌드할 document의 list를 제작한다.

```
{% raw %}# 참고 https://github.com/leoh0/k8s-website-to-pdf/blob/master/Dockerfile
FROM alpine/git as source

ARG BRANCH=master
WORKDIR /app

RUN git clone https://github.com/kubernetes/website && cd website && git checkout ${BRANCH}

RUN sed -i '/{% include header.html %}/d;/{% include_cached footer.html %}/d;/{% include footer-scripts.html %}/d;/^<!--  HERO  -->/,/^<\/section>/d;s/<div id="docsToc">/<div id="docsToc" style="display: none;">/g;/editPageButton/d;s/<div id="docsContent">/<div id="docsContent" style="width: 100%;">/g;/<p><a href=""><img src="https:\/\/kubernetes-site/,/{% endif %}/d' /app/website/_layouts/docwithnav.html

FROM jekyll/jekyll as build

COPY --from=source /app/website /srv/jekyll

ARG TARGET=/build

RUN mkdir -p ${TARGET} && chown jekyll.jekyll ${TARGET}

RUN jekyll build --destination ${TARGET}/_site && cat ${TARGET}/_site/docs/home/index.html ${TARGET}/_site/docs/setup/index.html ${TARGET}/_site/docs/concepts/index.html \
  ${TARGET}/_site/docs/tasks/index.html ${TARGET}/_site/docs/tutorials/index.html | \
  grep 'a class="item"' | grep 'href="/docs' | \
  uniq | cut -d'"' -f6 > ${TARGET}/_site/list{% endraw %}
```

이후엔 각 index.html이 web 기준이므로 로컬 파일 css, js를 참고 할 수 있게 경로 변경하고 wkhtmltopdf 로 pdf 생성한다.

다만 순서를 정렬하기 위해 앞에 숫자를 붙여서 제작한다.

이후에 ghostscript를 이용해서 letter size로 모든 pdf를 합친다. 나중에 합친 결과물 pdf를 뽑아내기 위해 커맨드를 세팅한다.

```
FROM madnight/docker-alpine-wkhtmltopdf as pdfs

ARG TARGET=/build

COPY --from=build ${TARGET}/_site /_site

WORKDIR /_site

RUN mkdir -p /out /out2 && apk add --no-cache ghostscript

RUN count=1 ; for l in $(cat list); do sed -i 's|/css/|/_site/css/|g;s|/js/|/_site/js/|g;s|/images/|/_site/images/|g' /_site${l}index.html || : ; wkhtmltopdf /_site${l}index.html /out/$(printf "%03d" $count)-$(echo $l | sed 's/^.\(.*\).$/\1/;s|/|-|g').pdf || : ; count=$((count+1)) ; done

WORKDIR /out

RUN gs -q -sPAPERSIZE=letter -dNOPAUSE -dBATCH -sDEVICE=pdfwrite -sOutputFile=/out2/out.pdf $(ls /out)

VOLUME /out3

ENTRYPOINT ["sh"]

CMD ["-c", "cp /out2/out.pdf /out3/"]
```

아무튼 이렇게 제작한 pdf는 2016페이지 이고 지난번 보다 200페이지가 증가했다.

역시나 읽지는..
