---
layout: post
title: "k8s tools for watching log and login to multiple containers"
date: 2017-07-27 00:58:28 +0900
comments: true
categories: 
- kubernetes
- k8s
- tools
- container
---

{% img /images/showterm-2017-07-27-01-10-29.png 780 290 %}

k8s 사용하면서 개인 취향에 맞게 작성한 자작 스크립트 몇가지 소개해 드리려고 합니다.

# watching log

k8s 에서 pod들의 로그 볼일 들이 많다보니 command 일일치다보니 불편했는데 `kubetail`이란 프로젝트가 있었습니다.    
덕분에 잘 쓰고 있었는데 인터페이스가 개인취향에 안맞아서 약간 수정해서 쓰고 있습니다.    
전반적으로 pod 선택 방법의 변경과 비슷한 이름으로 auto tailing 기능을 추가 했습니다.    

아래들이 대표적인 기능 입니다.

* 선택된것과 같은 pod들을 전체 log tailing 하고 변화가 있을시 auto reload 함    
  (다만 예외적으로 안되는 케이스 들이 아직 있긴 합니다..)

  ```bash
  $ kt
  ```
  * 자세한 영상은 [여기](http://showterm.io/df8a9f96e761012d3bb2c)를 참고하시면 됩니다.    
  {% img /images/kt.gif 800 %}

* `-m` 옵션시 자신이 원하는 pod들을 선택해서 log tailing 함. 다만, auto reload는 지원 하지 않음

  ```bash
  $ kt -m
  ```
  * 자세한 영상은 [여기](http://showterm.io/f4ab6a8ed080700ece976)를 참고하시면 됩니다.    
  {% img /images/ktm.gif 800 %}

* `-l` 옵션시 선택한 pod의 전체 로그를 본다. fzf를 이용해서 log를 탐색 한다.

  ```bash
  $ kt -l
  ```
  * 자세한 영상은 [여기](http://showterm.io/6381c317d2e42920c0227)를 참고하시면 됩니다.    
  {% img /images/ktl.gif 800 %}

준비물은 아래와 같습니다.

* **fzf**

  ```bash
  $ brew install fzf
  ```
* **kubectl**
  * https://kubernetes.io/docs/tasks/tools/install-kubectl/


설치 방법은 아래와 같습니다.

```bash
$ brew tap leoh0/kt && brew install kt
```

code는 아래에서 확인 가능합니다.

https://github.com/leoh0/kt

# login container

pod들 여러개에 동시에 login(bash, sh등) 하여 shell command를 사용하고 싶어서
기존의 cssh 같은 비슷한 메커니즘으로 스크립트 작성해서 사용하고 있습니다.

* 아래는 사용 영상입니다. 자세한 영상은 [여기](http://showterm.io/c58f9999d3ee6db03aa81)를 참고하시면 됩니다.
{% img /images/kl.gif 800 %}

설치 방법은 아래와 같이 rc나 profile에 등록해서 사용하시면 됩니다.
```bash
curl -s 'https://gist.githubusercontent.com/leoh0/'\
'c47dca1c98f998f0d0884c3560afac54/raw/'\
'1e1e9fb085d2c1a94293ae87e3922519d8342adb/k8s_login.sh' | \
    tee -a ~/.bash_profile && source ~/.bash_profile
```

아래는 전체 소스입니다.
https://gist.github.com/leoh0/c47dca1c98f998f0d0884c3560afac54
``` bash
function kl() {
  chkcommand() {
    command -v $1 >/dev/null 2>&1 || { echo >&2 "Plz install $1 first. Aborting."; return 1; }
  }
  chkcommand fzf || return 1
  chkcommand tmux || return 1
  chkcommand kubectl || return 1

  pods=$(kubectl get pods --all-namespaces | sed '1d' | fzf -x -m -e +s --reverse --bind=left:page-up,right:page-down --no-mouse | awk '{print $1","$2}');
  if [[ $pods != "" ]]; then
      init="true"
      tmuxname=k8s-ns-$(date +%s)
      tmux kill-session -t $tmuxname > /dev/null 2> /dev/null || true
      while read line ;do
        NS=$(echo $line | cut -d',' -f1)
        NAME=$(echo $line | cut -d',' -f2)
        connect="kubectl exec -ti $NAME -n $NS -- bash || kubectl exec -ti $NAME -n $NS -- sh "
        tmux new-session -d -s $tmuxname "export KUBECONFIG=${KUBECONFIG}; $connect"
        if [ "${init}" == "true" ]; then
          if (( $(tmux ls 2> /dev/null | grep "${tmuxname}" | wc -l) > 0 )) ; then
            init="false"
          fi
        else
          tmux split-window -t "${tmuxname}" "${connect}" && \
          tmux select-layout -t "${tmuxname}" "tiled"
        fi
      done <<< "$pods"
      tmux set-window-option -t "${tmuxname}" synchronize-panes on
      tmux -2 a -t $tmuxname
  fi
}
```

참고: [original kubetail](https://github.com/johanhaleby/kubetail)
