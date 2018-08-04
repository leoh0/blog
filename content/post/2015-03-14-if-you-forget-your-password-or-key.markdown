---
layout: post
title: "만약 사용자가 ssh password나 key등을 잃어 버려서 도저히 vm instance에 접속할 수 없을때.."
date: 2015-03-14 22:00:57 +0900
comments: true
categories: 
- openstack
- nova
- ssh password lost
- ssh login fail
- selinux
---


우선은 nbd로 attach 하여 mount 해서 접근 가능하다.   
이후에 아래와 같이 key를 추가해도 되고 각 os 버전에 맞게 패스워드를 새로 해슁하여 `/etc/shadow` 를 변경해도 된다.   
물론 마지막에 dettach 를 꼭 신경써서 해줘야 한다.

``` bash
# attach disk
qemu-nbd -c /dev/nbd0 /var/lib/nova/instances/10794bbb-7856-4ed6-ab39-32afbc01156a/disk
mount /dev/nbd0p1 /mnt

# insert a new key to target user
echo '''NEWKEY''' >> /mnt/home/USER/.ssh/authorized_keys

# dettach disk
umount /mnt
qemu-nbd -d /dev/nbd0p1
```

하지만 이상하게도 저런 수정을 했을때 아예 접속이 불가능한 경우들이 생긴다. sshd config가 잘못되었는지 고치기 시작하면 아예 ssh 조차도 뜨질 못한다.   
그이유는 아래와 같이 selinux의 보안 설정으로 위변조된 파일 사용시 차단되는 보안이 설정되어 있기 때문이다.

즉, selinux를 살펴 봤을때 아래 같이 selinux가 세팅되어 있을 수 있다.   
그렇다면 위와 같이 edit 했을때 관련 키, 계정들을 접근 못하게 된다.   

``` bash /etc/selinux/config
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=enforcing
# SELINUXTYPE= can take one of these two values:
#     targeted - Targeted processes are protected,
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```

그렇기 때문에 이런 케이스는 아래와 같이 `selinux`를 *disable* 해줘야 한다.

``` bash
# attach disk
qemu-nbd -c /dev/nbd0 /var/lib/nova/instances/10794bbb-7856-4ed6-ab39-32afbc01156a/disk
mount /dev/nbd0p1 /mnt

# insert a new key to target user
echo '''NEWKEY''' >> /mnt/home/USER/.ssh/authorized_keys

# disable selinux
sed -i 's/^SELINUX=.*$/SELINUX=disabled/g' /mnt/etc/selinux/config

# dettach disk
umount /mnt
qemu-nbd -d /dev/nbd0p1
```

