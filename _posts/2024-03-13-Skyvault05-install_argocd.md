---
layout: post
title: "Kubernetes에 Ingress Controller 설치하기"
description: "Kubernetes에 Ingress Controller 설치하기"
date: 2024-03-13
feature_image: images/road.jpg
tags: [kuberntes, ArgoCD, linux]
---
## ArgoCD 설치

### 설치 환경

- Ubuntu 22.04.3
- Kubernetes 1.29.2

이미 배포된 Kubernetes Cluster에 CI/CD를 위한 ArgoCD를 설치해보자!   

Menifest를 통해 설치할 계획이며 Service는 NodePort Type을 사용해 외부에 노출할 예정.

<!--more-->

### 설치 목록

- ArgoCD

### ArgoCD 설치
1. namespace생성, ArgoCD설치.

   ```bash
   $ kubectl create namespace argocd
   $ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```

2. ArgoCD CLI 설치.

   GUI를 사용하더라도 초기 비밀번호를 얻으러면 ArgoCD CLI를 설치해야 한다...
   ```bash
   $ curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
   $ sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
   ```

   설치 확인
   ```bash
   $ argocd
   ```

   설치파일 삭제
   ```bash
   $ rm argocd-linux-amd64
   ```

3. GUI로 접속하기 위해 일단 포트포워딩 설정
   임시로 접속해보기 위해 포트포워딩으로 설정하는 것이고 나중에 NodePort Type의 서비스로 교체할 예정. 포트포워딩은 재시작시 설정이 사라지므로 유의.

   ```bash
   $ kubectl port-forward --address <ip> svc/argocd-server -n argocd 8080:443
   ```

4. ArgoCD 초기 비밀번호 찾기
   ```bash
   $ argocd admin initial-password -n argocd
   ```

5. GUI 로그인
   
   설치된 Node의 IP:8080 으로 브라우저에서 접속    

   ID: admin   
   PWD: 초기 비밀번호   
   ![image-20240313213002944](images\image-20240313213002944-1710596552560-1.png)
   
6. 비밀번호 변경
   초기 비밀번호 확인 시 강조해서 비밀번호 바꾸기를 권장한다...   
   왼쪽 메뉴에서 User Info -> Update Password 를 누르면 아래와 같은 화면이 나온다. Current Password는 아까 확인한 기본 비밀번호를 입력하면 되고, New Password와 Confirm New Password에는 변경할 비밀번호를 입력한 후 Save New Password를 누르면 반영된다.
   ![image-20240313213440472](images\image-20240313213440472.png)

7. ArgoCD를 Service로 노출
   argocd-server Service의 type을 NodePort로 변경

   ```bash
    $ kubectl get service -n argocd
    NAME                                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
    argocd-applicationset-controller          ClusterIP   10.102.124.16    <none>        7000/TCP,8080/TCP            39m
    argocd-dex-server                         ClusterIP   10.96.225.83     <none>        5556/TCP,5557/TCP,5558/TCP   39m
    argocd-metrics                            ClusterIP   10.108.143.143   <none>        8082/TCP                     39m
    argocd-notifications-controller-metrics   ClusterIP   10.109.122.212   <none>        9001/TCP                     39m
    argocd-redis                              ClusterIP   10.106.144.25    <none>        6379/TCP                     39m
    argocd-repo-server                        ClusterIP   10.110.180.61    <none>        8081/TCP,8084/TCP            39m
    argocd-server                             ClusterIP   10.98.250.254    <none>        80/TCP,443/TCP               39m
    argocd-server-metrics                     ClusterIP   10.96.102.237    <none>        8083/TCP                     39m
    $ kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
    service/argocd-server patched
    $ kubectl get service -n argocd
    NAME                                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
    argocd-server                             NodePort    10.98.250.254    <none>        80:30907/TCP,443:31144/TCP   40m
   ```

   이후 이상이 없으면 argocd-server의 포트(현재 30907)로 접속이 된다.

   ![image-20240313213122441](images\image-20240313213122441.png)
   **NodePort**는 고정포트(위의 80:30907/TCP,443:31144/TCP)로 각 노드의 IP를 통해 서비스를 노출시킨다. 다만 포트 범위가 30000-32767 사이로 지정되고, 기본 IP는 Control-Plane으로 지정되기에, 클러스터 외부에서 접근하려면 [Control-Plane IP:NodePort]로 접속해야한다.