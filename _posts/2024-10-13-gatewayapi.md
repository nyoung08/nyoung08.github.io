---
layout: post
title: KANS3) gke gatewayapi
category:
  - study
  - gcp
tags:
  - study
  - gcp
---

이번 주차는 svc 다음으로 ingress와 gateway api 주제다. ingress는 이런저런 공부를하면서 접해서 어느정도 익숙하지만, gateway api는 사용해본적이 없어서 이를 주제로 숙제를 하기로 했다. 


# gatewayAPI

gateway api는 ingress가 조금 더 개선되어 나온 서비스라고 볼 수 있다. ingress는 단순히 http(s) 트래픽을 라우팅하기 위한 것으로 제한적인 기능들이 있었는데, 이를 보완하여 나온 것이 gatewayAPI이다.
몇가지 다른 점을 살펴보면, [🔗](https://konghq.com/blog/engineering/gateway-api-vs-ingress)
- http 트래픽만 처리하는 ingress와 달리 http 뿐만 아니라 tcp, udp등의 여러 프로토콜 관리가 가능하다. (아직 gke에서는 안되는 것으로 보이지만... 지원하는 gatewayclass가 모두 l7 lb다. [🔗](https://cloud.google.com/kubernetes-engine/docs/how-to/gatewayclass-capabilities))
- 주로 host와 path 기준으로 트래픽을 라우팅하는 ingress와 달리, gatewayAPI는 host, path, header, method, query 등 다양한 기준으로 복잡한 라우팅 설정이 가능하다. 추가로, 라우팅 설정 시 필터를 적용하여 요청,응답 헤더에 값을 추가하는 등의 작업이 가능하다.
- ingress는 외부에서 들어오는 트래픽만 관리하였었는데, gatewayAPI는 내부 간의 트래픽도 제어 가능하여 일관된 동작을 보장한다.


gatewayAPI의 컴포넌트들을 살펴보면, gateway class, gateway, route로 구성되어있다.
- gateway class: storage class처럼 네트워크 자원이 생성, 관리되는 방식을 정의
- gateway: gateway class로 연결된 LB를 만들 때, 필요한 구성에 대해 정의
- route: gateway에서 svc요청을 매핑하기 위한 프로토콜별 규칙으로 여기에 실제 라우팅 규칙이 써지는 것 (HTTP route, TCP route 등이 있음)

![1-0](/assets/img/kans3-2024/w6/1-0.png)
이미지 출처 [🔗](https://gateway-api.sigs.k8s.io/)

위의 이미지와 같이 각 리소스가 나눠지면서, 각 리소스를 담당하는 사람도 분리 할 수가 있다. 
가장 상위의 네트워크 인프라를 전반적으로 관리하는 담당자가 gateway class를 관리한다. 그 아래로는 cluster 운영자가 gateway를, 개발자들이 직접 라우팅 규칙을 적어 서로 간에 얽매이지 않고 서비스 할 수 있다. 


# gke 생성

gke에서 제공하는 gatewayAPI를 사용하기 위해서는 생성 시, 콘솔 기준 cluster의 networking 쪽에서 'enable Gateway API'를 체크를 해주거나 cli로는 ‘--gateway-api=standard’ 옵션을 추가하여 생성해야 한다. [🔗](https://cloud.google.com/kubernetes-engine/docs/how-to/deploying-gateways#enable-gateway)


```
# 각 lb타입 마다 gatewayclass로 생성되어있다.
❯ k get gatewayclass
NAME                               CONTROLLER                  ACCEPTED   AGE
gke-l7-global-external-managed     networking.gke.io/gateway   True       4h27m
gke-l7-gxlb                        networking.gke.io/gateway   True       4h27m
gke-l7-regional-external-managed   networking.gke.io/gateway   True       4h27m
gke-l7-rilb                        networking.gke.io/gateway   True       4h27m

❯ k describe gatewayclass gke-l7-global-external-managed
Name:         gke-l7-global-external-managed
Namespace:
Labels:       <none>
Annotations:  <none>
API Version:  gateway.networking.k8s.io/v1
Kind:         GatewayClass
...
Spec:
  Controller Name:  networking.gke.io/gateway
  Description:      New Global L7 External Load Balancer type.
Status:
  Conditions:
    Last Transition Time:  2024-10-12T17:34:58Z
    Message:
    Observed Generation:   1
    Reason:                Accepted
    Status:                True
    Type:                  Accepted
Events:                    <none>

```


작성중...
