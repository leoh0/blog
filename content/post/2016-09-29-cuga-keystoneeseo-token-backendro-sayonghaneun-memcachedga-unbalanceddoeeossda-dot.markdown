---
layout: post
title: "추가 - Keystone에서 Token Backend로 사용하는 Memcached가 Unbalanced되었다.."
date: 2016-09-29 23:30:20 +0900
comments: true
categories: 
- keystone
- memcached
- openstack
- skew
- token
- unbalance
---

예전에 아래글을 그냥 balancing 을 위해 id 를 잘 분배하자 라는식으로 결론 냈지만.

[Keystone에서 Token Backend로 사용하는 Memcached가 Unbalanced되었다..](http://leoh0.github.io/blog/2016/04/27/keystoneeseo-token-backendro-sayonghaneun-memcachedga-unbalanceddoeeossdamyeon-dot/)

user의 token리스트를 관리하지 않으면 부담은 훨씬 줄어든다.

물론 user의 password 변경 등으로 기존 모든 token을 revoke 시키는 경우때문에 이런 user 단위로 token리스트를 관리해야 하지만    
그걸 포기하면 그냥 코드 몇줄 추가로 간단하게 정리할 수 있다.

``` diff
diff --git a/keystone/token/persistence/backends/memcache.py b/keystone/token/persistence/backends/memcache.py
index e6b0fca..3f0de68 100644
--- a/keystone/token/persistence/backends/memcache.py
+++ b/keystone/token/persistence/backends/memcache.py
@@ -37,3 +37,6 @@ class Token(kvs.Token):
         kwargs['memcached_expire_time'] = CONF.token.expiration
         kwargs['url'] = CONF.memcache.servers
         super(Token, self).__init__(*args, **kwargs)
+
+    def _update_user_token_list(self, user_key, token_id, expires_isotime_str):
+        return []
```
