---
layout: post
title: KANS3) GKE GatewayAPI
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
- 


[🔗](https://konghq.com/blog/engineering/gateway-api-vs-ingress)
