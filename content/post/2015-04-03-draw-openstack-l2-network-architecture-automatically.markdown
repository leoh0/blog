---
layout: post
title: "draw openstack L2 network architecture automatically"
date: 2015-04-03 02:29:52 +0900
comments: true
categories:
- technology
tags:
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
{{< figure src="/images/draw-bridge-vlan.png" >}}

# openvswitch-flat
{{< figure src="/images/draw-ovs-flat.png" >}}

# openvswitch-vlan
{{< figure src="/images/draw-ovs-vlan.png" >}}

이걸 graphviz 로 그리면 다음과 같다.

# bridge-vlan
{{< figure src="/images/draw-bridge-vlan-g.png" >}}

# openvswitch-flat
{{< figure src="/images/draw-ovs-flat-g.png" >}}

# openvswitch-vlan
{{< figure src="/images/draw-ovs-vlan-g.png" >}}

이걸 3D 로 그리면 다음과 같다.

# bridge-vlan
{{< figure src="/images/draw-bridge-vlan-3d.png" >}}

# openvswitch-flat
{{< figure src="/images/draw-ovs-flat-3d.png" >}}

# openvswitch-vlan
{{< figure src="/images/draw-ovs-vlan-3d.png" >}}

는 사실 그냥 이전에 그려논 그림이다..

아무튼 해당 그림을 그리기 위해 제작한 스크립트 이다.   
아래 스크립트를 컴퓨트 노드에서 돌리면 해당 정보를 수집해서 그리게 된다. (물론 네트워크 노드도 가능..)   
귀찮아서 하드코딩한 부분들은 편하게 고쳐쓰시길..

{{< highlight bash "linenos=table" >}}
#!/bin/bash

sudo apt-get install -qqy ethtool libgraph-easy-perl graphviz > /dev/null

EXCEPT=/tmp/exceptlist
echo '' > $EXCEPT
result=""

function on_exit() {
  rm -f $EXCEPT
}

trap "on_exit" EXIT

# find ovs br <-> port
if [ "x$(which ovs-vsctl)" != "x" ]; then
  for br in $(sudo ovs-vsctl list-br); do
    for port in $(sudo ovs-vsctl list-ports $br); do
      result=$(echo "$result [$port]---->[$br] [$br]---->[$port] ")
    done
  done
fi

# find br <-> port
for br in $(brctl show | sed '1d' | grep '^[a-z]' | awk '{print $1}'); do
  for port in $(brctl show $br | sed '1d' | sed 's/.*\t.*\t.*\t\(.*\)/\1/g'); do
    result=$(echo "$result [$port]---->[$br] [$br]---->[$port] ")
  done
done

# ip namespace veth
for ns in $(ip netns); do
  for interface in $(ip netns exec $ns ip a | cut -d':' -f-2 | grep ^[1-9]); do
    index=$(ip netns exec $ns ethtool -S $interface 2> /dev/null | grep peer_ifindex | awk '{print $2}')
    ifname=$(ip netns exec $ns ip a | grep "^$index:" | awk '{print $2}' | cut -d':' -f1)
    if [ "x$ifname" == "x" ]; then
      ifname=$(ip a | grep "^$index:" | awk '{print $2}' | cut -d':' -f1)
      if [ "x$ifname" != "x" ]; then
        echo $ifname >> $EXCEPT
        result=$(echo "$result [$interface]---->[$ifname] [$ifname]---->[$interface] ")
      fi
    fi
  done
done

# ip veth
for interface in $(ip a | cut -d':' -f-2 | grep ^[1-9]); do
  if cat $EXCEPT | grep -q "^$interface$" ; then continue ; fi
  index=$(ethtool -S $interface 2> /dev/null | grep peer_ifindex | awk '{print $2}')
  ifname=$(ip a | grep "^$index:" | awk '{print $2}' | cut -d':' -f1)
  if [ "x$ifname" != "x" ]; then
    echo $ifname >> $EXCEPT
    result=$(echo "$result [$interface]---->[$ifname] [$ifname]---->[$interface] ")
  fi
done

# vm tap
for tap in $(ip a | cut -d':' -f-2 | grep ^[1-9]  | cut -d' ' -f2 | grep '^tap'); do
  vmuuid=$(grep -rl "$tap" /var/lib/nova/instances/*/libvirt.xml | cut -d'/' -f6)
  if [ "x$vmuuid" != "x" ]; then
    result=$(echo "$result [$tap]---->[VM-$vmuuid] [VM-$vmuuid]---->[$tap] ")
  fi
done

rm -f $EXCEPT

echo $result | graph-easy
echo $result | graph-easy -as dot | dot -Tpng -o l2path.png{{< /highlight >}}
