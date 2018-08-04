---
layout: post
title: "Mac os x - nslookup, host, dig works with /etc/resolv.conf, but ping, ssh doesnt work"
date: 2016-05-17 02:24:09 +0900
comments: true
categories:
- mac
- dns
- nslookup
- dig
- host
- ssh
- ping
- resolver
- vpn
- junos pulse
---

{% img /images/2016-05-17_03-24-13.jpg 650 167 %}

Mac os x 는 linux와 다른 resolving을 제공한다.

간단하게만 이야기 하면 /etc/resolv.conf 외에도 /etc/resolver/* 에 원하는 dns명을 기록해 놓으면 해당 domain name을 갖을시 해당하는 nameserver로 쿼리를 할 수가 있다.

예를 들어 아래와 같이 설정했다면 *.local 과 같은 도메인은 10.10.1.65로 dns 서버를 사용가능하다.

```
$ cat /etc/resolver/local
nameserver 10.10.1.65
```

다만 이건 nslookup, host, dig 와 같은 dns lookup command 들에서는 나타나지 않는다. 이런 커맨드는 linux와 같이 /etc/resolv.conf 의 dns 서버를 이용해서 lookup을 하게 된다. (/etc/hosts 도..)

아무튼 이 설명은 아래 커맨드 들의 man page를 보면 알수 있다.

```
$ man dig
...
Mac OS X NOTICE
       The dig command does not use the host name and address resolution or the DNS query routing mechanisms used by other processes running on Mac OS X.  The results of name or address queries printed by dig may
       differ from those found by other processes that use the Mac OS X native name and address resolution mechanisms.  The results of DNS queries may also differ from queries that use the Mac OS X DNS routing
       library.
...
```

```
$ man nslookup
...
Mac OS X NOTICE
       The nslookup command does not use the host name and address resolution or the DNS query routing mechanisms used by other processes running on Mac OS X.  The results of name or address queries printed by
       nslookup may differ from those found by other processes that use the Mac OS X native name and address resolution mechanisms.  The results of DNS queries may also differ from queries that use the Mac OS X DNS
       routing library.
...
```

```
$ man host
...
Mac OS X NOTICE
       The host command does not use the host name and address resolution or the DNS query routing mechanisms used by other processes running on Mac OS X.  The results of name or address queries printed by host may
       differ from those found by other processes that use the Mac OS X native name and address resolution mechanisms.  The results of DNS queries may also differ from queries that use the Mac OS X DNS routing
       library.
...
```

하지만 ssh, ping은 /etc/resolver/* 외에도 설정되어 있는 resolver dns 로 쿼리를 하게 된다. 그렇기 때문에 한쪽에서는(nslookup, dig, host) 되고 한쪽(ping, ssh 외 다수)에서는 안되는 케이스가 발생한다.

그렇다면 이것을 어떻게 확인 할 수 있을까?    
가장 간단한 방법은 아래와 같다.

```
$ scutil --dns
```

결과는 아래와 같이 resolver 들과 그들의 순위(order)를 확인 할 수 있다.

```
$ scutil --dns
...

resolver #2
  domain   : 1015140824.members.btmm.icloud.com
  options  : pdns
  timeout  : 5
  flags    : Request A records
Not Reachable
  order    : 150000

...

resolver #4
  domain   : local
  options  : mdns
  timeout  : 5
  flags    : Request A records
Not Reachable
  order    : 300000

resolver #5
  domain   : 254.169.in-addr.arpa
  options  : mdns
  timeout  : 5
  flags    : Request A records
Not Reachable
  order    : 300200

...
```

사족으로 여기에서도 만약에 local 순위가 300000(default) 가 마음에 안들면 아래와 같이 resolver 에 추가 하면 100000으로 변경이 가능하다.

```
$ cat /etc/resolver/local
nameserver 10.10.1.65
search_order 100000
```

이런 설정이 일반적으로는 필요하지 않지만 아래와 같은 vpn을 사용하는 케이스에서 발생했었다.

우선 아래와 같은 커맨드로 /etc/resolv.conf 에 원하는 search domain들을 network interface 당 추가가 가능하다. (아니면 환경 설정에서 추가..)

```
sudo networksetup -setsearchdomains 'WI-FI' local
```

이러면 대략 아래와 같이 local 을 search 구문으로 사용가능하다.

```
$ cat /etc/resolv.conf
...
search local
nameserver 8.8.8.8
```

일반적으로 이렇게 사용하는게 문제가 되지 않지만 만약 junos pulse 같은 vpn을 사용시 vpn에서 주어주는 search domain이 아닐시 자신의 기존 search domain 들은 원래 nameserver로 가도록 설정되어 있다.

예를 들면 이렇다.

A, B 라는 search domain을 (가)라는 dns를 사용하는데
만약 vpn에서 B, C 라는 search domain을 (나)라는 dns로 주어지면
아래와 같이 정리된다.

A, B, C search domain을 사용하는 것은 default 로 (나) 로 사용되나.    
이것보다 높은 order 로 A 가 (가)로 한개 더 설정된다.

이런 케이스에는 A domain은 (나)로 질의 하고 싶어도 (가)로 질의 하게 되는 문제가 있다.

vpn 측에서 모든 search domain을 내려주면 해결할 수 있지만 이런게 힘들 시에는 [여기](https://gist.github.com/b4ldr/f9d6aab4837ae18d908f) [여기](http://diaryproducts.net/about/operating_systems/mac_os_x/overriding_dhcp_or_vpn_assigned_dns_servers_in_mac_os_x_leopard) 와 같이 직접 scutil 을 통해서 VPN의 DNS 값들을 수정해 줄 수 있다. (아니면 위와 같이 /etc/resolver/local 과 같이 파일을 만들어도 됨)

추가적으로 아래와 같은 커맨드로 debugging 을 가능하다.

```
# enable operation logging.
sudo killall -USR1 mDNSResponder

# enable packet logging.
sudo killall -USR2 mDNSResponder

# clear the DNS cache.
sudo killall -HUP mDNSResponder

# dump mDNSRepsonder's internal state.
sudo killall -INFO mDNSResponder
```

졸려서 두서없이 정리함..
나중에 다시 좀더 자세하게 정리를..

