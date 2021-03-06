+++
title = "Use Admin Token Instead of Cert In Admin.conf For Kubernetes"
date = 2018-11-28T18:16:54+09:00
description = "After one year later using k8s, maybe you need this."
draft = false
toc = false
categories = ["technology"]
tags = ["k8s", "kubernetes", "admin.conf", "kubectl", "token", "service account", "cluster-role"]
images = [
  "https://kamsar.net/index.php/2017/10/All-about-xConnect-Security/cert-issuer.jpg"
]
+++

{{< figure src="https://kamsar.net/index.php/2017/10/All-about-xConnect-Security/cert-issuer.jpg" attr="" attrlink="" >}}

일반적으로 kubernetes에서 운영자가 kubernetes를 조작하기 위해서는 `admin.conf` 와 같이 전체 권한을 사용할 수 있는 파일을 이용해서 kubectl을 사용합니다.
이때 `admin.conf`는 일반적으로 kubeadm 등을 이용하면 certification expiration이 1년등에 제약이 있어서 1년 뒤에는 이 admin.conf를 이용할 시 사용이 불가능해서 재 발급 받아야 합니다.

물론 kubespray등을 쓰면 10년의 기간으로 생성하긴 하나 날짜가 기간한정이란게 여전히 불안함을 해소하지는 못합니다.

그래서 인증서 기반 방법을 피해서 token 기반으로 인증을 하면 좋지만 일반적으로 알려진 token auth file로 관리하려면 apiserver에 사용할 파일을 편집하는등 불편할 수 있습니다.

그래서 service account와 cluster-role를 활용해서 간단히 생성한 token을 `admin.conf`에 추가해서 PKI 걱정 없이 admin 권한을 사용할 수 있는 예제를 만들었습니다.

대충 결과물은 아래와 같습니다. 기존 `admin.conf`와 달리 user 아래 token을 사용하는걸 알 수 있습니다.

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN5RENDQWJDZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRFNE1URXlNakUxTkRBeU1Wb1hEVEk0TVRFeE9URTFOREF5TVZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBS3BZCnBSLzVnWkdNUmpKTUlWM0F4Rm1PaFdiTkhHZm5UUWxzRlh6TVpmaGxMNXA2VGZjc1dJb25TUUQ0S2YyZXI3STAKL21ZZ3EwbXBIV1I5MmNLZ1dRU0NDTDJRNTRLRVBaRGV5N3d4TXpreDJiQXFDWDRUbW05dHVwcmpCSVRFdDRNcQpTWUtNelhMWW04RVByUHA5cUU0MGxrZFlIdXZPdExHMnhWUGY4SDVwbWM4aVpiUHFUdFhpT3NjeHpjVjMveFF4CnQ5dDJDT25kbWNMWUFJVzlqcVMxb0xOVGRlV3RXdytsd1ZTWmw2clg4K3ROQldJdUFCeGdCYkEyakxud1hDbE8KdXE5V3hlNkc0R1IxUFVjTStscFNLQURuSjIyQmJ1VzhaRmlvYkxhNzIvOFlXcCtkS1pncDhpWDBwWXpGQ0JtTgp5WDk5MHZNT3NaWVp6TlIvWllrQ0F3RUFCYU1qTUNFd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0RRWUpLb1pJaHZjTkJRRUxCUUFEZ2dFQkFESm01NDh1SEp1YWE0dlFKK1I3bytvSk50MkEKdm8zMzNlREtFclNkNjNPa0Rvc0k3NDlRa1FTV0lZb29Ed3JWUEtzT3FhYnFiODZRTm41Z2hGNElyQi80ZlpVSgpuZWhEVkpkMzM4M2d3RXlmRExldVRIMDJPTzltMGtsSTdXSFZ0WUJ6S2Z6ZzBKaDdCekJueDEwUmp0M2d5QXExCmNEbitSWFl2eDBlZVFNRUpUQlVOYXptWFh6b2RsTktOcDlYL1FwaEpIb2NnRVBCV3NucENnUFlnR3ZoRG5nK2EKRnRUbUxDSElxM1dBOVVXdlJ6cFZaSHJHWExBQ284cHhva084bXJlUXFXSEdmNE5zc3ZZWnhrbThUTDkraGZrYwpiZUVrMnJiMFFPN2RLY0lTVzBjZEUrMFhqOUQxM2xkR014UjRYUGVYQitWU0Y3a2pDUGZPZjlwbWRCVT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQ==
    server: https://k8s.test.com:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: cluster-admin
  name: cluster-admin@kubernetes
