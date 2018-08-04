---
layout: post
title: "커맨드로 크롬에 떠 있는 유튜브 플레이중인 음악 mp3로 다운로드 하기"
date: 2015-05-19 09:49:32 +0900
comments: true
categories:
- brew
- chrome-cli
- youtube-dl
---

내가 듣는 음악 장르는 보통 스트리밍 서비스에서 찾을 수 가 없어서 보통 유튜브 자동재생을 켜놓고 음악을 듣는 편이다.

그런데 이런 음악을 찾기 귀찮아서 mp3 파일로 다운로드 해두고 싶은일이 생긴다.

{% img /images/unsun-whispers.png 452 239 unsun - whispers %}

[UNSUN - Whispers Youtube](https://www.youtube.com/watch?v=LapknbGS7Os)

물론 그냥 아래 같이 `youtube-dl`와 같은 방법을 써도 된다.

``` bash
brew install youtube-dl
youtube-dl -t --extract-audio --audio-format mp3 $YOUTUBEURL
```

하지만 일하는데 터미널에서 크롬으로 창을 포커스 바꾸고 url을 카피해서 저 커맨드를 넣는건 귀찮은 일이다.

그래서 이런걸 해결해주는 [chrome-cli](https://github.com/prasmussen/chrome-cli) 라는 솔루션을 이용한다.

물론 설치는 간단하게..

``` bash
brew install chrome-cli
```

그리고 그냥 아래 같은 방법을 쓰면 chrome 에서 열려 있는 유튜브 페이지들을 전부 조회해서 mp3로 다운로드 할 수 있다.

``` bash
function yd(){
  for l in $(chrome-cli list tabs | grep YouTube | cut -d '[' -f2- | cut -d':' -f2- | cut -d']' -f1); do
    url=$(chrome-cli info -t $l | grep '^Url: ' | cut -d' ' -f2)
    (cd /want/to/download/mp3; youtube-dl -t --extract-audio --audio-format mp3 $url)
  done
}
```

