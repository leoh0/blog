---
layout: post
title: "edit vagrant box for vagrant-libvirt"
date: 2015-02-24 13:22:36 +0900
comments: true
categories: 
- vagrant
- libvirt
---

# check file
``` bash
$ file trusty64_vagrant_box_image.img
trusty64_vagrant_box_image.img: QEMU QCOW Image (unknown version)
```

# init
``` bash
apt-get install -qqy lvm2
modprobe nbd
```

# attach
``` bash
cd /var/lib/libvirt/images
qemu-nbd -c /dev/nbd0 `pwd`/trusty64_vagrant_box_image.img
vgscan
vgchange -ay
mount /dev/mapper/vagrant--vg-root /mnt
```

# detach
``` bash
cd /
umount /mnt
vgchange -an vagrant-vg
qemu-nbd -d /dev/nbd0
```