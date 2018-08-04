---
layout: post
title: "containerize openstack"
date: 2015-04-24 01:08:16 +0900
comments: true
categories:
- openstack
- containerize
- docker
- kvm
- libvirt
- qemu
- kolla
---

docker 바깥으로 process를 낼 수 있는 방법들로 kvm 프로세스를 docker 바깥 host os 에 배치시키는 방법이 가능하네요.    
실질적으로 docker안에 kvm이 들어가게 된다면 docker process에 vm이 종속적이게 되어 불안정한 구성이 될테지만    
이런 방식으로 피해 갈 수도 있는 것을 확인했습니다.

{% img /images/containerize-openstack.png 1440 829 containerize-openstack %}

아무튼 사진과 같이 기묘한 형태로 (실제 host os 에는 없는 바이너리가 parent 1 을 물고 process 로 뜨게되는) 관리가 가능합니다.    
물론 보안 적인 결함에 대해서야 아직 끝도없이 이야기할 주제이겠지만 앞으로의 provisioning의 새로운 가능성에 대해서 관심이 가는건 사실인것 같습니다.

아무튼 열심히 삽질하다보니 kolla에서 이미 하고 있었어서 libvirt 를 containerize 하는데 도움을 받았습니다.

[https://github.com/stackforge/kolla](https://github.com/stackforge/kolla)

우선 저는 ubuntu로 kolla 소스가 아니라 따로 구성해서 실험했습니다.    
fedora와 ubuntu의 미묘한 차이가 있어서 ubuntu로 구성하려면 약간의 차이가 필요한것 같습니다.
