+++
title = "How to Debug Dead Container in K8s"
date = 2018-08-08T04:09:48+09:00
description = "till CrashLoopBackOff do us part"
categories = ["technology"]
tags = ["k8s", "kubernetes", "dead", "CrashLoopBackOff", "container", "debug"]
images = [
  "https://source.unsplash.com/collection/983219/1600x900"
] # overrides the site-wide open graph image
+++

{{< figure src="/images/dead_docker.png" caption="" attr="" attrlink="" >}}

> Bugs happen. That's a fact of life.
>
> ["ABSOLUTELY THE MOST IMPORTANT THING IN THE UNIVERSE WHEN IT COMES TO SOFTWARE DEVELOPMENT", linus](https://lkml.org/lkml/2018/8/3/621)

# k8s에서 pod 을 debugging 하는 일반적인 방법들

> Everyone knows that debugging is twice as hard as writing a program in the first place. 
>
>  “The Elements of Programming Style”, 2nd edition

우리는 소프트웨어는 언제나 문제가 생길 수 있다는 것을 알고 있고 많은 문제들을 debugging 을 통해서 해결합니다. container를 debugging 하는 것들은 준비여부(log, debugging tools, ...)가 중요해서 쉽지 않은 경우들이 대부분입니다. 다행히도 container를 debugging 하기 위한 방법들은 많은 발전을 했습니다.

그래서 이 글을 적을 때만 해도 이런 기술셋들 부터 정리 하려고 했는데 아래 글에서 아주 잘 정리해서 이를 대체 해도 될것 같습니다.

[THE STATE OF DEBUGGING MICROSERVICES ON KUBERNETES](https://radu-matei.com/blog/state-of-debugging-microservices-on-k8s/)

위 블로그 내용을 대략 간추려보면 아래와 같습니다.

public cloud에서는 아래와 같은 툴들을 지원해서 debugging 을 편하게 할수 있습니다.

* Azure의 [dev spaces](https://docs.microsoft.com/ko-kr/azure/dev-spaces/azure-dev-spaces)
* Google의 [stackdriver debugger](https://cloud.google.com/debugger/)

원격 디버깅은 squash의 kubernetes 버전을 이용하면 됩니다.

* [kubesquash](https://github.com/solo-io/kubesquash)

어플리케이션 디버깅을 도울수 있는 도구들은 아래와 같습니다.

* [KSync](https://github.com/vapor-ware/ksync): local filesystem를 k8s 내 container와 sync
* [Telepresence](https://github.com/telepresenceio/telepresence): k8s cluster로 들어오는 traffic을 local로 redirect
* [Draft](https://github.com/Azure/draft) and [Skaffold](https://github.com/GoogleContainerTools/skaffold): 일관된 재배포 방법 제공

k8s에서는 아래와 같은 기능들을 활용할 수 있습니다.

* [Share Process Namespace between Containers in a Pod](https://kubernetes.io/docs/tasks/configure-pod-container/share-process-namespace/) 를 이용해서 debugging할 container를 namespace에 sharing 시킴
* 향후엔 `debug` 커맨드를 지원해서 아래와 같은 커맨드로 임시적으로 debugger를 붙여서 디버깅을 할 수 있게 함. [Troubleshoot Running Pods](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/node/troubleshoot-running-pods.md)

   ```
   kubectl debug  --image=<your-debugger-image> <your-application-pod>
   ```

자세한 내용은 위 블로그에서 확인 가능합니다.

위에 정리한 글로 모든 문제를 해결하면 좋지만 실제로 서비스를 하다보면 위 정보들은 `살아 있는` 컨테이너를 디버깅 하기 위한 것임을 알게 됩니다. 그리고 `app`을 디버깅 하기위한 수단들이여서 뭔가 부족함을 알게됩니다.

# CrashLoopBackOff 된 컨테이너를 어떻게 디버깅 할 것인가

`CrashLoopBackOff`가 되면 pod은 지속적으로 `start` -> `crash` -> `start` -> `crash`를 반복하게 됩니다.

예를 들면 아래 같이 지속적으로 계속 리스타트 되고 있는 팟들은 디버깅 하다보면 계속 리스타트 되서 테스트를 하기 힘듭니다.
```text
$ kubectl get pod --all-namespaces | grep -v Running
NAMESPACE  NAME                         READY   STATUS             RESTARTS   AGE
test       my-very-important-pod-pgzgb  0/1     CrashLoopBackOff   6761       17d
...
```

그리고 CrashLoopBackOff의 원인은 아주 여러가지기 때문에 user들이 자신의 pod이 CrashLoopBackOff 이 되어서 구글에 검색해봐도 답은 다음과 같습니다.

> log를 잘 보세요.

사실 이말이 틀렸다고 할 수는 없으나(원론적으로는 가장 맞습니다.) 문제가 있는 상태에서 log로 해결하는건 쉬운일이 아닙니다.
예를 들어 해결하고자 하면 `기존 코드에 log 추가` -> `재빌드` -> `재배포` -> `문제가 아님을 확인` -> `기존 코드에 log 추가` -> `재빌드` -> `재배포` 를 반복하게 됩니다.

약간 더 추가하자면 `describe`까지 봐야 합니다. pods이 죽었을때 `message`와 `event`를 봐야 하기 때문입니다.

그래서 이 글에서는 이런 컨테이너가 `CrashLoopBackOff` 되고 있는 이유 중 대표적인 2가지에 대해서 설명하고 이를 해결하기 위한 도구, 방법들을 설명하려고 합니다.

1. liveness probe에 만족하지 못해서 pod이 리스타트 되는 경우
2. k8s pod의 command가 exit 0 외의 code로 실패하고 있는 경우

물론 이 2가지를 설명하기 전에 유저가 첫번째로 봐야하는 것은 로그입니다.

일반적으로 아래와 같은 로그 확인 방법으로만 로그를 보지만 출력이 안될 경우

    ```
    kubectl -n <namespace> logs <podname> -c <containername>
    ```

아래와 같이 `-p(--previous)` 를 추가해서도 확인해 보면 좋습니다. 이전 컨테이너 로그를 볼시 문제를 확인할 가능성이 조금이라도 높아지기 때문입니다.

    ```
    kubectl -n <namespace> logs <podname> -c <containername> -p
    ```

그리고 만약 이런 케이스에 지속적으로 restart 시키고 싶지 않다면 `restartPolicy`을 `Always` 외의 값들을 살펴보는 것도 좋습니다.

## liveness probe에 만족하지 못해서 pod이 리스타트 되는 경우

이 케이스는 즉 healthcheck가 실패해서 pod이 back-off 되고 있는 상태입니다. 그래서 간단하게 liveness probe를 제거함으로 pod의 리스타트를 막을 수 있습니다. 이 방법은 단순 무식하지만 잠시동안 시간을 벌어서 pod을 디버깅 할 수 있는 시간을 버는데는 아주 용이합니다.

이 팁은 아주 간단하지만 굉장히 효과적입니다. 물론 실제 pod이 서비스 되고 있는 경우 사용하기가 꺼림직 하지만 그래도 잠깐이라도 시간을 벌어 디버깅 하기에 좋습니다.

## k8s pod의 command가 exit 0 외의 code로 실패하고 있는 경우

사실은 이게 거의 이번 포스트의 핵심이라고 할 수 있습니다. 이런 상태가 되면 우선 pod(container)은 지속적으로 죽기때문에 `exec`등을 해볼 기회조차 없습니다.

심지어 어떤 팟들은 exec를 하고 싶으나 debug tool은 커녕 `sh`도 없는 컨테이너들이 있습니다.

예를 들면 아래는 heapster의 이미지 입니다. 보다시피 `scratch`로 되어 있어 아무런 툴들이 없습니다.

```dockerfile
FROM scratch

COPY heapster eventer /
COPY ca-certificates.crt /etc/ssl/certs/

#   nobody:nobody
USER 65534:65534
ENTRYPOINT ["/heapster"]
```

이런 container들은 아래와 같이 `sh`도 사용할 수 없습니다.

```text
$ kubectl exec -ti heapster-heapster-65489b24b5-kjlk4 -n kube-system -c heapster sh
OCI runtime exec failed: exec failed: container_linux.go:348: starting container process caused "exec: \"sh\": executable file not found in $PATH": unknown
command terminated with exit code 126
```

결국 2가지를 해결해야 합니다.

1. scratch image에 debugging tool을 심는다.
2. 커맨드를 항상 성공하는 것으로 교체한다.

그래서 여기서 잠깐 우선 1번째 `scratch image에 debugging tool을 심는다.`을 집고 넘어 갑니다.

### scratch image에 debugging tool을 심는다.

우선 결론은 간단합니다. [scratch debugger](https://github.com/kubernetes/contrib/tree/master/scratch-debugger)을 사용하면 됩니다. 현재 원본 레포의 코드가 망가진 상태여서 제가 수정한 [gist](https://gist.github.com/leoh0/a4f07c89ed41dca40ffb91b5e9ec1efd)를 참고하시면 좋습니다.

이 툴은 아래와 같이 `POD_NAME`, `POD_NAMESPACE`, `CONTAINER_NAME` 정도만 지정하면 해당 pod에 debugging tool(busybox)를 심습니다.
```
scratch-debugger/debug.sh POD_NAME [POD_NAMESPACE CONTAINER_NAME]
```

원리는 단순합니다. 특정 지정된 pod과 같은 node에 pod을 띄우는데 해당 pod이 docker socket을 마운트 해서 docker command를 이용해서 busybox 바이너리를 docker cp로 특정 지정된 pod에 심는 방법입니다. 이 방법을 이용하면 아무런 제약없이 binary를 심을 수 있습니다.

하지만 이 방법도 현재 한계가 있습니다.

1. 죽은 컨테이너를 대상으로 테스트 할 수 없다.
2. 바이너리 인젝트를 하려면 해당 컨테이너가 바이너리를 바인드 마운트 해야 제대로 접근가능한 경우들이 있다.

#### 죽은 컨테이너를 대상으로 테스트 할 수 없다.

말 그대로 죽은 컨테이너를 대상으로 쓸 수 없습니다. 그래서 `커맨드를 항상 성공하는 것으로 교체한다.`에서 이를 극복하는 방법을 소개 합니다.

#### 바이너리 인젝트를 하려면 해당 컨테이너가 바이너리를 바인드 마운트 해야 제대로 접근가능한 경우들이 있다.

이 방법도 heapster(와 같은 scratch image)를 debugging 하려면 바로 돌리긴 힘듭니다.
아래와 같이 /tmp/debug-tools/sh를 바로 사용할 수가 없기 때문입니다. 이건 바이너리 인젝트 방법을 바인드 마운트 방법을 쓰지 않아서 어쩔 수 없습니다.

```
$ debugk  heapster-heapster-65489b24b5-kjlk4 -n kube-system -c heapster
Debug Target Container:
  Pod:          heapster-heapster-65489b24b5-kjlk4
  Namespace:    kube-system
  Node:         kube-test
  Container:    heapster
  Container ID: f095214821b2645f43919c4628c6d9d3019ef6c85f7f86bb7ed8d6eb6b7e5cae
  Runtime:      docker

  "Installing busybox to /tmp/debug-tools ..."
waiting for debugger pod to complete (currently Pending)...
waiting for debugger pod to complete (currently Pending)...
waiting for debugger pod to complete (currently Pending)...
waiting for debugger pod to complete (currently Pending)...
waiting for debugger pod to complete (currently Running)...
waiting for debugger pod to complete (currently Running)...
waiting for debugger pod to complete (currently Running)...
waiting for debugger pod to complete (currently Running)...
pod "debugger-st86p" deleted
Installation complete.
To debug heapster-heapster-65489b24b5-kjlk4, run:
    kubectl --namespace=kube-system exec -i -t heapster-heapster-65489b24b5-kjlk4 -c heapster -- /tmp/debug-tools/sh -c 'PATH=${PATH}:/tmp/debug-tools sh'
Dumping you into the pod container now.

OCI runtime exec failed: exec failed: container_linux.go:348: starting container process caused "exec: \"/tmp/debug-tools/sh\": stat /tmp/debug-tools/sh: no such file or directory": unknown
command terminated with exit code 126
```

대신 아래와 같은 방법으로 `/tmp/debug-tools/busybox sh` 와 같은 방법으로 busybox를 바로 호출하는식으로 사용하면 접근이 가능합니다.

```
$ kubectl --namespace=kube-system exec -i -t heapster-heapster-65489b24b5-kjlk4 -c heapster -- /tmp/debug-tools/busybox sh
/ $
```

### 커맨드를 항상 성공하는 것으로 교체한다.

띄우기만 해도 실패하는 pod을 디버깅 하기 위해서는 커맨드를 교체해야 합니다. 우선 가장 효용성이 좋은건 아무래도 `sleep`입니다. sleep인 pod에 다른 자신이 원하는 커맨드들을 실행시켜서 결과를 보기 용이 하기 때문입니다.

하지만 커맨드를 교체할때 `sleep`과 같은 커맨드가 있어야 함으로 미리 위와같이 debug tool을 심어야 하는 경우들이 있습니다.

또한 커맨드를 교체하면 기존에 떠있는 팟들이 전체가 수정되야 하는 불편함이 있으니, 하나의 pod을 복사해서 새로 띄워서 사용하도록 합니다. 그리고 팟이 떴을때 문제가 될만한 소지가 있는 부분들을 수정, 제거 합니다.

제거하는 값들은 다음과 같습니다.

- .metadata.ownerReferences (replica set 종속성에서 띄어내서 서비스에 속하지 않는 pod으로 설정하도록 함)
- .metadata.selfLink
- .metadata.generateName
- .metadata.labels.pod-template-hash
- .spec.nodeName (이전 노드와 다른 노드에도 스케쥴링이 가능하도록 유도 예를 들어 포트 충돌 회피)
- .spec.containers[].livenessProbe (헬스체크 실패해도 영향 없도록)
- .spec.containers[].readinessProbe
- .status

수정하는 값들은 다음과 같습니다.

- .metadata.name (기존 pod과 이름을 다르게 함)
- .spec.initContainers (busybox binary inject)
- .spec.containers[].command (sleep command)
- .spec.containers[].volumeMounts (busybox binary inject)
- .spec.volumes (busybox binary inject)
- .spec.volumes[].persistentVolumeClaim (pvc에 동시접근을 막도록 emptyDir로 변경)

아래 스크립트는 [여기](https://gist.github.com/leoh0/0af21e5839a1ee7a799e8b2b40603ecd)를 참고하셔도 됩니다.

자세한 코드는 아래와 같습니다. 이 코드를 실행가능한 곳에 `debugdead.py` 와 같이 저장해 둡니다.

```python
#!/usr/bin/env python

import argparse
import os
import subprocess
import sys
import uuid
from tempfile import NamedTemporaryFile

import yaml


def main():
    parser = argparse.ArgumentParser(description='Dead container rising.')
    parser.add_argument('pod', type=str,
                        help='target pod name')
    parser.add_argument('-n', '--namespace', type=str,
                        default='default',
                        help='target namespace (default: default)')
    parser.add_argument('-c', '--container', type=str,
                        default='',
                        help='target container name')
    parser.add_argument('-k', '--keep', action="store_true",
                        help='keep yaml')

    args = parser.parse_args()
    container_name = args.container

    kwargs = {}
    kwargs.setdefault('stdout', subprocess.PIPE)
    kwargs.setdefault('stderr', subprocess.STDOUT)
    kwargs.setdefault('encoding', 'utf8')
    kwargs.setdefault('universal_newlines', True)
    kwargs['env'] = os.environ.copy()

    command = ['kubectl', 'get', 'pods', '-n', args.namespace,
               args.pod, "--export=true", "-o", "yaml"]
    pipe = subprocess.Popen(command, **kwargs)
    output, _ = pipe.communicate()

    try:
        data = yaml.load(output)
    except yaml.YAMLError as exc:
        print(exc)

    if 'metadata' not in data:
        print("There is no pod")
        return

    data['metadata'].pop('ownerReferences', None)
    data['metadata'].pop('selfLink', None)
    name = "debug-"
    if 'generateName' in data['metadata']:
        name += data['metadata']['generateName']
        name += str(uuid.uuid4())[:5]
    else:
        name += data['metadata']['name']
        name += str(uuid.uuid4())[:5]
    data['metadata']['name'] = name
    data['metadata'].pop('generateName', None)
    if data['metadata'].get('labels'):
        data['metadata']['labels'].pop('pod-template-hash', None)

    if not data['spec'].get('initContainers'):
        data['spec']['initContainers'] = []

    data['spec'].pop('nodeName', None)

    initcontainer = {}
    initcontainer['name'] = 'init-debugger'
    initcontainer['image'] = 'busybox'
    initcontainer['command'] = ['cp', '/bin/busybox', '/tmp/mydebug/busybox']
    initcontainer['volumeMounts'] = [
        {'name': 'mydebug', 'mountPath': '/tmp/mydebug/'}]

    data['spec']['initContainers'].append(initcontainer)

    real_command = ''
    for c in data['spec']['containers']:
        if container_name == '' or c['name'] == container_name:
            if 'command' in c:
                real_command = " ".join(c['command'])

            c['command'] = [
                '/bin/busybox',
                'sh', '-c',
                '/bin/busybox --install -s /tmp/mydebug/ && '
                '/tmp/mydebug/sleep 86400']
            if 'volumeMounts' not in c:
                c['volumeMounts'] = []
            c['volumeMounts'].append(
                {'name': 'insert-busybox', 'mountPath': '/tmp/mydebug/'})
            c['volumeMounts'].append(
                {'name': 'mydebug-busybox', 'mountPath': '/bin/busybox'})

            c.pop('livenessProbe', None)
            c.pop('readinessProbe', None)
            if container_name == '':
                container_name = c['name']
            break
    else:
        print("Error: can not found %s" % container_name)

    if 'volumes' not in data['spec']:
        data['spec']['volumes'] = []
    for v in data['spec']['volumes']:
        if v.get('persistentVolumeClaim', None):
            v.pop('persistentVolumeClaim', None)
            v['emptyDir'] = {}

    data['spec']['volumes'].append(
        {'name': 'insert-busybox', 'emptyDir': {}})
    data['spec']['volumes'].append(
        {'name': 'mydebug', 'hostPath': {'path': '/tmp/mydebug/'}})
    data['spec']['volumes'].append(
        {'name': 'mydebug-busybox',
         'hostPath': {'path': '/tmp/mydebug/busybox'}})

    data.pop('status', None)

    file = NamedTemporaryFile(suffix='.yaml', delete=False)

    yamlfile = yaml.dump(data, None, canonical=True)
    file.write(yamlfile.encode('utf8'))
    file.close()

    command2 = ['kubectl', 'create', '-n', args.namespace, '-f', file.name]
    pipe = subprocess.Popen(command2, **kwargs)
    output, _ = pipe.communicate()
    if args.keep:
        print("Check this changed pod yaml:")
        print(file.name)
    else:
        os.unlink(file.name)

    print("To debug %s, wait some second and run:" % args.pod)
    print("    kubectl exec -ti %s -n %s "
          "-c %s -- /tmp/mydebug/sh "
          "-c \"PATH=\$PATH:/tmp/mydebug/ sh\"" %
          (name, args.namespace, container_name))

    print("")
    print("If you want to run %s container's command"
          " then check below command" % args.pod)
    print("    " + real_command)
    print("")
    print("After finish debugging please delete debugging container:")
    print("    kubectl delete pods %s -n %s " %
          (name, args.namespace))


if __name__ == '__main__':
    sys.exit(main())
```

위의 스크립트를 `debugdead.py`라고 준비해서 실행할 수 있게 해두고 아래와 같이 사용합니다.

    ```
    debugdead.py POD_NAME [-n POD_NAMESPACE -c CONTAINER_NAME];
    ```

실제 사용예 입니다. 저는 왠만한 모든 cli에 [fzf](https://github.com/junegunn/fzf) 를 붙여서 UI(item select)로 사용합니다.

아래와 같이 스크립트를 준비해서 사용합니다. 아래 커맨드는 pod들을 컨테이너가 기준으로 분리해서 상태가 running 인것과 아닌것을 출력시키고 이를 fzf 로 선택하도록 합니다.
이 후 그 결과값을 위의 스크립트에 전달해서 사용합니다.


```bash
kdd ()
{
    pods=$(kubectl get pods --all-namespaces -o=go-template='
      {{ range .items }}
        {{$metadata:=.metadata}}
        {{$status:=.status}}
        {{ range .spec.containers }}
          {{$name:=.name}}
          {{ range $status.containerStatuses }}
            {{ if eq .name $name }}
              {{ range $key, $value := .state }}
                {{- printf "%v %v %v %v\n" $metadata.namespace $metadata.name $name $key }}
              {{ end }}
            {{ end }}
          {{ end }}
        {{ end }}
      {{ end }}' | column -t | sed '1d' | \
        fzf -x -e +s --reverse --no-mouse | awk '{print $1","$2","$3}');
    if [[ $pods != "" ]]; then
        namespace=$(echo ${pods} | cut -d',' -f1);
        name=$(echo ${pods} | cut -d',' -f2);
        container=$(echo ${pods} | cut -d',' -f3);
        debugdead.py ${name} -n ${namespace} -c ${container} $@;
    fi
}
```

이후 아래처럼 입력해서 디버깅할 pod을 찾으면 아래처럼 출력됩니다.
```
$ kdd
To debug heapster-heapster-65489b24b5-kjlk4, wait some second and run:
    kubectl exec -ti debug-heapster-heapster-65489b24b5-2fea7 -n kube-system -c heapster -- /tmp/mydebug/sh -c "PATH=\$PATH:/tmp/mydebug/ sh"

If you want to run heapster-heapster-65489b24b5-kjlk4 container's command then check below command
    /heapster --source=kubernetes:https://kubernetes.default --sink=influxdb:http://influxdb-influxdb.kube-system.svc:8086

After finish debuggin4 please delete debugging container:
    kubectl delete pods debug-heapster-heapster-65489b24b5-2fea7 -n kube-system
```

우선은 잠깐 생성이 안됐을때는 아래처럼 실패할겁니다.
debugging pod이 뜰 수 있도록 조금 기다려 준 후에 접근 합니다.
```
$ kubectl exec -ti debug-heapster-heapster-65489b24b5-2fea7 -n kube-system -c heapster -- /tmp/mydebug/sh -c "PATH=\$PATH:/tmp/mydebug/ sh"
error: unable to upgrade connection: container not found ("heapster")
$ kubectl exec -ti debug-heapster-heapster-65489b24b5-2fea7 -n kube-system -c heapster -- /tmp/mydebug/sh -c "PATH=\$PATH:/tmp/mydebug/ sh"
/ $
```

이후 원하는 커맨드를 실행해서 기존 죽은 팟의 문제점을 찾도록 합니다.

자세한 과정은 아래를 참고해서 보시면 됩니다.

<script src="https://asciinema.org/a/rwE73RWEJdJWdv5TnJZMZmmQm.js" id="asciicast-rwE73RWEJdJWdv5TnJZMZmmQm" async></script>

# 결론

살아있는 팟을 디버깅할 수 있는 툴들도 많지만 죽은 이유를 알아내야 할때는 아래 정도로 정리하면 좋을 것 같습니다.

1. log 확인 log가 없으면 log -p 를 사용해서 이전 컨테이너 로그까지 확인 (없으면 추가적으로 kubelet log, docker log, controller manager log 등도 확인)
2. describe를 이용해 message, event 등을 보고 파악함
3. debug tool이 없는 scratch image일 경우 바이너리를 인젝션 시켜서 사용할 수 있도록 함
4. 커맨드가 지속적으로 실패하는 팟은 팟을 복사해서 커맨드만 교체해서 테스트 할 수 있도록 함

긴 글 읽어주셔서 감사합니다.
