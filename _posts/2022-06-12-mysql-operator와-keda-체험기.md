---
layout: post
title: [DOIK] keda로 mysql server 자동확장 체험기
category: study
tags: doik
---

지난 네트워크 스터디에 이어 데이터베이스 오퍼레이터 스터디도 참여하게 되었다. 오퍼레이터 자체를 처음 접해서 헷갈리는 것들이 있지만… 그럼에도 중간과제는 찾아온다..! 2주차에서 알게된 mysql operator와 3주차에 알게된 KEDA를 사용하여 mysql 서버 파드를 자동확장 테스트를 하였다.

사용할 mysql operator와 keda를 간략하게 소개하자면,


mysql operator는 쿠버네티스 클러스터 내 mysql innoDB 클러스터를 관리해주는 operator 이다.  

```
# mysql operator와 mysql-innodbcluster 설치
> helm repo add mysql-operator https://mysql.github.io/mysql-operator/
> helm install mysql-operator mysql-operator/mysql-operator --namespace mysql-operator --create-namespace
> helm install mycluster mysql-operator/mysql-innodbcluster --set credentials.root.password='sakila' --set tls.useSelfSigned=true --namespace mysql-cluster --create-namespace
```

![1-2](/assets/img/doik1/1-2.png)

mysql-operator 네임스페이스에는 deployment로 배포된 InnoDB 클러스터를 관리해주는 mysql-operator가 있다.


![1-3](/assets/img/doik1/1-3.png)

mysql-cluster 네임스페이스에 있는 리소스들이 InnoDB 클러스터 구성한다.
statefulset/mycluster : mysql server instance
deployment/mycluster-router : mysql router(proxy 역할)
service/mycluster : mysql router로 접근됨
service/mycluster-instances : 특정 서버 접근 시 사용
configmap/mycluster-initconf : mysql config



