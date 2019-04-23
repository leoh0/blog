+++
title = "How to Migrate Ceph RBD to CSI Ceph RBD in K8S"
date = 2018-09-09T11:43:14+09:00
description = "CSI for Container Storage Interface not Crime Scene Investigation"
categories = ["technology"]
tags = ["k8s", "kubernetes", "ceph", "rbd", "csi", "migration"]
images = [
  "https://source.unsplash.com/collection/983219/1600x900"
] # overrides the site-wide open graph image
+++

{{< figure src="/images/csi-ceph.png" caption="" attr="" attrlink="" >}}

# Container Storage Interface가 왜 필요한가

Container Storage Interface(이하 CSI)는 k8s에서 확장 가능한 볼륨 사용을 위한 현존하는 최고의 방법입니다. 기존에 방법이 무엇이 문제였는지는 2가지로 접근 가능합니다.

1. 볼륨을 생성(or 삭제)할때
    * 볼륨을 생성할때는 기존에는 controller manager중에 controller가 볼륨을 생성하는식이였습니다. 예를 들어 Persistent volume claim(이하 PVC)이 들어올시 PVC를 watch 하는 controller가 이 변화를 감지하며 volume을 생성(provisioning) 했습니다. 하지만 대부분의 볼륨들을 생성 하려면 이를 위한 바이너리가 있어야 합니다.(예를 들어 ceph같은 경우 rbd 바이너리) 하지만 이런 바이너리들이 in-tree volume type들이 추가될때마다 같이 추가되면 용량이나 버전 관리에 힘든점이 있기때문에 이를 밖으로 빼서 관리하려고 했습니다.
    * 그래서 k8s에서는 [external-storage](https://github.com/kubernetes-incubator/external-storage/tree/master/ceph/rbd) 를 두고 사용자가 자신이 사용할 volume(StorageClass)에 따라 맞는 provisioner들을 선택해서 사용하는식으로 했습니다. 즉, 생성,삭제를 전담하는 controller를 띄워서 이를 수행하기 위한 방법을 만들었습니다. 하지만 이것만으로는 볼륨 생성,삭제만을 해결했고 볼륨을 연결할때의 문제는 해결되지 않았습니다.

2. 볼륨을 연결할때
    * 위의 문제가 해결되더라도 볼륨을 연결하기 위해서는 이종의 storage 타입마다 각각의 바이너리를 호스트에 설치해야하는 문제가 있었습니다. 또한 flexvolume으로 out-tree volume type또한 지원이 가능했으나 이것도 host 종속성을 피할 수 없었습니다.(root filesystem의 `/usr/libexec/kubernetes/kubelet-plugins/volume/exec` 같은 디렉토리에 바이너리를 넣어야함)
    * 결국 바이너리를 설치하기 위한 방법들은 daemonset 형태로 각각의 필요한 노드에 바이너리를 설치하는 방법으로 수행 가능했으나 나중에 볼륨을 연결할때 사용되야할 특별한 비밀키들을 k8s의 secret등으로 각 pvc가 있는 namespace에서 사용해야하는 등의 디자인의 문제들이 있었습니다. 그래서 이때까지만 해도 pvc를 사용하려면 ceph같은 경우 secret key들을 복사해서 사용하려는 namespace에 넣어야 했습니다.
    * 이런 방법은 결국 비밀키를 외부에서 얻어올 수 있도록 통신 가능한 소켓형태를 daemonset으로 각 노드에 배포하고 kubelet이 연결 가능하도록 디자인하게 변경되면서 마무리 됐습니다.

즉, 이 외에도 여러가지 driver(plugin)형태로 구현되기위한 디자인 변경점들을 가지고 CSI를 사용하면 k8s objects들 외에는 아무것도 host의 설치 변경 없이 사용하도록 되었습니다.

자세한건 [csi를 소개한 블로그](https://kubernetes.io/blog/2018/04/10/container-storage-interface-beta/) 내용들을 참고하면 됩니다.

# Container Storage Interface는 현재 사용가능 한가

우선 1.10 & CSI 0.3.0 으로 사용하나 별 문제가 없습니다. 1.12에 CSI가 GA이기 때문에 1.11 정도부터는 별 문제 없을 것으로 보입니다. [CSI Document](https://kubernetes-csi.github.io/docs/Home.html)에서는 1.10을 CSI 0.2.0 1.11에서 CSI 0.3.0을 권고합니다. 다만 0.2.0 -> 0.3.0에서 디자인 변경들이 있어서 0.3.0을 사용하는것을 더 추천합니다.

# 기존 Ceph RBD와 CSI Ceph RBD의 object 비교

## storageclass 비교

기존의 Ceph RBD의 storage class는 아래와 같습니다.

```bash
$ kubectl get sc ceph-rbd -o yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-rbd
parameters:
  adminId: admin
  adminSecretName: ceph-secret
  adminSecretNamespace: kube-system
  imageFeatures: layering
  imageFormat: "2"
  monitors: MON1:6789,MON2:6789,MON3:6789
  pool: kube-pool
  userId: user
  userSecretName: ceph-secret-user
provisioner: ceph.com/rbd
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

CSI용 Ceph RBD는 아래와 같습니다.

```bash
$ kubectl get sc csi-rbd -o yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-rbd
parameters:
  csiNodePublishSecretName: csi-rbd-secret
  csiNodePublishSecretNamespace: kube-system
  csiProvisionerSecretName: csi-rbd-secret
  csiProvisionerSecretNamespace: kube-system
  imageFeatures: layering
  imageFormat: "2"
  monitors: MON1:6789,MON2:6789,MON3:6789
  pool: kube-pool
provisioner: csi-rbdplugin
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

userId, userSecretName 와 같이 범용적이지 못한 부분이 제거되고 csiNodePublishSecretName와 같은 식으로 변경 되었습니다. 자세한 CSI spec은 [여기](https://github.com/container-storage-interface/spec/blob/master/spec.md)를 참고하면 됩니다.

## Persistent Volume 비교

ceph RBD와 CSI ceph RBD의 PV는 아래와 같이 차이가 있습니다.

{{< figure src="/images/diff_pv.png" caption="ceph RBD pv diff CSI ceph RBD pv" attr="" attrlink="" >}}

<center>
[ceph RBD pv diff CSI ceph RBD pv 크게보기](/images/diff_pv.png)
</center>

보라색으로 지운라인들은 해당 파일로 새로 생성할때 필요없거나 전략적으로 지우는 부분들입니다.
변경되거나 새로 입력해야하는 부분은 다음과 같습니다.

1. metadata.annotation는 CSI 기준으로 변경

2. finalizers는 CSI 기준으로 변경

3. name은 기존 uuid를 축약하도록 변경 (참고. `pvc-f1b328c9-b3f8-11e8-8946-fa163eadf2df` -> `pvc-f1b328c9-b3f8-11e8` -> `pvc-f1b328c9b3f811e8`)

4. csi값 기준으로 기존의 값들을 비슷하게 정의하면 됨 다만 6,7은 아래를 참고

5. storageClassName를 csi-rbd 로 변경

6. unique한 값이나 아래와 같은 방식으로 생성됨

    ```bash
    $ # https://github.com/kubernetes-csi/external-provisioner/blob/release-0.3.0/cmd/csi-provisioner/csi-provisioner.go#L99
    $ # identity := strconv.FormatInt(timeStamp, 10) + "-" + strconv.Itoa(rand.Intn(10000)) + "-" + *provisioner
    $ echo "$(($(date +%s%N)/1000000))-$(($RANDOM%10000))-csi-rbdplugin"
    ```

7. unique 한 값(e.g. UUID1)이면 됨

    ```bash
    $ # https://github.com/ceph/ceph-csi/blob/v0.1.0/pkg/rbd/controllerserver.go#L56-L61
    $ # uniqueID := uuid.NewUUID().String()
    $ # volumeID := "csi-rbd-" + uniqueID
    $ python -c "import uuid ; print('csi-rbd-' + str(uuid.uuid1()))"
    ```

참고. GO의 UUID는 UUID1을 쓰기때문에 아래와 같이 uuid로 부터 timestamp를 뽑아 낼 수 있습니다.

```bash
$ echo -n '207db805-ab31-11e8-9798-b4969112d5e4' | \
  python -c '''
import uuid,sys
from datetime import datetime, timedelta, timezone
data = sys.stdin.readline()
u=uuid.UUID(str(data))
date = datetime(1582, 10, 15) + timedelta(microseconds=u.time//10)
print(date.replace(tzinfo=timezone.utc))
'''
2018-08-29 02:13:32.165734+00:00
```

## Persistent Volume Claim 비교

ceph RBD와 CSI ceph RBD의 PVC는 아래와 같이 차이가 있습니다.

{{< figure src="/images/diff_pvc.png" caption="ceph RBD pvc diff CSI ceph RBD pvc" attr="" attrlink="" >}}

<center>
[ceph RBD pvc diff CSI ceph RBD pvc 크게보기](/images/diff_pvc.png)
</center>

보라색으로 지운라인들은 해당 파일로 새로 생성할때 필요없거나 전략적으로 지우는 부분들입니다.
변경되거나 새로 입력해야하는 부분은 다음과 같습니다.

1. metadata.annotation는 CSI 기준으로 변경

2. storageClassName를 csi-rbd 로 변경

3. volumeName은 기존 uuid를 축약하도록 변경 (참고. `pvc-f1b328c9-b3f8-11e8-8946-fa163eadf2df` -> `pvc-f1b328c9-b3f8-11e8` -> `pvc-f1b328c9b3f811e8`)

## rbd 커맨드 비교 & 준비

Ceph RBD와 CSI Ceph RBD 둘다 rbd 바이너리가 있는 곳에서 사용가능합니다.

사용하면서 아래와 같이 나오는 error는 무시해도됩니다. `rbd` 커맨드를 config 없이 사용하면 나타나는 워닝입니다.
```
2018-09-09 13:16:12.751360 7f6e3a685d80 -1 did not load config file, using default settings.
2018-09-09 13:16:12.803163 7f6e3a685d80 -1 auth: unable to find a keyring on /etc/ceph/ceph.client.admin.keyring,/etc/ceph/ceph.keyring,/etc/ceph/keyring,/etc/ceph/keyring.bin: (2) No such file or directory
```

### Ceph RBD의 rbd 커맨드

storage class가 맞는지 `CEPH_RBD`에 넣어서 사용하면 됩니다.

```bash
$ CEPH_RBD=${CEPH_RBD:-ceph-rbd}

$ read POOL MONITOR ID SECRET SECRET_NS <<< \
  $(kubectl get storageclass ${CEPH_RBD} \
    -o go-template='{{ .parameters.pool }}
 {{ .parameters.monitors }}
 {{ .parameters.adminId }}
 {{ .parameters.adminSecretName }}
 {{ .parameters.adminSecretNamespace }}')

$ KEY=$(kubectl get secret -n ${SECRET_NS} ${SECRET} \
    -o go-template='{{ .data.key }}' | base64 -d)

# 이후 아래와 같이 사용
$ rbd --pool $POOL --id $ID -m $MONITOR --key=$KEY list
```

### CSI Ceph RBD의 rbd 커맨드

비슷하게 CSI ceph rbd는 아래와 같이 확인 하면 됩니다. 이것도 `CSI_CEPH_RBD`를 넣어서 사용하면 됩니다. 이전과 바뀐건 adminId 같이 ID를 저장 안하고 secret의 key값에 계정이름을 넣는 다는 것입니다.

```bash
$ CSI_CEPH_RBD=${CSI_CEPH_RBD:-csi-rbd}

$ read POOL MONITOR SECRET SECRET_NS <<< \
  $(kubectl get storageclass ${CSI_CEPH_RBD} \
    -o go-template='{{ .parameters.pool }}
 {{ .parameters.monitors }}
 {{ .parameters.csiProvisionerSecretName }}
 {{ .parameters.csiProvisionerSecretNamespace }}')

$ ID=$(kubectl get secret -n ${SECRET_NS} ${SECRET} \
    -o go-template='{{ range $key, $value := .data }}{{ $key }}{{ end }}')

$ KEY=$(kubectl get secret -n ${SECRET_NS} ${SECRET} \
    -o go-template='{{ .data.'$ID' }}' | base64 -d)

# 이후 아래와 같이 사용
$ rbd --pool $POOL --id $ID -m $MONITOR --key=$KEY list
```

# How to migration

{{< figure src="/images/csipvc.png" caption="csi pvc로 마이그레이션 과정" attr="" attrlink="" >}}

<center>
[csi pvc로 마이그레이션 과정 크게보기](/images/csipvc.png)
</center>

전체적인 흐름은 일반적인 pod을 재시작 시키는 흐름과 같습니다. 예를 들어 pod이 delete되어 termination요청이 들어오고 삭제된 후 다시 scheduling되어서 pod이 생성되게 되는데 이 흐름을 이용합니다.
왜냐하면 우선 기존의 ceph-rbd에서 사용하던 마운트를 위한 바이너리 부터 마운트를 위한 모든게 바뀌었기때문에 떠있는 상태 그대로 migration은 불가능 합니다.
다만 scheduling 되는 사이에 모든 데이터를 csi형태로 마이그레이션 시키고 이를 떠있는 pod에 다시 연결하는 것이라고 보면 됩니다. 즉 ceph volume은 남긴 상태로 기존의 pod이 scheduling 되는 동안 PV, PVC를 삭제하고 CSI 형태로 재생성을 한 후에 pod에 연결하는 것입니다.

{{< figure src="/images/pvc.png" caption="일반적인 pvc 생성 과정" attr="" attrlink="" >}}

즉, 결국 여기까지 점검해보면 **가장 중요한건 이미 존재하는 ceph안의 volume(image)를 어떻게 k8s에 PV형태로 생성시키고 PVC형태로 만들어서 pod에 연결시키냐**가 주요포인트 입니다. 왜냐하면 일반적으로 dynamic provisioning은 PVC 생성으로 PV를 생성시키고 이를 통해 ceph에 volume을 생성시키지만 이 방법은 존재하는 ceph volume을 기준으로 PV와 PVC를 만들어 자신이 원하는 pod에 연결 시키는 기존의 방법을 역행하는 방법입니다.

{{< figure src="/images/reconcile_pvc.png" caption="ceph volume이 존재하는 상태에서 pvc 생성 과정" attr="" attrlink="" >}}

이 방법을 글로 정리하면 아래 정도 입니다.

* ceph volume과 PV과 연결되려면 아래의 조건이면 된됩니다.
    * ceph volume name(물론 pull, secret, access key가 맞는상태)과 PV의 metadata.name이 동일

* PV와 PVC가 연결되려면 아래의 조건이면 됩니다.
    * PV의 spec.claimRef 가 없는 상태로 (중요!)
    * PV의 metadata.name과 PVC의 spec.volumeName이 동일

# How to migration detail


1. 사전준비
    * CSI Ceph의 storageclass를 준비해 둡니다. 아래와 같이 기존에 ceph-rbd외에도 csi-rbd를 설정합니다. 이때 전략적으로 default storageclass를 조정하면 기존에 만약 storageclass없이 생성했던 PVC들을 이후부터는 csi-rbd로 붙일 수 있습니다.

        ```bash
        $ kubectl get sc
        NAME                 PROVISIONER                    AGE
        ceph-rbd             ceph.com/rbd                   33d
        csi-rbd (default)    csi-rbdplugin                  17d
        ```

        * CSI ceph은 아래 링크들을 확인하면 구축할 수 있습니다.
        * https://github.com/ceph/ceph-csi/tree/master/deploy/rbd/kubernetes
        * https://github.com/ceph/ceph-csi/tree/master/examples/rbd

    * 위의 RBD command(e.g. `rbd --pool $POOL --id $ID -m $MONITOR --key=$KEY list`)들을 준비해 둡니다. 이후에 저 커맨드로 기존의 ceph안의 볼륨 이름을 마이그레이션 이후 형태로 변경합니다. (`kubernetes-dynamic-pvc-bdb97c58-694a-11e6-91b6-080027242396` -> `pvc-bdb97c58694a11e6`)

2. 작업할 PV 선정 및 안전 설정
    * 작업할 PV를 설정합니다. 앞으로 모든 `rbd-image`<->`pv`<->`pvc`<->`pod`의 기준을 이 PV기준으로 수집합니다. 이후 `${PV}` 의 값으로 사용합니다.

        ```bash
        PV=pvc-f1b328c9-b3f8-11e8-8946-fa163eadf2df
        ```

    * 우선 기존의 PV들을 `delete` 이면 `retain`으로 변경해서 PV가 삭제되도 ceph에서 삭제가 안되도록 합니다. 왜냐하면 이후에 PV를 지우고 CSI용 PV로 재생성 해야하기 때문입니다.

        ```bash
        $ kubectl get pv
        NAME                                      CAPACITY  ACCESS MODES  RECLAIM POLICY  STATUS  CLAIM            STORAGECLASS   REASON   AGE
        pvc-f1b328c9-b3f8-11e8-8946-fa163eadf2df  1Gi       RWO           Delete          Bound   kube-system/nx   ceph-rbd                6h
        $ # retain으로 변경해서 지워지지 않도록 한다.
        $ kubectl get pv ${PV} -o yaml | \
            sed 's/persistentVolumeReclaimPolicy: Delete/persistentVolumeReclaimPolicy: Retain/g' | \
            kubectl replace -f -
        $ kubectl get pv
        NAME                                      CAPACITY  ACCESS MODES  RECLAIM POLICY  STATUS  CLAIM            STORAGECLASS   REASON   AGE
        pvc-f1b328c9-b3f8-11e8-8946-fa163eadf2df  1Gi       RWO           Retain          Bound   kube-system/nx   ceph-rbd                6h
        ```

3. 작업될 `rbd-image`, `pvc`, `pod`을 알아냄
    * 이후에 변경할 Ceph의 기존 volume 이름을 가져옵니다. `rbd-image`<->`pv`<->`pvc`<->`pod`이 관계중 `rbd-image`에 해당합니다. 이 format은 `kubernetes-dynamic-pvc-${UUID1}`과 같은 포맷으로 구성되어 있습니다. 이후에 CSI 0.3.0 spec의 이름으로 변경 시켜야 합니다. 이후부터는 `${RBDIMAGENAME}`으로 사용합니다.

        ```bash
        $ RBDIMAGENAME=$(kubectl get pv ${PV} -o go-template='{{ .spec.rbd.image }}')
        $ echo $RBDIMAGENAME
        kubernetes-dynamic-pvc-f201332c-b3f8-11e8-aeac-22581cfb0c23
        ```

    * 자신이 이후에 변경될 PV의 이름을 가져옵니다. 이후에 `${NEWPVNAME}`로 사용합니다.

        ```bash
        $ NEWPVNAME=$(echo "pvc-$(echo $PV | cut -d'-' -f2,3,4 | sed 's/-//g')")
        $ echo ${NEWPVNAME}
        pvc-f1b328c9b3f811e8
        ```

    * PV에 연결된 PVC의 이름을 가져옵니다. `rbd-image`<->`pv`<->`pvc`<->`pod`의 관계중 `pvc`에 해당합니다. 이때 pvc, pod이 다 사용할 namespace도 얻어옵니다. 이후에 각각 `${NAMESPACE}`, `${PVCNAME}` 으로 사용합니다.

        ```bash
        $ read NAMESPACE PVCNAME <<< \
          $(kubectl get pv $PV \
            -o go-template='{{ .spec.claimRef.namespace }}
         {{ .spec.claimRef.name }}')
        $ echo ${NAMESPACE} ${PVCNAME}
        kube-system nx
        ```

    * PVC에 연결된 pod의 이름을 가져옵니다. k8s 1.12의 kubectl에는 [이기능](https://github.com/kubernetes/kubernetes/pull/65837)이 포함되어 있어서 kubectl describe로 볼수 있지만 아닐시 그냥 모든 pod을 찾아서 pvc를 확인해서 찾습니다.

        ```bash
        $ PODNAME=$(kubectl get pods -n $NAMESPACE -o go-template="""
          {{- range .items -}}
            {{\$metadata:=.metadata}}
            {{- range .spec.volumes -}}
              {{- if .persistentVolumeClaim -}}
                {{- if eq .persistentVolumeClaim.claimName \"$PVCNAME\" -}}
                  {{- printf \"%s\\n\" \$metadata.name -}}
                {{- end }}
              {{- end }}
            {{- end }}
          {{- end }}""")
        $ echo ${PODNAME}
        nx-86954666cb-g8v5k
        ```

4. 변경할 PV와 PVC를 가져와서 csi version yaml로 생성 합니다.

    * PV를 CSI 형태로 변경하기위한 아래의 스크립트를 이용하면 이후에 `csi-pv-${NEWPVNAME}.yaml` 형태 파일로 생성 됩니다. `python csi-pv.py ${PV}` 로 생성 하면 됩니다. 스크립트 안에 `CSIKEY`는 실제 csi에서 ceph admin key 저장된 이름입니다. 만약 그 secret 이름이 `csi-rbd-secret` 가 아니면 변경해야 됩니다. `CSIKEY_NS` 도 동일하게 secret의 namespace의 이름이니 만약 `kube-system`이 아니면 변경해야 합니다.

        ```bash
        $ curl -sO https://gist.githubusercontent.com/leoh0/03ca401d690579fdf65c222fa5bfccef/raw/c5439a500c46803b599a0bccfc0f36d11664bd3a/csi-pv.py
        $ python csi-pv.py ${PV}
        check csi-pv-pvc-f1b328c9b3f811e8.yaml
        ```

        [gist 참고](https://gist.github.com/leoh0/03ca401d690579fdf65c222fa5bfccef)

    * PVC를 CSI 형태로 변경하기위한 아래의 스크립트를 이용하면 이후에 `csi-pvc-${NEWPVNAME}.yaml` 형태 파일로 생성 됩니다. `python csi-pvc.py ${PVCNAME} -n ${NAMESPACE}` 로 생성 하면 됩니다.

        ```bash
        $ curl -sO https://gist.githubusercontent.com/leoh0/cd17492cc5274ae7e41fdf56967e0177/raw/33269b4584a02caefc8640100592484f34494fe6/csi-pvc.py
        $ python csi-pvc.py ${PVCNAME} -n ${NAMESPACE}
        check csi-pvc-pvc-f1b328c9b3f811e8.yaml
        ```

        [gist 참고](https://gist.github.com/leoh0/cd17492cc5274ae7e41fdf56967e0177)

5. 이제 준비는 다 됐습니다. 우선 미리 삭제해도 괜찮은 PVC 부터 삭제합니다.

    * 실제 pvc를 삭제해도 바로 삭제 되지 않습니다. pod이 삭제될때까지 기다리는 상태이니 이상태까지는 우선 영향을 끼치진 않습니다. 다만 삭제를 시작하면 더이상 프로세스를 되돌리기 힘들기 때문에 여기서부터는 신중히 하는게 좋습니다.

        ```bash
        $ kubectl delete pvc -n ${NAMESPACE} ${PVCNAME}
        ```

6. 이제 중요한 pod과 pv를 삭제시켜서 새로운 csi 형태로 붙일 준비를 합니다.

    * 여기서 시작하면 이후부터는 빠르게 진행해야 합니다. pod의 migration이 시작되며 다시 pod이 뜨기 전까지 빨리 작업하면 재생성되며 바로 연결도 가능합니다. 다만 일부러 모든 것들중 pvc를 마지막에 생성시킵니다.(이 과정은 거의 마지막에 진행 하면서 다시 설명) 왜냐하면 pvc가 없으면 각각 pv 와 pod을 수정가능한 상태를 유지시킬 수 있습니다. pvc가 생겨나게 되면 pv를 변경할 수 없는 것들이 있습니다.

    * pod을 삭제하기 전에 scheduling 시킬 곳을 지정하여 그곳에 이미지를 새로 받아 놓으면 더욱 빨리 pod을 옮길 수 있습니다. 이건 여러 가지 전략들이 너무 긴 이야기라 나중에 따로 정리할 생각입니다.

        ```bash
        $ kubectl delete pods -n ${NAMESPACE} ${PODNAME}
        $ kubectl delete pv ${PV}
        ```

7. rbd이름을 변경합니다.

    * 미리 준비한 RBD 커맨드로 pv가 pod으로 부터 unmount, unmap 되어 떨어져서 rbd를 변경할 수 있는 상태인지 체크하여 이름을 변경 시킵니다.

        ```bash
        $ while true; do
          rbd --pool ${POOL} --id ${ID} -m ${MONITOR} --key=${KEY} \
            status ${RBDIMAGENAME} | grep -q 'watcher=' || \
            rbd --pool ${POOL} --id ${ID} -m ${MONITOR} --key=${KEY} \
            mv "${POOL}/${RBDIMAGENAME}" "${POOL}/${NEWPVNAME}"
          done
        ```

8. csi 형태 pv와 pvc를 재생성 합니다.

    * 미리 위에서 준비한 yaml를 이용하여 csi형태로 pv와 pvc를 생성합니다.

        ```bash
        $ kubectl create -f csi-pv-${NEWPVNAME}.yaml
        $ kubectl create -f csi-pvc-${NEWPVNAME}.yaml -n ${NAMESPACE}
        ```

9. pod이 잘 뜨는지 확인합니다.

    * pod이 pv를 mount 하는지 describe등으로 확인해야 합니다.

# 위의 내용을 스크립트로 정리

[gist 참고](https://gist.github.com/leoh0/418a4598e7ea8541eddead5a9f87622e)

```bash
#!/usr/bin/env bash

CEPH_RBD=${CEPH_RBD:-ceph-rbd}

read POOL MONITOR ID SECRET SECRET_NS <<< \
  $(kubectl get storageclass ${CEPH_RBD} \
    -o go-template='{{ .parameters.pool }}
 {{ .parameters.monitors }}
 {{ .parameters.adminId }}
 {{ .parameters.adminSecretName }}
 {{ .parameters.adminSecretNamespace }}')

KEY=$(kubectl get secret -n ${SECRET_NS} ${SECRET} \
    -o go-template='{{ .data.key }}' | base64 -d)

PV=$1

kubectl get pv ${PV} -o yaml | \
    sed 's/persistentVolumeReclaimPolicy: Delete/persistentVolumeReclaimPolicy: Retain/g' | \
    kubectl replace -f -

RBDIMAGENAME=$(kubectl get pv ${PV} -o go-template='{{ .spec.rbd.image }}')

NEWPVNAME=$(echo "pvc-$(echo $PV | cut -d'-' -f2,3,4 | sed 's/-//g')")

read NAMESPACE PVCNAME <<< \
  $(kubectl get pv $PV \
    -o go-template='{{ .spec.claimRef.namespace }}
 {{ .spec.claimRef.name }}')

PODNAME=$(kubectl get pods -n $NAMESPACE -o go-template="""
  {{- range .items -}}
    {{\$metadata:=.metadata}}
    {{- range .spec.volumes -}}
      {{- if .persistentVolumeClaim -}}
        {{- if eq .persistentVolumeClaim.claimName \"$PVCNAME\" -}}
          {{- printf \"%s\\n\" \$metadata.name -}}
        {{- end }}
      {{- end }}
    {{- end }}
  {{- end }}""")

if [ ! -f csi-pv.py ]; then
    curl -sO https://gist.githubusercontent.com/leoh0/03ca401d690579fdf65c222fa5bfccef/raw/c5439a500c46803b599a0bccfc0f36d11664bd3a/csi-pv.py
fi
python csi-pv.py ${PV}

if [ ! -f csi-pvc.py ]; then
    curl -sO https://gist.githubusercontent.com/leoh0/cd17492cc5274ae7e41fdf56967e0177/raw/33269b4584a02caefc8640100592484f34494fe6/csi-pvc.py
fi
python csi-pvc.py ${PVCNAME} -n ${NAMESPACE}

kubectl delete pvc -n ${NAMESPACE} ${PVCNAME}

# start disconnect
kubectl delete pods -n ${NAMESPACE} ${PODNAME}
kubectl delete pv ${PV}

while true; do
  rbd --pool ${POOL} --id ${ID} -m ${MONITOR} --key=${KEY} \
    status ${RBDIMAGENAME} 2>/dev/null | grep -q 'watcher=' || \
    rbd --pool ${POOL} --id ${ID} -m ${MONITOR} --key=${KEY} \
    mv "${POOL}/${RBDIMAGENAME}" "${POOL}/${NEWPVNAME}" 2>/dev/null && break
done

while true; do
    kubectl create -f csi-pv-${NEWPVNAME}.yaml && break
done
while true; do
    kubectl create -f csi-pvc-${NEWPVNAME}.yaml -n ${NAMESPACE} && break
done

NEWPODNAME=$(kubectl get pods --field-selector status.phase!=Terminating -n $NAMESPACE -o go-template="""
  {{- range .items -}}
    {{\$metadata:=.metadata}}
    {{- if ne .metadata.name \"$PODNAME\" -}}
      {{- range .spec.volumes -}}
        {{- if .persistentVolumeClaim -}}
          {{- if eq .persistentVolumeClaim.claimName \"$PVCNAME\" -}}
            {{- printf \"%s\\n\" \$metadata.name -}}
          {{- end }}
        {{- end }}
      {{- end }}
    {{- end }}
  {{- end }}""")


kubectl get pods -n ${NAMESPACE} ${NEWPODNAME}

kubectl get pv ${NEWPVNAME} -o yaml | \
    sed 's/persistentVolumeReclaimPolicy: Retain/persistentVolumeReclaimPolicy: Delete/g' | \
    kubectl replace -f -

echo "kubectl get pods -n ${NAMESPACE} ${NEWPODNAME}"
```

# 정리

이 방법은 ceph rbd를 기준으로 작성했지만 응용한다면 다른 storage들도 충분히 CSI로 업그레이드 하는데 사용할 수 있습니다. 또한 rbd image만 있는상태에서 pv, pvc를 만드는 방법은 cluster간 migration 에 응용 할 수 있습니다. 
그리고 실제 작업을 하면서 연습을 해서 충분히 숙지하고 진행하시는것을 추천합니다. 대부분 문제 없지만 아주 일부분에서 타이밍 이슈로 multiattach 에러등과 같은 문제들도 발생할 수 있습니다. 그때는 직접 rbd가 어떻게 물려 있는지 파악해서 분리할 수 있어야 진행에 문제 없기도 합니다.

아무튼 긴글 읽어 주셔서 감사합니다.



