---
layout: post
title: "추가 - edit vagrant box for vagrant-libvirt"
date: 2016-09-29 23:39:00 +0900
comments: true
categories: 
- vagrant
- libvirt
---

# copy image and files
``` bash
mkdir -p images
cp ~/.vagrant.d/boxes/${TARGETIMAGE}/0/libvirt/* images/
cd images
```

# init
``` bash
apt-get install -qqy lvm2
modprobe nbd
```

# attach
``` bash
qemu-nbd -c /dev/nbd0 `pwd`/box.img
vgscan
vgchange -ay
mount /dev/mapper/vagrant--vg-root /mnt
```

# bind
``` bash
binds="/dev /dev/pts /proc /sys /run"
for d in $binds; do
  mkdir -p /mnt${d}
  mount --bind ${d} /mnt${d}
done
```

# work
``` bash
chroot /mnt bash
# blah blah blah
exit
```

# unbind and detach
``` bash
cd /
for d in $(mount | awk '/\/mnt/{print $3}' | sort -r); do
  umount $d
done
vgchange -an vagrant-vg
qemu-nbd -d /dev/nbd0
```