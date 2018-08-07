+++
title = "How to Debug Dead Container in K8s"
date = 2018-08-04T23:21:48+09:00
description = "죽음이 우리를 갈라 놓을때 까지"
categories = ["technology"]
draft = true
tags = ["k8s", "kubernetes", "dead", "container", "debug"]
images = [
  "https://source.unsplash.com/collection/983219/1600x900"
] # overrides the site-wide open graph image
+++

{{< figure src="/images/dead_docker.png" caption="" attr="" attrlink="" >}}

> Everyone knows that debugging is twice as hard as writing a program in the first place. 
>
>  “The Elements of Programming Style”, 2nd edition

# k8s에서 pod 디버깅

우리는 소프트웨어는 언제나 문제가 생길 수 있다는 것을 알고 있고 많은 문제들을 debugging 을 통해서 해결한다. container를 debugging 하는 것들은 준비여부(log, debugging tools, ...)가 중요해서 쉽지 않은 경우들이 대부분이다. 그렇기에 이런 container를 debugging 하기 위한 방법들은 필요에 따라 많은 발전을 했다.

그래서 이 글을 적을때만해도 이런 기술 부터 정리 하려고 했는데 아래 글에서 아주 잘 정리해서 이를 대체 하고자 한다.

[THE STATE OF DEBUGGING MICROSERVICES ON KUBERNETES](https://radu-matei.com/blog/state-of-debugging-microservices-on-k8s/)

위 블로그 내용을 대략 간추리면 아래와 같다.

public cloud에서는 아래와 같은 툴들을 지원해서 debugging 을 편하게 지원한다.

* Azure의 dev spaces
* Google의 stack driver debugger

원격 디버깅은과 같은 squash의 kubernetes 버전을 이용한다.

* [kubesquash](https://github.com/solo-io/kubesquash)

어플리케이션 디버깅을 도울수 있는 도구들은 아래와 같다.

* [KSync](https://github.com/vapor-ware/ksync): local filesystem를 k8s 내 container와 sync
* [Telepresence](https://github.com/telepresenceio/telepresence): k8s cluster로 들어오는 traffic을 local로 redirect
* [Draft](https://github.com/Azure/draft) and [Skaffold](https://github.com/GoogleContainerTools/skaffold): 일관된 재배포 방법 제공

k8s에서는 아래와 같은 기능들을 활용할 수 있다.

* [Share Process Namespace between Containers in a Pod](https://kubernetes.io/docs/tasks/configure-pod-container/share-process-namespace/) 를 이용해서 debugging할 container를 namespace에 sharing 시킴
* 향후엔 `debug` 커맨드를 지원해서 아래와 같은 커맨드로 임시적으로 debugger를 붙여서 디버깅을 할 수 있게 함. [Troubleshoot Running Pods](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/node/troubleshoot-running-pods.md)

   ```
   kubectl debug  --image=<your-debugger-image> <your-application-pod>
   ```

자세한 내용은 위 블로그에서 확인 가능하다.

# 저런 솔루션들로 모든 문제를 해결하면 되지 않나?

하지만 실제로 서비스를 하다보면 위 정보들은 `살아 있는` 컨테이너를 디버깅 하기 위한 것임을 알게된다. 그리고 `어플리케이션`을 디버깅 하기위한 수단들이여서 뭔가 부족함을 알게된다.

예를 들면 아래 같이 지속적으로 계속 리스타트 되고 있는 팟들은 디버깅 하다보면 계속 리스타트 되서 끊겨서 테스트를 하기 힘들다.
{{< highlight text >}}
$ kubectl get pod --all-namespaces | grep -v Running
NAMESPACE  NAME                         READY   STATUS             RESTARTS   AGE
test       my-very-important-pod-pgzgb  0/1     CrashLoopBackOff   6761       17d
...
{{< /highlight >}}

`CrashLoopBackOff`은 container가 crash해서 다시 backing off

이런 컨테이너가 `CrashLoopBackOff` 되고 있는 이유는 여러가지가 있을 수 있지만 대표적으로는 2가지로 본다.

k8s 안에서 죽은 컨테이너를 디버깅 하는 것은 쉬운 일이 아닙니다. 

```
kubectl logs nucleo-python-sample-full-4a418ac-76cb6c64cb-d7t9m -n charlie-choe -p
```