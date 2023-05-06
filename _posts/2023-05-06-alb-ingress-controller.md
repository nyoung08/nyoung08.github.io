---
layout: post
title: aews) alb ingress controller
category:
  - study
  - aws
tags:
  - study
  - aws
---


이전 스터디에서 [aws ingress controller 체험기](https://nyoung08.github.io/study/aws/2023/03/18/aws-ingress/)를 작성했었는데, 이번주 스터디 이후 도전과제로 주어진 [블로그](https://aws.amazon.com/ko/blogs/containers/exposing-kubernetes-applications-part-2-aws-load-balancer-controller/)글을 보고 테스트하였다. 환경은 지난번에는 kops로 구성된 클러스터였지만, 이번에는 eks 클러스터에서 테스트하였다.


![1-0](/assets/img/aews/2w/1-0.png)

ingress controller를 설치하였다면 aws에서는 ingress를 생성시 alb로 생성하게된다. 위에 그림에서와 같이 ingress controller가 생성되는 ingress를 보고 alb를 배포해주게되고, alb에서 바로 라우팅하여 트래픽을 전달하는 단일 진입점 역할을 하게된다. 이 때 ingress는 ingress class의 구성 값을 참조하게 된다.


# irsa 설정


ingress controller 설치에 앞서 controller에게 권한이 필요하기 때문에 irsa설정이 필요하다. 배포해둔 기존 클러스터를 업데이트하였다.

```

(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# cat albcontroller.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: myeks
  region: ap-northeast-2
iam:
  withOIDC: true
  serviceAccounts:
    - metadata:
        name: aws-load-balancer-controller
        namespace: kube-system
      attachPolicyARNs:
          - arn:aws:iam::$ARN_NUM:policy/AWSLoadBalancerControllerIAMPolicy


# sa를 따로 만들지 않고 위의 yaml을 적용시키면 sa이 생성되면서 iam policy arn이 붙는다.
# eksctl은 cloud formation으로 동작하는데, 기존 스택에 의존하는 것이 있는 것 같다..
# 해당 클러스터가 eksctl로 만들었었는데 --override-existing-serviceaccounts 옵션이 필요했다.
(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# eksctl create iamserviceaccount --config-file=albcontroller.yaml  --override-existing-serviceaccounts --approve

(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# k describe sa aws-load-balancer-controller -n kube-system
Name:                aws-load-balancer-controller
Namespace:           kube-system
Labels:              app.kubernetes.io/managed-by=eksctl
Annotations:         eks.amazonaws.com/role-arn: arn:aws:iam::$ARN_NUM:role/eksctl-myeks-addon-iamserviceaccount-kube-sy-Role1-10O6HFOBNXOQN

```

![1-1](/assets/img/aews/2w/1-1.png)


# aws lb controller crd 설치

```

# install helm
(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# chmod 700 get_helm.sh
(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# ./get_helm.sh
Helm v3.11.3 is already latest

# install crd
(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# kubectl apply -k \
>     "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"
customresourcedefinition.apiextensions.k8s.io/ingressclassparams.elbv2.k8s.aws created
customresourcedefinition.apiextensions.k8s.io/targetgroupbindings.elbv2.k8s.aws created

# install ingress controller
(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# helm repo add eks https://aws.github.io/eks-charts
"eks" has been added to your repositories
(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller \
>     --namespace kube-system \
>     --set clusterName=myeks  --set serviceAccount.create=false \
>     --set serviceAccount.name=aws-load-balancer-controller
Release "aws-load-balancer-controller" does not exist. Installing it now.
NAME: aws-load-balancer-controller
LAST DEPLOYED: Thu May  4 10:21:44 2023
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
AWS Load Balancer controller installed!

(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# kubectl -n kube-system rollout status deployment aws-load-balancer-controller
deployment "aws-load-balancer-controller" successfully rolled out
(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# kubectl get deployment -n kube-system aws-load-balancer-controller
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   2/2     2            2           28s

```

# test pod, svc 배포

```
# yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${SERVICE_NAME}
  namespace: ${NS}
  labels:
    app.kubernetes.io/name: ${SERVICE_NAME}
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: ${SERVICE_NAME}
  replicas: 1
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ${SERVICE_NAME}
    spec:
      terminationGracePeriodSeconds: 0
      containers:
        - name: ${SERVICE_NAME}
          image: hashicorp/http-echo
          imagePullPolicy: IfNotPresent
          args:
            - -listen=:3000
            - -text=${SERVICE_NAME}
          ports:
            - name: app-port
              containerPort: 3000
          resources:
            requests:
              cpu: 0.125
              memory: 50Mi
---
apiVersion: v1
kind: Service
metadata:
  name: ${SERVICE_NAME}
  namespace: ${NS}
  labels:
    app.kubernetes.io/name: ${SERVICE_NAME}
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: ${SERVICE_NAME}
  ports:
    - name: svc-port
      port: 80
      targetPort: app-port
      protocol: TCP
---

(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# SERVICE_NAME=first NS=test  envsubst < test.yaml | kubectl apply -f -
(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# SERVICE_NAME=second  NS=test  envsubst < test.yaml | kubectl apply -f -
(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# SERVICE_NAME=third NS=test  envsubst < test.yaml | kubectl apply -f -(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# SERVICE_NAME=fourth  NS=test  envsubst < test.yaml | kubectl apply -f -
(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# SERVICE_NAME=error NS=test  envsubst < test.yaml | kubectl apply -f -

```

# ingress 설정

```
# yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
  namespace: test
  annotations:
    alb.ingress.kubernetes.io/load-balancer-name: test-ingress
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/healthcheck-path: /healthz
spec:
  ingressClassName: alb
  defaultBackend:
    service:
      name: error
      port:
        name: svc-port
  rules:
    - http:
        paths:
          - path: /first
            pathType: Prefix
            backend:
              service:
                name: first
                port:
                  name: svc-port
          - path: /second
            pathType: Prefix
            backend:
              service:
                name: second
                port:
                  name: svc-port
    - host: test.nyoung.xyz
      http:
        paths:
          - path: /third
            pathType: Prefix
            backend:
              service:
                name: third
                port:
                  name: svc-port
          - path: /fourth
            pathType: Prefix
            backend:
              service:
                name: fourth
                port:
                  name: svc-port

```

ingress의 annotation의 정보들은 이전 과제에서 한번 정리했었다. [🔗](https://nyoung08.github.io/study/aws/2023/03/18/aws-ingress/)


![1-2](/assets/img/aews/2w/1-2.png)

```

(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# export ALB_URL=$(kubectl get -n test ingress/test-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# curl ${ALB_URL}/first
first
(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# curl ${ALB_URL}/second
second

# 호스트기반 라우팅 확인은 /etc/hosts에 넣어주거나 -H 옵션을 붙여 확인 가능하다.
(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# cat /etc/hosts
...
43.201.194.5 test.nyoung.xyz

(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# curl test.nyoung.xyz/third
third

(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# curl ${ALB_URL}/third -H 'Host:test.nyoung.xyz'
third

```

### ingress group

하나의 alb에 붙어서 분기처리가 잘 되는 것을 확인했다. 지금은 하나의 namespace 였지만 여러 namespace의 경우 어떻게 될까. ingress는 namespace단위라 ns마다 ingress를 생성할텐데, 그럴때마다 alb가 생성되면 너무 많은 lb가 생성되어 관리가 어려워질 수 있다. 이 때 ingress group을 두어 하나의 alb에서 여러 ns의 ingress가 붙을 수 있도록 구성할 수 있다.


```

# ingress.yaml
metadata:
  name: test-ingress
  namespace: test
  annotations:
    alb.ingress.kubernetes.io/group.name: test-group
...

# ingress2.yaml
metadata:
  name: test2-ingress
  namespace: test2
  annotations:
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/group.name: test-group
    alb.ingress.kubernetes.io/group.order: '10'
    # 낮을 수록 우선순위가 높으며, 같은 ingress group내에서는 고유한 숫자여야함
...

# 위에서 사용한 $ALB_URL을 그대로 사용하여 호출
(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# curl ${ALB_URL}/test2
test2

# 동일한 Address를 사용 중
eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# k get ingress -A
NAMESPACE   NAME            CLASS   HOSTS             ADDRESS                                                   PORTS   AGE
test        test-ingress    alb     test.nyoung.xyz   test-ingress-232841134.ap-northeast-2.elb.amazonaws.com   80      2d3h
test2       test2-ingress   alb     *                 test-ingress-232841134.ap-northeast-2.elb.amazonaws.com   80      4m26s

```
 
![1-3](/assets/img/aews/2w/1-3.png)



# ingress class

ingress가 배포될 때 어떤 ingress class를 바라보고 배포할지 선택하게 된다. 위에서는 기본으로 생성되어있는 alb를 사용하여 ingress를 생성했었었다 (.spec.ingressClassName: alb)

```
# aws lb controller 설치시 기본으로 생성된 ingress class

(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# k describe ingressclass
Name:         alb
Labels:       app.kubernetes.io/instance=aws-load-balancer-controller
              app.kubernetes.io/managed-by=Helm
              app.kubernetes.io/name=aws-load-balancer-controller
              app.kubernetes.io/version=v2.5.1
              helm.sh/chart=aws-load-balancer-controller-1.5.2
Annotations:  meta.helm.sh/release-name: aws-load-balancer-controller
              meta.helm.sh/release-namespace: kube-system
Controller:   ingress.k8s.aws/alb

# aws lb controller 설치시 기본으로 생성된 ingress params
(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# k describe IngressClassParams
Name:         alb
Namespace:
Labels:       app.kubernetes.io/instance=aws-load-balancer-controller
              app.kubernetes.io/managed-by=Helm
              app.kubernetes.io/name=aws-load-balancer-controller
              app.kubernetes.io/version=v2.5.1
              helm.sh/chart=aws-load-balancer-controller-1.5.2
Annotations:  meta.helm.sh/release-name: aws-load-balancer-controller
              meta.helm.sh/release-namespace: kube-system
API Version:  elbv2.k8s.aws/v1beta1
Kind:         IngressClassParams

```


### ingress class 기본 지정

.spec.ingressClassName 없이 그냥 ingress를 생성할 수 있도록 annotation을 추가하여 생성했다.


```

(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# cat ingressclass.yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: test-ingressclass
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: ingress.k8s.aws/alb

# ingress class name 지정없이 ingress 생성 후 확인 시, 위의 ingress class가 사용된 것을 확인 할 수 있다.
(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# k get ingress test3-ingress -n test2 --output=jsonpath={.spec.ingressClassName}
test-ingressclass

```

### ingress class params

ingress 사용을 제어하고 싶다면 ingress class가 참조하는 ingress class params를 사용할 수 있다. [🔗](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.2/guide/ingress/ingress_class/#ingressclassparams-specification)
아래 테스트에서는 ns가 test2인 환경에서 외부 lb만 생성되도록 설정 후 확인해보았다.


```

(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# cat ingressclass.yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: test-ingressclass
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: ingress.k8s.aws/alb
  parameters:
    apiGroup: elbv2.k8s.aws
    kind: IngressClassParams
    name: test-class-cfg
---
apiVersion: elbv2.k8s.aws/v1beta1
kind: IngressClassParams
metadata:
  name: test-class-cfg
spec:
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: test2
  scheme: internet-facing


# test2가 아닌 ns에서는 위 ingress class를 달고 생성이 불가하다. 
(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# k apply -f ingress-5.yaml -n test
Error from server (invalid ingress class: namespaceSelector of IngressClassParams test-class-cfg mismatch): error when creating "ingress-5.yaml": admission webhook "vingress.elbv2.k8s.aws" denied the request: invalid ingress class: namespaceSelector of IngressClassParams test-class-cfg mismatch

# test2의 ns에서 internal로 ingress를 생성했다. 예상대로라면 안되야 하는데... 잘만들어진다. 
# 확인해보면 address는 elb가 붙고, annotation에는 internal로 붙어있는 이상한 ingress가 되었다.
(eunyoung@myeks:N/A) [root@myeks-bastion-EC2 ~]# k describe ingress test5-ingress -n test2
Name:             test5-ingress
Labels:           <none>
Namespace:        test2
Address:          k8s-test2-test5ing-700c802c5f-1032460810.ap-northeast-2.elb.amazonaws.com
Ingress Class:    test-ingressclass
Default backend:  <default>
Rules:
  Host        Path  Backends
  ----        ----  --------
  *
              /test2   test2:svc-port (192.168.2.191:3000)
Annotations:  alb.ingress.kubernetes.io/healthcheck-path: /healthz
              alb.ingress.kubernetes.io/scheme: internal
              alb.ingress.kubernetes.io/target-type: ip

```

콘솔로 보았을 때에는 elb로 생성된 것으로 확인되고 외부에서 호출도 잘된다😇 yaml 그대로 annotation에 붙었지만, 실제 alb가 생성될 때에는 ingress class가 설정한대로 alb가 생성된 것 같다.

![1-4](/assets/img/aews/2w/1-4.png)
![1-5](/assets/img/aews/2w/1-5.png)



---
참고
- [https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html](https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html)
- [https://aws.amazon.com/ko/blogs/containers/exposing-kubernetes-applications-part-2-aws-load-balancer-controller/](https://aws.amazon.com/ko/blogs/containers/exposing-kubernetes-applications-part-2-aws-load-balancer-controller/)
- [https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html](https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html)
- [https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.3/guide/ingress/annotations/#group.order](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.3/guide/ingress/annotations/#group.order)
- [https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.3/guide/ingress/annotations/#group.name](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.3/guide/ingress/annotations/#group.name)
- [https://aws.amazon.com/ko/blogs/containers/exposing-kubernetes-applications-part-2-aws-load-balancer-controller/](https://aws.amazon.com/ko/blogs/containers/exposing-kubernetes-applications-part-2-aws-load-balancer-controller/)
- [https://github.com/kubernetes-sigs/aws-load-balancer-controller/blob/main/docs/install/v2_2_4_full.yaml](https://github.com/kubernetes-sigs/aws-load-balancer-controller/blob/main/docs/install/v2_2_4_full.yaml)
- [https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.2/guide/ingress/ingress_class/#ingressclassparams-specification](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.2/guide/ingress/ingress_class/#ingressclassparams-specification)

