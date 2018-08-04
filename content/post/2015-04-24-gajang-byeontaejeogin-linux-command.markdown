---
layout: post
title: "가장 변태적인 linux command"
date: 2015-04-24 00:28:38 +0900
comments: true
categories:
- ip
---

ip command 는 linux network 쪽을 위해 굉장히 많이 사용되는 커맨드 이다.
```bash
Usage: ip [ OPTIONS ] OBJECT { COMMAND | help }
       ip [ -force ] -batch filename
where  OBJECT := { link | addr | addrlabel | route | rule | neigh | ntable |
                   tunnel | tuntap | maddr | mroute | mrule | monitor | xfrm |
                   netns | l2tp | tcp_metrics | token }
       OPTIONS := { -V[ersion] | -s[tatistics] | -d[etails] | -r[esolve] |
                    -f[amily] { inet | inet6 | ipx | dnet | bridge | link } |
                    -4 | -6 | -I | -D | -B | -0 |
                    -l[oops] { maximum-addr-flush-attempts } |
                    -o[neline] | -t[imestamp] | -b[atch] [filename] |
                    -rc[vbuf] [size]}
```

이 커맨드의 가장 변태적이라고 생각한 부분은 다음이다..

```bash
ip a
```
```bash
ip ad
```
```bash
ip add
```
```bash
ip addr
```
```bash
ip addre
```
```bash
ip addres
```
```bash
ip address
```

전부 동일한 결과가...

`link`, `route` 등도 다 저렇게 되있다고 보면된다.

별건 아닌데 참 `괴상한` `친절한` 커맨드라는 생각이 들어서 ㅋㅋ
