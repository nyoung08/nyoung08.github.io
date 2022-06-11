---
layout: post
title: KANS) gke dataplane v2 체험기 (cilium)
category: study
tags: study
---
 
7주차에 cilium 을 배우면서 gke 위에서는 어떻게 돌지 궁금하여 졸업과제 겸 체험해봤다. 
찾아보니 dataplaneV2는 eBFP기반인 cilium을 사용하는 옵션이라한다.
kube-proxy없이 구현되어 iptables 아닌 eBPF에 의해 커널에서 네트워크 패킷이 제어된다.


gke 생성시 네트워킹>DataplaneV2사용설정을 체크하거나 명령어로 생성시에는 --enable-dataplane-v2 플래그를 넣어주면된다.
클러스터 생성 후 kube-system 네임스페이스를 확인해보면 dataplaneV2 컨트롤러 데몬셋인 anetd을 확인할 수 있다. 데몬셋을 확인해보면 구글에서 만들어둔 cilium:v1.10.3-gke.15 이미지를 사용하는 것으로 보인다.

![3-0-0](/assets/img/3-0-0.png)


anetd의 pod에서 cilium status로 정보를 확인해보면 'kubeProxyReplacement: strict' 환경으로 설치 되어있는 것을 확인 할 수 있다. 
그 밖의 컨피그 정보는 configmap을 확인해보면 된다. 여기서 쓰인 이미지를 잘 모르겠지만 따로 설치했을때 확인 가능한것이 달랐다. 예를 들어 아래 나오는 DSR 기능을 사용 가능하지만 status나 config에서 dsr mode를 확인할 수 없었다. 


