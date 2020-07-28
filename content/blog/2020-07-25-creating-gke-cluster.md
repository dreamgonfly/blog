---
title: "GKE 클러스터 생성하기"
date: 2020-07-28T16:13:00+09:00
slug: "creating-gke-cluster"
description: "Creating GKE cluster using console"
keywords: ["kubernetes", "gcp"]
draft: false
tags: ["kubernetes", "gcp"]
math: false
toc: true
---

GKE(Google Kubernetes Engine)는 Google Cloud Platform이 제공하는 managed Kubernetes 서비스입니다. 이 글에서는 GKE에서 새 클러스터를 생성하는 방법을 단계 별로 알아보겠습니다. 그 전에 먼저, GKE가 다른 Kubernetes-as-a-Service인 AWS의 EKS, Azure의 AKS에 비해서 갖는 특징부터 살펴볼게요.

## GKE vs. EKS vs. AKS

이 절의 목적은 세 서비스의 모든 면을 상세히 비교하는 것이 아닙니다. 다른 서비스에 비해 GKE가 갖는 특징적인 부분 위주로 살펴보겠습니다. 제 주관적인 의견이 포함되어 있을 수 있습니다.

### Made by Google

Kubernetes는 Google이 만든 오픈 소스 프로젝트입니다. GCP 역시 Google이 제공하는 클라우드 컴퓨팅 서비스이죠. 그렇기 때문에 다른 클라우드 서비스와 비교해서 GCP의 GKE는 조금 특별한 점을 갖습니다.

이 특징이 가장 잘 드러나는 점이 GKE의 릴리즈 날짜입니다. GKE는 2015년 8월에 처음 GA(General Availability)로 릴리즈되었습니다. EKS와 AKS가 모두 2018년에 릴리즈된 것과 비교하면 매우 이른 시기입니다. 참고로 Kubernetes가 1.0 버전을 릴리즈한 것이 2015년 7월입니다. 즉, GKE는 Kubernetes 1.0 릴리즈 이후 한 달만에 출시된 것입니다. Kubernetes와 GCP 모두 Google의 프로젝트이기 때문에 가능했던 일일 것입니다.

### GCP is GKE-first

GKE는 GCP에 있는 유일한 container orchestration 서비스로서 중요한 의미를 갖습니다. 다른 GCP 서비스들도 orchestration이 필요하다면 GKE와의 연동을 디폴트로 지원하죠. 이에 비해 AWS는 자체적으로 container orchestration 서비스인 ECS를 갖고 있습니다. 그래서 그런지 여러 면에서 AWS가 Kubernetes에 기반한 EKS보다 자사가 만든 ECS를 밀어주려고 한다는 인상을 지우기가 힘듭니다. AWS의 다른 서비스에서 ECS를 먼저 지원하거나 더 안정적으로 지원해주곤 하기 때문입니다. (이 글을 쓰는 현재 AWS Fargate Spot 서비스는 ECS만 지원하며 EKS는 지원하지 않습니다. 언제 EKS를 지원할지도 아직 불명확합니다.)

### Automatic upgrade

GKE는 다른 서비스들과 다르게 클러스터 Kubernetes 버전의 automatic upgrade를 지원합니다. 즉, automatic upgrade를 선택했다면 새로운 버전이 릴리즈될 때 GKE가 스스로 master와 node pool의 버전을 올리고 맞춰주는 것입니다. Automatic upgrade는 GKE를 생성할 때 Release Channel을 선택해서 활성화할 수 있습니다.

### 1 free cluster

GKE에서 1개의 zonal cluster (zone 하나에 있는 클러스터)까지는 무료입니다. (Azure의 AKS도 마찬가지로 무료인 것으로 알고 있습니다.) 이에 비해 AWS의 EKS는 시간당 0.1달러로 한달에 72달러의 비용을 내야합니다. 

## Creating GKE cluster

이 절에서는 GKE 클러스터를 생성하는 과정은 스크린샷과 함께 알아보겠습니다. 설명할 필요가 있는 부분은 설명 덧붙여 놓았습니다.

GCP에서 project를 생성하고 GKE 기능을 enable하고 나서, Create cluster 버튼을 클릭하면 아래 화면으로 진입할 수 있습니다.

### Cluster basics

![cluster-basics](/images/creating-gke-cluster/cluster-basics.png)

#### Location type

Zonal location type은 zone 이라는 Google Cloud의 데이터 센터에 master node가 하나 존재하는 형태입니다. Cluster master의 고가용성(High Availability)를 보장하지 않습니다. 이에 비해 Regional location type은 고가용성을 보장하기 위해 region 안에 zone 별로 하나씩 여러 개의 master node를 생성합니다.

#### Multi-zonal cluster vs. Regional cluster

Default node locations를 여러개 지정해서 multi-zonal cluster를 구성할 수도 있습니다. Multi-zonal cluster와 regional cluster는 무엇이 다를까요? Multi-zonal cluster는 master가 하나의 zone에만 존재하며 node는 여러 zone에 같은 개수로 존재합니다. Node들의 고가용성은 보장하지만 cluster master node의 고가용성을 보장하지 않는 것입니다. 이에 비해 regional cluster는 master node가 여러 zone에 위치합니다.

#### Release channel

Release channel을 선택하면 Kubernetes 버전의 automatic upgrade를 지원받을 수 있습니다. Rapid channel은 오픈 소스 Kubernetes가 GA(General Availability)된 뒤 몇 주 안에 릴리즈됩니다. Regular channel은 Rapid가 업데이트된 후 2~3개월 후에 릴리즈되며, Stable channel은 Regular보다 다시 2~3월 후에 문제가 없다는 것이 확인된 후 릴리즈됩니다. 

