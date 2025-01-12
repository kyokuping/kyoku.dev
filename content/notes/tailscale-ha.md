+++
title = "Tailscale을 이용한 HA 구성"
date = 2025-01-07
description = "Subnet Router와 loopback 어댑터를 활용한 HA 환경 구성"
[taxonomies]
categories = ["infra"]
tags = ["tailscale", "network"]
[extra]
lang = "ko"
toc = true
+++

여러 Control Plane 노드를 가지는 HA 쿠버네티스 클러스터 구성을 위해서는 노드들 간에 공유될 수 있는 단일 엔드포인트가 필요한데,
이를 위해 Tailscale의 Subnet Router 기능을 이용한 HA 환경을 구성했다.

## 왜 Tailscale을 사용하여 HA를 구성하였나?

현재 클러스터 구성 시 서버 간 네트워킹을 위해 Tailscale을 이용하고 있는데,
공식 문서가 권한 HA 구성 방식인 keepalived와 HAProxy를 활용한 셋업은 Tailscale 라우터가 지원하지 않는 VRRP 프로토콜을 사용하기 때문에 부적합했다.
대신 Tailscale 자체 기능을 활용해 HA를 구성하여 이 문제를 해결하고자 했다.

## 문제 상황
- 1개의 Debian과 3개의 Armbian 서버로 구성되어 있다.
- 모든 서버가 동일한 가상 IP로부터 요청을 받을 수 있어야 한다.

## 서버의 Network Manager 파악

가상 IP 설정을 위해, 각 서버들이 사용 중인 Network Manager를 확인해 보았다.
`ifupdown`, `netplan`, `wicd` 등등이 사용된다. `systemctl status ${네트워크 메니저}` 명령어를 사용하여 사용 중인 Network Manager를 확인하였다.
현재 환경 기준으로 Debian 서버는 `systemd-networkd`를 사용하고 있고, 다른 Armbian 서버들은 `NetworkManager`를 사용하고 있다.


## `systemd-networkd` 를 통한 loopback 설정

서버의 네트워크 설정을 수정하기 위해 다음 문서를 참고하여 `systemd-networkd`를 통해 loopback을 설정했다.
참고 문서: [`systemd-networkd` 설정](https://wiki.debian.org/SystemdNetworkd)

1. network device 설정을 위해 `/etc/systemd/network/lo1.netdev` 파일을 생성한다.
```ini
[NetDev]
Name=lo1
Kind=dummy
```
2. network link 설정을 위해 `/etc/systemd/network/lo1.network` 파일을 생성한다.
```ini
[Match]
Name=lo1

[Network]
Address=${사설_IP_대역대_내의_임의의_CIDR}
```
3. `networkctl reload` 명령을 실행하여 1, 2의 설정을 적용한 후, `networkctl status` 명령을 실행하여 loopback 설정을 확인한다.
![img](/assets/images/tailscale-ha/networkctl-status.png)

## `NetworkManager`를 통한 loopback 설정

서버의 네트워크 설정을 수정하기 위해 다음 문서를 참고하여 `NetworkManager`를 통해 loopback을 설정했다.
참고 문서: [`NetworkManager` 설정](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/creating-a-dummy-interface_configuring-and-managing-networking), [`NetworkManager` Reference Manual](https://networkmanager.dev/docs/api/latest/nm-settings-keyfile.html)

1. `nmcli` 명령어를 사용하여 dummy 생성
```sh
sudo nmcli connection add type dummy \
    ifname lo1 \
    con-name lo1 \
    ipv4.method manual \
    ipv4.addresses ${사설_IP_대역대_내의_임의의_CIDR} \
    ipv6.method ignore
```

## Tailscale 설정

Tailscale을 통해 위에서 생성한 loopback을 tailnet 전체로 노출했다.
참고 문서: [Set up high availability](https://tailscale.com/kb/1115/high-availability#subnet-router-high-availability)

1. Tailscale subnet router 설정
```sh
sudo tailscale up --advertise-routes ${위_단계에서_생성한_loopback의_CIDR}
```
2. Subnet router 설정 허용
콘솔의 "Edit Route Settings" 내에서 새로 추가한 subnet router 대역을 허용 처리한다.
![img](/assets/images/tailscale-ha/tailscale-console.png)
