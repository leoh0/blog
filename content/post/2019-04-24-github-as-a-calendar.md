---
title: "Github as a Calendar"
date: 2019-04-24T08:55:44+09:00
draft: true
toc: false
categories: ["technology"]
tags: ["git", "github", "calendar"]
images: [
  "http://leoh0.github.io/images/2019-04-24-085809_650x202_scrot.png"
]
---

{{< figure src="http://leoh0.github.io/images/2019-04-24-085809_650x202_scrot.png" attr="" attrlink="" >}}

# play with your github contribution

개발자들이라면 일반적으로 github 링크를 공유할일 들이 있습니다.
그런데 아무래도 github의 contribution calendar를 이쁘게 잘 채워놓는 것은 쉬운일이 아니라고 생각합니다.
이럴때 평소면 상관없지만 github 링크를 적는일이 있다면 신경쓰일수 밖에 없어집니다.

하지만 git은 commit 시간을 지정해서 작성 가능하기 때문에 원하는 시간에 맞게 commit들을 생성가능 하고 이를 통해서 github contribution을 꾸밀 수 있습니다.

그리고 이런것들을 쉽게 도와 주는 다양한 툴들이 있습니다.

* [github-contributions](https://github.com/IonicaBizau/github-contributions)
* [github-spray](https://github.com/Annihil/github-spray)

만약 이런걸 모르는 사람이 있다면 적어도 당신에게 흥미로워 할것같습니다.

# github as a calendar

그렇다면 재미를 위해 이걸 이용해서 오늘의 날짜를 표현해주는 script를 공유해봅니다.

아래 스크립트는 debian 계열에서 우선 동작하도록 되어 있는데 `timezone` 패키지만 설치되어 있으면 다른곳에서도 동작 가능 합니다.

### 설정

스크립트를 사용하기 전 아래 5개의 환경변수를 셋업 해야 합니다.

* COMMIT_USERNAME

    COMMIT_USERNAME 은 커밋에 적힐 username입니다.

    ```
    export COMMIT_USERNAME="Eohyung Lee"
    ```

* COMMIT_USEREMAIL

    COMMIT_USEREMAIL 은 커밋에 적힐 email입니다. github에 등록되어 있는 email을 사용하는게 좋습니다.

    ```
    export COMMIT_USEREMAIL=liquidnuker@gmail.com
    ```

* GITHUB_USERNAME

    GITHUB_USERNAME 은 github계정입니다.

    ```
    export GITHUB_USERNAME=leoh0
    ```

* GITHUB_REPOSITORY

    GITHUB_REPOSITORY 은 이 가짜 커밋 데이터들을 생성할 레포지토리 입니다.
    **이 레포지토리는 매번 github calendar를 깨끗히 초기화 시키기 위해 생성전에 삭제합니다.**

    ```
    export COMMIT_USERNAME="Eohyung Lee"
    ```

* GITHUB_APITOKEN

    GITHUB_APITOKEN 은 github을 API로 이용하기 위한 token입니다.
    [이 링크](https://help.github.com/en/articles/creating-a-personal-access-token-for-the-command-line)를 참고하시면 token 생성방법을 확인 가능합니다.

    **이 토큰은 반드시 `delete_repo`, `repo`의 권한을 가지고 있어야 합니다.**

    ```
    export GITHUB_APITOKEN=*******************************************
    ```

* 옵션

    추가로 `export USEPRIVATE=true`를 사용하면 private repository를 이용할 수 있습니다. private repository를 이용하는 이유는 쓸데없이 follower 들에게 알람이 가지 않도록 하기 위함 입니다. 다만 이 옵션을 사용할 경우엔 contribution settings 에서 아래와 같이 `Private contributions` 옵션을 켜줘야 합니다.

    {{< figure src="http://leoh0.github.io/images/2019-04-25-103218_755x242_scrot.png" attr="" attrlink="" >}}

### 실행

저는 이런 것들을 돌리기 위해서 gcp 에서 [always free](https://cloud.google.com/free/docs/gcp-free-tier#always-free-usage-limits)인 자원을 사용합니다. `f1-micro`에 특정 지역인 경우 무료이기 때문에 이런 장난감을 돌리기 용의 합니다. 그래서 우선 gcp 에서 f1-micro에 ubuntu-18.04 이미지로 사용했습니다.

부팅 이후 아래와 같이 실행하면 cron이 등록 됩니다. 이후 정각부터 cron으로 구동되어 github contribution을 업데이트 해주게 됩니다. 보통 정각에 구동되면 반영되기 까지 3-4분 정도 소요됩니다.

다른 환경에서 구동시키려면 아래 스크립트를 돌리기 전에 스크립트 내용을 아시면 좋습니다. 

```
curl https://gist.githubusercontent.com/leoh0/06f763c23b46e1f07c05a4df7c9abdc2/raw/9b7c756d4e957fb31d35a215a654ef200f62f882/github-contribution-as-a-calendar.sh | bash
```

### Review code

[gist link](https://gist.github.com/leoh0/06f763c23b46e1f07c05a4df7c9abdc2)

```bash
#!/bin/bash

set -e
set -o pipefail

USEPRIVATE=${USEPRIVATE:-false}

if [[ -z "$COMMIT_USERNAME" \
    || -z "$COMMIT_USEREMAIL" \
    || -z "$GITHUB_USERNAME" \
    || -z "$GITHUB_REPOSITORY" \
    || -z "$GITHUB_APITOKEN" ]]; then
    echo "You need set below variables before run this script"
    echo
    echo "# COMMIT_USERNAME is for your commit user name"
    echo "export COMMIT_USERNAME="
    echo
    echo "# COMMIT_USEREMAIL is for your commit email"
    echo "export COMMIT_USEREMAIL="
    echo
    echo "# GITHUB_USERNAME is for github user name"
    echo "export GITHUB_USERNAME="
    echo
    echo "# GITHUB_REPOSITORY is for pushed dummy datas which included commits for github contribution calendar."
    echo "# Attention!: This repository is always *DELETED* before it is created because of cleansing github contribution calendar. So please use disposable repository."
    echo "export GITHUB_REPOSITORY="
    echo
    echo "# GITHUB_APITOKEN is for github API"
    echo "# This token MUST include follow permissions. delete_repo, repo"
    echo "# https://help.github.com/en/articles/creating-a-personal-access-token-for-the-command-line"
    echo "export GITHUB_APITOKEN="
    exit 1
fi

# setup docker
if ! command -v docker >/dev/null ; then
    curl https://releases.rancher.com/install-docker/18.09.sh | sh

    sudo usermod -aG docker "$USER"
fi

# setup github-contribution bin
mkdir -p "$HOME/bin"

if [[ "$USEPRIVATE" == true ]]; then
    PRIVATE=", \\\"private\\\": true"
fi

cat << EOF > "$HOME/bin/update-github-contribution.sh"
#!/bin/bash

set -e
set -o pipefail

if [[ "\$(env TZ=Asia/Seoul date +%H)" != "00" ]]; then
    echo "Only working at KST(Asia/Seoul) midnight"
    exit 0
fi

curl -s -S -n -XDELETE \\
    "https://api.github.com/repos/$GITHUB_USERNAME/$GITHUB_REPOSITORY"

curl -s -S -f -n -XPOST \\
    -d "{\"name\": \"$GITHUB_REPOSITORY\"$PRIVATE}" \\
    "https://api.github.com/user/repos"

docker run \\
        --rm \\
        --name github-spray \\
        -i \\
        -e "USERNAME=$COMMIT_USERNAME" \\
        -e "USEREMAIL=$COMMIT_USEREMAIL" \\
        -v $HOME/.netrc:/root/.netrc \\
        leoh0/github-spray \\
        -t "\$(env TZ=Asia/Seoul date '+%x')" \\
        --multiplier 10 \\
        --push \\
        --origin https://github.com/$GITHUB_USERNAME/$GITHUB_REPOSITORY.git
EOF

chmod +x "$HOME/bin/update-github-contribution.sh"

# setup netrc
cat << EOF > "$HOME/.netrc"
machine github.com
  login $GITHUB_USERNAME
  password $GITHUB_APITOKEN

machine api.github.com
  login $GITHUB_USERNAME
  password $GITHUB_APITOKEN
EOF

# setup date for using timezone
if [ ! -d "/usr/share/zoneinfo" ]; then
    export DEBIAN_FRONTEND=noninteractive
    ERROR="ERROR: Installing timezone package is failed so you need to install "
    ERROR="$ERROR timezone package by yourself before run this script."
    sudo apt install -y tzdata || ( echo "$ERROR" ; exit 1 )
fi

# setup cron
if ! crontab -l | grep -q update-github-contribution.sh; then
    # add to cron
    crontab -l > tempcron

    # work every our for supporting different timezone
    echo "0 * * * * $HOME/bin/update-github-contribution.sh 2>&1 | \
/usr/bin/logger -t github-contribution" >> tempcron

    crontab tempcron
    rm -f tempcron
fi
```
