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
```

### 테스트 서비스 생성

테스트로는 http 샘플을 올려서 확인하기 좋은 Httpbin을 올렸다. 되게 좋다. . [🔗](https://httpbin.org/)
링크로 들어가보면, 원하는 값을 path로 확인 할 수가 있다.

```
❯ kubectl apply -f https://raw.githubusercontent.com/solo-io/solo-blog/main/gateway-api-tutorial/01-httpbin-svc.yaml
namespace/httpbin created
serviceaccount/httpbin created
service/httpbin created
deployment.apps/httpbin created

❯ kubectl get deploy,pod,svc,endpointslices,sa -n httpbin
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/httpbin   1/1     1            1           26s

NAME                           READY   STATUS    RESTARTS   AGE
pod/httpbin-5855dc8bdd-q64pr   1/1     Running   0          26s

NAME              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/httpbin   ClusterIP   34.118.232.181   <none>        8000/TCP   26s

NAME                                           ADDRESSTYPE   PORTS   ENDPOINTS    AGE
endpointslice.discovery.k8s.io/httpbin-pl7cf   IPv4          80      10.200.1.5   26s

NAME                     SECRETS   AGE
serviceaccount/default   0         28s
serviceaccount/httpbin   0         28s

# yaml 수정으로 service/httpbin을 nodePort 타입으로 변경
NAME              TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/httpbin   NodePort   34.118.232.181   <none>        8000:30000/TCP   2m3s


❯ gcloud compute instances list | grep nyoung
gke-nyoung-test-clus-nyoung-test-pool-4bc5262f-1w03  asia-northeast3-a  e2-medium                                    10.0.0.27      34.64.48.125  RUNNING
gke-nyoung-test-clus-nyoung-test-pool-4bc5262f-zjhj  asia-northeast3-a  e2-medium                                    10.0.0.26      34.22.95.78   RUNNING

# 외부에서 노드 포트로 확인하면, 응답 ip가 노드 서버로 나온다.
❯ curl 34.22.95.78:30000/ip
{
  "origin": "10.0.0.26"
}

# node 내부에서 접근하여 확인하면, 응답 ip는 pod gw ip(서로 간 통신시 사용되는 브릿지 네트워크의 gw ip)
eunyoung@gke-nyoung-test-clus-nyoung-test-pool-4bc5262f-1w03 ~ $ curl 34.118.232.181:8000/ip
{
  "origin": "10.200.1.1"
}
```


### gateway와 route 생성

아까 위에서 페르소나 설정한걸 생각하면서 operation namespace를 생성하여 이곳에 gateway을 배포,
서비스가 위치한 httpbin namespace에 route를 배포하였다.

```
# gateway
❯ cat gw.yaml
kind: Gateway
apiVersion: gateway.networking.k8s.io/v1beta1
metadata:
  name: httpbingw
  namespace: operation
spec:
  gatewayClassName: gke-l7-global-external-managed
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    # 다른 namespace에 있는 것을 허용해줘야 붙을 수 있다.
    # namespace label로 이름을 지정하거나, all로 모든 ns 허용 혹은 same으로 동일 ns 한정 할 수 있다.
    allowedRoutes:
      namespaces:
        from: All
    
❯ k apply -f gw.yaml
gateway.gateway.networking.k8s.io/httpbingw created

❯ k get gateway -A
NAMESPACE   NAME        CLASS                            ADDRESS         PROGRAMMED   AGE
operation   httpbingw   gke-l7-global-external-managed   34.117.246.19   True         3m58s
```

lb가 생겼지만, 경로에 아무것도 할당되지 않은 상태

![1-1](/assets/img/kans3-2024/w6/1-1.png)


```
# 라우팅 규칙 추가 
❯ cat route.yaml
kind: HTTPRoute
apiVersion: gateway.networking.k8s.io/v1beta1
metadata:
  name: httpbinroute
spec:
  parentRefs:
  - kind: Gateway
    name: httpbingw
    namespace: operation
  hostnames:
  - "nyoung.xyz"
  rules:
  - matches:
    - path:
        type: Exact
        value: /get
    backendRefs:
    - name: httpbin
      port: 8000
      
❯ k apply -f route.yaml
httproute.gateway.networking.k8s.io/httpbinroute created


# 호스트로 확인해보려고 로컬 /etc/hosts에 추가
34.54.128.119 nyoung.xyz


작성중..



작성중...
