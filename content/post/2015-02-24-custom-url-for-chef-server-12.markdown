---
layout: post
title: "custom URL for chef server 12"
date: 2015-02-24 23:29:19 +0900
comments: true
categories: 
- chef
- chef server 12
---

chef server ssl verification 때문에 그냥 domain 인증서를 제대로 적용하려고 했는데 엄청 삽질했다. 아마 chef 11 에서 chef 12 로 올라가면서 config 위치가 변경된거 같다. 

[여기](http://www.bitlancer.com/2014/10/custom-chef-server-url/)에서 관련 정보를 얻을 수 있었는데 차이점은 /etc/chef-server/chef-server.rb 가 아닌 /etc/opscode/chef-server.rb 파일을 수정해서 적용 가능했다.

아래 처럼 해당 파일을 수정한다.

``` ruby /etc/opscode/chef-server.rb
...
server_name = "chef.yourdomain.com"
api_fqdn = server_name
bookshelf['vip'] = server_name
nginx['url'] = "https://#{server_name}"
nginx['server_name'] = server_name
nginx['ssl_certificate'] = "/var/opt/chef-server/nginx/ca/#{server_name}.crt"
nginx['ssl_certificate_key'] = "/var/opt/chef-server/nginx/ca/#{server_name}.key"
lb['fqdn'] = server_name
```

이후에 해당 컨피그 내용으로 적용한다.

``` bash
chef-server-ctl reconfigure
```

이후 테스트해보면 끝.
