---
layout: post
title: gcp) private service connect 체험기
category: gcp
tags: gcp
---


서로 완전히 독립된 vpc에서 아무런 peering 없이 내부 통신을 할 수 있는 방법이 있다하여 테스트 해보았다. 


# 테스트 준비

테스트를 위해 먼저 source instance(consumer)와 target web server(producer) + lb를 구성해뒀다. target server는 neg로 lb에 붙인 구성이다.[🔗](https://cloud.google.com/load-balancing/docs/internal/setting-up-internal-zonal-neg#creating_gce_vm_ip_zonal_negs)


``` 

# 현재 구성되어 있는 리소스 정보들

❯ gcloud compute instances list --format="table(name,networkInterfaces.networkIP,networkInterfaces.network)"
NAME      NETWORK_IP         NETWORK
consumer  ['192.168.128.2']  ['https://www.googleapis.com/compute/v1/projects/PROJECT_ID/global/networks/test2-vpc']
producer  ['192.168.0.2']    ['https://www.googleapis.com/compute/v1/projects/PROJECT_ID/global/networks/test1-vpc']

❯ gcloud compute network-endpoint-groups list
NAME      LOCATION           ENDPOINT_TYPE  SIZE
test-neg  asia-northeast3-a  GCE_VM_IP      1

❯ gcloud compute backend-services list --filter="test-neg"
NAME          BACKENDS                                          PROTOCOL
test-backend  asia-northeast3-a/networkEndpointGroups/test-neg  TCP

❯ gcloud compute forwarding-rules list --filter="test-backend"
NAME     REGION           IP_ADDRESS    IP_PROTOCOL  TARGET
test-lb  asia-northeast3  192.168.0.10  TCP          asia-northeast3/backendServices/test-backend


# producer server
❯ curl localhost
this page served from producer server
❯ curl  192.168.0.10
this page served from producer server

# consumer server에서 lb호출 시.. 당연히 안됨.
❯ curl  192.168.0.10
curl: (28) Failed to connect to 192.168.0.10 port 80: Connection timed out

``` 


# psc 구성

 
![1-0](/assets/img/gcp/psc/1-0.png)

가운데의 psc endpoint와 service attachment를 생성하여 psc를 구성해준다.

service attachment는 publish할 서비스의 lb와 psc용 subent이 필요하다. 이 때의 subnet은 consumer의 ip가 snat되어 들어올 대역대이다. publish할 서비스의 수가 많아질 경우 해당 대역대의 크기를 고려해야한다. [🔗](https://cloud.google.com/vpc/docs/private-service-connect#subnet-nat-config)

psc endpoint는 lb compoenets 중 하나인 forwarding rule로 생성된다. 생성 후 콘솔에서 확인해보면 target이 service attachment로 붙어있는 것을 확인 할 수 있다.


``` 

# 1. psc용 subnet 생성 
❯ gcloud compute networks subnets create subnet4psc --network=test1-vpc --region=asia-northeast3 --range=10.0.0.
0/24 --purpose PRIVATE_SERVICE_CONNECT
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/regions/asia-northeast3/subnetworks/subnet4psc].
NAME        REGION           NETWORK    RANGE        STACK_TYPE  IPV6_ACCESS_TYPE  INTERNAL_IPV6_PREFIX  EXTERNAL_IPV6_PREFIX
subnet4psc  asia-northeast3  test1-vpc  10.0.0.0/24

# 2. service attachment 생성
# endpoint가 연결하려 할 때, 특정 project만 허용할 수도 무조건 모두에게 허용 할 수도 있음. 여기선 특정 project만 허용하는 방식.
❯ gcloud compute service-attachments create test-service --region=asia-northeast3 --producer-forwarding-rule=test-lb  --connection-preference=ACCEPT_MANUAL --consumer-accept-list=ALLOW_PROJECT_ID=1 --nat-subnets subnet4psc 
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/regions/asia-northeast3/serviceAttachments/test-service].
NAME          REGION           TARGET_SERVICE  CONNECTION_PREFERENCE
test-service  asia-northeast3  test-lb         ACCEPT_MANUAL

# 3. endpoint로 사용할 ip 고정
❯ gcloud compute addresses create ip4psc --region=asia-northeast3 --subnet=test2-b-subnet --addresses 192.168.12
9.10
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/regions/asia-northeast3/addresses/ip4psc].

# 4. endpoint 생성
❯ gcloud compute forwarding-rules create test-ep --region=asia-northeast3 --network=test2-vpc --address ip4psc --target-service-attachment=projects/PROJECT_ID/regions/asia-northeast3/serviceAttachments/test-service
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/regions/asia-northeast3/forwardingRules/test-ep].


# 생성한 endpoint 정보
❯ gcloud compute forwarding-rules list --filter="name=test-ep"
NAME     REGION           IP_ADDRESS      IP_PROTOCOL  TARGET
test-ep  asia-northeast3  192.168.129.10               asia-northeast3/serviceAttachments/test-service

# consumer server에서 해당 endpoint ip 호출 결과
❯ curl 192.168.129.10
this page served from producer server

``` 

target server에서 패킷을 떠봐도 snat되었기 때문에 원래의 source ip를 알 수 없다. 만약 ip를 보고 싶다면 service attachment에서 proxy protocol 사용 설정과 백엔드에서의 proxy protocol 헤더를 처리할 수 있도록 구성이 필요하다. 해당 방법을 친절하게 잘써준 블로그..[🔗](https://medium.com/google-cloud/exposing-the-client-behind-psc-2471a851ae23)


---
참고
- [https://cloud.google.com/vpc/docs/manage-private-service-connect-services](https://cloud.google.com/vpc/docs/manage-private-service-connect-services)
- [https://medium.com/google-cloud/exposing-the-client-behind-psc-2471a851ae23](https://medium.com/google-cloud/exposing-the-client-behind-psc-2471a851ae23)
- [https://cloud.google.com/load-balancing/docs/internal/setting-up-internal-zonal-neg?hl=ko#creating_gce_vm_ip_zonal_negs](https://cloud.google.com/load-balancing/docs/internal/setting-up-internal-zonal-neg?hl=ko#creating_gce_vm_ip_zonal_negs)
