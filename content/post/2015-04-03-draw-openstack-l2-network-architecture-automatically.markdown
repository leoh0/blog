---
layout: post
title: "draw openstack L2 network architecture automatically"
date: 2015-04-03 02:29:52 +0900
comments: true
categories:
- openstack
- openvswitch
- bridge
- graph-easy
- graphviz
- network architecture
- neutron
---

iptables를 좀 보기 편하게 할 수 없는가를 이야기하다가 [여기](http://atoato88.hatenablog.com/entry/2014/01/25/133852)사이트를 보게되었다.   
그래서 감동을 받아서 이에 뭔가 남기고자 삽질을 했다. (어짜피 요새 deploy 테스트 하다보면 남는게 시간이다 보니..)   
대략 openstack neutron의 L2 architecture 에 구성요소들을 좀 보기 편하게 그린것이다.   
지금 tunnel architecture를 가진건 없어서 br-tun 쪽은 그리려고 테스트 하진 않았다. 다만 ovs 나 bridge 모드에서 대략적인 그림은 맘에 들게 그려지는 것 같다.   

ascii 로 그린 architecture 들이다.

# bridge-vlan
{% img /images/draw-bridge-vlan.png 1312 544 bridge-vlan %}

# openvswitch-flat
{% img /images/draw-ovs-flat.png 2526 780 openvswitch-flat %}

# openvswitch-vlan
{% img /images/draw-ovs-vlan.png 2752 544 openvswitch-vlan %}

이걸 graphviz 로 그리면 다음과 같다.

# bridge-vlan
{% img /images/draw-bridge-vlan-g.png 757 131 bridge-vlan %}

# openvswitch-flat
{% img /images/draw-ovs-flat-g.png 1101 491 openvswitch-flat %}

# openvswitch-vlan
{% img /images/draw-ovs-vlan-g.png 1594 131 openvswitch-vlan %}

이걸 3D 로 그리면 다음과 같다.

# bridge-vlan
{% img /images/draw-bridge-vlan-3d.png 2716 1564 bridge-vlan %}

# openvswitch-flat
{% img /images/draw-ovs-flat-3d.png 2844 1668 openvswitch-flat %}

# openvswitch-vlan
{% img /images/draw-ovs-vlan-3d.png 2844 2032 openvswitch-vlan %}

는 사실 그냥 이전에 그려논 그림이다..

아무튼 해당 그림을 그리기 위해 제작한 스크립트 이다.   
아래 스크립트를 컴퓨트 노드에서 돌리면 해당 정보를 수집해서 그리게 된다. (물론 네트워크 노드도 가능..)   
귀찮아서 하드코딩한 부분들은 편하게 고쳐쓰시길..

{% gist 8499b653f479766378d8 %}