---
layout: post
title: "Management multiple openstack cluster with automatic openstack rc file loader script"
date: 2016-05-10 01:08:56 +0900
comments: true
categories: 
- openstack
- rc
- openrc
- automatic
- multiple cluster
---

{% img /images/2016-05-10_03-40-27.jpg 368 272 %}

## openstack rc file ##
여러 openstack 클러스터를 관리하려면 [openstack rc file](http://docs.openstack.org/user-guide/common/cli_set_environment_variables_using_openstack_rc.html)(이하 openrc file)을 잘 관리해야한다.    
이런 관리를 위해서 [supernova](http://supernova.readthedocs.io/en/latest/) 와 같은 rc file 관리해주는 툴들을 사용하게 된다.    
(개인적으로 비슷한 시기에 bash로 비슷한 아이디어로 구현해서 써서 사용하진 않았지만 이러한 관리 툴이 필요하다면 supernova를 참고하면 좋을것같다.)    
이런 툴들은 기본적으로 shell에 환경변수를 추가하는 방식이기에 폴더가 변경된다고 자동으로 로딩된다기 보다는    
유저가 어떤 rc file을 사용할지 로딩하여(트리거링하여) 써야한다.

## openstack rc file (normal version) ##

예를 들면 아래는 일반적으로 사용하는 익숙 한 예이다.

``` bash
al@MacBook-Pro-2:~
$ source stage_v1_openrc

al@MacBook-Pro-2:~
$ env | grep OS_
OS_SERVICE_TOKEN=token4service4v1
OS_REGION_NAME=stage-v1
OS_SERVICE_ENDPOINT=https://stage-v1.example.com/v2.0
OS_PASSWORD=password4admin4v1
OS_AUTH_URL=https://stage-v1.example.com/v2.0/
OS_USERNAME=admin-v1
OS_TENANT_NAME=admin-v1

al@MacBook-Pro-2:~
$ source stage_v2_openrc

al@MacBook-Pro-2:~
$ env | grep OS_
OS_SERVICE_TOKEN=token4service4v2
OS_REGION_NAME=stage-v2
OS_SERVICE_ENDPOINT=https://stage-v2.example.com/v2.0
OS_PASSWORD=password4admin4v2
OS_AUTH_URL=https://stage-v2.example.com/v2.0/
OS_USERNAME=admin-v2
OS_TENANT_NAME=admin-v2
```

위와 같이 `source`(혹은 `.`) 을 이용해서 미리 기록해둔 파일(stage\_v1\_openrc, stage\_v2\_openrc)를 export 해서 환경변수에 기록해서 사용한다.    
이런방식은 어째뜬 파일을 관리해야 하고 파일을 항상 사용해야 하기때문에 다른 디렉토리에서 사용하기에 불편함이 있다.

## openstack rc file (advanced version) ##

그래서 내가 만들었던 툴은 아래와 같이 사용했다.

``` bash
al@MacBook-Pro-2:~ O:stage_v2
$ rcvm stage_v1
OPENRC: stage_v1

al@MacBook-Pro-2:~ O:stage_v1
$ env | grep OS_
OS_SERVICE_TOKEN=token4service4v1
OS_REGION_NAME=stage-v1
OS_SERVICE_ENDPOINT=https://stage-v1.example.com/v2.0
OS_PASSWORD=password4admin4v1
OS_AUTH_URL=https://stage-v1.example.com/v2.0/
OS_USERNAME=admin-v1
OS_TENANT_NAME=admin-v1

al@MacBook-Pro-2:~ O:stage_v1
$ rcvm stage_v2
OPENRC: stage_v2

al@MacBook-Pro-2:~ O:stage_v2
$ env | grep OS_
OS_SERVICE_TOKEN=token4service4v2
OS_REGION_NAME=stage-v2
OS_SERVICE_ENDPOINT=https://stage-v2.example.com/v2.0
OS_PASSWORD=password4admin4v2
OS_AUTH_URL=https://stage-v2.example.com/v2.0/
OS_USERNAME=admin-v2
OS_TENANT_NAME=admin-v2
```

사실 source 같은 커맨드와 지정한 파일을 사용하는거 외에 사용방식은 비슷하다.    
`<커맨드> <구성>` 이런 호출을 통해서 rc를 변경한다. (위 보다 디렉토리 제약과 파일 관리의 부담이 조금 줄었다.)

## 기존 방법의 불편한 점 ##

위의 두가지 방법들은 한 클러스터를 집중적으로 관리할때 편리하다.

여기서 만약 다른 클러스터 작업이 필요하면 대략 아래 같은 방법의 옵션들을 사용하게 된다.

    1. shell을 추가적으로 띄워서 다른 환경변수를 로딩한다.
    2. supernova의 사용법중 하나인매번 `supernova <environment> <command>` 와 같은 형태로 환경변수를 로딩하여 호출해야 한다.

하지만 이렇게 관리하다보면 여러 클러스터에 간단한 작업을 할때마다 여러창을 띄우던지 아니면 의식적으로 환경을 변경하면서 작업해야한다.

*그래서 아래 같은 방법을 생각하게 되었다.*

## 디렉토리 기반 openrc 자동 로딩 방법 ##

그래서 생각한 아이디어는 git 이나 chef 등과 같이 parent directory 에 특정 구성 파일이 있으면    
이하 디렉토리에서 해당 환경을 자동으로 로딩할 수 있으면 편하겠다고 생각했다.    
가장 간단한 방법은 아마 nova 와 같은 커맨드에 `pwd` 등을 확인해서 파일을 로딩하는식으로 구현할까 했었다.    
하지만 이건 새로운 커맨드가 나올때마다 등 관리가 잘 안될게 뻔하기 때문에 이런 방법을 쓰지않고 있었다.

그러던 와중에 [pyenv](https://github.com/yyuu/pyenv)를 잘 쓰고 있었는데 여기에서 python command 들을 사용할때 hook을 걸 수 있는것을 알았다.    
그리고 해당 hook에 부모 디렉토리에 원하는 파일(openrc)이 있으면 로딩하는 식으로 구성하니 아주 말끔히 원하는식으로 작동하게 되었다.    

hook 위치는 아래를 참고하면 된다.

mac에서 brew 로 pyenv를 설치했다면 아래와 같다.

``` bash
# mac os x
$ cat $(brew --prefix pyenv)/pyenv.d/exec/openstack.bash
```

ubuntu는 아래와 같다. (non root user시)

``` bash
# ubuntu
$ cat ~/.pyenv/pyenv.d/exec/openstack.bash
```

해당 파일내용은 아래와 같다.
``` bash
openstack_root=$(pwd -P 2>/dev/null || command pwd)
while [ ! -e "$openstack_root/openrc" ];
do
  openstack_root=${openstack_root%/*}
  if [ "$openstack_root" == "" ]; then
    break
  fi
done

if [ "$openstack_root" != "" ]; then
  while read var; do unset "$var"; done< <(env | awk -F= '/^OS_/{print $1}')

  . "$openstack_root/openrc"
fi
```

약간에 설명을 하자면 해당 커맨드를 실행하는 해당 디렉토리에 `openrc` 파일이 있으면 `OS_`로 시작하는 모든 환경 변수를 초기화(unset)하고 openrc 를 export 한다.    
만약 해당 디렉토리에 해당 파일(openrc)가 없으면 상위 디렉토리가 존재할때까지 recursive 하게 올라간다.

## 디렉토리 기반 openrc 자동 로딩 데모 ##

아래는 데모이다.    
천천히 보려면 이 링크에서 커맨드를 확인하면서 볼 수 있다. (http://showterm.io/112d21ab5f83d5843f7b2)
{% youtube r_Hgitz0Tn0 560 315 %}

중간에 보이는바와 같이 `env | grep OS_` 로 환경변수가 비어 있으나    
`python` 커맨드 실행시 안에 환경변수들이 채워져 있는 것을 확인 할 수 있다.    
아래 커맨드로 python 실행시 해당 환경변수중 `OS_` 시작하는 변수들의 값을 출력해 본것이다.

``` bash
python -c "import os; print('\n'.join([str(\"%s=%s\" %(i,j)) for i,j in os.environ.iteritems() if i.startswith('OS_')]))"
```

이런식으로 앞서 말한 방법과 함께 두가지 방법을 이용하면 보다 쾌적한 클러스터 관리를 할 수 있게 된다.

## DIY ##

아래 스크립트를 이용하면 ubuntu에서 테스트 설치해서 테스트 가능하다.

``` bash install_openrc_changer.sh https://gist.github.com/leoh0/21d61d3bebe394d278e6f18d5465415d link
#!/usr/bin/env bash

sudo apt-get install -qqy git make build-essential libssl-dev zlib1g-dev libbz2-dev \
  libreadline-dev libsqlite3-dev wget curl llvm libncurses5-dev \
  libncursesw5-dev libxml2-dev libxslt1-dev libffi-dev

curl -sL https://raw.github.com/yyuu/pyenv-installer/master/bin/pyenv-installer | \
  bash

export PATH="~/.pyenv/bin:$PATH"
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"

echo 'export PATH="~/.pyenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(pyenv init -)"' >> ~/.bashrc
echo 'eval "$(pyenv virtualenv-init -)"' >> ~/.bashrc

PYTHON_CONFIGURE_OPTS="--enable-unicode=ucs4" ~/.pyenv/bin/pyenv install 2.7.9

~/.pyenv/bin/pyenv global 2.7.9

cat << OPENRC_CHOOSER > ~/.pyenv/pyenv.d/exec/openstack.bash
openstack_root=\$(pwd -P 2>/dev/null || command pwd)
while [ ! -e "\$openstack_root/openrc" ];
do
  openstack_root=\${openstack_root%/*}
  if [ "\$openstack_root" == "" ]; then
    break
  fi
done

if [ "\$openstack_root" != "" ]; then
  while read var; do unset "\$var"; done< <(env | awk -F= '/^OS_/{print \$1}')

  . "\$openstack_root/openrc"
fi
OPENRC_CHOOSER

echo 'need relogin'
exit
```

## 결론 ##

pyenv + 스크립트 한개 = 디렉토리 기반 자동 openrc file loader 제작 가능

### ps ###

더 좋은 방법은 언제든지 환영합니다.
