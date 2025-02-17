+++
title = "HA 쿠버네티스 클러스터 구성"
date = 2025-02-07
description = "tailscale을 이용한 HA 구성"
[taxonomies]
categories = ["k8s"]
tags = ["tailscale", "k8s"]
[extra]
lang = "ko"
toc = true
+++

control-plane 3대와 worker-node 1대로 이루어진 *k8s HA 클러스터를 구성* 해보았다.

+ [HA](https://kubernetes.io/ko/docs/setup/production-environment/tools/kubeadm/ha-considerations) : 쿠버네티스 시스템의 안정성과 지속정을 위해 3개 이상의 다중 contorl-plane 노드를 구성하여, 하나의 노드에 장애가 발생하더라도 다른 노드가 작업을 이어받아 운영을 지속할 수 있게 하는 구성이다.
+ [Control Plane](https://kubernetes.io/docs/concepts/overview/components/#control-plane-components) : 전체적인 쿠버네티스 시스템을 제어하는 노드.

## 구성 환경

- orangepi5 2대, rpi4B 1대 (Armbian)
- VM 1대 (Debian)
총 4대의 서버로 구성된 쿠버네티스 클러스터를 구성하였다.

사용하고 있는 버전은 다음과 같다.
- containerd v1.6.20
- kubeadm v1.32.0
- kubectl v1.32.0
- kubelet v1.32.0

## Tailscale
Tailscale VPN을 이용하여 네트워크 레벨의 HA를 구성하도록 계획했다.
Tailscale의 Subnet Router 기능을 이용하면 Failover 정책 기반의 HA를 간편하게 구성할 수 있어 이를 사용하기로 했다.
Tailscale을 이용하면 로드밸런싱이 지원되지 않는다는 단점이 생기지만, k8s API가 트래픽을 많이 받는 종류의 서비스는 아니라고 판단하여 치명적인 단점이 아니라고 판단했다.

## kubeadm
[kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm)을 이용해 k8s 클러스터를 구성하였다.
kubeadm은 가장 표준적인 k8s인 배포 툴로서, 온프레미스 환경에 적합하고 표준 k8s 기능을 모두 지원하며 CNI 선택이나 클러스터 설정이 자유로워 사용하기 적합하다고 판단했다.
k0s나 Kubespray 등도 고려해 보았지만 커스터마이징이 자유롭지 않거나 Ansible에 대한 깊은 이해가 요구되는 등의 단점이 크게 느껴져 이번 구성에는 사용하지 않았다.

## 과정

### 0. 환경 설정
참고 문서 : [kubernetes-Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm)

설치 전 k9s API에서 사용할 IP/서브넷 값들을 선택해야 한다. 각 값들은 서로 겹치지 않는 선에서 사설 IP 대역 내의 값을 임의로 선택하면 된다.

+ Control Plane Endpoint: Tailscale VPN 안에서 k8s API 엔드포인트를 띄우기 위한 IP이다. `10.200.0.1`을 사용하였다.
+ Service Subnet: k8s 클러스터 내부의 서비스들이 사용하는 가상 네트워크 서브넷으로, 기본값은 `10.96.0.0/12`이다. 기본값을 그대로 사용하였다.
+ Pod Subnet: k8s 클러스터 내부의 팟들이 사용하는 가상 네트워크 서브넷이다. `10.50.0.0/16`을 사용하였다.


### 1. Tailscale 설정
각 노드들을 한 네트워크로 연결하기 위해 Tailscale VPN을 활성화 한다.
`--accept-routes` 옵션을 통해 네트워크 내에 공유된 서브넷 라우터를 사용할 수 있도록 허용해야 한다.

```zsh
$ tailscale up --accept-routes
```

+ 기존에 다른 옵션들을 사용중이었다면 `--reset` 옵션을 추가한다.

### 2-1. kubeadm init : 마스터 노드
master node를 위한 config는 `InitConfiguration`과 `ClusterConfiguration` 두 부분으로 나뉜다.

```yaml
# kubeadm_init_config.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: ${TAILSCALE_IP}
nodeRegistration:
  kubeletExtraArgs:
    - name: node-ip
      value: ${TAILSCALE_IP}
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
networking:
  serviceSubnet: ${SERVICE_SUBNET}
  podSubnet: ${POD_SUBNET}
controlPlaneEndpoint: "${CONTROL_PLANE_ENDPOINT}:6443"
```

- `TAILSCALE_IP` 값은 해당 커맨드를 실행하는 기기의 Tailscale 상의 IP를 입력하면 된다. `tailscale ip` 커맨드를 통해 확인할 수 있다.

#### a. InitConfiguration 구성요소
Control Plane 초기화 시 적용되는 설정이다.

- `localAPIEndpoint.advertiseAddress`: 해당 Control Plane 노드가 다른 노드들과 통신할 때 사용할 IP. 노드 간에 Tailscale을 사용하여 통신하기 위해 Tailscale 상의 IP로 설정한다.
- `nodeRegistration.kubeletExtraArgs`: kubelet 실행시 전달할 파라미터들을 설정한다. `node-ip` 옵션을 통해 노드가 사용할 IP 주소를 설정할 수 있는데, 노드 간에 Tailscale을 사용하여 통신하기 위해 Tailscale 상의 IP로 설정한다.

#### b. ClusterConfiguration 구성요소
클러스터 전체에 적용되는 설정들이다.

- `networking.serviceSubnet`, `podSubnet`: 위에서 선택한 Service Subnet과 Pod Subnet 값을 입력한다.
- `controlPlaneEndpoint`: Control Plane API에 접근하기 위한 엔드포인트를 설정한다. HA 구성을 위해 위에서 선택했던 IP를 입력한다.

### 2-2. kubeadm join: 컨트롤 플레인 노드

master node 외의 control-plane node들을 위한 `JoinConfiguration`을 작성한다.

```yaml
# kubeadm_join_config.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: JoinConfiguration
discovery:
  bootstrapToken:
    token: ${TOKEN}
    apiServerEndpoint: "${LOOPBACK_IP}:6443"
    caCertHashes:
      - "${CA_HASH}"
controlPlane:
  localAPIEndpoint:
    advertiseAddress: ${TAILSCALE_IP}
  certificateKey: ${CERT_KEY}
nodeRegistration:
  kubeletExtraArgs:
    - name: node-ip
      value: ${TAILSCALE_IP}
    # OS 환경에 따라 다음을 추가한다
    - name: resolv-conf
      value: /etc/resolv.conf
```

#### a. JoinConfiguration 구성요소
노드들을 기존 클러스터에 조인시킬 때 적용되는 설정들이다.

- `discovery` : 기존 컨트롤 플레인 노드를 찾는 방법과 인증 정보를 설정한다.
- `controlPlane` : 컨트롤 플레인 노드로 구성하기 위한 설정을 정의한다.
- `nodeRegisteration` : kubelet 관련 설정을 지정한다.

+ `caCertHashes` 값은 마스터 노드에서 `kubectl get csr`로 얻을 수 있다.
+ `token` 값은 마스터 노드에서 `kubeadm token create`로 얻을 수 있다.


### 2-3. kubeadm join: 워커 노드
앞서 컨트롤 플레인 노드 join 시 활용한 `JoinConfiguration` 설정에서 `controlPlane` 하위의 설정만 지워서 사용하면 된다.

### 3. Control Plane Endpoint 셋업

Control Plane 노드들에 한해, 위에서 선택한 Control Plane Endpoint가 실제로 동작할 수 있도록 하기 위해
해당 IP로 요청을 날렸을 때 패킷이 올바른 노드로 라우팅될 수 있도록 설정이 필요하다.

#### a. Control Plane Endpoint를 Loopback 주소로 설정
먼저 해당 IP를 Loopback 주소로 설정하여야 하는데, 루프백을 설정하면 해당 IP를 로컬에서 사용할 경우 자기 자신을, 루프백이 없는 곳에서 사용할 경우 Tailscale에 의해 라우팅된 노드를 바라보게 된다.
이때 배포판에 따라 다른 Network Manager가 사용되기에 설정 방법이 갈린다. 내가 가진 환경에 대해 이전에 정리해 놓은 짧은 노트가 있다.

연관글 : [Tailscale을 이용한 HA 구성](https://kyoku.dev/notes/tailscale-ha)

#### b. Tailscale Subnet Router 설정
Tailscale 네트워크 내 다른 노드들의 서브넷 라우팅 경로를 수락하지 않도록 하고, 현재 노드가 Control Plane Endpoint를 통해 라우팅될 수 있도록 한다.

```zsh
$ tailscale up --accept-routes=false --advertise-routes=${CONTROL_PLANE_ENDPOINT}/32
```

## 4. CNI 설치
CNI 플러그인으로 [cilium](https://cilium.io/)을 사용하였다. 별도의 시스템 구축 없이 모니터링하기 편리하고 설치가 간단해서 선택하게 되었다.
[cilium 문서](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/)를 참조 바람.

## 5. 설치 후 `kubectl` 커맨드가 정상적으로 실행되다가 Connection Refused를 반복하는 문제가 발생

### 원인 분석

OOM 문제, CORS 문제, loopback 문제 등 여러 가지 원인을 의심하였지만, containerd 설정이 적합하지 못하여 문제가 발생했다.


### containerd 설정 확인 및 수정
`containerd config dump | grep "SystemdCgroup" ` 명령어를 통해 containerd 런타임의 설정을 확인하였다.
SystemdCgroup 옵션이 활성화되어 있지 않다는 사실을 확인했고, 해결을 위해 containerd 설정을 수정하였다.
containerd 설정 파일 수정 후 `systemctl restart containerd` 명령어를 통해 containerd를 재시작하였다.

```toml
# /etc/containerd/config.toml
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true
```
