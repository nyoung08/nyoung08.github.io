---
layout: post
title: pkos) aws ingress controller 체험기
category: 
  - study
  - aws
tags:
  - study
  - aws
---


참고로, 테스트 환경은 eks가 아닌 kops로 ec2 위에 배포한 환경이다.
테스트 구성은 ingress에서 /tetris와 /mario로 분기처리된다.


# 테스트 서버 배포


```
# deploy mario
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mario
  labels:
    app: mario
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mario
  template:
    metadata:
      labels:
        app: mario
    spec:
      containers:
      - name: mario
        image: pengbai/docker-supermario
---
apiVersion: v1
kind: Service
metadata:
   name: mario
   annotations:
     alb.ingress.kubernetes.io/healthcheck-path: /mario/index.html
spec:
  selector:
    app: mario
  ports:
  - port: 9002
    protocol: TCP
    targetPort: 8080
  type: ClusterIP
---

# deploy tetris 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tetris
  labels:
    app: tetris
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tetris
  template:
    metadata:
      labels:
        app: tetris
    spec:
      containers:
      - name: tetris
        image: bsord/tetris
---
apiVersion: v1
kind: Service
metadata:
   name: tetris
spec:
  selector:
    app: tetris
  annotations:
     alb.ingress.kubernetes.io/healthcheck-path: /tetris/index.html
  ports:
  - port: 9001
    protocol: TCP
    targetPort: 80
  type: ClusterIP
```

아래와 같이 잘 생성된 것을 확인 할 수 있다.


```
(nyoung:N/A) [root@kops-ec2 ~]# k get po,svc,ep -owide
NAME                          READY   STATUS    RESTARTS   AGE   IP              NODE                  NOMINATED NODE   READINESS GATES
pod/mario-687bcfc9cc-fq5r9    1/1     Running   0          20h   172.30.59.211   i-05957c279a51717a0   <none>           <none>
pod/tetris-7f86b95884-q9j29   1/1     Running   0          20h   172.30.59.212   i-05957c279a51717a0   <none>           <none>

NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE   SELECTOR
service/kubernetes   ClusterIP   100.64.0.1       <none>        443/TCP    22h   <none>
service/mario        ClusterIP   100.67.173.85    <none>        9002/TCP   21h   app=mario
service/tetris       ClusterIP   100.70.120.163   <none>        9001/TCP   21h   app=tetris

NAME                   ENDPOINTS            AGE
endpoints/kubernetes   172.30.41.74:443     21h
endpoints/mario        172.30.59.211:8080   21h
endpoints/tetris       172.30.59.212:80     21h
```


# ingress 생성


```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    alb.ingress.kubernetes.io/certificate-arn: $CERT-ARN
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-port: traffic-port
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
spec:
  ingressClassName: alb
  rules:
  - host: nyoung.xyz
    http:
      paths:
      - path: /tetris
        pathType: Prefix
        backend:
          service:
            name: tetris
            port:
              number: 9001
      - path: /mario
        pathType: Prefix
        backend:
          service:
            name: mario
            port:
              number: 9002
```

 
사용한 annotation을 살펴보면
- alb.ingress.kubernetes.io/certificate-arn: $CERT-ARN
  : ssl certificate arn
- alb.ingress.kubernetes.io/scheme: internet-facing
  : 외부에서 lb로 접근 가능  (default: internal)
- alb.ingress.kubernetes.io/healthcheck-port: traffic-port
  : 트래픽이 흐르는 포트로 healthcheck
- alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'
  : listen port 설정
- alb.ingress.kubernetes.io/ssl-redirect: '443'
  : http-https redirect 설정
- alb.ingress.kubernetes.io/target-type: ip
  : 외부에서 pod로 트래픽이 바로 전달됨 

> ✏️   kubernetes svc 문서를 보면 externalTrafficPolicy의 default 값이 cluster로 되어있다. 기본적으로 외부에서 접근시 pod로 바로 가는 것이 아니라, 아무 노드에 가서 iptables를 보고 pod가 있는 node로 가게되어 홉이 두개가 발생하게된다. 해당 문제로 대한 latency를 줄이기 위해 aws에서 지원하는 target-type을 ip로 변경했다. 콘솔에서도 확인해보면 lb의 target이 node가 아닌 pod ip로 되어있는 것을 확인 할 수 있다. 참고로, gke의 경우(1.17버전이상) 기본으로 network endpoint group으로 생성되어 lb에서 바로 pod로 패킷이 전달된다.



배포 후 접속을 위해 아래 작업이 필요하다. 
현재 해당 파드들의 이미지에는 각 경로가 없기때문에 생성이 필요하다.

```
(nyoung:N/A) [root@kops-ec2 ~]# k exec $podname -it -- bash

# mario
mario@mario-687bcfc9cc-fq5r9:/usr/local/tomcat$ mkdir ./webapps/ROOT/mario
mario@mario-687bcfc9cc-fq5r9:/usr/local/tomcat$ mv webapps/ROOT/* webapps/ROOT/mario
# tetris
root@tetris-7f86b95884-q9j29:/# mkdir ./usr/share/nginx/html/tetris
root@tetris-7f86b95884-q9j29:/# mv usr/share/nginx/html/* usr/share/nginx/html/tetris 
```


들어가보면 두페이지 모두 제대로 작동하는 것을 확인 할 수 있다.

![1-0](/assets/img/pkos/ingress/1-0.png)

![1-1](/assets/img/pkos/ingress/1-1.png)


```
# https로 redirect도 잘 걸려있음을 확인 가능                                                                                                                     12:14:14
❯ curl -I http://nyoung.xyz/mario/
HTTP/1.1 301 Moved Permanently
Server: awselb/2.0
Date: Sat, 18 Mar 2023 03:14:18 GMT
Content-Type: text/html
Content-Length: 134
Connection: keep-alive
Location: https://nyoung.xyz:443/mario/

# https로 호출시 200 반환
❯ curl -I https://nyoung.xyz/mario/
HTTP/2 200
date: Sat, 18 Mar 2023 03:14:14 GMT
content-type: text/html
content-length: 2781
accept-ranges: bytes
etag: W/"2781-1568822980000"
last-modified: Wed, 18 Sep 2019 16:09:40 GMT

```


콘솔에서도 아래와 같이 lb가 잘 생성되어있음을 확인 가능하다.

![1-2](/assets/img/pkos/ingress/1-2.png)

![1-3](/assets/img/pkos/ingress/1-3.png)



---
참고
- [https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/guide/tasks/ssl_redirect/](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/guide/tasks/ssl_redirect/)
- [https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/guide/ingress/annotations/](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/guide/ingress/annotations/)
- [https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html#target-type](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html#target-type)
- [https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/](https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/)
- [https://cloud.google.com/kubernetes-engine/docs/concepts/container-native-load-balancing](https://cloud.google.com/kubernetes-engine/docs/concepts/container-native-load-balancing)
