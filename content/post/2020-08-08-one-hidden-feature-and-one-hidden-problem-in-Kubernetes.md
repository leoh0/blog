---
title: "2020 08 08 One Hidden Feature and One Hidden Problem in Kubernetes"
date: 2020-08-08T01:12:11+09:00
draft: false
toc: false
categories: ["technology"]
tags: ["k8s", "kubernetes", "service", "external", "ip", "ipvs"]
images: [
  "https://source.unsplash.com/collection/983219/1600x900"
]
---

# kubernetes에 다양한 외부 네트워크 연동 방법

kubernetes 는 어플리케이션을 외부에 연계하기 위해 ingress, service 등의 기능들을 제공합니다.
각각 ingress 는 L7 에 가까운 기능 그리고 service 는 L4 에 가까운 기능을 제공합니다.

일반적인 외부에 연동하는 방법들을 살펴보면 아래와 같습니다.

`ingress`는 보통 LB의 ip는 지정되어 있고 특정한 포트들을 지정해서 서비스 합니다. 80, 443과 같은 http, https 와 같은 포트를 서비스 하고 물론 tcp, udp등을 지원 하나 포트가 변경될때마다 ingress 의 controller 들이 다시 로딩되야 하여 포트를 추가하기는 번거롭습니다.
다만 물론 virtual host, virtual path 등으로 여러 종류의 서비스들이 연계 될 수 있습니다.

`service`는 이와 달리 4가지 L4관련된 기능들을 제공합니다. 보통 특별한 도움이 있으면(외부 LB, metal lb 등) loadbalancer와 같은 타입으로 서비스 가능하고 만약 k8s 내부에서만 사용할 것이라면 cluster ip를 사용합니다. 이 외에도 외부 연동을 하려고 하면 nodeport와 같은 방법을 사용하고 또한 외부 ip를 사용가능 하다면 external ip를 사용하게 됩니다.

이 외에도 pod 자체를 `host network` 혹은 `host port`로 열어서 외부 연동을 할 수 있습니다.

# k8s에서 아무포트나 열어서 내부 서비스를 외부에 연계 하는 방법이 있을까

k8s를 서비스 하다보면 이런 욕구들이 들때가 있습니다.

> mysql 서비스를 올려서 외부에 서비스 하고 싶은데 metal lb나 외부 lb를 사용 할 수 없고 포트는 3306으로 열어야 함
 
저 같은 경우도 저런 케이스를 이야기 하면 보통은 안된다고 합니다. mysql 같은 서비스들은 L4로 동작해야 되기 때문에 ingress 없이 ip:port가 요구 될 때들이 있기에 ingress 사용은 어렵고, 외부 LB가 없기에 nodeport를 사용해야 하나 nodeport range는 일반적으로 30000 번대를 사용하게 됩니다. 물론 이 range도 조정 가능하나 기존 포트대역과 겹칠 우려등이 있어서 일반적으로 이 range를 조정하기는 껄끄럽습니다. 그래서 결국 host network 이나 host port를 사용해야 하나 라는 고민을 하게 됩니다. 물론 host network host port를 사용가능 하나 결국 해당 ip 등과 함께 사용되고 admin 권한으로 떠야 하기에 찝찝한 부분들이 있습니다.

> kubernetes api를 6443 포트로 열어 놨는데 똑같은 api를 6444로 한벌 더 서비스 할 수 있으면

일반적인 kubernetes api들은 node ip로 6443으로 떠 있습니다. 근데 6443을 특정 이유로 외부로 사용을 못해서 다른 포트로 서비스해야 할시 기존의 6443을 닫고 6444로 포트를 변경해서 pod을 다시 띄우게 됩니다. 하지만 실제 외부 코드들 연동을 하다보면 6443을 그대로 사용한다던지 해서 6443을 클러스터 내에서는 호출 할 수 있기에 닫고 싶지는 않고 6444만 추가로 열고 싶은 니즈가 있을 수 있습니다. 이런건 보통 생각했을때 kubernetes에서 처리하기는 좀 애매하다고 생각되게 됩니다. nodeport range도 아니고 이미 떠있는 pod을 변경하지 않고 추가적인 포트만 띄워야 한다고 생각하면 그냥 iptables rule을 조작하는게 간단하다고 생각할 수 있습니다.

# kubernetes에 잘 알려지지 않은 내부 서비스를 외부에 연계 하는 방법

우선 단순히 답은 `Service` 의 `external ip`를 사용하는 것입니다. 일반적으로 설명이 external ip가 외부 lb 연동시 사용이 된다고 설명되어 있으나 여기서의 방법은 `external ip`들을 `node ip`들을 이용하는 것입니다.

예를 들어서 아래와 같은 서비스를 생성 했다고 하면 selector는 master apiserver들을 바라보고 있으니 해당 pod의 6443을 6444로 열어 주게 됩니다.

```
apiVersion: v1
kind: Service
metadata:
  name: port-forward
  namespace: kube-system
spec:
  externalIPs:
  - ${MASTER1_IP}
  - ${MASTER2_IP}
  - ${MASTER3_IP}
  ports:
  - name: http
    port: 6444
    protocol: TCP
    targetPort: 6443
  selector:
    component: kube-apiserver
    tier: control-plane
```

이렇게 위와 같이 적용할 시 master api는 각 master api node에서 6443과 6444 로 서빙 하게 됩니다. 그리고 재미 있는 것은 사실 이렇게 여는 포트는 `netstat` 으로는 숨겨지게 됩니다. 

```
# netstat -tapn | grep 6444

```

