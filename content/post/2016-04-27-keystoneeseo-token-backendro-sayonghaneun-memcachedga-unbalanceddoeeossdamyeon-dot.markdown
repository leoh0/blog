---
layout: post
title: "keystone에서 token backend로 사용하는 memcached가 unbalanced되었다.."
date: 2016-04-27 00:00:45 +0900
comments: true
categories: 
- openstack
- keystone
- token
- memcached
- unbalance
- skew
---

해당 클러스터는 kilo 버전으로 구성 되었었고 token 을 memcached 에 저장하고 있었다.    
또한 kilo 부터 dogpile.cache 는 거의 고정으로 들어가 있가 있어서 해당 모듈을 사용했다.    
이런 상황을 디버깅 했던 경험을 정리해 본다.
 
아래는 문제가 되었던 memcached host의 in/out bound 그래프이다.    
수치는 가려서 스케일만 감으로 볼 수 있게 남겼다.

**A 서버**
{% img /images/2016-04-26_23-50-30.png 508 122 A 서버 %}

**B 서버**
{% img /images/2016-04-26_23-50-41.png 499 122 B 서버 %}

최초엔 keystone과 memcached connection 이 unbalance 할것이라고 생각했으나 그런 정황은 없었다.    (connection 개수가 일정) 그리고 특별히 keystone 로그에도 별다른 문제가 보이지 않았다.    
memcache key 개수는 심지어 **A 서버**가 많았다.

이 외에도 온갖 삽질을 했지만 관련이 없었다.    
그 후 결국 아래와 같은 방법으로 디버깅 할 수 있었다.

각 호스트에서 아래 같이 memcache slab id 별로 dump를 떠보니 `bf81985d70a6416897edbade7a8bfc0a5a579af4 ` 와 같이 유독 큰 (578966 b) 키가 **B 서버**에만 존재 하고 있었다.

대부분의 token 데이터는 `3f786850e387550fdab836ed7e6dc881de23001b` 정도와 같이 PKI가 아닌 UUID token data여서 10000 b 정도를 구성하고 있었기 때문에 `크기`가 더 눈에 띈다.

```
$ for i in $(echo 'stats items' | nc localhost 11211 | cut -d':' -f2 | sort -u | grep -v END); do
    echo "stats cachedump $i 1" | nc localhost 11211
done
...
ITEM 3f786850e387550fdab836ed7e6dc881de23001b [11651 b; 1461686308 s]
END
ITEM 6e49b86a7502dae881f3b9466ecbdfa4743c7eb9 [578966 b; 1461688011 s]
END
...
```

그렇다면 이 키를 열어 보면 아래와 같다.    
( 참고로 아래 커맨드를 쓰기위해선 `dogpile.cache` 가 인스톨 되어 있어야 한다. )

```
$ echo get 6e49b86a7502dae881f3b9466ecbdfa4743c7eb9 | nc localhost 11211 | python -c '
import sys
import cPickle
import json
try:
    data=cPickle.load(sys.stdin)
    print json.dumps(data)
except (cPickle.UnpicklingError, EOFError):
    print ""
' | python -mjson.tool
[
    [
        [
            "386c0e0a01bb4069904d9c11771516a2",
            "2016-04-26T15:41:25.000000Z"
        ],
        [
            "9ba06233d0894aa4a06d4302800035c1",
            "2016-04-26T15:41:24.000000Z"
        ],
        [
            "1510940842f943b798f4bb9f7964aa67",
            "2016-04-26T15:41:24.000000Z"
        ],
   
        ...
   
        [
            "d57b2946174c4a4391496a7f9af7e0c5",
            "2016-04-26T16:41:18.000000Z"
        ]
    ],
    {
        "ct": 1461685284.976144,
        "v": 1
    }
]
```

위와 같이 되어 있고 알고 보면 특정 토큰들과 그 토큰이 issue 된 시간이 적혀 있는 [리스트](https://github.com/openstack/keystone/blob/stable/kilo/keystone/token/persistence/backends/kvs.py#L155-L188) 이다.    
이 키의 리스트는 유저별로 token의 expire time을 관리하는 값으로 해당 user에게 token이 발급 되거나 expire 될때마다 해당 리스트를 memcache로 부터 가져와서(`get`) 다시 업로드(`set`) 한다.    
그렇기 때문에 해당 키값이 결국 유저와 관계 있다는 것을 추측할 수 있었다.

예를 들어 아래 처럼 특정 유저를 본다면 아래 처럼 `id`를 갖고 있을 것이다.

```
$ openstack user show ceilometer
+----------+----------------------------------+
| Field    | Value                            |
+----------+----------------------------------+
| enabled  | True                             |
| id       | eef939600bc111e69aeb57d4fa849231 |
| name     | ceilometer                       |
| username | ceilometer                       |
+----------+----------------------------------+
```

eef939600bc111e69aeb57d4fa849231 이값은 아래와 같이 prefix가 붙고 hash 되서 key 값으로 사용된다.
 
```   
$ echo -n 'usertokens-eef939600bc111e69aeb57d4fa849231' | sha1sum
6e49b86a7502dae881f3b9466ecbdfa4743c7eb9  -
```

즉, `6e49b86a7502dae881f3b9466ecbdfa4743c7eb9`은 가 key이 기때문에 위의 토큰 리스트는 ceilometer의 토큰 리스트인걸 알 수 있다.    

마지막으로 아래와 같이 계산해 보면 어떤 멤캐쉬에 들어갈 지 알수 있다. ([cmemcache_hash](https://github.com/linsomniac/python-memcached/blob/master/memcache.py#L63-L66) 참고)    
여기에서는 `3065` 가 나왔기 때문에 멤캐쉬 서버가 두대이면 두번째(`3016%2=1`) 서버로 들어가게 된다.
 
```   
$ echo -n '6e49b86a7502dae881f3b9466ecbdfa4743c7eb9' | python -c '
import sys,binascii
print (
    (((binascii.crc32(sys.stdin.read()) & 0xffffffff)
       >> 16) & 0x7fff) or 1)
'
3065
```

나의 케이스는 불운 하게도 이런 많은 토큰을 같은 유저(ceilometer, neutron, nova 등)가 전부 해쉬값이 홀수가 나와서 한 memcached host에 할당되었고, 이때문에 한쪽으로 skew 가 있었다.

이런걸 피할려면 결국 memcached 개수를 늘이던가 토큰이 많은 유저의 uuid를 분배시킬 수 있도록 해야 할것같다.
물론 저런 거대한 토큰 리스트를 관리 안하는 것이 더 나아 보이지만 이 코드는 현재 master(mitaka 이후)까지도 유지되어 있는 상태이다.

우리도 ceilometer 를 쓰기 전까지는 이런 문제가 없었으나 ceilometer를 추가하면서 문제가 발생하기 시작했다.    
아무래도 ceilometer 상위 버전쪽에서는 토큰의 재활용을 높이는 부분들이 있어야 할것이다.

사족으로 토큰이 어떤 내용을 담고 있는지는 아래 같은 스크립트로 찾으면 편하다.

```
MEMCACHES='serverA serverB'
for h in $MEMCACHES; do
  echo $h
  sha=$(echo -n "token-$1" | sha1sum | cut -d' ' -f1)
  echo get $sha | nc $h 11211 | python -c '
import sys
import cPickle
import json
try:
    data=cPickle.load(sys.stdin)
    data[0]["expires"] = data[0]["expires"].strftime("%Y-%m-%d %H:%M:%S")
    print json.dumps(data[0])
except (cPickle.UnpicklingError, EOFError):
    print ''
' | python -mjson.tool
done
```