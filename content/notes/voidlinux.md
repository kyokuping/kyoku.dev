+++
title = "M1 air에 Void Linux를 올려보았다"
date = 2025-02-10
description = ""
[taxonomies]
categories = ["linux"]
tags = ["OS", "AppleSilicon"]
[extra]
lang = "ko"
toc = true
+++

이전부터 관심을 갖고 있던 리눅스 배포판 중 하나인 Void Linux가 Apple Silicon을 공식적으로 지원하게 되었다는 소식을 알게 되어 M1 맥북 에어 위에 올려보았다!
[<Void Linux is officially the first distro to officially support Apple Silicon!> -Reddit]("https://www.reddit.com/r/AsahiLinux/comments/1ijgal1/void_linux_is_officially_the_first_distro_to/")

## Void Linux
Void Linux는 **runit** 시스템을 사용하는 리눅스 배포판으로, 비교적 적은 리소스를 사용하여 가볍고 안정적이다.
Arch Linux와 유사하게 롤링 릴리즈 모델을 사용하고 있지만 Arch Linux와는 달리 **스테이블 롤링 릴리즈**를 지향하여 비교적 안정적으로 소프트웨어를 사용할 수 있다.
서버 환경에서는 보편적으로 사용되어 트러블슈팅하기 편리한 다른 배포판을 사용하지만, 개인 개발 환경에서는 Systemd 외의 다른 init 시스템을 경험해보고 싶어 Void Linux를 선택했다.
참고 자료 : [Void Handbook - About]("https://docs.voidlinux.org/about/index.html")

다음을 참고하여 설치를 진행해보았다! : [Void Handbook - Apple Silicon]("https://docs.voidlinux.org/installation/guides/arm-devices/apple-silicon.html")

## 설치 전
부팅 USB를 만들고, macOS에서 Asahi Linux 부트로더를 설치 및 재정비 하여야한다.
두 단계를 모두 마치면 USB를 연결한 후 [시동 디스크 변경]("https://support.apple.com/ko-kr/guide/mac-help/mchlp1034/mac")을 통해 Asahi Linux로 부팅한다.

