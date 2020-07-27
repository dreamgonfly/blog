---
title: "GKE 클러스터 생성하기"
date: 2020-07-25T09:50:55+09:00
slug: "creating-gke-cluster"
description: "Creating GKE cluster using GUI & CLI"
keywords: ["kubernetes", "gke"]
draft: true
tags: ["kubernetes"]
math: false
toc: true
---

GKE(Google Kubernetes Engine)는 Google Cloud Platform이 제공하는 managed Kubernetes 서비스입니다. 이 글에서는 GKE가 다른 Kubernetes-as-a-Service인 AWS의 EKS, Azure의 AKS에 비해서 갖는 특징에 대해 알아보고, GKE에서 새 클러스터를 생성하는 방법을 단계 별로 알아보겠습니다.

## GKE vs. EKS vs. AKS

이 절의 목적은 세 서비스의 모든 면을 상세히 비교하는 것이 아닙니다. 다른 서비스에 비해 GKE가 갖는 특징적인 부분 위주로 살펴보겠습니다. 제 주관적인 의견이 포함되어 있을 수 있습니다.

### Made by Google

Kubernetes는 Google이 만든 오픈 소스 프로젝트입니다. GCP 역시 Google이 제공하는 클라우드 컴퓨팅 서비스이죠. 그렇기 때문에 다른 클라우드 서비스와 비교해서 GCP의 GKE는 조금 특별한 점을 갖습니다.

이 특징이 가장 잘 드러나는 점이 GKE의 릴리즈 날짜입니다. GKE는 2015년 8월에 처음 GA(General Availability)로 릴리즈되었습니다. EKS와 AKS가 모두 2018년에 릴리즈된 것과 비교하면 매우 이른 시기입니다. 참고로 Kubernetes가 1.0 버전을 릴리즈한 것이 2015년 7월입니다. 즉, GKE는 Kubernetes 1.0 릴리즈 이후 한 달만에 출시된 것입니다. Kubernetes와 GCP 모두 Google의 프로젝트이기 때문에 가능했던 일일 것입니다.

### GKE-first GCP

GKE는 GCP에 있는 유일한 container orchestration 서비스로서 중요한 의미를 갖습니다. 다른 GCP 서비스들도 orchestration이 필요하다면 GKE와의 연동을 디폴트로 지원하죠. 이에 비해 AWS는 자체적으로 container orchestration 서비스인 ECS를 갖고 있습니다. 그런데 여러면에서 AWS가 Kubernetes에 기반한 EKS보다 자사가 만든 ECS를 밀어주려고 한다는 인상을 지우기가 힘듭니다. AWS의 다른 서비스에서 ECS를 먼저 지원하거나 더 안정적으로 지원해주곤 하기 때문입니다. (이 글을 쓰는 현재 AWS Fargate Spot 서비스는 ECS만 지원하며 EKS는 지원하지 않습니다. 언제 EKS를 지원할지도 아직 불명확합니다.)

### Automatic upgrade

GKE는 다른 서비스들과 다르게 클러스터 Kubernetes 버전의 automatic upgrade를 지원합니다. 즉, automatic upgrade를 선택했다면 새로운 버전이 릴리즈될 때 GKE가 스스로 master와 node pool의 버전을 올리고 맞춰주는 것입니다.

Automatic upgrade는 GKE를 생성할 때 선택 가능하며 Release Channel이라는 것을 고를 수 있습니다. Release Channel에는 Rapid , Regular, Stable 세 종류가 있습니다. Rapid는 새 버전이 릴리즈하자마자 바로 업그레이드 하며, Regular는 그보다 2~3개월 늦게, Stable은 Regular보다 2~3개월 늦게 업그레이드가 되어 안정성을 보장합니다.

### 1 free cluster

GKE에서 1개의 zonal cluster (zone 하나에 있는 클러스터)까지는 무료입니다. (Azure의 AKS도 마찬가지로 무료인 것으로 알고 있습니다.) 이에 비해 AWS의 EKS는 시간당 0.1달러로 한달에 72달러의 비용을 내야합니다. 

## Creating GKE cluster

이 절에서는 GKE 클러스터를 생성하는 과정은 스크린샷과 함께 알아보겠습니다. 설명할 필요가 있는 부분은 설명 덧붙

우선 project를 생성하고 GKE 기능을 enable하였다고 가정합니다.

### Cluster basics

![cluster-basics](/Users/dreamgonfly/Codes/personal/blog/static/images/creating-gke-cluster/cluster-basics.png)

#### Location type

Zonal location type은 zone 이라는 Google Cloud 의 데이터 센터에 master node가 하나 존재하는 형태입니다. Regional location type은 고가용성을 보장하기 위해 region 안에 zone 별로 master node를 여러개 둡니다. Master 뿐만 아니라 node들도 

Regional location type을 선택하면 master



### Node pool details![node-pools-details](/Users/dreamgonfly/Codes/personal/blog/static/images/creating-gke-cluster/node-pools-details.png)





### Node pool nodes

![node-pools-node](/Users/dreamgonfly/Codes/personal/blog/static/images/creating-gke-cluster/node-pools-node.png)





### Node pool security

![node-pools-security](/Users/dreamgonfly/Codes/personal/blog/static/images/creating-gke-cluster/node-pools-security.png)

![node-pools-node-networking](/Users/dreamgonfly/Codes/personal/blog/static/images/creating-gke-cluster/node-pools-node-networking.png)





### Node pool metadata

Node pool metadata

![node-pools-metadata](/Users/dreamgonfly/Codes/personal/blog/static/images/creating-gke-cluster/node-pools-metadata.png)





### Automation

![automation](/Users/dreamgonfly/Codes/personal/blog/static/images/creating-gke-cluster/automation.png)







### Networking

![networking](/Users/dreamgonfly/Codes/personal/blog/static/images/creating-gke-cluster/networking.png)





### Security

![security](/Users/dreamgonfly/Codes/personal/blog/static/images/creating-gke-cluster/security.png)





### Metadata

![metadata](/Users/dreamgonfly/Codes/personal/blog/static/images/creating-gke-cluster/metadata.png)



### Features

![features](/Users/dreamgonfly/Codes/personal/blog/static/images/creating-gke-cluster/features.png)