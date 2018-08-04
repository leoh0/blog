---
layout: post
title: "make l2 network for vm without neutron"
date: 2015-02-02 09:20:23 +0900
comments: true
categories: 
- openstack
- neutron
- ovs
- openvswitch
---

우선 아래와 같은 컨피그를 이용할때..
## nova ##
* linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver

## neutron ##
* type_drivers = vlan
* mechanism_drivers = openvswitch
* firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

## 1. 기본 상태:
우선 실제 물리 머신에 아래와 같이 vm을 위한 ethernet인 eth1이 존재한다.

{% img /images/neutron-nova-network-1.jpg 497 395 %}

### 변수 생성
``` bash
vmuuid=9ee8db25-6a13-4ca8-8a4e-495db24492ca
vlan=2001
localvlan=1
ip=$(nova show $vmuuid | grep ' network' | awk '{print $5}')
interfaceid=$(neutron port-list | grep "\"$ip\"" | awk '{print $2}')
mac=$(neutron port-list | grep "\"$ip\"" | awk '{print $5}')
ovsid=${interfaceid:0:11}
```

## 2. 유저 생성 단계:
eth1에 연결된 br-eth1 bridge를 생성한다.
또한 gre, vlan등의 다양한 type들의 네트워크를 연결시켜 줄 수 있는 br-int bridge를 생성한다.

{% img /images/neutron-nova-network-2.jpg 497 395 %}

### user가 생성 필요
``` bash
ovs-vsctl --timeout=10 -- --may-exist add-br br-eth1
ovs-vsctl --timeout=10 -- --may-exist add-port br-eth1 eth1
```

## 3. neutron openvswitch plugin 시작:
br-eth1과 br-int간의 bridge 연결을 위한 port를 생성하며 veth로 해당 port를 연결하고 해당 bridge에 기본적인 flow를 추가해 준다.

{% img /images/neutron-nova-network-3.jpg 497 395 %}

### neutron - br-int setup
``` bash
ovs-vsctl --timeout=10 -- --may-exist add-br br-int
ovs-vsctl --timeout=10 -- set-fail-mode br-int secure
ovs-vsctl --timeout=10 -- --if-exists del-port br-int patch-tun
ovs-ofctl del-flows br-int
ovs-ofctl add-flow br-int hard_timeout=0,idle_timeout=0,priority=1,actions=normal
ovs-ofctl add-flow br-int hard_timeout=0,idle_timeout=0,priority=0,table=22,actions=drop
```

### neutron - physical bridge setup
``` bash
ovs-ofctl del-flows br-eth1
ovs-ofctl add-flow br-eth1 hard_timeout=0,idle_timeout=0,priority=1,actions=normal
```

### neutron - int <-> phy setup
``` bash
ovs-vsctl --timeout=10 -- --if-exists del-port br-int int-br-eth1
ovs-vsctl --timeout=10 -- --if-exists del-port br-eth1 phy-br-eth1
ip link delete int-br-eth1
ip link delete phy-br-eth1
udevadm settle --timeout=10
ip link add int-br-eth1 type veth peer name phy-br-eth1
ovs-vsctl --timeout=10 -- --may-exist add-port br-int int-br-eth1
ovs-vsctl --timeout=10 -- --may-exist add-port br-eth1 phy-br-eth1
intport=$(ovs-ofctl show br-int | grep int-br-eth1 | awk '{print $1}' | cut -d'(' -f1)
phyport=$(ovs-ofctl show br-eth1 | grep phy-br-eth1 | awk '{print $1}' | cut -d'(' -f1)
ovs-ofctl add-flow br-int hard_timeout=0,idle_timeout=0,priority=2,in_port=${intport},actions=drop
ovs-ofctl add-flow br-eth1 hard_timeout=0,idle_timeout=0,priority=2,in_port=${phyport},actions=drop
ip link set int-br-eth1 up
ip link set phy-br-eth1 up
```

## 4. nova에서 인스턴스 생성:
nova에서는 qbr bridge를 생성하며 해당 bridge에 qvb port를 생성하고 br-int에 qvo port를 생성하여 두 port를 veth로 연결한다.(xml에 관련해서 여기에서 생성된다.)

{% img /images/neutron-nova-network-4.jpg 497 395 %}

### nova - instance up
``` bash
brctl addbr qbr${ovsid}
brctl setfd qbr${ovsid} 0
brctl stp qbr${ovsid} off
echo '0' > /sys/class/net/qbr${ovsid}/bridge/multicast_snooping

ip link add qvb${ovsid} type veth peer name qvo${ovsid}
ip link set qvb${ovsid} up
ip link set qvb${ovsid} promisc on
ip link set qvo${ovsid} up
ip link set qvo${ovsid} promisc on

ip link set qbr${ovsid} up
brctl addif qbr${ovsid} qvb${ovsid}
ovs-vsctl --timeout=120 -- --if-exists del-port qvo${ovsid} -- add-port br-int qvo${ovsid} -- set Interface qvo${ovsid} external-ids:iface-id=${interfaceid} external-ids:iface-status=active external-ids:attached-mac=${mac} external-ids:vm-uuid=${vmuuid}
```

## 5. neutron openvswitch plugin에서 ovsdb-client monitoring을 통한 풀링:
br-int의 변화를 감지하여 해당 qvo port를 위한 vlan tagging을 해주며 이를 위한 flow를 br-int와 br-eth1에 추가한다.

{% img /images/neutron-nova-network-5.jpg 497 395 %}

### neutron - instance up
``` bash
ovs-ofctl add-flow br-int hard_timeout=0,idle_timeout=0,priority=3,in_port=${intport},dl_vlan=${vlan},actions=mod_vlan_vid:${localvlan},normal
ovs-ofctl add-flow br-eth1 hard_timeout=0,idle_timeout=0,priority=4,in_port=${phyport},dl_vlan=${localvlan},actions=mod_vlan_vid:${vlan},normal

ovs-vsctl --timeout=10 set Port qvo${ovsid} tag=${localvlan}
```

## 6. vm 부팅:
vm이 시작하면서  libvirt의 xml파일안에 정의 되어 있는 qbr bridge에 tap port가 생성되어서 vm까지 네트워크가 완성된다.(이 xml은 이미 4번 단계에서 완성되어 있다.)

{% img /images/neutron-nova-network-6.jpg 497 395 %}
