---
layout: post
title: "mac에서 ntfs disk에 write 하기"
date: 2015-11-03 09:38:22 +0900
comments: true
categories: 
- mac
- ntfs
---
GPT 일때 가능

{{< highlight bash "linenos=table" >}}
#!/usr/bin/env bash

if mount | grep -q 'ntfs' ; then
  if ! mount | grep -q 'read-only' ; then
    echo -e "\033[01;36mExist already in /etc/fstab.\033[00m"
    VOLUME=$(mount | grep "(ntfs, " | sed 's|/dev/disk[0-9]s[0-9] on \(.*\) (ntfs,.*$|\1|g')
    open "$VOLUME"
    exit
  fi

  DISK=$(mount | grep 'read-only' | awk '{print $1}')
  UUID=$(diskutil info $DISK | grep UUID | awk '{print $3}')
  echo -e "\033[01;36mRegistry into /etc/fstab.\033[00m"
  echo "UUID=$UUID none ntfs rw,auto,nobrowse" | sudo tee -a /etc/fstab > /dev/null
  diskutil umount $DISK
  diskutil mount $DISK
  VOLUME=$(mount | grep "(ntfs, " | sed 's|/dev/disk[0-9]s[0-9] on \(.*\) (ntfs,.*$|\1|g')
  open "$VOLUME"
else
  echo -e "\033[01;36mThere is no ntfs disk in here.\033[00m"
fi
{{< /highlight >}}
