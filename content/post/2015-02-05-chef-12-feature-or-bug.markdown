---
layout: post
title: "chef 12: node attribute style problem - feature or bug?"
date: 2015-02-05 17:21:56 +0900
comments: true
categories: 
- chef 12
- attribute
- symbol
- string
- hash
---

chef 12 client를 이용하면서 기존 recipe 들을 사용중 갑자기 작동 안하는 문제가 있었다.   
chef 12로 변화되며 추가된 기능인가 싶어 [chef 12 release note](https://docs.chef.io/release_notes.html)를 뒤져보았지만 사실 관련된 부분을 찾지 못했다.

대략 현상은 아래와 같다..

``` ruby chef12
chef > attributes_mode
chef:attributes > default['haproxy']['conf_dir'] = 'aa'
chef:attributes > puts node['haproxy']['conf_dir']
aa
chef:attributes > puts node[:haproxy][:conf_dir]
aa

chef:attributes > node.set['haproxy']['conf_dir'] = 'bb'

chef:attributes > puts node['haproxy']['conf_dir']
bb
chef:attributes > puts node[:haproxy][:conf_dir]
aa
```

위와 같이 `string`으로 된 hash와 `symbol`로 된 hash의 값이 merge가 안되는 문제이다.

chef 11까지는 아래와 같이 둘중 어떤 것을 사용해도 같은 변수로 관리됐다.


``` ruby chef11
chef > attributes_mode
chef:attributes > default['haproxy']['conf_dir'] = 'aa'
chef:attributes > puts node['haproxy']['conf_dir']
aa
chef:attributes > puts node[:haproxy][:conf_dir]
aa

chef:attributes > node.set['haproxy']['conf_dir'] = 'bb'

chef:attributes > puts node['haproxy']['conf_dir']
bb
chef:attributes > puts node[:haproxy][:conf_dir]
bb
```

옛날 부터 [foodcritic에서도 둘중에 한 포맷으로 정해서 사용하라는 룰](http://www.foodcritic.io/#FC019)이 있었지만 정리를 안했더니 이런 일이 생긴건가 라고 반성을 하면서..
(사실 현재도 안돌리고 있긴한.. 안돌리다 돌리려니 너무 고칠게 많아서 차라리 리팩토링 하고 돌려야지 하다보니..... ㅠㅠ)

그렇다면 어느쪽으로 style을 모는게 좋을까...??

## string vs symbol (for node attribute style)

> 개인적으로나 chef에서는 string을 선호

물론 사실 개인적인 생각엔 아직까지도 어느정도 취향(?)의 영역이 있다. 
foodcritic(chef lint tool)에서는 [string을 사용을 규정](http://www.foodcritic.io/#FC001)하고는 있긴하다.
하지만 아래와 같이 빼고 사용하면 그만인 부분이다.

``` bash rule FC001
foodcritic -t ~FC001 .
```

그렇다면 사용자들이 선호하는 것은 어떨까?   
아래와 같이 3종류의 attribute 사용법에 대한 [user survey](https://www.evernote.com/shard/s5/sh/1fc5a0c9-bdd0-44f4-8f5a-ed2ddc9d2cfd/a13f36acd7cfa2a468f7829e5549209f)에 대한 조사는 아래와 같이 symbol이 우세하다.
{% img /images/fc001-survey-20121029-144804.jpg 1524 1164 fc001-survey-20121029-144804.png %}

그리고 실제 [ruby style guide](https://github.com/bbatsov/ruby-style-guide)에서도 물론 특정한 규정은 없어서..   
그렇다고 괜히 `string`을 선호하는 건 아니다. 자세한 이유는 [여기1](https://github.com/acrmp/foodcritic/issues/1) [여기2](https://github.com/acrmp/foodcritic/issues/86) 를 읽어 보면 도움이 될듯 하다. 

## 그래서 내 recipe를 지금이라도 당장 다 뜯어 고쳐야 하는건가요?

> 아니요

위의 style 문제는 사실 feature가 아닌 bug 이다. ([이때 추가된..](https://github.com/chef/chef/commit/097d5eb1bf4b3cbcc9bfc937c5e3441dee5c9f5c))

[deep_merge_cache fixes for bugs in 12.0.0](https://github.com/chef/chef/pull/2753)

22일 전에 이미 패치는 된 상태이나 chef 11에 대해 followup하느라 12.1.0이 나오질 않고 있는것 같다.   
그렇기 때문에 chef-client를 12.0.0 이상으로 쓸 생각이라면 잠깐 보류하는게 좋을듯 하다.   


하지만 이번 기회에 이런 문제를 야기 할 수 있는 `symbol`을 `string`으로 바꿔야 하지 않을까하는 교훈(?)을 주는 일이지 않을까 싶다.   