모니터링 툴인 hubble ui를 설치했다. cilium configmap에 보면 hubble enabled가 설정되어있다. 
해당 [링크](https://github.com/rueian/gke-hubble-export/blob/master/example.yaml)의 야믈을 다운로드 받아 설치만 해주면 된다.
중간에보면 cluster role binding하는 부분에 service account의 namespace가 default로 지정되어있는 부분이 있는데, 다른 네임스페이스를 쓸 거라면 이부분을 바꿔줘야한다.

```
❯ k create ns hubble-ui

❯ k apply -f hubble.yaml -n hubble-ui
daemonset.apps/gke-hubble-export created
configmap/hubble-relay-config created
serviceaccount/hubble-relay created
deployment.apps/hubble-relay created
service/hubble-relay created
serviceaccount/hubble-ui created
clusterrole.rbac.authorization.k8s.io/hubble-ui created
clusterrolebinding.rbac.authorization.k8s.io/hubble-ui created
configmap/hubble-ui-envoy created
deployment.apps/hubble-ui created
service/hubble-ui created

# 외부 접속을 위해 svc 타입을 lb로 바꿔준다.
❯ k patch svc hubble-ui -n hubble-ui --type='json' -p '[{"op":"replace","path":"/spec/type","value":"LoadBalancer"}]'
service/hubble-ui patched

❯ k get svc -n hubble-ui -w
NAME           TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
hubble-relay   ClusterIP      10.8.14.125   <none>        80/TCP         16s
hubble-ui      LoadBalancer   10.8.1.120    <pending>     80:31946/TCP   12s
hubble-ui      LoadBalancer   10.8.1.120    34.134.179.151   80:31946/TCP   46s
```

![3-1-0](/assets/img/3-1-0.png)


테스트를 위해 간단한 앱을 올렸다.

```
❯ k create ns samplechat

❯ k create deploy chat-app --image=ayesha306/chat-app --port=8000 --replicas=2 -n samplechat
deployment.apps/chat-app created

❯ k expose deploy chat-app --type=LoadBalancer --port=8000 -n samplechat
service/chat-app exposed


❯ k get svc -n samplechat -w
NAME       TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
chat-app   LoadBalancer   10.8.5.183    <pending>     8000:30295/TCP   25s
chat-app   LoadBalancer   10.8.5.183   34.122.24.208   8000:30295/TCP   39s

❯ k get po -o wide -n samplechat
NAME                        READY   STATUS    RESTARTS   AGE    IP         NODE                                       NOMINATED NODE   READINESS GATES
chat-app-7645ccdf54-5hcxq   1/1     Running   0          7m2s   10.4.1.8   gke-cluster-1-default-pool-204698c2-kq7f   <none>           <none>
chat-app-7645ccdf54-p2qk8   1/1     Running   0          7m2s   10.4.0.6   gke-cluster-1-default-pool-204698c2-1nwk   <none>           <none>
```

![3-2-0](/assets/img/3-2-0.png)

![3-2-1](/assets/img/3-2-1.png)


디테일을 보면 출발지 ip, 도착지 파드 ip와 이름 등을 확인할 수 있다.

![3-2-2](/assets/img/3-2-2.png)



다른 [문서](https://cloud.google.com/blog/products/containers-kubernetes/bringing-ebpf-and-cilium-to-google-kubernetes-engine)를 보다 dataplaneV2로 생성시 DSR(Direct Server Return) 모드를 사용 할 수 있다하여 이것도 함께 확인해봤다. 

```
❯ while true; do curl -s  34.122.24.208:8000 ;echo "-----";sleep 1;done

❯ k exec  anetd-5dqmq -n kube-system -it -- cilium monitor -vv
Defaulted container "cilium-agent" out of: cilium-agent, clean-cilium-state (init)
Listening for events on 2 CPUs with 64x4096 of shared memory
Press Ctrl-C to quit
------------------------------------------------------------------------------
level=info msg="Initializing dissection cache..." subsys=monitor
Ethernet	{Contents=[..14..] Payload=[..54..] SrcMAC=22:83:87:bb:f3:49 DstMAC=06:aa:c2:60:73:4d EthernetType=IPv4 Length=0}
IPv4	{Contents=[..20..] Payload=[..32..] Version=4 IHL=5 TOS=96 Length=52 Id=0 Flags=DF FragOffset=0 TTL=56 Protocol=TCP Checksum=14222 SrcIP=14.32.241.170 DstIP=10.4.1.8 Options=[] Padding=[]}
TCP	{Contents=[..32..] Payload=[] SrcPort=64987 DstPort=8000(irdmi) Seq=152841599 Ack=2128586483 DataOffset=8 FIN=false SYN=false RST=false PSH=false ACK=true URG=false ECE=false CWR=false NS=false Window=2068 Checksum=46793 Urgent=0 Options=[TCPOption(NOP:), TCPOption(NOP:), TCPOption(Timestamps:1997545547/4179403526 0x7710204bf91ca306)] Padding=[]}
CPU 00: MARK 0x0 FROM 3888 to-endpoint: 66 bytes (66 captured), state establishedidentity world->18579, orig-ip 14.32.241.170, to endpoint 3888
------------------------------------------------------------------------------
...
# SrcIP와 orig-ip에서 snat되지않은 출발지 ip를 확인할 수 있다.

```

이 밖에 다른 것들은 어떻게 수정해야할지를 잘 모르겠다. configmap이나 데몬셋의 어노테이션을 수정해도 관리되는 부분이라 그런지 적용이 안되었다. 
그렇다면 gke 위에 직접 올려보면 어떨까 싶어서 [cilium 가이드](https://docs.cilium.io/en/v1.11/gettingstarted/k8s-install-helm/) 따라서 해봤는데, 버전에 따라 위에서 진행한 hubble-ui와 dsr 등은 가능하지만 그 외의 다른 플래그들을 넣어서 설치하면 자꾸 오류가 난다.



---
참고
- [https://cloud.google.com/kubernetes-engine/docs/concepts/dataplane-v2](https://cloud.google.com/kubernetes-engine/docs/concepts/dataplane-v2)
- [https://cloud.google.com/blog/products/containers-kubernetes/bringing-ebpf-and-cilium-to-google-kubernetes-engine](https://cloud.google.com/blog/products/containers-kubernetes/bringing-ebpf-and-cilium-to-google-kubernetes-engine)
- [https://arthurchiao.art/blog/k8s-l4lb/](https://arthurchiao.art/blog/k8s-l4lb/)
