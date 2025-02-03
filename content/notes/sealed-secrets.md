+++
title = "Sealed-secrets"
date = 2025-02-03
description = "더 안전하게 Secret 관리하기"
[taxonomies]
categories = ["k8s"]
tags = ["secrets"]
[extra]
lang = "ko"
toc = true
+++

## Sealed-secrets란

k8s의 리소스를 Git을 통해 관리할 경우 Secret 유출을 막기 위해 사용한다
유사한 목적의 다른 툴인 Helm-Secret은 로컬에서 값을 복호화하여 helm install 하기에 차트 배포 권한을 가진 사람이 원본 Secret 값을 확인할 수 있다는 문제가 존재한다.
그러나 Sealed-secrets는 오로지 k8s operator만 secret을 복호화할 수 있기에 보안 측면에서 helm-secret의 방식보다 안전하다.

참고 문서 : [bitnami-labs/sealed-secerts Github](https://github.com/bitnami-labs/sealed-secrets)

## 사용법

1. sealed-secrets operator 설치

helm-hub에 있는 차트를 이용하여 kubectl apply 하여도 좋다.

```zsh
$ helm create secrets-operator
$ cd secrets-operator
$ helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
# Charts.yaml에 dependency 작성
$ helm install -n kube-system secrets-operator .
```

2. secret.yaml 작성

k8s의 secret 규칙에 따라 작성한다.

3. sealed-secret.yaml 생성

```zsh
kubeseal -f secret.yaml -w sealed-secret.yaml --controller-name secrets-opeator-sealed-secrets
```

SealedSecret 타입의 암호화된 sealed-secret.yaml 파일이 만들어진다!

4. sealed-secret.yaml 배포

`kubectl apply -f`, `helm install` 등의 방법을 통해 클러스터 상에 SealedSecret을 배포한다.