current-context: cluster-admin@kubernetes
kind: Config
users:
- name: cluster-admin
  user:
    token: eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJjbHVzdGVyLWFkbWluLXRva2VuLXZzNXZ3Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImNsdXN0ZXItYWRtaW4iLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhW2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiIwMDAzYWI5MC1lZTZkLTExZTgtYTVjMS1mYTE2M2U1NDNmNjUiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06Y2x1c3Rlci1hZG1pbiJ9.Jvt2WhSZqkB9zX1szKmERXTlf_pIJircgOgJA9NeLnECo8KhxFSJzm404Ua4BS6v2JkCkGL5-lwKuKNL5_2N1hIn3Q9F5irZf2GpYrxt6_oigRvgErWCfoAcFYngPxVoJrWY7CmizhrQLht0AhLzKAHpzLWGVNjkk7Kmx5rjxJrGRE01UAYUXxtGBW3PX7x61vsuBOYau8hNIIIRZJPdPh05hxZc82t8XOjE1qtKsiyzn3pFPLRmH_hPSKT8GDZhlQ2qLhazAejJkWgAu2r4WSucuSePSh6J37zBPgat-7GUHOZL_HNDTBNtplAEEsOgrLj_cOKvyuTPryRZw_8WtA
...
```

만드는 예는 아래와 같습니다.
[gist 링크](https://gist.github.com/leoh0/ae7db2adad939e1c0c805cd1879ebbb6)
```bash
#!/usr/bin/env bash

# In general, operators need a `admin.conf` file when they use the kubectl
# for managing k8s.
# In `admin.conf` file, there is a user section for authenticating k8s which is
# configured by PKI datas like `certificate-data` and `key-data`.

# This PKI is great authentication system in k8s, but it has some limitations.

# For example, you need to care creating new certs before it is expired and you
# can't revoke this `admin.conf` file until it is expired. If you're using
# `kubespray`, it's expiration date setted 10 years after so you don't need to
# create this `admin.conf` again in that time but it still has revoking
# `admin.conf` problem.

# And also, if you interest about replacing authentication method other than PKI
# in admin.conf, you can use `token-auth-file` config in APIserver for replacing
# PKI.
# But, it is related with APIserver so little annoying when you are using this
# config.

# So, I propose another method for solving this problem.
# This method used `JWT token` which was generated by `service-account` which
# has a `cluster-admin` role.

kubectl create serviceaccount cluster-admin -n kube-system
kubectl create clusterrolebinding cluster-admin-sa \
  --clusterrole=cluster-admin \
  --serviceaccount=kube-system:cluster-admin

SECRET_NAME=$(kubectl get sa -n kube-system cluster-admin \
  -o go-template="{{ (index .secrets 0).name }}")

TOKEN=$(kubectl get secrets -n kube-system $SECRET_NAME \
  -o go-template="{{ .data.token }}" | base64 -d)

kubectl config set-credentials cluster-admin --token=$TOKEN

CLUSTER_NAME=$(kubectl config get-clusters | tail -n1)

kubectl config set-context cluster-admin@${CLUSTER_NAME} \
  --cluster=${CLUSTER_NAME} --user=cluster-admin
kubectl config use-context cluster-admin@${CLUSTER_NAME}
```