### 1. Rufus를 이용해 부팅 USB 굽기
[Rufus](https://rufus.ie/)가 설치된 USB를 연결한 후 [Apple Silicon Void Linux ISO]("https://voidlinux.org/download/#arm%20platforms")에서 다운받은 ISO를 담는다.
나는 musl에 대한 경험이 적고(TT) Xfce 사용을 피하고 싶어 glibc base 이미지를 이용했다. Xfce 이미지를 사용하면 Desktop Environment로 Xfce를 사용하여 더 쉽게 설치가 가능해 보인다....
윈도우 외의 환경을 사용 중이라 Rufus 사용이 쉽지 않다면 [BalenaEtcher]("https://etcher.balenca.io") 등의 프로그램을 이용해 부팅 USB를 만들 수 있다.

### 2. Asahi Linux 부트로더
[Asahi Linux]("https://asahilinux.org/")는 리눅스 커널을 Apple Silicon으로 포팅하는 프로젝트이다. Void를 설치하려면 먼저 "UEFI only" 세팅으로 Asahi Linux 부트로더를 설치해야 한다.
```zsh
$ curl https://alx.sh > alx.sh
$ sh ./alx.sh
```

이미 Asahi를 통해 Fedora가 설치되어 있다면 삭제는 *다음의 문서를 참고하여 실행*한다. [Delete an Insallation - Asahi Linux wiki](https://github.com/AsahiLinux/docs/wiki/Delete-an-Installation)

```zsh
# 디스크 정보 확인
$ curl https://alx.sh | sh
# Asahi APFS 컨테이너 삭제
$ diskutil apfs deleteContainer ${APFS_컨테이너_이름}
# EFI 또는 리눅스 파일 시스템 파티션 삭제
$ diskutil eraseVolume free free ${EFI_파티션_이름}
$ diskutil eraseVolume free free ${리눅스_파일_시스템_파티션_이름}
```

## Void Linux 설치

### 1. 인터넷 연결
Void Linux ISO에는 `wpa_supplicant`가 내장되어 있다. 이를 사용하여 Wi-Fi에 연결한다.

#### a. Wi-Fi 인터페이스 확인
```sh
$ ip link
```
커맨드를 실행하면 무선랜 인터페이스 이름을 확인할 수 있다.

#### b. `wpa_passphrase` 로 설정 파일 생성
`wpa_passpharse`로 암호화된 설정 파일을 생성한다.
```sh
$ wpa_passphrase ${네트워크_SSID} ${네트워크_패스워드} > /etc/wpa_supplicant/wpa_supplicant.conf
```

#### c. `wpa_supplicant` 실행
```sh
$ wpa_supplicant -B -i ${인터페이스_이름} -c /etc/wpa_supplicant/wpa_supplicant.conf
```
`-B`는 백그라운드 실행, `-i`는 인터페이스 지정, `-c`는 사용지 지정 옵션이다.

#### d. `dhcpcd` 실행
```sh
$ dhcpcd ${인터페이스_이름}
```

#### e. 연결 확인
```sh
$ ping 8.8.8.8
```
위의 과정으로 연결에 실패한 경우, `wpa_cli`를 사용하여 설정을 확인해 보고 연결을 다시 시도해본다.

### 2. 파티셔닝 - fdisk
Asahi Linux 부트로더 설치 과정에서 EFI 시스템 파티션(EF00)이 이미 생성되었기 때문에, 이는 별도로 생성하지 않는다.
Void Linux를 설치하기 위해 추가 파티션(8300, Linux FileSystem)을 생성한다.
디스크 파티셔닝을 위해 fdisk를 사용했다.

```sh
$ fdisk /dev/${파티셔닝할_디스크}
Command(m to help): n # 새 파티션 생성.
Command(m to help): t # 파티션 유형 변경.
Command(m to help): w # 파티션 설정 저장.
```

### 3. 파티션 포맷팅  - mkfs
Linux File System은 ext4로 포맷팅한다.
ext4는 Linux 시스템에서 사용되는 파일 시스템으로, 데이터를 저장하는 방식을 정의한다. 다른 리눅스 배포판과 같이 Void Linux도 ext4를 기본 파일 시스템으로 채택하는 것으로 보인다.
```sh
$ mkfs.ext4 /dev/${새로_생성한_파티션}
```

### 4. NewRoot
위 단계에서 새로 생성한 파티션을 `/mnt`에 마운트하고, EFI파티션을 `/boot/efi`에 마운트 한다.
```sh
$ mount /dev/${새로_생성한_파티션} /mnt
$ mkdir -p /mnt/boot/efi
$ mount /dev/${EFI_파티션} /mnt/boot/efi
```

### 5. base installation
XBPS(X Binary Package System) 방식을 사용하여 진행하였다.

미러를 선택하기 위한 셸 변수를 저장한다. 각 변수는 시스템 타입과 아키텍처를 고려하여 설정한다.
```sh
$ REPO=https://repo-default.voidlinux.org/current/aarch64 #mirror URL
$ ARCH=aarch64
```
xbps 패키지 관리자가 사용하는 공개키인 RSA 키를 루트 디렉토리로 복사한다. xbps 패키지를 설치하고 검증하기 위한 과정이다.
```sh
$ mkdir -p /mnt/var/db/xbps/keys
$ cp /var/db/xbps/keys/* /mnt/var/db/xbps/keys/

```
Void의 패키지 매니저인 XBPS를 사용하여 다음 패키지를 설치한다.
- `base-system` : Void Linux의 메타패키지. Void Linux 시스템의 필수 패키지 그룹으로 xbps, runit 등등을 포함한다.
- `asahi-base` : Asahi Linux의 메타패키지. Apple Silicon을 위한 리눅스 커널과 모듈을 위한 패키지이다.
- `asahi-scripts` : Asahi Linux 전용 패키지로 Apple Silicon 장비를 위한 관리 스크립트. 하드웨어와 상호작용을 위한 패키지로 추정된다.

```sh
$ XBPS_ARCH=$ARCH xbps-install -S -r /mnt -R "$REPO" base-system asahi-base asahi-scripts
```
`-S`는 패키지 데이터베이스 동기화와 설치를 위한 옵션, `-r`은 루트 디렉토리 지정을 위한 옵션, `-R`은 디렉토리 저장소 설정을 위한 옵션이다.

### 6. configuration

#### a. 파일시스템
`xgenfstab`
`xgenfstab`을 사용하여 마운트 된 파일시스템을 기반으로 fstab 파일을 생성한다.
```sh
$ xgenfstab -U /mnt > /mnt/etc/fstab
```

#### b. chroot 환경 진입
`xchroot`을 사용하여 chroot 환경을 설정하고 진입할 수 있다.
```sh
$ xchroot /mnt /bin/bash
```

#### c. 설치 구성

##### vim 설치
이미 설치되어 있는 vi보다 편한 환경에서 작업하고 싶어 vim을 설치했다.

```sh
$ xbps-install -S vim
```

##### 호스트 네임 설정 `/etc/hostname`
원하는 이름을 문자열로 저장하면 된다.

```sh
$ echo ${호스트_이름} > /etc/hostname
```

##### `/etc/rc.conf`
시스템 설정에 대한 파일이다. 튜닝하고 싶은 값이 있다면 적절히 설정해주면 된다.

##### `/etc/default/libc-locales`
glibc 배포판의 경우 locale 설정 파일 내 주석을 해제하여 원하는 locale을 설정한다.
```sh
$ xbps-reconfigure -f glibc-locales
```

##### 사용자 추가 및 비밀번호 설정
```sh
$ useradd -m -u ${UID} -s /bin/bash ${사용자_이름} -g wheel
$ passwd ${사용자_이름}
```
+ 시스템 계정은 1000 이하의 UID를 사용한다.
+ wheel 그룹은 sudo 권한을 가지는 그룹이다.

이제 해당 사용자로 sudo를 사용할 수 있게 되었으니, 루트 계정은 로그인할 수 없도록 잠근다.
```sh
$ sudo passwd -l root
```

### 7. GRUB 설치
Apple Silicon 기기들은 모두 UEFI 부트 시스템을 사용하니, Void 문서의 관련 부분을 참고하면 된다.
```sh
$ xbps-install -S grub-arm64-efi
$ grub-install --target=arm64-efi --efi-directory=/boot/efi --bootloader-id="Void" --removable
```
Asahi Linux의 경우 이동식 디스크 위에서도 부팅가능하도록 하는 옵션인 `--removable` 옵션 사용이 권장된다.

### 8. 끝!

```sh
$ xbps-reconfigure -fa # 설치된 모든 패키지를 재구성하여 설정 파일을 최신 상태로 유지.
```
리부트하여 설치 성공 여부를 확인한다.
```sh
$ exit # chroot 환경에서 나오기 위한 exit.
$ umount -R /mnt
$ shutdown -r now
```

## 사담
재작년에 Arch Linux를 설치한 이후로 별 무리 없이 랩탑에서 개발환경으로 잘 사용하고 있었으나, Asahi 지원 소식을 계기로 맥북에 Void Linux를 설치하여 둘을 비교하며 사용할 수 있게 되었다.
지난 Arch Linux 설치 시에는 Arch의 더 큰 커뮤니티와 더 잘 관리되는 친절한 위키 덕에 리눅스에 대한 완벽한 이해가 없어도 ~~아주 조금의 문제를 겪고 난 후~~ 설치할 수 있었다.
이번 Void Linux 설치 시에는 Void 위키의 설명과 이전의 Arch Linux 설치 경험으로 다행히도 전보다 쉽고 재미있게 설치를 마칠 수 있었다.
물리적으로는 13인치 맥북에서 설치를 하려고 하니 콘솔 폰트가 너무 작아서 힘들었다... ~~setfont로 폰트를 조정할 수 있다고 한다~~
runit 시스템과 xbps 패키지 시스템을 경험해 보고 난 후 더 자세한 후기를 업데이트하도록 하겠다.