이 방법을 이해하게 되면 node 들의 ip를 이용해서 쉽게 내가 원하는 서비스를 원하는 포트로 열어서 외부에 서비스 시킬 수 있습니다. 그리고 또한 이 방법의 장점은 이 ip 들도 HA 할 수 있다는 점입니다. 그리고 손쉽게 externalIPs 의 값을 floating IP 처럼 변경해서 사용 가능 하다는 점이 있습니다. 물론 단점은 node가 추가 삭제 될 시 이런 엔트리도 관리 되야 된다는 점이 있지만 생각보다 간단한 아이디어로 필요한 기능들을 하기 좋은 방법이라고 생각합니다.

다만, 아래의 문제를 제외하면 입니다. 그래서 이 방법을 사용하시기 전에 아래와 같은 상황인지 다시 한번 확인 하고 사용해야 합니다.

# IPVS kube proxy 이용시 이 방법을 사용하면 안되는 이유

**단도직입적으로 IPVS를 사용하고 있다면 이 방법을 사용하면 안됩니다.** 

우선 IPVS 모드를 많이 사용해보신 분들은 대부분 [external ip 가 내부에서 블랙홀 처럼 작용되는 문제](https://github.com/kubernetes/kubernetes/issues/75290)가 있다는 사실을 알고 계실 겁니다. 그래서 이 방법(위의 yaml)을 만약 적용한다면 100% 클러스터가 망가지게 됩니다.

왜냐하면 ipvs에서는 external ip를 ipvs interface에 추가해서 모든 트래픽을 해당 인터페이스로 처리 하게 되어서 저 포트 외의 것들도 모두 해당 인터페이스에서 처리하려고 하기 때문입니다. 그래서 실제 위 같은 yaml를 추가하고 만약 etcd를 master 서버에서 사용중이라면 etcd 의 모든 peer 싱크가 깨지게 됩니다. 왜냐하면 peer 들에 보내는 패킷이 자신한테 가기 때문이죠.

만약 이상태가 되면 그래서 복구절차가 좀 복잡합니다.


1. 각 마스터의 kubelet을 내림 (kube-proxy가 살아나지 못하도록)

    ```
    systemctl stop kubelet
    ```

1. 각 마스터의 kube-proxy를 제거 (ipvs entry 추가하지 못하도록)

    ```
    docker stop $(docker ps | grep kube-proxy | head -n1 | awk '{print $1}')
    ```

1. 각 마스터의 ipvs 엔트리를 제거 

    ```
    ip a d ${MASTER1_IP} dev kube-ipvs0
    ip a d ${MASTER2_IP} dev kube-ipvs0
    ip a d ${MASTER3_IP} dev kube-ipvs0
    ```

1. 위의 작업을 마스터 3군데에서 다 진행시 etcd가 연결되어 kubectl command가 가능해짐 그때 위의 서비스를 삭제

    ```
    kubectl delete svc port-forward -n kube-system
    ```

1. 이후 다시 kubelet 을 시작하여 kube-proxy, kube-master, etcd 등 복구

    ```
    systemctl start kubelet
    ```

1. 이후 모든 worker node에서 위의 행동을 반복 (보통 ipvs에서 엔트리만 제거 해도 될 수 있지만 특수 한 경우 kube-proxy에서 계속 추가 할 수 있어서 kube proxy도 재시작 해주는게 안전함)

    ```
    systemctl stop kubelet
    docker stop $(docker ps | grep kube-proxy | head -n1 | awk '{print $1}')
    ip a d ${MASTER1_IP} dev kube-ipvs0
    ip a d ${MASTER2_IP} dev kube-ipvs0
    ip a d ${MASTER3_IP} dev kube-ipvs0
    systemctl start kubelet
    ```

# 진짜 문제는 여기서 부터

이 문제의 진짜 문제점은 `soft multi-tenancy model` *(namespace 들로 구분해서 tenant 들에게 나눠줌)* 을 사용하면서 `IPVS` 모드를 사용하고 있을 때 입니다. 만약 사용자가 해당 클러스터에서 자신의 namespace 밖에 조작을 못하더라도 자신의 namespace 안에서 서비스 생성만으로 쉽게 master 노드의 etcd를 망가트릴 수 있습니다. 왜냐하면 externalIPs 에 ip만 추가되어도 ipvs에서 블랙홀 효과를 낼 수 있기 때문입니다.

이 뿐만 아니라 클러스터 내에서 장난을 칠 방법이 무궁무진 해집니다. 특정 ip 대역을 `admission webhooks` 들을 만들어서 service 생성시 검사해서 막는다 하더라도 사실 이게 node ip 뿐만 아니라 온갖 ip 들을 등록해서 장난을 칠 수 있기 때문에 결국 externalIPs 기능을 온전히 쓰기 어려워 진다고 봅니다.

아무래도 현재 IPVS는 다양한 기능 때문에 선호하게 되면서도 이런 문제가 있다보니 특정한 목적에는 사용이 꺼려지는 부분이 있습니다. 향후에는 이런문제가 해결 할 수 있는 방법이 있으면 좋겠습니다. 다만 그래도 IPVS혼자 만으로는 힘들고 주변 iptables 등과 같은 도움을 이용해야 하다보니 우선은 깔끔한 방법이 나오기는 힘든것 같다고 생각합니다.

혹시라도 제가 이문제를 겪은게 작년이여서 더 좋은 방법이 나왔다면 알려주시면 감사하겠습니다. 아무래도 이 이후부터는 IPVS를 사실 안쓰기 시작해서 요새는 좋은 방법이 있는지 모르겠습니다. :)

긴 글 읽어 주셔서 감사합니다.
