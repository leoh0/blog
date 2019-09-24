---
title: "How to use static ip in kubernetes nginx ingress without default backend"
date: 2019-09-24T20:55:44+09:00
draft: false
toc: false
categories: ["technology"]
tags: ["k8s", "kubernetes", "ingress", "static", "ip"]
images: [
  "https://source.unsplash.com/collection/983219/1600x900"
]
---

kubernetes ingress는 일반적으로 virtual host, virtual path등을 위한 서비스들을 제공하고 이는 DNS기반으로 작동되게 되어 있습니다.

만약 이런 ingress를 ip기반으로 사용하고 싶다고 하더라도 일반적으로 [ingress validation](https://github.com/kubernetes/kubernetes/blob/release-1.16/pkg/apis/networking/validation/validation.go#L247)에 막혀있습니다.

그래서 일반적으로는 [nginx ingress static-ip example](https://github.com/kubernetes/ingress-nginx/tree/master/docs/examples/static-ip)이런 예제와 같이 default backend를 통해 ip로 들어오는 리퀘스트가 처리 됩니다.

그렇다면 진짜로 ip로 들어오는 리퀘스트를 default backend가 아닌 다른 service로 연결할 수 없을까요? 물론 이 포스트를 쓰는 목적이지만 제한적이지만 답은 있습니다.

우선 힌트는 위의 [ingress validation](https://github.com/kubernetes/kubernetes/blob/release-1.16/pkg/apis/networking/validation/validation.go#L247) 라고 할 수 있는데, 뭐냐하면 사실 위에서 ParseIP를 통해 ip address가 파싱되는지를 확인해서 이로 설정되는 것을 막고 있지만 실제 ip address는 다양한 방법으로 표기 가능합니다.

예를 들어서 172.17.0.6 이라는 ip가 있다면 다음과 같은 방법들이 가능합니다.

* 172.17.6
* 2886795270 (172*256*256*256+17*256*256+0*256+6)
* 0xac.0x11.0x0.0x6
* 0xac.0x11.0x6
* 0xac110006
* 0254.0021.0.0006

그리고 이 모든 방법으로 호스트 등록이 가능합니다.

예를 들면 아래와 같이 `172.17.0.6` 와 같이 daemonset 으로 nginx ingress 가 hostnetwork로 열어서 생성했다고 했을때.
```
$ kubectl get pods -o wide
NAME                                                  READY   STATUS    RESTARTS   AGE   IP            NODE                 NOMINATED NODE   READINESS GATES
apple-app                                             1/1     Running   0          33m   10.244.0.13   kind-control-plane   <none>           <none>
test-nginx-ingress-controller-hmr7q                   1/1     Running   0          60m   172.17.0.6    kind-control-plane   <none>           <none>
test-nginx-ingress-default-backend-559589ff74-8pk5b   1/1     Running   0          60m   10.244.0.4    kind-control-plane   <none>           <none>
```

만약 이 ingress를 ip에 바로 request를 던질시 default backend의 404가 응답됩니다.
```
$ curl 172.17.0.6
default backend - 404
```

하지만 아래 처럼 ingress들을 생성했다고 한다면
```
$ kubectl get ing 
NAME     HOSTS        ADDRESS   PORTS   AGE
nginx    172.17.6               80      53m
nginx2   2886795270             80      52m
```

아래와 같이 바로 ip만으로도 정상적인 응답을 하게 됩니다.
```
$ curl 172.17.6
apple
$ curl 2886795270 
apple
```

# 해설

사실 이게 가능한건 curl의 특성때문입니다. 실제 `http://2886795270`를 브라우저에서 호출하면 `default backend - 404`를 호출합니다.
하지만 curl에서 작동하는 이유는 `2886795270`와 같이 기본 ip address가 아닌 표기법(notation)을 이용할 시 ip로 자동으로 변환해서 호출하며 Host header에 해당 host 이름을 넣어주기 때문입니다.
그래서 curl로는 쉽게 작동되지요, 물론 이처럼 다른 방법들도 전부 해당 ip address를 호출될시 Host header를 집어 넣는 방법으로 작동 가능합니다.

```
$ curl 2886795270 -v
*   Trying 172.17.0.6:80...
* TCP_NODELAY set
* Connected to 2886795270 (172.17.0.6) port 80 (#0)
> GET / HTTP/1.1
> Host: 2886795270
> User-Agent: curl/7.65.3
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: nginx/1.15.9
< Date: Tue, 24 Sep 2019 11:54:31 GMT
< Content-Type: text/plain; charset=utf-8
< Content-Length: 6
< Connection: keep-alive
< X-App-Name: http-echo
< X-App-Version: 0.2.3
< 
apple
* Connection #0 to host 2886795270 left intact
```

재미로 보셨길! :)
