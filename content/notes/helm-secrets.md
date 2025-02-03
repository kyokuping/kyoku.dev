+++
title = "helm-secrets로 Secret 관리하기"
date = 2025-02-03
description = "안전하게 Secret 관리하기"
[taxonomies]
categories = ["k8s"]
tags = ["helm", "sops"]
[extra]
lang = "ko"
toc = true
+++

Helm 차트에 들어갈 Secret 값들을 Git 상에서 안전하게 관리하기 위하여 암호화/복호화가 필요한데,
이를 해결하기 위해 Helm의 플러그인 Helm-Secrets을 이용하여 시크릿 값들을 보호했다.

## Helm-Secrets란

Helm 명령어 실행 시 변수 파일 내의 값을 암호화/복호화 하기 위한 Helm 플러그인이다.

참고 문서 : [`jkroepke/helm-secrets` GitHub](https://github.com/jkroepke/helm-secrets.git)

## 사용법

0. `age`로 key 만들기

현재 사용 중인 쿠버네티스 클러스터는 홈서버에 올라가 있는데,
클라우드 플랫폼을 사용하지 않는 상황에, Vault와 같은 별도의 Key Management Service 역시 사용하고 있지 않았다.
그래서 개인 키를 통한 로컬에서의 간편한 암호화를 지원하는 툴인 age를 활용하였다.
age를 사용하려면 먼저 개인 키를 생성해야 하는데, sops가 사용할 키이기에 sops가 키를 읽어오도록 설정되어 있는 위치에 키를 만들어야 한다.

- SOPS, Secrets OPerationS : 파일 내의 특정 필드의 데이터를 암호화하기 위한 파일 에디터. yaml, json, env 등의 파일을 지원한다.
- age : SOPS가 실제 데이터 암호화를 위해 사용하는 암호화 도구. 공개 키 방식을 사용한다.

```zsh
#  ~/.config/sops/age에 keys.txt 생성
$ age-keygen -o ~/.config/sops/age/keys.txt
```

참고 문서 : [`FiloSottile/age` GitHub](https://github.com/FiloSottile/age.git)
참고 문서 : [`getsops/sops` GitHub](https://github.com/getsops/sops?tab=readme-ov-file#encrypting-using-age)

1. `.sops.yaml` 작성

sops의 config 파일에 해당한다.
0에서 생성한 키 값을 다음과 같은 양식으로 추가하여 파일을 작성한다.

```yaml
creation_rules:
  - sops: >-
      agexxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
      agexxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

참고 문서 : [`getsops/sops` GitHub](https://github.com/getsops/sops?tab=readme-ov-file#encrypting-using-age)

2. `secrets.yaml.dec` 작성

Values.yaml과 같은 형식으로 암호화할 값들을 작성한다.

3. `helm secret encrypt`로 `secrets.yaml.dec`을 `secrets.yaml`로 암호화

```zsh
$ helm plugin install https://github.com/jkroepke/helm-secrets
$ helm secrets encrypt secrets.yaml.dec > secrets.yaml
```

4. `helm secrets upgrade [Release Name] [Chart Path] -n [Namespace] -f secrets.yaml --install`

`helm upgrade` 대신 `helm secrets upgrade`를 실행하면 필요에 따라 secrets.yaml을 복호화하여 원본 값을 추출한 후 원본 커맨드를 실행한다.

## 정리

helm-secrets는 간편하게 암호화가 가능하여 홈서버에는 적합한 방식이나, 여러 팀 구성원이 운영하기에는 권한 관리 등에서 다소 무리가 있어
여러 인원이 관리하는 클러스터의 경우에는 sealed-secrets 사용이 권장된다. 이상!
