---
layout: post
title: "ceph pg incomplete: rbd image-format 2 data recovery"
date: 2015-01-16 00:54:53 +0900
comments: true
categories: 
- openstack
- block storage
- ceph
- pg
- incomplete
---

## ceph: pg incomplete is worst nightmare

{% img http://a1.res.cloudinary.com/hqq9ey1mh/image/upload/c_limit,w_793/v1414983220/z3vn1zndif6v7q2u08w1.png 500 500 "2014 open user survey block storage" %}

[2014년 유저 설문조사](http://superuser.openstack.org/articles/openstack-user-survey-insights-november-2014)에서 찾을 수 있듯이 **ceph**은 openstack의 block storage의 de facto standard 라고 말할 수 있다.

ceph을 사용한지 조금 되었지만 큰 문제가 한번도 없어서 일명 **믿음의 ceph**이라고 칭송하며 내부구조도 살필일 없이 블랙박스로 두고 잘 쓰고 있었었다.

하지만 근래에 급작스런 몇가지 문제로 ceph의 `placement group(pg)` 들이 `incomplete` 상태로 떨어졌고..   

말그대로 절망했다.

왜냐하면 `incomplete` 상태로 떨어진 pg들이 절대 다른 상태로 돌아올수 없으며 해당 pg에 접근하는 모든 request는 slow request화 되며 응답을 주지 않는다. 그렇기 때문에 해당 pg가 있는 pool은 모든 request가 hang에 걸릴 수 있게 되어 pool 전체가 사용불능 상태로 변하게 된다.


## incomplete?

조금 더 `incomplete`가 어떤 상태인지를 잠깐 이해해야 할것 같다. 우선 `incomplete`의 정의를 살펴보면 ceph documents의 [pg-states](http://ceph.com/docs/master/rados/operations/pg-states/)에서 찾아볼 수 있다.

> Ceph detects that a placement group is missing a necessary period of history from its log. 
> If you see this state, report a bug, and try to start any failed OSDs that may contain the needed information.

즉, 해당 pg 가 log에서 특정 부분 히스토리를 잃어 버렸을때 발생하는 것이고, 만약 이 일이 일어나면 해당 정보를 가지고 있을만한 실패한 OSD들을 시작하라고 설명이 되어 있다.(버그 리포트는 덤..)     
그래서 위의 설명처럼 `incomplete` 상태는 OSD를 중지했을때도 잠깐 발생할 수 있는데 여기서 말하는 `incomplete`는 그런게 아니라 무슨짓을 해도 다시 `incomplete`로 돌아오는 상태를 말한다..

이 의미를 조금 더 의미있게 해석하면 pg가 `incomplete` 상태로 떨어졌을때 `peering`을 할 수 있는 상태로 돌아오지 못한다는 것은 이미 복구 불가능한 pg가 생겼고 이는 복구 불가능한 조각이 생겼다는 것을 의미한다.   
그러므로 전체 파일중 특정 조각이 문제가 되고 이 파일을 접근하는 모든 client request는 hang이 걸리게 된다.   
볼륨같은 큰 데이터(많은 조각을 갖는 데이터)는 몇 개의 pg만 `incomplete`로 떨어져도 결국 모든 client의 request가 hang이 걸리게 된다.

{% img /images/ceph.png 1688 264 "ceph logical flow" %}

그러므로 `incomplete` 된 pg가 있으면 pool 전체를 사용할 수 가 없다.(pool을 초기화 하기 전까지..)   
왜냐하면 어떠한 rbd object는 rados object로 분할되고 rados object들은 해당 pool에 분할 되어져서 들어간다.   
해당 pool은 pg 들로 이루어 지는데 그중 한 pg 조각만 문제가 있어도 그 pg 조각에 들어간 한 rados object에 접근이 안되고 그렇기 때문에 rbd object 자체를 쓸수가 없게 되기 때문에 해당 pool을 쓸 수 없게 된다.   
온갖 메일링 리스트와 구글에서 검색한 방법을 사용했지만 효과는 없었고   
다음에 이 상태로 빠지지 않을 수 있는 교훈만 얻을 수 있었다.   

이 글을 보면 이게 얼마나 간단하지 않은 일인지 알게 된다..   
[Ceph PG Incomplete = Cluster unusable](https://www.mail-archive.com/ceph-users@lists.ceph.com/msg15916.html)

현재로서는 여기 [feature 요청](http://tracker.ceph.com/issues/10098) 같이 데이터를 포기하더라도 pg를 재생성할 수 있는 기능이 추가되기를 기대해야 하는 상황이다.   
(현재는 `ceph pg force_create_pg`를 사용해도 `creating` 상태에서 `incomplete` 상태로 되돌아 온다..)   

물론 이상태까지 오게 하지 않는게 최선의 방법이지만 이미 이런 복구 불가능한 데이터가 생기면   
그리고 그 데이터가 볼륨인데 pg 한조각이 유실되면 해당 데이터를 사용할 수 없게 된다.(read write중 hang)   
그렇기 때문에 결국 이런 볼륨을 살려내기 위해서는 강제로 볼륨 데이터를 `export`하고 이를 바탕으로 새로운 풀을 제작해서 다시 `import`할 수 밖에 없다.  
물론 볼륨 데이터를 `export` 하면 hang이 걸려서 이 내용과 같이 특수한 방법이 필요하다.

그래서 오늘의 주제는 `incomplete` 와 같이 pool 전체를 사용할 수 없는 상태에서 최대한 데이터를 복구해 내는 방법을 소개하고자 한다.

## image format

우선 이 방법을 이해하기 위해서는 기본적으로 `rbd image format`을 이해해야한다.   
우선 [Ceph 아키텍처 해설](http://blog.bit-isle.jp/bird/2014/06/10)에서 기본적인 지식을 습득할 수 있다.   
우선 `rbd image format`의 종류는 아래와 같이 2가지 종류이고 위의 글에서 2가지 포맷의 제한점을 알 수 있다.

### rbd image format 1

``` c ceph/src/include/rbd_types.h https://github.com/ceph/ceph/blob/master/src/include/rbd_types.h#L36-46 link
/*
 * old-style rbd image 'foo' consists of objects
 *   foo.rbd      - image metadata
 *   rb.<idhi>.<idlo>.00000000
 *   rb.<idhi>.<idlo>.00000001
 *   ...          - data
 */

#define RBD_SUFFIX	 	".rbd"
#define RBD_DIRECTORY           "rbd_directory"
#define RBD_INFO                "rbd_info"
```

### rbd image format 2

``` c ceph/src/include/rbd_types.h https://github.com/ceph/ceph/blob/master/src/include/rbd_types.h#L24-34 link
/* New-style rbd image 'foo' consists of objects
 *   rbd_id.foo              - id of image
 *   rbd_header.<id>         - image metadata
 *   rbd_data.<id>.00000000
 *   rbd_data.<id>.00000001
 *   ...                     - data
 */

#define RBD_HEADER_PREFIX      "rbd_header."
#define RBD_DATA_PREFIX        "rbd_data."
#define RBD_ID_PREFIX          "rbd_id."
```

우선 이글은 현재 사용중인 `rbd image format 2`의 포맷의 복구 방법에 대해서 설명할 계획이다.

## image format 2 분석

만약 당신이 glance나 cinder를 사용중이라면 uuid 형태의 이름을 갖는 image와 volume들이 있을 것이다.   
이 정보는 ceph rbd 안에서는 아래와 같이 조회 가능하다.

``` bash
# rbd -p images ls
08909734-66fa-48e3-ab5e-2e2b8bb3a58c
```

위와 같이 image나 volume이 사용하고 있는 pool의 list를 조회해 보면 uuid 들이 나오는데 이 리스트 데이터들이 실제 image와 volume의 uuid 이다.
그렇다면 이런 image와 volume을 조회하면 다음과 같다.

``` bash
# rbd -p images info 08909734-66fa-48e3-ab5e-2e2b8bb3a58c
rbd image '08909734-66fa-48e3-ab5e-2e2b8bb3a58c':
    size 810 MB in 102 objects
    order 23 (8192 kB objects)
    block_name_prefix: rbd_data.5a57484353d0cd
    format: 2
    features: layering
```

이 내용은 실제 rbd에 저장된 uuid의 데이터가 어떤지 정보를 보여주는데   
여기에서 `08909734-66fa-48e3-ab5e-2e2b8bb3a58c` 라는 uuid의 이미지는 `810 MB`의 크기를 가지며 `102`개의 조각이다.   
그리고 `2**23=8192kB`의 크기로 쪼개져 있다. 즉, `810 MB`는 `약 8192kB * 102` 이다.   
다음으로 `block_name_prefix: rbd_data.5a57484353d0cd`에서 `5a57484353d0cd`가 prefix 값이다.   
format: 2 는 말그대로 image format 2 이고 features은 default값이다.

rbd는 rados object들로 이루어져 있으며 위의 정보를 바탕으로 아래와 같이 3종류의 object로 구성된다.

* **rbd_id.08909734-66fa-48e3-ab5e-2e2b8bb3a58c**
* **rbd_header.5a57484353d0cd**
* **rbd_data.5a57484353d0cd.0000000000000000 ~ rbd_data.5a57484353d0cd.0000000000000021 (102개)**

### rbd_id.08909734-66fa-48e3-ab5e-2e2b8bb3a58c

rbd_id.<uuid\> 파일을 다운받아서 열어보면 그 안에 `prefix` 값이 저장되어 있는 것을 알 수 있다.   
저 값은 random 값이 들어 있어서 이 방법이 아니면 찾을 수가 없는 값이다.

``` bash
# cat rbd_id.08909734-66fa-48e3-ab5e-2e2b8bb3a58c
5a57484353d0cd
```

### rbd_header.5a57484353d0cd

rbd_header.<prefix\> 파일은 아무것도 내용이 없다. 실제 파일 내용이며 xattr 정보도 쓸만 한게 없다.   
하지만 이정보는 파일로 관리하기 위해 존재하는 가상의 정보이지 실제는 leveldb에 저장되어 있다.(구버전은 테스트를 못해서 구버전엔 xattr을 썻을 지도..)
아무튼 이 안에서 features, object_prefix, order, size, snap_seq 를 뽑아 낼 수 있다.

``` bash
# rados listomapvals -p images rbd_header.5a57484353d0cd
features
value: (8 bytes) :
0000 : 01 00 00 00 00 00 00 00                         : ........
object_prefix
value: (27 bytes) :
0000 : 17 00 00 00 72 62 64 5f 64 61 74 61 2e 35 61 35 : ....rbd_data.5a5
0010 : 37 34 38 34 33 35 33 64 30 63 64                : 7484353d0cd
order
value: (1 bytes) :
0000 : 17                                              : .
size
value: (8 bytes) :
0000 : 00 8c a3 32 00 00 00 00                         : ...2....
snap_seq
value: (8 bytes) :
0000 : 00 00 00 00 00 00 00 00                         : ........
```

### rbd_data.5a57484353d0cd.0000000000000000 ~ rbd_data.5a57484353d0cd.0000000000000021 (102개)

실제 데이터이며 이 정보를 이어 붙이면 우리가 원하는 데이터를 얻을 수 있다.   
하지만 이 정보 들은 존재하지 않는 파일들도 있을 수도 있다.(안쓰는 블럭부분의 오브젝트는 생성안되어 있음)
아무튼 뒤의 16진수 16자리 개수 만큼 데이터를 생성할 수 있다.
우리는 102개의 오브젝트 인걸 알고 있으니 0000000000000000 ~ 0000000000000021 까지해서 102개 이다.

#### Q. 왜 `rados -p image ls` 같은 커맨드로 오브젝트의 정확한 리스트를 검사하지 않는가?

만약 incomplete 상태가 되면 `rados ls` 같은 커맨드는 incomplete 인 pg 에 request 를 던지고 그 request는 hang이 되기때문에 응답을 받기가 힘들것이다..   
그렇기 때문에 이런 작업으로 실제 존재할 데이터를 추정해야 한다.

## 작업 순서

image나 volume 의 `uuid`이름과 `pool`의 이름을 알고 있으면 시작할 수 있다.

### 1. pool 이름 -> pool 번호, pg 개수

아래와 같은 정보에서 보면 `pool 35`와 `pg_num 512` 정보로 pool 번호가 `35`인것 그리고 pg 개수가 `512` 인것을 알 수 있다.
추가로 `object_hash rjenkins`로 `rjenkins` 해쉬를 사용함을 알 수 있다.

``` bash
# ceph osd dump
...
pool 35 'images' replicated size 2 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 512 pgp_num 512 last_change 46681 flags hashpspool stripe_width 0
...
```

### 2. rbd -p <pool\> info <uuid\> -> prefix, 파일 개수, 사이즈

앞에서 한번 설명했지만 다시 정리하면 아래와 같은 상황에서 prefix는 `5a57484353d0cd` 파일 개수는 `102` 파일 사이즈는 `8192 kB` 이다.

``` bash
# rbd -p images info 08909734-66fa-48e3-ab5e-2e2b8bb3a58c
rbd image '08909734-66fa-48e3-ab5e-2e2b8bb3a58c':
    size 810 MB in 102 objects
    order 23 (8192 kB objects)
    block_name_prefix: rbd_data.5a57484353d0cd
    format: 2
    features: layering
```
물론 incomplete pg에 해당 header가 들어 있다면.. 그 파일은 포기해야 한다.

### 3. rbd_data.<prefix\>.<seq\> 파일 찾기

실제 데이터는 rbd_data.<prefix\>.<seq\> 파일을 연결하면 복구 할 수 있다.
그렇다면 어떻게 저 파일들을 찾을 수 있을까..

우선 아래 커맨드로 찾을 수 있다. 저 아래정보에서 `->` 뒤편의 정보를 참고 하면 된다.   
pg 35.9f6e4c67 는 35번 풀의 9f6e4c67 해쉬 값을 갖는 데이터를 말하며   
9f6e4c67 값이 rbd_data.b0e882ae8944a.0000000000000134 를 해쉬 한 값이다.   
그렇다면 67은 무엇이냐면 9f6e4c67 를 pg_num으로 `modular` 연산 해서 나온 값이다.   
우리는 실제 디렉토리는 35.67을 사용하니까 저 값을 알 고 있으면 된다.   
또한 뒤의 up 3,11 에서 3번 osd 11번 osd가 가지고 있는 것을 알 수 있다.   

``` bash
# ceph osd map images rbd_data.b0e882ae8944a.0000000000000134
osdmap e53074 pool 'images' (35) object 'rbd_data.b0e882ae8944a.0000000000000134' -> pg 35.9f6e4c67 (35.67) -> up ([3,11], p3) acting ([3,11], p3)
```

다만 `ceph osd map ..` 커맨드는 `ceph pg dump` 와 같은 pg를 뒤져야 하는 부분이 있기 때문에 느릴 수 있다.   
이때문에 이 기능은 아래와 같이 pg dump에 대한 내용을 파일로 저장하고 아래 같이 해쉬 스크립트만 따로 돌리는게 훨씬 빠를 것이다.

아래 python코드는 ceph에서 사용하는 [robert jenkins hash](http://burtleburtle.net/bob/hash/evahash.html) 를 [포팅](http://stackoverflow.com/a/3611698)한 스크립트 이다.   
물론 rjenkins hash 아니면 linux hash이나 기본이 rjenkins hash 이다.   

``` python rjhash.py https://gist.github.com/leoh0/aac0bb046c49a108c541 link
'''Implements a straight Jenkins lookup hash - http://burtleburtle.net/bob/hash/doobs.html

Usage: 
    from jhash import jhash
    print jhash('My hovercraft is full of eels')

Returns: unsigned 32 bit integer value

Prereqs: None

Tested with Python 2.6.
Version 1.00 - der@dod.no - 23.08.2010

Partly based on the Perl module Digest::JHash
http://search.cpan.org/~shlomif/Digest-JHash-0.06/lib/Digest/JHash.pm

Original copyright notice:
    By Bob Jenkins, 1996.  bob_jenkins@burtleburtle.net.  You may use this
    code any way you wish, private, educational, or commercial.  It's free.

    See http://burtleburtle.net/bob/hash/evahash.html
    Use for hash table lookup, or anything where one collision in 2^^32 is
    acceptable.  Do NOT use for cryptographic purposes.
'''

def mix(a, b, c):
    '''mix() -- mix 3 32-bit values reversibly.
For every delta with one or two bits set, and the deltas of all three
  high bits or all three low bits, whether the original value of a,b,c
  is almost all zero or is uniformly distributed,
* If mix() is run forward or backward, at least 32 bits in a,b,c
  have at least 1/4 probability of changing.
* If mix() is run forward, every bit of c will change between 1/3 and
  2/3 of the time.  (Well, 22/100 and 78/100 for some 2-bit deltas.)'''
    # Need to constrain U32 to only 32 bits using the & 0xffffffff 
    # since Python has no native notion of integers limited to 32 bit
    # http://docs.python.org/library/stdtypes.html#numeric-types-int-float-long-complex
    a &= 0xffffffff; b &= 0xffffffff; c &= 0xffffffff
    a -= b; a -= c; a ^= (c>>13); a &= 0xffffffff
    b -= c; b -= a; b ^= (a<<8); b &= 0xffffffff
    c -= a; c -= b; c ^= (b>>13); c &= 0xffffffff
    a -= b; a -= c; a ^= (c>>12); a &= 0xffffffff
    b -= c; b -= a; b ^= (a<<16); b &= 0xffffffff
    c -= a; c -= b; c ^= (b>>5); c &= 0xffffffff
    a -= b; a -= c; a ^= (c>>3); a &= 0xffffffff
    b -= c; b -= a; b ^= (a<<10); b &= 0xffffffff
    c -= a; c -= b; c ^= (b>>15); c &= 0xffffffff
    return a, b, c

def jhash(data, initval = 0):
    '''hash() -- hash a variable-length key into a 32-bit value
  data    : the key (the unaligned variable-length array of bytes)
  initval : can be any 4-byte value, defaults to 0
Returns a 32-bit value.  Every bit of the key affects every bit of
the return value.  Every 1-bit and 2-bit delta achieves avalanche.'''
    length = lenpos = len(data)

    # empty string returns 0
    if length == 0:
        return 0

    # Set up the internal state
    a = b = 0x9e3779b9 # the golden ratio; an arbitrary value
    c = initval        # the previous hash value
    p = 0              # string offset

    # ------------------------- handle most of the key in 12 byte chunks
    while lenpos >= 12:
        a += (ord(data[p+0]) + (ord(data[p+1])<<8) + (ord(data[p+2])<<16) + (ord(data[p+3])<<24))
        b += (ord(data[p+4]) + (ord(data[p+5])<<8) + (ord(data[p+6])<<16) + (ord(data[p+7])<<24))
        c += (ord(data[p+8]) + (ord(data[p+9])<<8) + (ord(data[p+10])<<16) + (ord(data[p+11])<<24))
        a, b, c = mix(a, b, c)
        p += 12
        lenpos -= 12

    # ------------------------- handle the last 11 bytes
    c += length
    if lenpos >= 11: c += ord(data[p+10])<<24
    if lenpos >= 10: c += ord(data[p+9])<<16
    if lenpos >= 9:  c += ord(data[p+8])<<8
    # the first byte of c is reserved for the length
    if lenpos >= 8:  b += ord(data[p+7])<<24
    if lenpos >= 7:  b += ord(data[p+6])<<16
    if lenpos >= 6:  b += ord(data[p+5])<<8
    if lenpos >= 5:  b += ord(data[p+4])
    if lenpos >= 4:  a += ord(data[p+3])<<24
    if lenpos >= 3:  a += ord(data[p+2])<<16
    if lenpos >= 2:  a += ord(data[p+1])<<8
    if lenpos >= 1:  a += ord(data[p+0])
    a, b, c = mix(a, b, c)

    # ------------------------- report the result
    return c

if __name__ == "__main__":
    hashstr = sys.argv[1]
    myhash = jhash(hashstr)
    myhash2 = myhash % int(sys.argv[2])
    print "%x" % myhash2
```

즉, 위와 같이 스크립트로 해슁하면 `rbd_data.b0e882ae8944a.0000000000000134` 값이 `67` 임을 찾을 수 있을 것이다.
35번 pool 임을 알고 있으니 35.67 pg 인것을 확인 가능하다.
아래 같이 `ceph pg dump` 는 파일로 저장해서 사용하는 것이 훨씬 빠를 것이다.. (물론 클러스터 변화를 주지 않는 상태에서.. )

``` bash
# ceph pg dump > /tmp/pgdump # just once

# cat /tmp/pgdump | grep ^35.67
dumped all in format plain
35.67	0	0	0	0	0	2	2	active+clean	2015-01-15 06:48:51.979426	46681	53074:6358	[3,11]	0	[3,11]	0	46681	2015-01-15 06:48:51.979318	46681	2015-01-14 06:47:33.392701
```

그렇다면 실제 데이터는 어디에 저장될까..
대략 기본적으로 아래와 같은 경로에 저장된다.

ceph-3 의 `3`은 osd 번호이며   
`35.67`_head 는 위에서 계산한 hash 값에 pg_num으로 modular한 값이다.   
`DIR_7` `DIR_6` `DIR_C` 는 뒤의 `9F6E4C67` 의 suffix 역순이다.(이 디렉토리는 파일 개수에 따라 생기고 없어질수도 있다.)   
물론 `9F6E4C67`는 아까의 hash 값이다.   
마지막 __`23` 은 pool 번호 35의 16진수 값이다.

``` bash
/var/lib/ceph/osd/ceph-3/current/35.67_head/DIR_7/DIR_6/DIR_C/rbd\udata.b0e882ae8944a.0000000000000134__head_9F6E4C67__23
```

중간에 DIR 과 같은 디렉토리가 들어가야 하기때문에 결국 이 파일을 찾으려면 find로 찾는게 간편하다.

### 4. 파일 합치기

아래와 같은 스크립트를 참고하면 편하다. 간단하게 파일번호로 offset 계산해서 합치는 스크립트 이다.      
해당 스크립트는 로컬에 있는 파일리스트를 조회해서 합치는 것임으로 위에서 미리 찾아서 한 폴더로 몰아놓으면 편하다.   

``` bash rbd_restore.sh https://raw.githubusercontent.com/smmoore/ceph/master/rbd_restore.sh link
#!/bin/sh
#
# AUTHORS
# Shawn Moore <smmoore@catawba.edu>
# Rodney Rymer <rrr@catawba.edu>
#
#
# REQUIREMENTS
# GNU Awk (gawk)
#
#
# NOTES
# This utility assumes one copy of all object files needed to construct the rbd
# are located in the present working direcory at the time of execution.  
# For example all the rb.0.1032.5e69c215.* files.
#
# When listing the "RBD_SIZE_IN_BYTES", be sure you list the full potential size, 
# not just what it appears to be. If you do not know the true size of the rbd,
# you can input a size in bytes that you know is larger than the disk could be
# and it will be a large sparse file with un-partioned space at the end of the
# disk.  In our tests, this doesn't occupy any more space/objects in the cluster
# but the rbd could be resized from within the rbd (VM) to grow.  Once you bring
# it up and are able to find the true size, you can resize with "rbd resize ..".
#
# To obtain needed utility input information if not already known run:
# rbd info RBD
#
# To find needed files we run the following command on all nodes that might have
# copies of the rbd objects:
# find /${CEPH} -type f -name rb.0.1032.5e69c215.*
# Then copy the files to a single location from all nodes.  If using btrfs be
# sure to pay attention to the btrfs snapshots that ceph takes on it's own.
# You may want the "current" or one of the "snaps".
#
# We are actually taking our own btrfs snapshots cluster osd wide at the same
# time with parallel ssh and then using "btrfs subvolume find-new" command to
# merge them all together for disaster recovery and also outside of ceph rbd
# versioning.
#
# Hopefully once the btrfs send/recv functionality is stable we can switch to it.
#
#
# This utility works for us but may not for you.  Always test with non-critical
# data first.
#

# Rados object size
obj_size=4194304

# DD bs value
rebuild_block_size=512

rbd="${1}"
base="${2}"
rbd_size="${3}"
if [ "${1}" = "-h" -o "${1}" = "--help" -o "${rbd}" = "" -o "${base}" = "" -o "${rbd_size}" = "" ]; then
  echo "USAGE: $(echo ${0} | awk -F/ '{print $NF}') RESTORE_RBD BLOCK_PREFIX RBD_SIZE_IN_BYTES"
  exit 1
fi
base_files=$(ls -1 ${base}.* 2>/dev/null | wc -l | awk '{print $1}')
if [ ${base_files} -lt 1 ]; then
  echo "COULD NOT FIND FILES FOR ${base} IN $(pwd)"
  exit
fi

# Create full size sparse image.  Could use truncate, but wanted
# as few required files and dd what a must.
dd if=/dev/zero of=${rbd} bs=1 count=0 seek=${rbd_size} 2>/dev/null

for file_name in $(ls -1 ${base}.* 2>/dev/null); do
  seek_loc=$(echo ${file_name} | awk -F_ '{print $1}' | awk -v os=${obj_size} -v rs=${rebuild_block_size} -F. '{print os*strtonum("0x" $NF)/rs}')
  dd conv=notrunc if=${file_name} of=${rbd} seek=${seek_loc} bs=${rebuild_block_size} 2>/dev/null
done
```

### 5. mount 해서 테스트 해보기

아래 와 같이 loop device 를 이용해서 mount 해본다.   
하지만 partition 테이블이 망가져서 `/dev/mapper/loop0p1`를 못쓰고 그냥 `/dev/loop0` device로 mount 해야 할 수 도 있다.

``` bash
v="volume-uuid"
losetup /dev/loop0 $v
kpartx -a /dev/loop0
mkdir -p /mnt/$v
mount /dev/mapper/loop0p1 /mnt/$v
```

물론 들어가보면 엄청나게 많은 파일, 폴더들이 깨져 있으므로.. `fsck -y /mnt/$v` 와 같이 파일 시스템 체크를 돌려서 살려내는것이 필요하다.

## 결론

이런 정보를 바탕으로 우리는 3가지의 rados object들로 rbd 이미지를 재구성 할 수 있다.   
하지만 이 방법은 운이 따라줘야 하는 부분들이 당연히 있다.   
예를 들어 아래 pg 들이 망가지면 복구가 불가능 할 수도 있다.

* rbd_header 파일이 있는 pg
* rbd_data에서 초반 super block이 있는 pg

아무튼 이 글을 앞으로도 절망할 지 모르는 사람들을 위해 바친다.

## ps.

마지막으로 firefly는 v0.80.8 package를 2015년 1월 13일에 release 했다.   
업그레이드 하는게 좋을것이다.





