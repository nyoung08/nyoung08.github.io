---
layout: post
title: gpc) iam condition으로 
category: gcp
tags: gcp
---


사용자에게 프로젝트의 특정 인스턴스만 볼 수 있게 권한을 주려면, iam에서 condition으로 제한할 수 있다.


# 리소스와 이름으로 제한

명명규칙을 사용하고 있다면 해당 방법으로 특정 인스턴스에만 권한을 줄 수 있다.


``` 
# iam condition editor
# test로 시작하는 disk 혹은 instance

(
  resource.type == 'compute.googleapis.com/Disk' &&
    resource.name.startsWith('projects/{PROJECT_ID}/regions/us-central1/disks/test')) || (
  resource.type == 'compute.googleapis.com/Instance' &&
    resource.name.startsWith('projects/{PROJECT_ID}/zones/us-central1-a/instances/test'))||
(resource.type != "compute.googleapis.com/Disk" &&
resource.type != "compute.googleapis.com/Instance")

``` 

![1-0](/assets/img/gcp/iam_condition/1-0.png)

콘솔에서는 위의 이미지와 같이 입력하면 된다.


정말 딱 인스턴스만 볼 수 있도록 아래 세가지 권한만 부여하여 테스트하였다.

- compute.instances.get
- compute.instances.list
- compute.projects.get


![1-1](/assets/img/gcp/iam_condition/1-1.png)

최소한의 compute.instance에 대한 값만 줬기 떄문에 프로젝트명을 확인 할 수 없다.
이 때 콘솔도 url뒤에 파라미터 값을 고쳐서 들어갔다. 
최소 project list를 줘야 직접 프로젝트를 선택하고 들어갈 수 있을 것으로 보인다.

이름으로 제한을 걸었어도 위와 같이 모든 인스턴스가 보이지만,
막상 들어가면 아래처럼 권한이 없어 인스턴스를 볼 수 없다.

![1-2](/assets/img/gcp/iam_condition/1-2.png)



# 태그로 제한

태그를 사용할 경우 리소스 타입과 같이 쓰게 되면 조건 표현식에서 오류가 발생하여 tag의 key와 value를 써서 권한을 부여했다.

``` 
# iam condition editor
# tag가 env:dev인 instance

resource.matchTag("{org_num}/env", "dev") &&
resource.hasTagKey("{org_num}/env")

``` 

![1-3](/assets/img/gcp/iam_condition/1-3.png)

테스트 결과는 이름으로 부여했을 때와 같다.
전체 인스턴스 리스트는 확인 가능하고, 인스턴스 세부는 권한이 있는 것만 확인 가능하다.


---
참고
- [https://cloud.google.com/iam/docs/configuring-resource-based-access#resource-name-instance](https://cloud.google.com/iam/docs/configuring-resource-based-access#resource-name-instance)
- [https://cloud.google.com/iam/docs/conditions-attribute-reference#resource-tags](https://cloud.google.com/iam/docs/conditions-attribute-reference#resource-tags)
