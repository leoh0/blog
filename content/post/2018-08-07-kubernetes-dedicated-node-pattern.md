+++
title = "Kubernetes dedicated node pattern for Daemonsets"
date = 2018-08-07T21:25:37+09:00
description = "tl;dr label&nodeselector 와 taint&toleration을 같이 걸면 전용노드를 만들 수 있다."
categories = ["technology"]
tags = ["kubernetes", "Daemonset", "nodeselector", "taint", "toleration", "dedicated", "node", "pattern"]
images = [
  "http://leoh0.github.io/images/pattern1.png"
] # overrides the site-wide open graph image
+++

# deploy Daemonset to dedicated nodes

k8s를 운영하다 보면 전용 노드를 사용할 필요성을 느낄때가 있습니다. 특정 기능을 수행하기 위해서 라던지 아니면 특정 포트를 `hostport` 혹은 `hostnetwork`등으로 독점적으로 사용하던지 등의 경우들입니다. 이럴땐 `Deployment`보다 `Daemonset`을 이용하면 노드당 1개의 pod을 띄우기 때문에 쉽게 해당 노드들에 특정 기능을 수행하게 할 수 있으나 Daemonset의 특성상 모든 node에 들어가게 되거나 아니면 내가 원하는 특수한 node에 들어가지 못하는 경우 들이 있습니다.

그래서 이런 경우를 해결하기 위한 가이드를 작성해 봤습니다. 우선 이번 글은 **전용 노드(Daemonset들이 정확하게 그 node에만 구성)를 만드는것**을 설명합니다.

{{< figure src="/images/pattern1.png" >}}

예를 들면 위와 같이 왼쪽 pod은 왼쪽 2개의 node에 스케쥴링이 되길 원하고, 오른쪽 pod은 오른쪽 2개의 node에 스케쥴링 되고 싶어하는 상태를 가정합니다.

# label & nodeSelector

{{< figure src="/images/pattern2.png" >}}

우선 **label**과 **nodeSelector**는 위와 같은 역할을 합니다. 왼쪽과 같이 만약 `nodeSelector가 있는 pod`(daemonset)일 경우 `해당하는 label이 있는 node`에만 스케쥴링 가능합니다.
오른쪽 pod과 같이 만약 nodeSelector가 없으면 결국 모든 node에 배포 됩니다.

즉, node에 label을 해둔다면 특정 nodeSelector가 있는 pod들을 스케쥴링 가능합니다. 하지만 아직도 nodeSelector가 없는 pod들이 스케쥴링 되는 문제가 남아 있습니다.

# taint & toleration

{{< figure src="/images/pattern3.png" >}}

**taint**와 **toleration**은 개인적으로 이 단어를 각각 `오점`과 `관용`으로 해석해서 기억하고 있습니다.

> 특정한 오점을 가진 node를 수용할 수 있는 관용을 가진 pod이 스케쥴링 될 수 있다.

여기서는 저 말뜻과 같이 특정 taint가 걸려 있으면 toleration을 갖지 못한 pod들은 스케쥴링이 되지 못합니다. 즉, 저런 toleration을 가진 pod들만 taint을 가진 노드를 포함한 모든 노드에 스케쥴링 가능합니다.

만약 이기능을 사용하면 특정 pod들을 들어오지 못하게 하는 노드들을 생성 할 수 있으나 아직도 toleration이 있는 pod들이 다른 노드에도 스케쥴링 되는 문제가 있습니다.

# label & nodeSelector + taint & toleration

{{< figure src="/images/pattern4.png" >}}

즉, 이정도만 설명하면 눈치채시겠지만 이 두 기능을 섞으면 정확히 원하는 pod(Daemonset)을 원하는 노드에 스케쥴링 시킬 수 있습니다.
이런 방법들을 응용하면 또한 여러 가지 방법으로 사용가능하니 여러 패턴을 생각해보는 것도 좋을 것 같습니다.

# label & selector와 taint & toleration

다시 한번 정리하면 아래와 같습니다.

* label & selector 사용시 해당 노드에 스케쥴링 가능한가?

|                      | node labeled | node not labeled      |
|----------------------|--------------|-----------------------|
| matched selector     |  O           |          X            |
| not matched selector |  X           |          X            |

* taint & toleration 사용시 해당 노드에 스케쥴링 가능한가?

|                        | node tainted | node not tainted      |
|------------------------|--------------|-----------------------|
| matched toleration     |  O           |          O            |
| not matched toleration |  X           |          O            |

* selector가 적용안된 상태일때 노드에 스케쥴링 가능한가?

|                      | node labeled | node not labeled      |
|----------------------|--------------|-----------------------|
| NO selector          |  O           |          O            |

* toleration이 없는 상태일때 노드에 스케쥴링이 가능한가?

|                        | node tainted | node not tainted      |
|------------------------|--------------|-----------------------|
| NO toleration          |  X           |          O            |

위와 같은 패턴을 이용해서 ingress node등을 구성하는 등의 패턴으로 이용할 수 있습니다. 추가로 아래 발표했던 자료를 참고해 보시면 조금 더 도움이 될것 같습니다.

<center>
<iframe src="//www.slideshare.net/slideshow/embed_code/key/HMOPqcLxHkDhme" width="595" height="485" frameborder="0" marginwidth="0" marginheight="0" scrolling="no" style="border:1px solid #CCC; border-width:1px; margin-bottom:5px; max-width: 100%;" allowfullscreen> </iframe> <div style="margin-bottom:5px"> <strong> <a href="//www.slideshare.net/ssuser5ad078/making-cloud-native-platform-by-kubernetes-83626072" title="Making cloud native platform by kubernetes" target="_blank">Making cloud native platform by kubernetes</a> </strong> from <strong><a href="https://www.slideshare.net/ssuser5ad078" target="_blank">어형 이</a></strong> </div>
</center>
