---
layout: post
title: "Kubernetes Ingress Controller 설치하기"
description: "인그레스 컨트롤러 설치하기"
date: 2024-03-16
feature_image: images/road.jpg
tags: [kuberntes, Ingress, Metallb, linux]
---
## Kubernetes 설치

### 설치 환경

- Ubuntu 22.04.3
- Kubernetes 1.29.2

Kubernetes 환경에서 배포한 Application의 HTTPS 통신에 필요한 인증서 관리와 Load Balancing을 위해 Ingress를 사용할 예정.   

On-premises 환경에선 Ingress를 사용하기 위해서는 Ingress Controller를 설치해야한다.

<!--more-->

### 설치 목록

- Ingress Controller
  Ingress Controller의 구현체로는 Ingress-Nginx를 사용할 예정. Nginx-Ingress와 혼동하지 않도록 유의.
- Metallb
  LoadBalancer Type의 Service로 Ingress Controller를 노출할 계획!  LoadBalancer 활성화 하기 위해서 설치.  
  On-premises에선 LoadBalacer를 사용할려면 구현체를 설치해줘야 한다 그래서 Metallb를 사용할 예정.

### Ingress Controller 설치

1. Manifest를 사용해 Ingress-Nginx를 설치.
   ```bash
   $ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/baremetal/deploy.yaml
   ```

2. 설치 확인
   ```bash
   $ kubectl get pods -n=ingress-nginx
   ```

### MetalLB 설치

Ingerss와 마찬가지로 LoadBalancer Type의 Service를 사용할 때도 구현체가 필요하다. 필자는 On-premises환경이라 MetalLB를 설치할 예정.   
Cloud환경에선 플랫폼에서 LB를 직접 제공하는 경우가 많다.

1. kube-proxy 설정을 다음과 같이 바꿔준다.   
   Kubernetes 1.14.2버전 이후로는 IPVS Mode로 kube-proxy를 사용중이면 StrictARP를 활성화 해줘야 한다.

   ```bash
   $ kubectl edit configmap -n kube-system kube-proxy
   ---------------------------------------------------
   apiVersion: kubeproxy.config.k8s.io/v1alpha1
   kind: KubeProxyConfiguration
   mode: "ipvs"
   ipvs:
     strictARP: true # 기본값이 false 이니 true로 바꿔준다.
   ```

2. Menifest를 사용해서 MetalLB 설치.
   ```bash
   $ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.3/config/manifests/metallb-frr.yaml
   ```

3. 설치확인.
   ```bash
   $ kubectl get pod -n metallb-system
   ```

4. IpAddressPool CR(사용자 리소스)를 만들어준다. spec은 원하는 IP 혹은 IP대역대로 적어주면 된다.
   ```bash
   # https://metallb.universe.tf/configuration/#defining-the-ips-to-assign-to-the-load-balancer-services에서 제공하는 예시
   # 파일명은 metallb-ipap.yaml로 정함
   apiVersion: metallb.io/v1beta1
   kind: IPAddressPool
   metadata:
     name: first-pool
     namespace: metallb-system
   spec:
     addresses:
     - 192.168.10.0/24
     - 192.168.9.1-192.168.9.5
     - fc00:f853:0ccd:e799::/124
   ```

   ```bash
   $ kubectl apply -n metallb-system -f metallb-ipap.yaml
   ```

5. L2Advertisement를 만들어준 후, 마찬가지로 yaml 파일을 적용시켜준다.
   ````bash
   # https://metallb.universe.tf/configuration/_advanced_l2_configuration/ 에서 제공하는 예시
   # 파일명은 metallb-l2.yaml로 정함
   apiVersion: metallb.io/v1beta1
   kind: L2Advertisement
   metadata:
     name: example
     namespace: metallb-system
   spec:
     ipAddressPools:
     - first-pool
   ````

   ```bash
   $ kubectl apply -n metallb-system -f metallb-l2.yaml
   ```

6. LoadBalancer Type의 Service를 만들어서 External-IP 할당이 잘 되는지 확인하면 된다.

