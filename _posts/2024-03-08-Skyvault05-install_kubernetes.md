---
layout: post
title: "Kubernetes 설치하기"
description: "쿠버네티스 설치하기"
date: 2024-03-08
feature_image: images/road.jpg
tags: [kuberntes]
---
## Kubernetes 설치

### 설치 환경

- Ubuntu 22.04.3

VMWare를 사용해서 Control Plain과 Woker Node를 각각 다른 VM에 설치할 계획이다. 최종적으로는 Jenkins와 ArgoCD를 이용해서 CI/CD Pipeline을 구축하는 것이 목표.

<!--more-->

### 설치 목록

- Containerd 
  + Kubernetes 1.24이전 버전에선 Dockershim이 기본 Container Rumtime으로 기본 설치가 됐으나 이후 버전부터는 Container Rumtime을 따로 설치해줘야 하기 때문에 Docker에서 Container Rumtime으로 채택하고 있는 Containerd를 사용할 예정
- kubeadm
- kubectl
- kubelet

### 설치

1. Containerd 설치

   - Containerd 설치를 위해 Package Manager에 Repository추가.
     ```bash
     sudo apt-get update
     sudo apt-get install ca-certificates curl
     sudo install -m 0755 -d /etc/apt/keyrings
     sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
     sudo chmod a+r /etc/apt/keyrings/docker.asc
     
     # Add the repository to Apt sources:
     echo \
       "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
       $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
       sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
     sudo apt-get update
     ```

     

   - 설치
     ```bash
     sudo apt-get install containerd.io
     ```

   - 설치 확인
     ```bash
     systemctl status containerd
     ```

   - Containerd의 기본 설정파일 생성
     ```bash
     sudo rm /etc/containerd/config.toml
     containerd config default | sudo tee /etc/containerd/config.toml
     ```

   - cgroup driver를 systemd로 교체
     config.toml 파일의 SystemdCgroup 항목을 true로 변경

     ```bash
     sudo vim /etc/containerd/config.toml
     ```

     ```toml
     [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
                 SystemdCgroup = true
     ```

   - 설정파일 적용을 위한 재시작
     ```bash
     sudo systemctl restart containerd
     ```



2. Kubeadm, Kubelet, Kubectl 설치

   + swap memory 비활성화
     필자는 swap memory가 이미 비활성화 돼있었지만 활성화 돼있다면 /etc/fstab 비활성 화 할 수 있다.   
     swapoff 명령어로 다음 부팅 전 까지만 적용되니 /etc/fstab 파일을 수정할 것

   + 쿠버네티스 설치에 필요한 패키지들 설치. 기존 Containerd를 설치하면서 설치된 ca-certificates, curl은 제외.
     ```bash
     sudo apt-get install -y gpg apt-transport-https
     ```

   + Package Repository를 위한 퍼블릭 키 받아오기
     ```bash
     curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
     ```

   + Repository 추가
     ```bash
     echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
     ```

   + Kubeadm, Kubelet, Kubectl 설치
     ```bash
     sudo apt-get update
     sudo apt-get install -y kubelet kubeadm kubectl
     # 현재 버전 고정
     sudo apt-mark hold kubelet kubeadm kubectl
     ```

   + CNI 네트워크 활성화를 네트워크 설정
     ```bash
     sudo vim /etc/sysctl.conf
     ```

     아래와 같이 설정

     ```
     # 내용추가
     net.bridge.bridge-nf-call-ip6tables = 1 
     net.bridge.bridge-nf-call-iptables = 1
     # 주석제거
     net.ipv4.ip_forward=1
     net.ipv6.conf.all.forwarding=1
     ```

     ![image-20240309232322554](images\image-20240309232322554.png)

     ```bash
     sudo modprobe br_netfilter
     sudo sysctl -p
     ```

   + Control Plain 생성
     CNI의 구현체를 Flannel을 하기 위해  pod network cidr을 10.244.0.0/16로 설정

     ```bash
     sudo kubeadm init --pod-network-cidr=10.244.0.0/16
     ```

     Control Plain이 생성 완료되면 Kubectl 설정을 위한 명령어들과 Worker Node들의 Join을 위한 명령어(인증 토큰들 포함)이 나온다. 메세지를 저장해두는 것을 추천.

   + Kubectl과 Kubernetes api서버 통신을 위한 설정
     ```bash
     mkdir -p $HOME/.kube
     sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
     sudo chown $(id -u):$(id -g) $HOME/.kube/config
     ```

   + Flannel 설치
     ```
     kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
     ```

   + Control Plain 생성 확인
     ```bash
     kubectl get nodes
     kubectl get pod --namespace=kube-system -o wide
     ```

   # 작성중....

