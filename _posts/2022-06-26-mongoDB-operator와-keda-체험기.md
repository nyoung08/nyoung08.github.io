---
layout: post
title: DOIK) keda와 함께하는 mongoDB operator 체험기
category: study
tags: study
---

중간과제에서 테스트해봤던 keda를 사용하여 이번엔 mongoDB operator에 사용을 해봤다. 
테스트 해 본 결과부터 말하자면, keda로는 replica를 확장하는게 아니라 지정한 이상의 값이 들어왔을 때 특정 작업을 트리거하는 정도로 사용할 수 있을 것 같다. 지난 중간과제 때 테스트했던 mysql operator의 경우 deployment를 지정하여 확장가능했었지만, mongoDB operator의 경우 statefulset이 있음에도 불구하고 해당 컨트롤러를 지정하여 확장 불가능하고 트리거될 때 생성할 파드의 템플릿을 작성하게 되어있다. 



🥭
사용하게 될 mongoDB operator의 경우 percona에서 제공하는 오퍼레이터를 사용하였다. mongoDB의 엔터프라이즈 버전에서만 제공되는 몇 가지를 무료로 사용할 수 있다. [🔗](https://docs.percona.com/percona-server-for-mongodb/5.0/comparison.html)

#### mongoDB [🔗](https://www.percona.com/doc/kubernetes-operator-for-psmongodb/kubernetes.html)

```
# mongoDB 설치
> git clone -b v1.12.0 https://github.com/percona/percona-server-mongodb-operator
> cd percona-server-mongodb-operator

# crd 생성
> kubectl apply --server-side -f deploy/crd.yaml
# 네임스페이스 생성 및 컨텍스트 설정
> kubectl create namespace psmdb
> kubectl config set-context $(kubectl config current-context) --namespace=psmdb
# rbac 생성
> kubectl apply -f deploy/rbac.yaml
# operator 생성
> kubectl apply -f deploy/operator.yaml
# secrets 생성
> kubectl create -f deploy/secrets.yaml
# cluster 생성 
> kubectl apply -f deploy/cr.yaml
```

![1-1](/assets/img/doik2/1-1.png)

- deployment/percona-server-mongodb-operator: mongoDB 클러스터를 관리해주는 오퍼레이터
- statefulset/my-cluster-name-cfg : 설정 서버
- statefulset/my-cluster-name-mongos : 라우터 서버
- statefulset/my-cluster-name-rs0 : 몽고디비 서버
*이번 테스트에서는 샤딩 없이 단일 서버 환경임으로 my-cluster-name-rs0만 사용하게 된다.



#### keda [🔗](https://keda.sh/docs/2.7/concepts/)

```
> helm repo add kedacore https://kedacore.github.io/charts
> helm install keda kedacore/keda --version 2.7.2 --namespace keda --create-namespace
```

![1-2](/assets/img/doik2/1-2.png)

- deployment/keda-operator : 에이전트
- deployment/keda-operator-metrics-apiserver : 메트릭 서버



### 테스트
먼저, 테스트를 하기 위해 클라이언트 파드를 생성 후 유저와 데이터베이스 등을 생성했다.

```
# 테스트 파드 생성
> kubectl run -i --rm --tty percona-client --image=percona/percona-server-mongodb:4.4.13-13 --restart=Never -- bash -il

# 유저 생성을 위해 rs0의 서비스 dns로 mongodb 접속 
# 연결 형식 https://www.mongodb.com/docs/manual/reference/connection-string
# deploy/secrets.yaml에서 시스템유저정보 확인 가능하다
# https://www.percona.com/doc/kubernetes-operator-for-psmongodb/users.html
> mongo "mongodb+srv://userAdmin:userAdmin123456@my-cluster-name-rs0.psmdb.svc.cluster.local/admin?replicaSet=rs0&ssl=false"

# db 생성 및 데이터 추가할 유저 생성
> db.createUser({user: "ny0ung" , pwd: "n0te" , roles: [ "userAdminAnyDatabase", "dbAdminAnyDatabase","readWriteAnyDatabase"]})
# keda에서 확인할 유저 생성
> db.createUser({user: "keda" , pwd: "test" , roles: [ { role: "read", db: "testdb" }],mechanisms: ["SCRAM-SHA-1"]})

# 위에서 만든 유저로 접속하여 디비 및 콜렉션 생성
> mongo "mongodb+srv://ny0ung:n0te@my-cluster-name-rs0.psmdb.svc.cluster.local/admin?replicaSet=rs0&ssl=false"

# db, collection 생성
> use testdb
> db.createCollection("testcollection", {capped:true, size:10000})
```


#### keda.yaml [🔗](https://keda.sh/docs/2.7/scalers/mongodb/)

```

apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: mongodb-job
spec:

# 확장 시 사용될 템플릿을 직접 작성해줘야함
  jobTargetRef:
    template:
      spec:
        containers:
          - name: mongodb-update
            image: percona/percona-server-mongodb:4.4.13-13
            command: ["/bin/sh","-il"]
            imagePullPolicy: IfNotPresent
        restartPolicy: Never
    backoffLimit: 1

# rs0 StatefulSet을 지정하여 scale out 하려했지만 적용되지 않음
#  scaleTargetRef:
#    apiVersion: apps/v1  
#    kind: StatefulSet         
#    name: my-cluster-name-rs0     
    
  pollingInterval: 1              
  maxReplicaCount: 10             
  successfulJobsHistoryLimit: 10   
  failedJobsHistoryLimit: 10      

  triggers:
    - type: mongodb
      metadata:
        connectionStringFromEnv: connectionString
        dbName: "testdb"
        collection: "testcollection"
        query: '{"name":"test2","status":"running"}'
        queryValue: "1"
      authenticationRef:
        name: mongodb-trigger
---

apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: mongodb-trigger
spec:
  secretTargetRef:
    - parameter: connectionString
      name: mongodb-secret
      key: connect
---

apiVersion: v1
kind: Secret
metadata:
  name: mongodb-secret
type: Opaque
data:
  # 위에서 만들었던 유저정보와 호스트 정보가 담긴 연결 uri
  connect: bW9uZ29kYitzcnY6Ly9rZWRhOnRlc3RAbXktY2x1c3Rlci1uYW1lLXJzMC5wc21kYi5zdmMuY2x1c3Rlci5sb2NhbC90ZXN0ZGI/cmVwbGljYVNldD1yczAmc3NsPWZhbHNl

```

![2-1](/assets/img/doik2/2-1.png)

준비 완료된 상태 
(캡처하는걸 잊어서 테스트 끝난 뒤의 상태. ..)



#### 📈 자동 확장 테스트

```
# 접속
> mongo "mongodb+srv://ny0ung:n0te@my-cluster-name-rs0.psmdb.svc.cluster.local/admin?replicaSet=rs0&ssl=false"

# 데이터 추가
> db.testcollection.insertMany(
	 [
	  { region: "us-central1" },
	  { name: "test2", status: "running" },
	  { name: "test3", status: "running", region: "us-central1" },
	  { status: "pending", resource: "abc" },
	  { name: "test2", status: "running" },
	 ]
	)
```

![3-1](/assets/img/doik2/3-1.png)

⬇️

![3-2](/assets/img/doik2/3-2.png)


#### 🚧 해당 테스트를 진행하며 마주했던 오류들
오류1
Warning  KEDAScalerFailed      31s (x14 over 73s)  scale-handler  failed to ping mongoDB, because of connection() error occurred during connection handshake: auth error: sasl conversation error: unable to authenticate using mechanism "SCRAM-SHA-1": (AuthenticationFailed) Authentication failed.

▶️  user 생성시 mechanism 옵션 추가 [🔗](https://www.mongodb.com/docs/manual/reference/method/db.createUser/)


---
참고
- [https://www.percona.com/doc/kubernetes-operator-for-psmongodb/kubernetes.html](https://www.percona.com/doc/kubernetes-operator-for-psmongodb/kubernetes.html)
- [https://www.percona.com/doc/kubernetes-operator-for-psmongodb/users.html](https://www.percona.com/doc/kubernetes-operator-for-psmongodb/users.html)
- [https://keda.sh/docs/2.7/scalers/mongodb/](https://keda.sh/docs/2.7/scalers/mongodb/)
- [https://www.mongodb.com/docs/manual/reference/method/db.createUser/](https://www.mongodb.com/docs/manual/reference/method/db.createUser/)
- [https://www.mongodb.com/docs/manual/reference/connection-string/#dns-seed-list-connection-format](https://www.mongodb.com/docs/manual/reference/connection-string/#dns-seed-list-connection-format) 
