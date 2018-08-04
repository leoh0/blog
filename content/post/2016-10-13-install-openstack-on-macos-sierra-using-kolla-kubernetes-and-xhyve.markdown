---
layout: post
title: "Install openstack on macos Sierra using kolla-kubernetes with xhyve"
date: 2016-10-13 00:02:06 +0900
comments: true
categories:
- mitaka
- openstack
- install openstack
- kolla
- kolla-kubernetes
- kubernetes
- macos
- sierra
- docker
- container
---

<a href="http://showterm.io/404f651b005c52298bc9f">{% img /images/2016-10-13_00-04-57.jpg 924 533 %}</a>
[클릭해서 과정 보기](http://showterm.io/404f651b005c52298bc9f)

`kubernetes` 로 `openstack mitaka`를 설치해 봤습니다.

위의 이미지를 클릭하면 전과정을 보실 수 있습니다. 참고하시면 좋을 것 같습니다.

대략 31분 정도 걸려서 minikube 구성 부터 전체 셋업 및 vm 2개 띄워서 로그인 테스트 합니다.

그리고 저는 `macos Sierra` 에서 `xhyve` 로 8G 머신을 만들어서 사용을 했습니다.    
linux 머신이 있으면 kvm을 이용하시는 것도 좋을것 같습니다.    
windows 환경에선 xhyve 외에도 virtual box 등을 이용하셔도 좋을 것 같습니다.

셋업할때 `kubernetes-cli` 는 `1.3.6` 버전과 `minikube` 는 `0.11.0` 버전을 이용합니다.

앞서 말한 2가지 버전만 잘 지키고 bootstraping 이나 container가 올라올때 잘 기다리고 진행하면 크게 어렵게 진행 하실 수 있습니다.

자세한건 [kolla-kubernetes의 도큐먼트](https://github.com/openstack/kolla-kubernetes/blob/master/doc/source/minikube-quickstart.rst)를 참고하시면 도움이 되실 겁니다.


kolla-kubernetes 를 이용한 방법으로 기존 kolla 와 비슷한 플로우로 진행합니다.    
아무튼 그냥 devstack 처럼 테스트 용으로 보시면 좋을겁니다.
