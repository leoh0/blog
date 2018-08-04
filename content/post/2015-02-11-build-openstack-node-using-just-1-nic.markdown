---
layout: post
title: "NIC 1개로 compute node를 vlan type으로 neutron을 사용하여 구성하기 위한 팁"
date: 2015-02-11 17:12:39 +0900
comments: true
categories: 
- openstack
- neutron
- compute-node
- openvswitch
---

gre 같은 tunnel 을 사용한다면 NIC 하나로 구성 할 수 있겠지만 그게 아니라면 일반적으로는 management 용 NIC 한개와 service 용 NIC 한개가 필요하다.

우선 아래와 같은 상태가 2 NIC을 사용하는 일반적인 구성이다. 

그림 처럼 eth0은 management를 위한 ip로 이용되며 eth1을 guest interface(vlan 이라면 0.0.0.0)로 사용할 수 있다.

{% img /images/1nic-neutron-1.png 449 487 %}


단도직입적으로 ethernet 한개로는 아래와 같이 구성하면 된다.

우선 eth0은 0.0.0.0 으로 ip를 사용안하는 대신 br0 부분에서 기존의 management용 ip를 가져간다.

{% img /images/1nic-neutron-2.png 449 487 %}

아래와 같은 흐름으로 진행하면 된다.

``` bash
# br0 switch 추가 (자동으로 br0 switch 에는 br0 interface가 달려있는 상태로 생성됨)
ovs-vsctl add-br br0
# br0 switch 에 eth0 interface를 추가 시켜줌
ovs-vsctl add-port br0 eth0
# ip 및 route 세팅
/sbin/ifconfig eth0 0.0.0.0 up
/sbin/ifconfig br0 x.x.x.x/xx up

# br-eth0 인터페이스를 생성 (br-ex 도 비슷)
ovs-vsctl add-br br-eth0
# br0 switch와 br-eth0 을 연결 시킬 veth 생성
ip link add br0-veth type veth peer name br-eth0-veth
ovs-vsctl add-port br0 br0-veth
ovs-vsctl add-port br-eth0 br-eth0-veth
```

만약 network node 라면 br-eth0 대신 br-ex 를 사용하고 추가적으로 nat 를 사용하기 위한 ip 하나만 할당하면 된다.
