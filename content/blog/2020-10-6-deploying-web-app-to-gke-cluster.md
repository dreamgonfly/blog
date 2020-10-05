---
title: "GKE에 웹 어플리케이션 배포하기"
date: 2020-10-05T16:23:00+09:00
slug: "deploying-web-app-to-gke-cluster"
description: "Deploying a web application to GKE cluster"
keywords: ["kubernetes", "gcp"]
draft: false
tags: ["kubernetes", "gcp"]
math: false
toc: true
---

배포(deployment)는 개발의 마지막 단계로 코드가 서비스로 바뀌는 순간이라고 할 수 있습니다. [지난 글](https://dreamgonfly.github.io/blog/creating-gke-cluster/)에서는 GKE 클러스터를 생성하는 방법을 알아보았습니다. 이를 이어서 이번 글에서는 GKE를 이용하여 간단한 웹 어플리케이션의 개발부터 배포까지 전 과정을 코드로 살펴보겠습니다. 구체적으로 Python, Docker, Github packages, Kubernetes service와 ingress, https를 위한 managed certificate 설정 방법을 다룹니다.

## Web application

이 글에서 사용할 예시로 Python FastAPI를 이용한 웹 어플리케이션을 만들어 보겠습니다. [FastAPI](https://fastapi.tiangolo.com/) 는 flask와 비슷한 인터페이스를 가졌지만 더 빠르고, 타입 기반이고, 문서화가 잘 되어 있는 Python 웹 프레임워크입니다. GKE에 배포하는 웹 어플리케이션이 어떤 언어나 프레임워크를 썼는지가 중요하지는 않으므로 이 부분은 얼마든지 바꾸어도 됩니다.

`api.py`

```python
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
async def root():
    return {"message": "Hello World"}
  
@app.get("/ping")
async def root():
    return "pong"
```

코드에서 엔드포인트를 두 개 만들었습니다. 이 중 /ping 엔드포인트는 health check용입니다. Kubernetes와 GCP의 Load balancer에서 어플리케이션이 정상 작동하는지 확인하는 데 이 엔드포인트를 사용할 것입니다.

> Health check용 엔드포인트 맨 뒤에 슬래시(/) 여부는 중요합니다. Health check를 성공하려면 response의 http status code가 200이어야 하는데 만약 엔드포인트가 `/ping/` 이고 health check를 `/ping` 으로 수행한다면 307 temporary redirect로 응답하게 되어 health check가 실패한 것으로 간주되기 때문입니다.

## Docker image

위에서 만든 어플리케이션을 컨테이너 이미지로 빌드합니다.

`server.Dockerfile`

```dockerfile
FROM python:3.7-stretch

WORKDIR /app

RUN apt-get clean \
    && apt-get -y update

RUN pip install --no-cache-dir --upgrade pip
RUN pip install --no-cache-dir fastapi==0.60.1 uvicorn==0.11.8

COPY . .

ENV LANG C.UTF-8

CMD [ "uvicorn", "api:app", "--host", "0.0.0.0", "--port", "8000"]
```

```
docker build . --file server.Dockerfile --tag server:latest
```

## Github Packages (or Container Registry)

쿠버네티스에 컨테이너 이미지를 배포하기 위해서는 먼저 쿠버네티스가 참조할 수 있는 컨테이너 레지스트리에 이미지가 등록되어 있어야 합니다. 컨테이너 레지스트리는 DockerHub일 수도 있고, GCR(Google cloud Container Registry)일 수도 있습니다. 여기에서는 [Github Packages](https://github.com/features/packages)를 사용하겠습니다.

> Github Packages는 최근에 Github Container Registry로 대체되었습니다. 하지만 전반적인 개념은 동일하므로 Github Packages에서 사용했던 명령어를 그대로 사용하겠습니다. Github Container Registry는 docker.pkg.github.com 대신 ghcr.io를 레지스트리 서버로 사용합니다.

### Personal access token (PAT) 생성하기

Personal access token은 비밀번호를 대체하여 사용자를 인증합니다. 

Github Settings > Developer settings > Personal access tokens에서 생성할 수 있습니다.

![personal_access_tokens_tab](/images/deploying-web-app-to-gke-cluster/personal_access_tokens_tab.png)

### Docker login

위에서 생성한 토큰을 GITHUB_TOKEN.txt에 저장하고 아래 명령어로 로그인합니다.

```bash
$ cat ~/GITHUB_TOKEN.txt | docker login https://docker.pkg.github.com -u USERNAME --password-stdin
```

### Tag

```bash
$ docker tag IMAGE_ID docker.pkg.github.com/OWNER/REPOSITORY/IMAGE_NAME:VERSION
```

#### Publish

```bash
$ docker push docker.pkg.github.com/OWNER/REPOSITORY/IMAGE_NAME:VERSION
```

## Secret for registry credential

쿠버네티스가 private한 컨테이너 레지스트리에서 컨테이너 이미지를 가져올 수 있으려면 인증이 필요합니다. 이를 위해 `regcred` 라는 이름의 Secret을 생성하겠습니다. 이 Secret을 Pod마다 마운트하여 컨테이너 이미지를 가져오게 됩니다.

```
kubectl create secret docker-registry regcred --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>
```

Docker-server에는 `docker.pkg.github.com` 라고 적으면 됩니다.

## Deployment

컨테이너 이미지를 publish했으니 이제 배포할 차례입니다. 위에서 Publish한 도커 이미지를 사용하여 쿠버네티스 디플로이먼트를 만듭니다.

`deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-deployment
  labels:
    app: example
spec:
  replicas: 1
  selector:
    matchLabels:
      app: example
  template:
    metadata:
      labels:
        app: example
    spec:
      containers:
        - name: example-container
          image: "REGISTRY/OWNER/REPOSITORY/IMAGE_NAME:VERSION"
          env:
            - name: ENVIRONMENT
              value: latest
          ports:
            - containerPort: 8000
              protocol: TCP
          readinessProbe:
            httpGet:
              path: /ping
              port: 8000
            initialDelaySeconds: 3
            periodSeconds: 15
      imagePullSecrets:
        - name: regcred
```

```bash
$ kubectl apply -f deployment.yaml
```

- 어플리케이션이 8000 포트로 서빙하므로 containerPort 8000번을 열어줍니다.
- readinessProbe 설정에 따라 8000번 포트에  어플리케이션이 서비스할 준비가 된 상태인지 판
- imagePullSecrets를 통해 위에서 만든 regard secret을 통해 컨테이너 레지스트리에서 이미지를 가져올 수 있도록 인증합니다.

## Service

어플리케이션이 배포되었지만 아직 외부에서 접근 가능한 상태는 아닙니다. 이를 가능케하기 위해 Service를 생성해 줍니다.

Ingress와 연결하기 위해서는 반드시 NodePort로 타입을 지정해야 합니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: example-service
  labels:
    app: example
spec:
  type: NodePort
  ports:
    - port: 8000
      targetPort: 8000
      protocol: TCP
  selector:
    app: example
```

## Ingress

쿠버네티스 Ingress는 여러 서비스 앞단에서 "스마트 라우터" 역할을 할 수 있는 쿠버네티스 객체입니다. Ingress를 이용해 하나의 IP 주소에서 각기 다른 path를 다른 서비스에 연결할 수도 있습니다.

아래는 example.com이라는 도메인 이름으로 접속한 연결의 모든 path를 example-service로 연결하는 예시입니다. 디폴트 백엔드 역시 example-service로 설정되었습니다.

`ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: example-ingress
spec:
  backend:
    serviceName: example-service
    servicePort: 8000
  rules:
  - host: example.com
    http:
      paths:
      - path: /*
        backend:
          serviceName: example-service
          servicePort: 8000
```

GKE에서 ingress는 여러 GCP 리소스를 생성합니다. Ingress가 생성하는 리소스의 목록을 잠시 살펴보면 아래와 같습니다.

> - A forwarding rule and IP address.
> - Compute Engine firewall rules that permit traffic for load balancer health checks and application traffic from Google Front Ends or Envoy proxies.
> - A target HTTP proxy and a target HTTPS proxy, if you configured TLS.
> - A URL map which with a single host rule referencing a single path matcher. The path matcher has two path rules, one for `/*` and another for `/discounted`. Each path rule maps to a unique backend service.
> - NEGs which hold a list of Pod IPs from each Service as endpoints. These are created as a result of the `my-discounted-products` and `my-products` Services. The following diagram provides an overview of the Ingress to Compute Engine resource mappings.

![gke-ingress-mapping](/images/deploying-web-app-to-gke-cluster/gke-ingress-mapping.svg)



## Credential

마지막으로 서비스에 HTTPS로 접근하기 위해 인증서 설정 단계가 남았습니다. 이 단계를 진행하기 위해서는 먼저 두가지 조건이 필요합니다. 이 글에서 이 두 가지는 생략했습니다.

- ingress에 연결된 IP 주소를 static ip 주소로 설정해야 합니다. 이는 GCP 콘솔에서 IP addresses 서비스를 통해 할 수 있습니다.
- IP 주소가 도메인 이름에 연결되어 있어야 합니다. 도메인을 구입한 뒤 설정을 통해 IP를 연결하면 됩니다.

위 두 조건이 만족되었다는 전제 하에서 Google-managed SSL 인증서를 생성해 봅시다. 이 인증서는 도메인 검증 (DV, Domain Validation) 인증서로 구글이 생성하고 갱신하며 관리해 주는 인증서입니다.

ManagedCertificate는 v1beta2 API를 따릅니다. GKE 클러스터 버전 1.18.9 이후부터는 v1 API에서도 사용 가능하다고 합니다.

`certificate.yaml`

```yaml
apiVersion: networking.gke.io/v1beta2
kind: ManagedCertificate
metadata:
  name: example-certificate
spec:
  domains:
    - example.com
```

Ingress 설정에 IP 주소와 certificate 이름을 넣어줍니다.

`ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    kubernetes.io/ingress.global-static-ip-name: example-ip-name
    networking.gke.io/managed-certificates: example-certificate
spec:
  backend:
    serviceName: example-service
    servicePort: 8000
  rules:
  - host: example.com
    http:
      paths:
      - path: /*
        backend:
          serviceName: example-service
          servicePort: 8000
```

이것으로 모든 준비는 끝났습니다. 이제 인증서가 발급되기를 기다리는 일만 남았습니다. 처음 인증서를 설정했을 때는 인증서 상태가 Provisioning으로 뜹니다. 일정 시간(약 30분 이상)이 지난 후에 이 상태가 Active로 바뀌면 설정이 완료된 것입니다.

```bash
$ kubectl describe managedcertificate example-certificate
```

- 인증서 발급 중

![provisioning](/images/deploying-web-app-to-gke-cluster/provisioning.png)

- 인증서 발급 완료

![active](/images/deploying-web-app-to-gke-cluster/active.png)

## References

- GKE Ingress for HTTP(S) Load Balancing: https://cloud.google.com/kubernetes-engine/docs/concepts/ingress
- Kubernetes NodePort vs LoadBalancer vs Ingress? When should I use what?: https://medium.com/google-cloud/kubernetes-nodeport-vs-loadbalancer-vs-ingress-when-should-i-use-what-922f010849e0
- Using Kubernetes Port, TargetPort, and NodePort: https://www.bmc.com/blogs/kubernetes-port-targetport-nodeport/
- Using Google-managed SSL certificates: https://cloud.google.com/kubernetes-engine/docs/how-to/managed-certs