### Node pool details

![node-pools-details](/images/creating-gke-cluster/node-pools-details.png)

#### Node pool

Node pool은 같은 종류의 노드의 집합입니다. EKS의 node group에 해당합니다. 

#### Surge upgrade

Surce upgrade는 node의 Kubernetes version 업그레이드 시 node를 몇개까지 더 늘이고 줄일 수 있는지를 결정합니다. Max surge가 1이라는 말은 추가로 1개 node를 더 생성할 수 있다는 뜻입니다. Max unavailable이 0이라는 말은 사용 가능한 node 수가 업그레이드 전의 node 수보다 더 적게 되지는 않도록 하겠다는 뜻입니다. 즉, max surge가 1이고 max unavailable이 0이면 GKE는 새 버전의 node 1개를 추가한 뒤 하나의 node를 종료하고, 다시 새 버전의 node 1개를 추가하기를 반복하면서 node를 업그레이드 하게 됩니다.

### Node pool nodes

![node-pools-node](/images/creating-gke-cluster/node-pools-node.png)

![node-pools-node-networking](/images/creating-gke-cluster/node-pools-node-networking.png)

#### Machine type

현재 가장 저렴하면서도 Kubernetes cluster를 감당할 수 있는 machine type은 general-purpose N1 Series g1-small (1 vCPU, 1.7GB memory)입니다. 

#### Boot disk size

저렴하면서도 실용적으로 쓸 수 있는 boot disk 크기는 32기가 정도입니다.

#### Enable preemptible nodes

Preemptible nodes란 최대 24시간까지만 지속되는 인스턴스입니다. 이 옵션을 체크하면 cluster에서 preemptible nodes를 사용할 수 있습니다.

Security와 metadata 등의 옵션은 필요한 경우 설정하세요. 대부분의 경우 디폴트로 남겨두어도 좋습니다.

### Automation

![automation](/images/creating-gke-cluster/automation.png)

#### Maintenance window

Maintenance window를 설정하지 않았을 때 GKE는 Kubernetes version upgrade를 어느 때나 진행할 수 있습니다. Maintenance window는 auto upgrade를 진행할 시간을 지정할 수 있는 기능입니다. 트래픽이 가장 적은 새벽 시간 등으로 설정해 두시면 좋습니다.

#### Node auto-provisioning

Node auto-provisioning은 자동으로 새로운 node pool을 생성하고 삭제하는 기능입니다. 이 기능을 활성화하지 않으면 GKE는 미리 정해진 node pool 중에서만 새 node를 생성합니다. 하지만 node auto-provisioning 기능을 사용하면 새로운 node pool을 GKE가 직접 생성할 수 있습니다.

### Networking

![networking](/images/creating-gke-cluster/networking.png)

#### Private cluster

Private cluster 내 node들은 public ip address를 갖지 않고 public internet과 연결되지 않습니다. 따라서 inbound, outbound 연결을 할 수 없습니다. Private cluster 내 특정 node에 outbound internet access를 주고 싶다면 NAT 등을 통해야 합니다.

#### VPC-native traffic routing

VPC-native traffic routing을 활성화하면 Kubernetes의 네트워크 구조를 GCP의 VPC와 자동으로 연동합니다. 따라서 NAT 없이도 GKE를 다른 GCP 서비스와 바로 연결할 수 있게 됩니다. 다른 GCP 서비스들과 internal IP로 편리하게 통신할 수 있도록 하기 위해서 이 기능을 활성화하는 것이 좋습니다.

### Features

![features](/images/creating-gke-cluster/features.png)

#### Cloud Operations

Cloud Operations는 Google Cloud 어플리케이션을 로깅 및 모니터링하는 서비스입니다. 과거 Stackdriver라고 불리다가 Cloud Operations로 이름이 바뀌었습니다. 활성화해두시면 대시보드 등으로 클러스터를 쉽게 모니터링할 수 있습니다.

## Connecting to GKE cluster

클러스터를 만들었으면 써보아야겠죠. 생성된 클러스터를 로컬 kubectl과 연결하는 방법은 다음과 같습니다.

```bash
gcloud init  # Account, project, default region을 설정합니다.
gcloud container clusters get-credentials {cluster-name} --zone {zone-name} --project {project-name}

```

위 커멘드를 실행하면 kubeconfig에 새 context 정보가 기록되며 kubectl로 GKE cluster에 접근할 수 있게 됩니다.

## Run a pod

연결된 클러스터 샘플 어플리케이션을 실행해 보고 싶다면 다음 커멘드로 가능합니다.

```
kubectl run hello-server --image gcr.io/google-samples/hello-app:1.0 --port 8080
```

## References

- https://stackoverflow.com/questions/57215726/google-cloud-gke-multi-zone-cluster-vs-regional-clusters
- https://cloud.google.com/kubernetes-engine/docs/how-to/alias-ips
- https://cloud.google.com/kubernetes-engine/docs/concepts/private-cluster-concept
- [https://medium.com/@jwlee98/gcp-gke-차근-차근-알아보기-1탄-gke-개요](https://medium.com/@jwlee98/gcp-gke-차근-차근-알아보기-1탄-gke-개요-382dc69b2ec4)
- [https://medium.com/@jwlee98/gcp-gke-차근-차근-알아보기-2탄-gke-서비스-및-확장-해보기](https://medium.com/@jwlee98/gcp-gke-차근-차근-알아보기-2탄-gke-서비스-및-확장-해보기-5c9b137e72c8)
- https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl#using-gcloud-init

