---
layout: post
title: T101) http lb와 mig 구성
category: study
tags: study
---


- backend: 상태 파일을 gcs에서 관리하도록 백엔드 설정
- env1: 모듈을 불러와서 환경별 설정
- modules: 리소스들을 모듈로 정의
![1-1](/assets/img/t101/fin/1-1.png)

환경별 상태관리 방법은 지난 포스팅을 참고하면 된다.[🔗](https://nyoung08.github.io/study/2022/11/13/%ED%99%98%EA%B2%BD%EB%B3%84-%EC%83%81%ED%83%9C%EA%B4%80%EB%A6%AC%EB%A5%BC-%EC%9C%84%ED%95%9C-%EB%B0%B1%EC%97%94%EB%93%9C%EC%99%80-%EB%AA%A8%EB%93%88-%EC%82%AC%EC%9A%A9%EA%B8%B0/)



이번 졸업과제에서는 관리형 인스턴스 그룹에 http 로드밸런서를 붙이는 모듈을 작성하였다. [🔗](https://github.com/nyoung08/t101/tree/main/final)
중간과제에서 작성했던 것도 이렇게 설명을 붙이고 싶었는데, 지금보다 더 이해도가 없었어서 코드가 너무 엉망이라 못했었다. 그것도 꼭 다시 수정해둬야지 . . ! 


# modules


### VPC

```
# vpc 생성
resource "google_compute_network" "test_vpc" {
  name                    = var.vpcname
  project                 = var.project
  provider                = google-beta
  # 자동 생성시 모든 리전에 서브넷이 생성되기 때문에 커스텀으로 생성
  auto_create_subnetworks = false
}

# 인스턴스 그룹이 위치할 서브넷 생성
resource "google_compute_subnetwork" "test_subnet" {
  name          = var.subnetname
  project       = var.project
  provider      = google-beta
  ip_cidr_range = var.iprange
  region        = var.region
  network       = google_compute_network.test_vpc.id
}

# lb에 사용할 ip 예약
resource "google_compute_global_address" "test_ip4lb" {
  provider = google-beta
  project  = var.project
  name     = "ip4lb"
}

# health check 방화벽
resource "google_compute_firewall" "test_fw4hc" {
  name          = "fw4hc"
  provider      = google-beta
  project       = var.project
  direction     = "INGRESS"
  network       = google_compute_network.test_vpc.id
  # gcp health check 대역대
  source_ranges = ["130.211.0.0/22", "35.191.0.0/16"]
  allow {
    protocol = "tcp"
  }
  target_tags = ["allow-health-check"]
}
```


###LB

```
# lb 전달 규칙
resource "google_compute_global_forwarding_rule" "test_forwardingrule" {
  name                  = "test-forwardingrule"
  project               = var.project
  provider              = google-beta
  ip_protocol           = "TCP"
  # lb 타입 지정
  load_balancing_scheme = "EXTERNAL"
  port_range            = "80"
  target                = google_compute_target_http_proxy.test_httpproxy.id
  ip_address            = google_compute_global_address.test_ip4lb.id
}

# lb 이름
resource "google_compute_target_http_proxy" "test_httpproxy" {
  name     = var.lbname
  project  = var.project
  provider = google-beta
  url_map  = google_compute_url_map.test_urlmap.id
}

# lb url map
resource "google_compute_url_map" "test_urlmap" {
  name            = "test-url-map"
  project         = var.project
  provider        = google-beta
  # 기본적으로 뒤에 지정해둔 백엔드서비스로 가게 지정
  # 여러 패스로 구성하게 된다면 여기서 패스별 백엔드 지정이 가능하다.
  default_service = google_compute_backend_service.test_backend.id
}

# lb 백엔드 서비스
resource "google_compute_backend_service" "test_backend" {
  name                    = "test-backend-service"
  project                 = var.project
  provider                = google-beta
  protocol                = "HTTP"
  load_balancing_scheme   = "EXTERNAL"
  # 인스턴스 그룹 생성시 사용할 포트를 이름으로 매핑하고 해당 이름은 백엔드 서비스 연결시 사용한다.
  port_name               = "test-port" 
  timeout_sec             = 10
  enable_cdn              = false
  health_checks           = [google_compute_health_check.test_hc.id]
  backend {
    # 백엔드로 붙일 인스턴스 그룹
    # self_link가 아닌 전체 url이 들어가야하기 때문에 instance_group이 붙어야한다.
    group           = google_compute_instance_group_manager.test_mig.instance_group
    balancing_mode  = "UTILIZATION"
    capacity_scaler = 1.0
  }
}
```


### GCE

```
# 인스턴스 템플릿 생성
resource "google_compute_instance_template" "test_template" {
  name         = "test-mig-template"
  project      = var.project
  provider     = google-beta
  machine_type = var.machine_type
  tags         = ["allow-health-check"]

  network_interface {
    network    = google_compute_network.test_vpc.id
    subnetwork = google_compute_subnetwork.test_subnet.id
    access_config {
    }
  }
  disk {
    source_image = "debian-cloud/debian-10"
    auto_delete  = true
    boot         = true
  }

  # 확인 할 웹서버용 시작스크립트
  metadata = {
    startup-script = <<-EOF1
      #! /bin/bash
      set -euo pipefail

      export DEBIAN_FRONTEND=noninteractive
      apt-get update
      apt-get install -y nginx-light jq

      NAME=$(curl -H "Metadata-Flavor: Google" "http://metadata.google.internal/computeMetadata/v1/instance/hostname")
      IP=$(curl -H "Metadata-Flavor: Google" "http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/ip")
      METADATA=$(curl -f -H "Metadata-Flavor: Google" "http://metadata.google.internal/computeMetadata/v1/instance/attributes/?recursive=True" | jq 'del(.["startup-script"])')

      cat <<EOF > /var/www/html/index.html
      <pre>
      Name: $NAME
      IP: $IP
      Metadata: $METADATA
      </pre>
      EOF
    EOF1
  }
  lifecycle {
    create_before_destroy = true
  }
}

# health check
resource "google_compute_health_check" "test_hc" {
  name     = "test-hc"
  project  = var.project
  provider = google-beta
  http_health_check {
    port_specification = "USE_SERVING_PORT"
  }
}

# managed instance group 생성
# 비관리형 인스턴스 그룹은 google_compute_instance_group으로 생성가능하다.
resource "google_compute_instance_group_manager" "test_mig" {
  name     = var.migname
  provider = google-beta
  project  = var.project
  zone     = var.zone
  named_port {
    name = "test-port"
    port = 80
  }
  version {
    # 위에서 생성한 템플릿을 지정
    instance_template = google_compute_instance_template.test_template.id
    name              = "primary"
  }
  # 인스턴스 생성시 해당 이름은 베이스로 뒤에 랜덤한 값이 붙는다.
  base_instance_name = "vm"
  target_size        = var.instancecount
}
```



---
참고
- [https://cloud.google.com/load-balancing/docs/https/ext-http-lb-tf-module-examples](https://cloud.google.com/load-balancing/docs/https/ext-http-lb-tf-module-examples)
- [https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_instance_group_manager#instance_group](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_instance_group_manager#instance_group)
- [https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_backend_service](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_backend_service)
